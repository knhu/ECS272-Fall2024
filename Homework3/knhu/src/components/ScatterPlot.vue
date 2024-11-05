<template>
  <div>
    <svg ref="svg"></svg>
    <!-- Tooltip div that will be positioned absolutely -->
    <div v-if="tooltipVisible" :style="tooltipStyles" class="tooltip">{{ tooltipContent }}</div>
    <label for="car-make-filter">Select up to 5 Car Makes:</label>
    <select id="car-make-filter" v-model="selectedMakes" multiple @change="checkSelectionLimit">
      <option value="Select option">(Select option)</option>
      <option v-for="make in availableMakes" :key="make" :value="make">{{ make }}</option>
    </select>

    <!-- Legend container -->
    <div class="legend-container">
      <span v-for="make in availableMakes" :key="make" class="legend-item">
        <span :style="{ backgroundColor: colorScale(make) }" class="legend-color-box"></span>
        {{ make }}
      </span>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";

export default {
  props: ["data", "selectedMake"],
  data() {
    return {
      margin: { top: 40, right: 20, bottom: 80, left: 80 }, // Increased bottom and left margins for axis titles
      width: 900,
      height: 500,
      availableMakes: [],
      selectedMakes: ["Select option"],
      loadedData: [],
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
      colorScale: d3.scaleOrdinal(d3.schemeCategory10)
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
      const makeCounts = Array.from(
        d3.group(data, d => d.make),
        ([make, values]) => ({ make, count: values.length })
      )
        .sort((a, b) => d3.descending(a.count, b.count))
        .slice(0, 10)
        .map(d => d.make);
      
      this.availableMakes = makeCounts;
      this.selectedMakes = ["Select option"];
    },
    checkSelectionLimit() {
      if (this.selectedMakes.includes("Select option")) {
        this.selectedMakes = ["Select option"];
      } else if (this.selectedMakes.length > 5) {
        this.selectedMakes.pop();
      }
      this.$emit("update-selected-makes", this.selectedMakes);
      this.drawChart();
    },
    drawChart() {
      const svg = d3.select(this.$refs.svg);
      svg.selectAll("*").remove();

      const width = this.width - this.margin.left - this.margin.right;
      const height = this.height - this.margin.top - this.margin.bottom;

      const isDefaultView = this.selectedMakes.includes("Select option");
      const filteredData = this.loadedData.filter(d =>
        (isDefaultView || this.selectedMakes.includes(d.make)) && d.condition
      );

      const x = d3.scaleLinear().domain([0, 50]).range([0, width]);
      const y = d3.scaleLinear().domain(d3.extent(filteredData, d => +d.sellingprice)).nice().range([height, 0]);

      this.colorScale.domain(this.availableMakes);

      const chart = svg
        .attr("width", this.width)
        .attr("height", this.height)
        .append("g")
        .attr("transform", `translate(${this.margin.left}, ${this.margin.top})`);

      // X-axis with transition
      const xAxis = chart.append("g")
        .attr("transform", `translate(0, ${height})`)
        .call(d3.axisBottom(x).ticks(10).tickFormat(d => (d === 50 ? "46-49" : `${d}`)));

      // X-axis title
      xAxis.append("text")
        .attr("x", width / 2)
        .attr("y", 50) // Position below the axis
        .attr("fill", "black")
        .style("font-size", "14px")
        .text("Condition Rating Bins");

      // Y-axis with transition
      const yAxis = chart.append("g")
        .call(d3.axisLeft(y));

      // Y-axis title
      yAxis.append("text")
        .attr("x", -height / 2)
        .attr("y", -60) // Position to the left of the axis
        .attr("transform", "rotate(-90)")
        .attr("fill", "black")
        .style("font-size", "14px")
        .text("Selling Price");

      // Circles with animations for entering, updating, and exiting data
      const circles = chart.selectAll("circle")
        .data(filteredData, d => d.id)
        .join(
          enter => enter.append("circle")
            .attr("cx", d => x(+d.condition))
            .attr("cy", d => y(+d.sellingprice))
            .attr("r", 0) // Start with radius 0 for smooth entrance
            .attr("fill", d => this.colorScale(d.make))
            .attr("opacity", 0.7)
            .call(enter => enter.transition()
              .duration(1000)
              .attr("r", 3)
            ),
          update => update
            .call(update => update.transition()
              .duration(1000)
              .attr("cx", d => x(+d.condition))
              .attr("cy", d => y(+d.sellingprice))
            ),
          exit => exit
            .call(exit => exit.transition()
              .duration(1000)
              .attr("r", 0)
              .remove()
            )
        )
        .on("mouseover", (event, d) => {
          this.tooltipContent = `Make: ${d.make} | Condition: ${d.condition} | Price: $${d.sellingprice}`;
          this.tooltipVisible = true;
          this.tooltipStyles.opacity = 1;
          this.tooltipStyles.left = `${event.pageX + 10}px`;
          this.tooltipStyles.top = `${event.pageY - 28}px`;
        })
        .on("mousemove", event => {
          this.tooltipStyles.left = `${event.pageX + 10}px`;
          this.tooltipStyles.top = `${event.pageY - 28}px`;
        })
        .on("mouseout", () => {
          this.tooltipVisible = false;
          this.tooltipStyles.opacity = 0;
        });

      // Animate the X-axis when data updates
      xAxis.transition().duration(1000).call(d3.axisBottom(x).ticks(10));
      
      // Animate the Y-axis when data updates
      yAxis.transition().duration(1000).call(d3.axisLeft(y));

      // Chart title
      svg.append("text")
        .attr("x", this.width / 2)
        .attr("y", 20)
        .attr("text-anchor", "middle")
        .style("font-size", "16px")
        .text("Condition Rating vs. Selling Price by Car Make");
    },
  },
};
</script>

<style scoped>
select {
  margin-bottom: 10px;
  width: 100%;
}
circle {
  transition: opacity 0.2s;
}
circle:hover {
  opacity: 1;
}
.tooltip {
  font-size: 14px;
  line-height: 1.5;
  opacity: 0;
  transition: opacity 0.2s ease;
}
.legend-container {
  display: flex;
  flex-wrap: wrap;
  margin-top: 10px;
  max-width: 900px;
}
.legend-item {
  display: flex;
  align-items: center;
  margin-right: 15px;
  font-size: 12px;
}
.legend-color-box {
  width: 10px;
  height: 10px;
  margin-right: 5px;
  border-radius: 2px;
}
</style>
