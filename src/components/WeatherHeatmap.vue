<template>
  <div class="heatmap-container">
    <h2>西雅圖氣象資料 - 年度每日均溫</h2>
    <p>資料來源：Vega Datasets 數據</p>

    <!-- 載入中與錯誤提示 -->
    <div v-if="loading" class="loading">正在載入資料...</div>
    <div v-if="errorMsg" class="error">{{ errorMsg }}</div>

    <!-- SVG 畫布容器 -->
    <svg ref="svgRef"></svg>

    <!-- 提示框 (Tooltip) -->
    <div class="tooltip" ref="tooltipRef"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as d3 from 'd3'

const svgRef = ref(null)
const tooltipRef = ref(null)
const loading = ref(true)
const errorMsg = ref(null)

onMounted(async () => {
  try {
    // 讀取氣象資料(西雅圖)
    const csvUrl = "https://cdn.jsdelivr.net/npm/vega-datasets@3/data/seattle-weather.csv"
    const rawData = await d3.csv(csvUrl)

    loading.value = false

    // 計算每日平均溫 (最高溫 + 最低溫 / 2)
    const weatherData = rawData.map(d => {
      const avgTemp = (parseFloat(d.temp_max) + parseFloat(d.temp_min)) / 2
      return {
        date: d.date,
        temp: parseFloat(avgTemp.toFixed(1))
      }
    })
    .filter(d => d.date.startsWith("2015"))

    // 尺寸設定
    const width = 900
    const height = 200
    const cellSize = 15

    const svg = d3.select(svgRef.value)
      .attr("width", width + 100)
      .attr("height", height + 130)


    const g = svg.append("g")
      .attr("transform", "translate(50, 40)")

    // 取得最小與最大溫度作為色彩區間
    const minTemp = d3.min(weatherData, d => d.temp)
    const maxTemp = d3.max(weatherData, d => d.temp)

    const colorScale = d3.scaleSequential()
      .domain([minTemp, maxTemp])
      .interpolator(d3.interpolateYlOrRd)

    const parseDate = d3.timeParse("%Y-%m-%d")
    const tooltip = d3.select(tooltipRef.value)

    // 繪製熱力圖方塊
    g.append("g")
      .selectAll("rect")
      .data(weatherData)
      .enter()
      .append("rect")
      .attr("class", "cell")
      .attr("width", cellSize)
      .attr("height", cellSize)
      .attr("x", d => {
        const date = parseDate(d.date)
        if (!date) return 0
        const weekOfYear = d3.timeWeek.count(d3.timeYear(date), date)
        return weekOfYear * (cellSize + 2)
      })
      .attr("y", d => {
        const date = parseDate(d.date)
        if (!date) return 0
        return date.getDay() * (cellSize + 2)
      })
      .attr("fill", d => colorScale(d.temp))
      .on("mouseover", (event, d) => {
        tooltip.style("display", "block")
               .html(`日期: <b>${d.date}</b><br>平均氣溫: <span style="color:#ffcc00">${d.temp} °C</span>`)
      })
      .on("mousemove", (event) => {
        tooltip.style("top", (event.pageY - 15) + "px")
               .style("left", (event.pageX + 15) + "px")
      })
      .on("mouseout", () => {
        tooltip.style("display", "none")
      })

    const days = ["日", "一", "二", "三", "四", "五", "六"]
    days.forEach((day, i) => {
      g.append("text")
        .attr("x", -20)
        .attr("y", i * (cellSize + 2) + 12)
        .style("font-size", "10px")
        .style("fill", "#666")
        .text(day)
    })

    // 色彩對應溫度圖例
    const legendGroup = g.append("g")
      .attr("transform", `translate(0, ${height + 25})`)

    const legendWidth = 300
    const legendHeight = 12

    const defs = svg.append("defs")
    const linearGradient = defs.append("linearGradient")
      .attr("id", "temp-gradient-seattle")

    const numStops = 10
    for (let i = 0; i <= numStops; i++) {
      const interp = i / numStops
      const tempVal = minTemp + interp * (maxTemp - minTemp)
      linearGradient.append("stop")
        .attr("offset", `${interp * 100}%`)
        .attr("stop-color", colorScale(tempVal))
    }

    legendGroup.append("rect")
      .attr("x", 0)
      .attr("y", 0)
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .style("fill", "url(#temp-gradient-seattle)")
      .style("stroke", "#ccc")
      .style("stroke-width", "0.5px")

    const legendScale = d3.scaleLinear()
      .domain([minTemp, maxTemp])
      .range([0, legendWidth])

    const legendAxis = d3.axisBottom(legendScale)
      .ticks(5)
      .tickFormat(d => `${d.toFixed(1)} °C`)

    legendGroup.append("g")
      .attr("transform", `translate(0, ${legendHeight})`)
      .call(legendAxis)
      .selectAll("text")
      .style("font-size", "10px")

    legendGroup.append("text")
      .attr("x", 0)
      .attr("y", -6)
      .style("font-size", "12px")
      .style("fill", "#333")
      .style("font-weight", "bold")
      .text("溫度色階對應表：")

  } catch (error) {
    loading.value = false
    errorMsg.value = "載入氣象資料失敗，請檢查網路連線或主控台錯誤。"
    console.error("載入氣象資料失敗:", error)
  }
})
</script>

<style scoped>
.heatmap-container {
  font-family: sans-serif;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

h2 {
  color: #333;
  margin-bottom: 5px;
}

p {
  color: #666;
  font-size: 14px;
  margin-bottom: 10px;
}

.loading {
  color: #007bff;
  font-weight: bold;
  margin-bottom: 15px;
}

.error {
  color: #dc3545;
  font-weight: bold;
  margin-bottom: 15px;
}

:deep(.cell) {
  stroke: #fff;
  stroke-width: 1px;
}

:deep(.cell:hover) {
  stroke: #333;
  stroke-width: 2px;
}

.tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.85);
  color: #fff;
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 12px;
  pointer-events: none;
  display: none;
  box-shadow: 0 2px 6px rgba(0,0,0,0.3);
  z-index: 10;
}
</style>