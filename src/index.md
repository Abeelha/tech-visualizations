---
toc: false
---

```js
const benchmarks = FileAttachment("data/gpu_benchmarks.csv").csv({typed: true});
const apiScores = FileAttachment("data/gpu_api_scores.csv").csv({typed: true});
const releases = FileAttachment("data/nvidia_releases.csv").csv({typed: true});
```

```js
const gpuCount = (await benchmarks).length;
const topGpu = (await benchmarks).sort((a, b) => b.G3Dmark - a.G3Dmark)[0];
const brands = [...new Set((await benchmarks).map(d => d.brand))].filter(b => b);
const totalReleases = (await releases).reduce((sum, d) => sum + d.release, 0);

const benchmarkData = (await benchmarks)
  .filter(d => d.G3Dmark && d.brand && d.brand !== "Other")
  .map(d => ({
    gpuName: d.gpuName,
    g3dmark: +d.G3Dmark,
    brand: d.brand,
    price: +d.price || 0,
    testDate: +d.testDate || 0
  }));

const topG3D = [...benchmarkData].sort((a, b) => b.g3dmark - a.g3dmark).slice(0, 15);

const processedApi = (await apiScores)
  .filter(d => d.Manufacturer === "Nvidia" || d.Manufacturer === "AMD")
  .map(d => ({
    manufacturer: d.Manufacturer === "Nvidia" ? "NVIDIA" : d.Manufacturer,
    device: d.Device,
    opencl: +d.OpenCL || 0
  }))
  .filter(d => d.opencl > 0);

const topOpenCL = [...processedApi].sort((a, b) => b.opencl - a.opencl).slice(0, 10);

const processedData = benchmarkData
  .filter(d => d.price > 0 && d.g3dmark > 0)
  .map(d => ({
    ...d,
    valueScore: (d.g3dmark / d.price).toFixed(2)
  }));

const topValue = [...processedData].sort((a, b) => +b.valueScore - +a.valueScore).slice(0, 10);

const priceRanges = [
  {range: "Under $500", min: 0, max: 500, color: "#009966"},
  {range: "$500-$1,000", min: 500, max: 1000, color: "#3b82f6"},
  {range: "$1,000-$2,000", min: 1000, max: 2000, color: "#f59e0b"},
  {range: "Over $2,000", min: 2000, max: Infinity, color: "#dc2626"}
];

const bestPerRange = priceRanges.map(r => {
  const gpus = processedData.filter(d => d.price >= r.min && d.price < r.max);
  if (gpus.length === 0) return null;
  const bestByPerf = gpus.sort((a, b) => b.g3dmark - a.g3dmark)[0];
  return {
    range: r.range,
    color: r.color,
    bestPerf: bestByPerf,
    minScore: d3.min(gpus, d => d.g3dmark),
    maxScore: d3.max(gpus, d => d.g3dmark),
    avgScore: d3.mean(gpus, d => d.g3dmark),
    count: gpus.length
  };
}).filter(d => d !== null);

const releaseData = (await releases)
  .map(d => ({ year: +d.release_year, count: +d.release }))
  .sort((a, b) => a.year - b.year);

const nvidiaPerf = benchmarkData
  .filter(d => d.brand === "NVIDIA" && d.testDate >= 2012 && d.testDate <= 2024);

const perfByYear = d3.rollups(
  nvidiaPerf,
  v => ({
    avgScore: d3.mean(v, d => d.g3dmark),
    topGpu: v.sort((a, b) => b.g3dmark - a.g3dmark)[0]?.gpuName
  }),
  d => d.testDate
).map(([year, stats]) => ({year, ...stats})).sort((a, b) => a.year - b.year);

const colorMap = { "NVIDIA": "#76b900", "AMD": "#ed1c24", "Intel": "#0071c5" };
```



<div class="key-stats">
  <div class="key-stat">
    <div class="value">${gpuCount}</div>
    <div class="label">GPUs Tracked</div>
  </div>
  <div class="key-stat">
    <div class="value">${brands.length}</div>
    <div class="label">Manufacturers</div>
  </div>
  <div class="key-stat">
    <div class="value">${topGpu.G3Dmark.toLocaleString()}</div>
    <div class="label">Top G3D Score</div>
  </div>
  <div class="key-stat">
    <div class="value">${totalReleases}</div>
    <div class="label">NVIDIA Releases</div>
  </div>
