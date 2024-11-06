<template>
  <div class="sankey-container">
    <svg ref="svg"></svg>
    <div class="filter-container">
      <label for="car-make-filter">Select up to 5 Car Makes:</label>
      <select id="car-make-filter" v-model="selectedMakes" multiple @change="checkSelectionLimit">
        <option value="Select option">(Select option)</option>
        <option v-for="make in availableMakes" :key="make" :value="make">{{ make }}</option>
      </select>
    </div>
    <!-- Tooltip div that will be positioned absolutely -->
    <div v-if="tooltipVisible" :style="tooltipStyles" class="tooltip">{{ tooltipContent }}</div>
  </div>
</template>

<script>
import * as d3 from "d3";
import { sankey, sankeyLinkHorizontal } from "d3-sankey";

export default {
  data() {
    return {
      margin: { top: 60, right: 100, bottom: 20, left: 80 },
      width: 1400,
      height: 900,
      topMakes: 10,
      selectedMakes: ["Select option"],
      loadedData: [],
      availableMakes: [],
      tooltipVisible: false,
      tooltipContent: "",
      tooltipStyles: {
        position: "absolute",
        padding: "8px",
        font: "12px sans-serif",
        background: "#333",
        color: "#fff",
        borderRadius: "4px",
        pointerEvents: "none",
        opacity: 0,
      },
    };
  },
  mounted() {
    this.loadDataAndDrawChart();
    window.addEventListener("resize", this.drawChart);
  },
  methods: {
    async loadDataAndDrawChart() {
      const data = await d3.csv("../../data/car_prices.csv");
      this.initializeAvailableMakes(data);
      this.loadedData = data;
      this.drawChart();
    },
    initializeAvailableMakes(data) {
      const makeCounts = d3.rollups(data, v => v.length, d => d.make)
        .sort((a, b) => d3.descending(a[1], b[1]))
        .slice(0, 10)
        .map(d => d[0]);

      this.availableMakes = makeCounts;
      this.selectedMakes = ["Select option"];
    },
    checkSelectionLimit() {
      if (this.selectedMakes.includes("Select option")) {
        this.selectedMakes = ["Select option"];
      } else if (this.selectedMakes.length > 5) {
        this.selectedMakes.pop();
      }
      this.drawChart();
    },
    drawChart() {
      const svg = d3.select(this.$refs.svg);
      svg.selectAll("*").remove();

      const width = this.width - this.margin.left - this.margin.right;
      const height = this.height - this.margin.top - this.margin.bottom;

      const g = svg
        .attr("width", this.width)
        .attr("height", this.height)
        .append("g")
        .attr("transform", `translate(${this.margin.left}, ${this.margin.top})`);

      const isDefaultView = this.selectedMakes.includes("Select option");

      const filteredData = this.loadedData.filter(d =>
        isDefaultView || this.selectedMakes.includes(d.make)
      );

      const sankeyData = this.getSankeyData(filteredData);

      const sankeyLayout = sankey()
        .nodeWidth(30)
        .nodePadding(15)
        .extent([[0, 0], [width, height]]);

      const graph = sankeyLayout(sankeyData);

      const makeColor = "#66c2a5";
      const modelColor = "#fc8d62";
      const bodyColor = "#8da0cb";

      const colorByType = d => {
        if (this.availableMakes.includes(d.name)) return makeColor;
        if (d.name.includes("Body")) return bodyColor;
        return modelColor;
      };

      // Draw nodes with animation
      g.append("g")
        .selectAll("rect")
        .data(graph.nodes, d => d.name)
        .join(
          enter => enter.append("rect")
            .attr("x", d => d.x0)
            .attr("y", d => d.y0)
            .attr("width", d => d.x1 - d.x0)
            .attr("height", d => d.y1 - d.y0)
            .attr("fill", colorByType)
            .attr("stroke", "#333")
            .attr("stroke-width", 1.5)
            .attr("opacity", 0)
            .on("mouseover", (event, d) => this.showTooltip(event, d, true))
            .on("mousemove", event => this.moveTooltip(event))
            .on("mouseout", () => this.hideTooltip())
            .transition()
            .duration(1000)
            .attr("opacity", 1),
          update => update
            .transition()
            .duration(1000)
            .attr("y", d => d.y0)
            .attr("height", d => d.y1 - d.y0),
          exit => exit.transition()
            .duration(1000)
            .attr("opacity", 0)
            .remove()
        );

      // Draw links with animation
      g.append("g")
        .selectAll("path")
        .data(graph.links, d => `${d.source.name}-${d.target.name}`)
        .join(
          enter => enter.append("path")
            .attr("d", sankeyLinkHorizontal())
            .attr("fill", "none")
            .attr("stroke", d => colorByType(d.source))
            .attr("stroke-width", d => Math.max(1, d.width))
            .attr("stroke-opacity", 0)
            .on("mouseover", (event, d) => this.showTooltip(event, d, false))
            .on("mousemove", event => this.moveTooltip(event))
            .on("mouseout", () => this.hideTooltip())
            .transition()
            .duration(1000)
            .attr("stroke-opacity", 0.3),
          update => update
            .transition()
            .duration(100)
            .attr("d", sankeyLinkHorizontal())
            .attr("stroke-width", d => Math.max(1, d.width)),
          exit => exit.transition()
            .duration(1000)
            .attr("stroke-opacity", 0)
            .remove()
        );

      // Node labels
      g.append("g")
        .selectAll("text")
        .data(graph.nodes)
        .join("text")
        .attr("x", d => d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6)
        .attr("y", d => (d.y0 + d.y1) / 2)
        .attr("dy", "0.35em")
        .attr("text-anchor", d => d.x0 < width / 2 ? "start" : "end")
        .attr("fill", "#000000")
        .style("font-size", "10px")
        .text(d => d.name);

      // Title and labels with added padding
      svg.append("text")
        .attr("x", this.width / 2)
        .attr("y", this.margin.top / 2)
        .attr("text-anchor", "middle")
        .style("font-size", "16px")
        .text("Car Make-Model-Body Type Relationships");

      svg.append("text")
        .attr("x", this.margin.left)
        .attr("y", this.margin.top - 10)
        .attr("fill", "#000000")
        .style("font-size", "14px")
        .text("Make");

      svg.append("text")
        .attr("x", this.width / 2)
        .attr("y", this.margin.top - 10)
        .attr("text-anchor", "middle")
        .style("font-size", "14px")
        .text("Model");

      svg.append("text")
        .attr("x", this.width - this.margin.right)
        .attr("y", this.margin.top - 10)
        .attr("text-anchor", "end")
        .style("font-size", "14px")
        .text("Body Type");
    },
    showTooltip(event, d, isNode) {
      if (isNode) {
        // Node tooltip (Make, Model, or Body Type)
        const dataForNode = this.loadedData.filter(row => row.make === d.name || row.model === d.name || row.body === d.name);
        const numModels = new Set(dataForNode.map(row => row.model)).size;
        const avgPrice = d3.mean(dataForNode, row => +row.sellingprice).toFixed(2);
        const avgCondition = d3.mean(dataForNode, row => +row.condition).toFixed(2);

        this.tooltipContent = `${d.name} 
          \nModels Available: ${numModels}
          \nAverage Selling Price: $${avgPrice}
          \nAverage Condition: ${avgCondition}`;
      } else {
        // Link tooltip (Make to Model or Model to Body Type)
        this.tooltipContent = `${d.source.name} → ${d.target.name}
          \nNumber of Connections: ${d.value}`;
      }

      this.tooltipVisible = true;
      this.tooltipStyles.opacity = 1;
      this.tooltipStyles.left = `${event.pageX + 10}px`;
      this.tooltipStyles.top = `${event.pageY - 28}px`;
    },
    moveTooltip(event) {
      this.tooltipStyles.left = `${event.pageX + 10}px`;
      this.tooltipStyles.top = `${event.pageY - 28}px`;
    },
    hideTooltip() {
      this.tooltipVisible = false;
      this.tooltipStyles.opacity = 0;
    },
    getSankeyData(data) {
      const sankeyData = { nodes: [], links: [] };
      const nodeIndex = {};
      const linkSet = new Set();

      const addNode = name => {
        if (!(name in nodeIndex)) {
          sankeyData.nodes.push({ name });
          nodeIndex[name] = sankeyData.nodes.length - 1;
        }
        return nodeIndex[name];
      };

      // Normalize body type names to avoid duplicates
      const normalizeBodyType = bodyType => bodyType.toLowerCase().replace(/\s+/g, " ").trim();

      // Filter top car makes
      const makeCounts = d3.rollups(data, v => v.length, d => d.make)
        .sort((a, b) => d3.descending(a[1], b[1]))
        .slice(0, this.topMakes)
        .map(d => d[0]);

      const filteredMakesData = data.filter(d => makeCounts.includes(d.make));

      const modelCounts = {};
      filteredMakesData.forEach(d => {
        if (!modelCounts[d.make]) modelCounts[d.make] = {};
        modelCounts[d.make][d.model] = (modelCounts[d.make][d.model] || 0) + 1;
      });

      const topModels = {};
      Object.keys(modelCounts).forEach(make => {
        topModels[make] = Object.entries(modelCounts[make])
          .sort((a, b) => d3.descending(a[1], b[1]))
          .slice(0, this.topMakes) // Limit to top models per make
          .map(d => d[0]);
      });

      const bodyTypeCounts = {};
      filteredMakesData.forEach(d => {
        if (!bodyTypeCounts[d.model]) bodyTypeCounts[d.model] = {};
        const normalizedBody = normalizeBodyType(d.body);
        bodyTypeCounts[d.model][normalizedBody] = (bodyTypeCounts[d.model][normalizedBody] || 0) + 1;
      });

      const topBodyTypes = {};
      Object.keys(bodyTypeCounts).forEach(model => {
        topBodyTypes[model] = Object.entries(bodyTypeCounts[model])
          .sort((a, b) => d3.descending(a[1], b[1]))
          .slice(0, this.topMakes) // Limit to top body types per model
          .map(d => d[0]);
      });

      const addLink = (source, target, value) => {
        if (!linkSet.has(`${source}-${target}`)) {
          sankeyData.links.push({ source, target, value });
          linkSet.add(`${source}-${target}`);
        }
      };

      filteredMakesData.forEach(d => {
        const makeIndex = addNode(d.make);
        if (topModels[d.make] && topModels[d.make].includes(d.model)) {
          const modelIndex = addNode(d.model);
          addLink(makeIndex, modelIndex, modelCounts[d.make][d.model]);

          const normalizedBodyType = normalizeBodyType(d.body);
          if (topBodyTypes[d.model] && topBodyTypes[d.model].includes(normalizedBodyType)) {
            const bodyIndex = addNode(normalizedBodyType);
            addLink(modelIndex, bodyIndex, bodyTypeCounts[d.model][normalizedBodyType]);
          }
        }
      });

      return sankeyData;
    },
  },
};
</script>

<style scoped>
.sankey-container {
  display: flex;
  align-items: flex-start;
}

.filter-container {
  margin-left: 20px;
}

select {
  width: 200px;
  margin-top: 20px;
}

.tooltip {
  font-size: 14px;
  line-height: 1.5;
  opacity: 0;
  transition: opacity 0.2s ease;
  white-space: pre-wrap; /* Preserve line breaks in tooltip */
}
</style>