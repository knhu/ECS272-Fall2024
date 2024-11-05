<!-- src/App.vue -->

<template>
  <div class="dashboard">
    <BarChart :data="barChartData" />
    <ScatterPlot :data="scatterPlotData" />
    <SankeyDiagram :data="sankeyData" />
  </div>
</template>

<script>
import * as d3 from 'd3-fetch';
import BarChart from './components/BarChart.vue';
import ScatterPlot from './components/ScatterPlot.vue';
import SankeyDiagram from './components/SankeyDiagram.vue';

export default {
  components: { BarChart, ScatterPlot, SankeyDiagram },
  data() {
    return {
      barChartData: [],
      scatterPlotData: [],
      sankeyData: []
    };
  },
  async created() {
    const rawData = await d3.csv('../../data/car_prices.csv');
    
    // Transform data for BarChart (Top 10 makes by count)
    const makeCounts = d3.rollup(rawData, v => v.length, d => d.make);
    this.barChartData = Array.from(makeCounts, ([make, count]) => ({ make, count }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 10);

    // Transform data for ScatterPlot (Odometer vs Selling Price)
    this.scatterPlotData = rawData.map(d => ({
      odometer: +d.odometer,
      sellingprice: +d.sellingprice
    }));

    // Transform data for SankeyDiagram (Make -> Model -> Body)
    this.sankeyData = rawData.map(d => ({
      make: d.make,
      model: d.model,
      body: d.body
    }));
  }
};
</script>

<style>
.dashboard {
  display: flex;
  flex-wrap: wrap;
  gap: 20px;
  justify-content: space-around;
}
</style>