</div>

---

## Performance Overview

```js
display(html`<div class="chart-section">
  <div class="chart-container-full">
    <h3>Top 15 GPUs by G3Dmark Score</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      const height = isMobile ? 380 : 400;
      const padding = isMobile ? 1 : 2;
      const minW = isMobile ? 40 : 60;
      const minH = isMobile ? 25 : 35;
      const container = d3.create("div").style("position", "relative");
      const svg = container.append("svg").attr("width", width).attr("height", height);

      const root = d3.treemap()
        .size([width, height])
        .padding(padding)
        .round(true)
        (d3.hierarchy({children: topG3D}).sum(d => d.g3dmark));

      const leaves = svg.selectAll("g")
        .data(root.leaves())
        .join("g")
        .attr("transform", d => "translate(" + d.x0 + "," + d.y0 + ")");

      leaves.append("rect")
        .attr("width", d => d.x1 - d.x0)
        .attr("height", d => d.y1 - d.y0)
        .attr("fill", d => colorMap[d.data.brand] || "#888")
        .attr("rx", isMobile ? 3 : 4)
        .attr("opacity", 0.85);

      leaves.append("clipPath")
        .attr("id", (d, i) => "clip-top-" + i)
        .append("rect")
        .attr("width", d => d.x1 - d.x0)
        .attr("height", d => d.y1 - d.y0);

      leaves.append("text")
        .attr("clip-path", (d, i) => "url(#clip-top-" + i + ")")
        .attr("x", isMobile ? 4 : 6)
        .attr("y", isMobile ? 14 : 18)
        .attr("font-size", d => {
          const w = d.x1 - d.x0;
          const h = d.y1 - d.y0;
          if (w < minW || h < minH) return 0;
          return isMobile ? Math.min(10, w / 8) : Math.min(12, w / 10);
        })
        .attr("fill", "white")
        .attr("font-weight", "600")
        .style("text-shadow", "0 1px 3px rgba(0,0,0,0.6)")
        .text(d => {
          const w = d.x1 - d.x0;
          const maxChars = isMobile ? Math.floor(w / 6) : 20;
          return d.data.gpuName.length > maxChars ? d.data.gpuName.substring(0, maxChars - 2) + ".." : d.data.gpuName;
        });

      leaves.append("text")
        .attr("clip-path", (d, i) => "url(#clip-top-" + i + ")")
        .attr("x", isMobile ? 4 : 6)
        .attr("y", isMobile ? 26 : 34)
        .attr("font-size", d => {
          const w = d.x1 - d.x0;
          const h = d.y1 - d.y0;
          if (w < minW || h < (isMobile ? 35 : 50)) return 0;
          return isMobile ? 8 : 10;
        })
        .attr("fill", "rgba(255,255,255,0.9)")
        .style("text-shadow", "0 1px 3px rgba(0,0,0,0.6)")
        .text(d => d.data.g3dmark.toLocaleString());

      leaves.append("title")
        .text(d => d.data.gpuName + "\nG3Dmark: " + d.data.g3dmark.toLocaleString() + "\nBrand: " + d.data.brand);

      return container.node();
    })}
    <div style="display: flex; gap: 1.5rem; justify-content: center; margin-top: 0.5rem; font-size: 11px;">
      <span><span style="display: inline-block; width: 12px; height: 12px; background: #76b900; border-radius: 2px; margin-right: 4px;"></span>NVIDIA</span>
      <span><span style="display: inline-block; width: 12px; height: 12px; background: #ed1c24; border-radius: 2px; margin-right: 4px;"></span>AMD</span>
    </div>
  </div>

  <div class="chart-container-full">
    <h3>Top 10 OpenCL Performance</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      const height = isMobile ? 300 : 320;
      const padding = isMobile ? 1 : 2;
      const minW = isMobile ? 35 : 50;
      const minH = isMobile ? 22 : 30;
      const container = d3.create("div").style("position", "relative");
      const svg = container.append("svg").attr("width", width).attr("height", height);

      const root = d3.treemap()
        .size([width, height])
        .padding(padding)
        .round(true)
        (d3.hierarchy({children: topOpenCL}).sum(d => d.opencl));

      const leaves = svg.selectAll("g")
        .data(root.leaves())
        .join("g")
        .attr("transform", d => "translate(" + d.x0 + "," + d.y0 + ")");

      leaves.append("rect")
        .attr("width", d => d.x1 - d.x0)
        .attr("height", d => d.y1 - d.y0)
        .attr("fill", d => colorMap[d.data.manufacturer] || "#3b82f6")
        .attr("rx", isMobile ? 3 : 4)
        .attr("opacity", 0.85);

      leaves.append("clipPath")
        .attr("id", (d, i) => "clip-ocl-" + i)
        .append("rect")
        .attr("width", d => d.x1 - d.x0)
        .attr("height", d => d.y1 - d.y0);

      leaves.append("text")
        .attr("clip-path", (d, i) => "url(#clip-ocl-" + i + ")")
        .attr("x", isMobile ? 3 : 5)
        .attr("y", isMobile ? 13 : 16)
        .attr("font-size", d => {
          const w = d.x1 - d.x0;
          const h = d.y1 - d.y0;
          if (w < minW || h < minH) return 0;
          return isMobile ? Math.min(9, w / 8) : Math.min(10, w / 12);
        })
        .attr("fill", "white")
        .attr("font-weight", "600")
        .style("text-shadow", "0 1px 3px rgba(0,0,0,0.6)")
        .text(d => {
          const w = d.x1 - d.x0;
          const maxChars = isMobile ? Math.floor(w / 5) : 18;
          return d.data.device.substring(0, maxChars);
        });

      leaves.append("text")
        .attr("clip-path", (d, i) => "url(#clip-ocl-" + i + ")")
        .attr("x", isMobile ? 3 : 5)
        .attr("y", isMobile ? 24 : 30)
        .attr("font-size", d => {
          const w = d.x1 - d.x0;
          const h = d.y1 - d.y0;
          if (w < minW || h < (isMobile ? 32 : 45)) return 0;
          return isMobile ? 8 : 9;
        })
        .attr("fill", "rgba(255,255,255,0.9)")
        .style("text-shadow", "0 1px 3px rgba(0,0,0,0.6)")
        .text(d => d.data.opencl.toLocaleString());

      leaves.append("title")
        .text(d => d.data.device + "\nOpenCL: " + d.data.opencl.toLocaleString());

      return container.node();
    })}
  </div>
</div>`)
```

---

## Value Analysis

```js
display(html`<div class="chart-section">
  <div class="chart-container-full">
    <h3>Performance Range by Price Tier</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      return Plot.plot({
        width,
        height: isMobile ? 240 : 280,
        marginLeft: isMobile ? 95 : 110,
        marginRight: 60,
        marginBottom: 50,
        style: { background: "transparent" },
        x: { label: "G3Dmark Score", grid: true },
        y: { label: null },
        marks: [
          Plot.ruleX([0]),
          Plot.barX(bestPerRange, {
            x1: "minScore", x2: "maxScore", y: "range",
            fill: "color", fillOpacity: 0.3, rx: 4
          }),
          Plot.tickX(bestPerRange, {
            x: "avgScore", y: "range",
            stroke: "color", strokeWidth: 3
          }),
          Plot.dot(bestPerRange, {
            x: "maxScore", y: "range", fill: "color", r: 6, stroke: "white", strokeWidth: 2,
            tip: true, title: d => "Best: " + d.bestPerf.gpuName + "\nScore: " + d.maxScore.toLocaleString()
          }),
          Plot.text(bestPerRange, {
            x: "maxScore", y: "range",
            text: d => d.bestPerf.gpuName.substring(0, 18),
            dx: 8, textAnchor: "start", fontSize: 9, fill: "#79716B"
          })
        ]
      });
    })}
    <div style="display: flex; gap: 1.5rem; justify-content: center; margin-top: 0.5rem; font-size: 11px; color: #79716B;">
      <span>Bar = Score Range</span>
      <span>| = Average</span>
      <span>Dot = Top performer</span>
    </div>
  </div>

  <div class="chart-container-full">
    <h3>Top 10 Best Value GPUs</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      return Plot.plot({
        width,
        height: isMobile ? 340 : 380,
        marginLeft: isMobile ? 140 : 180,
        marginRight: 100,
        style: { background: "transparent" },
        x: { label: "Value Score (G3D points per $)", grid: true },
        y: { label: null },
        marks: [
          Plot.barX(topValue, {
            x: d => +d.valueScore, y: "gpuName",
            fill: d => colorMap[d.brand] || "#888",
            sort: {y: "-x"}, rx: 4, tip: true
          }),
          Plot.text(topValue, {
            x: d => +d.valueScore, y: "gpuName",
            text: d => d.valueScore + " | $" + d.price.toFixed(0),
            dx: 5, textAnchor: "start", fontSize: 10, fill: "#79716B"
          }),
          Plot.ruleX([0])
        ]
      });
    })}
  </div>
</div>`)
```

---

## NVIDIA Historical Trends

```js
display(html`<div class="chart-section">
  <div class="chart-container-full">
    <h3>GPU Releases by Year</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      return Plot.plot({
        width,
        height: isMobile ? 280 : 320,
        marginLeft: isMobile ? 45 : 50,
        marginRight: isMobile ? 20 : 30,
        marginBottom: isMobile ? 50 : 45,
        style: { background: "transparent" },
        x: { label: "Year", tickFormat: d => String(d), ticks: isMobile ? 6 : 11 },
        y: { label: "Number of Releases", grid: true },
        marks: [
          Plot.barY(releaseData, {
            x: "year", y: "count", fill: "#76b900", rx: 4, tip: true
          }),
          Plot.text(releaseData, {
            x: "year", y: "count",
            text: d => d.count,
            dy: -8, fontSize: 11, fontWeight: "600", fill: "#76b900"
          }),
          Plot.ruleY([0])
        ]
      });
    })}
  </div>

  <div class="chart-container-full">
    <h3>Average GPU Performance by Year</h3>
    ${resize((width) => {
      const isMobile = width < 640;
      return Plot.plot({
        width,
        height: isMobile ? 280 : 320,
        marginLeft: isMobile ? 60 : 70,
        marginRight: isMobile ? 20 : 30,
        marginBottom: isMobile ? 50 : 45,
        style: { background: "transparent" },
        x: { label: "Year", tickFormat: d => String(d), ticks: isMobile ? 5 : 8 },
        y: { label: "Avg G3Dmark Score", grid: true },
        marks: [
          Plot.areaY(perfByYear, {
            x: "year", y: "avgScore", fill: "#76b900", fillOpacity: 0.2, curve: "monotone-x"
          }),
          Plot.line(perfByYear, {
            x: "year", y: "avgScore", stroke: "#76b900", strokeWidth: 3, curve: "monotone-x"
          }),
          Plot.dot(perfByYear, {
            x: "year", y: "avgScore", fill: "#76b900", r: 6, stroke: "white", strokeWidth: 2,
            tip: true, title: d => String(d.year) + "\nAvg Score: " + Math.round(d.avgScore).toLocaleString() + "\nTop GPU: " + d.topGpu
          }),
          Plot.ruleY([0])
        ]
      });
    })}
  </div>
</div>`)
```

---

## Data Sources

- **GPU Benchmarks**: TechPowerUp GPU Database, PassMark
- **Price Data**: Current market pricing from major retailers
- **Release History**: NVIDIA official product database

<div class="dashboard-footer">
  <span>Built with Observable Framework</span>
  <span>Sources: <a href="https://www.techpowerup.com/gpu-specs/" target="_blank">TechPowerUp</a>, <a href="https://www.passmark.com/" target="_blank">PassMark</a>, <a href="https://www.nvidia.com/" target="_blank">NVIDIA</a></span>
</div>
