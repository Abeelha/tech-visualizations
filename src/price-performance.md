---
toc: false
---

<div class="portaljs-banner">
  <div class="portaljs-banner-content">
    <span class="portaljs-banner-icon">🌀</span>
    <div class="portaljs-banner-text">
      <p class="portaljs-banner-title">Create beautiful data portals with PortalJS</p>
      <p class="portaljs-banner-description">The open-source framework for building data catalogs, dashboards, and visualizations.</p>
    </div>
  </div>
  <a href="https://www.portaljs.com/" target="_blank" rel="noopener noreferrer" class="portaljs-banner-cta">
    Get Started Free <span class="portaljs-banner-cta-arrow">→</span>
  </a>
</div>

```js
const benchmarks = FileAttachment("data/gpu_benchmarks.csv").csv({typed: true});
```

```js
const processedData = (await benchmarks)
  .filter(d => d.price && d.G3Dmark && d.price > 0 && d.G3Dmark > 0)
  .map(d => ({
    gpuName: d.gpuName,
    g3dmark: +d.G3Dmark,
    price: +d.price,
    brand: d.brand || "Other",
    category: d.category || "Unknown",
    tdp: +d.TDP || 0,
    gpuValue: +d.gpuValue || 0,
    powerPerf: +d.powerPerformance || 0,
    valueScore: (+d.G3Dmark / +d.price).toFixed(2)
  }));

const byValue = [...processedData].sort((a, b) => b.valueScore - a.valueScore);
const topValue = byValue.slice(0, 15);
const byPerformance = [...processedData].sort((a, b) => b.g3dmark - a.g3dmark);

const avgPrice = d3.mean(processedData, d => d.price);
const avgValue = d3.mean(processedData, d => +d.valueScore);
const bestValue = byValue[0];
const topPerformer = byPerformance[0];
const gpuCount = processedData.length;

const brandAvgValue = d3.rollups(
  processedData,
  v => ({
    avgValue: d3.mean(v, d => +d.valueScore),
    avgPrice: d3.mean(v, d => d.price),
    avgScore: d3.mean(v, d => d.g3dmark),
    count: v.length
  }),
  d => d.brand
).map(([brand, stats]) => ({brand, ...stats}))
 .sort((a, b) => b.avgValue - a.avgValue);

const categoryData = d3.rollups(
  processedData,
  v => ({
    count: v.length,
    avgPrice: d3.mean(v, d => d.price),
    avgScore: d3.mean(v, d => d.g3dmark),
    avgValue: d3.mean(v, d => +d.valueScore)
  }),
  d => d.category
).map(([cat, stats]) => ({category: cat, ...stats}))
 .filter(d => d.category !== "Unknown")
 .sort((a, b) => b.avgValue - a.avgValue);

const priceRanges = [
  {range: "$0-500", min: 0, max: 500},
  {range: "$500-1000", min: 500, max: 1000},
  {range: "$1000-2000", min: 1000, max: 2000},
  {range: "$2000-5000", min: 2000, max: 5000},
  {range: "$5000+", min: 5000, max: Infinity}
];

const byPriceRange = priceRanges.map(r => {
  const gpus = processedData.filter(d => d.price >= r.min && d.price < r.max);
  return {
    range: r.range,
    count: gpus.length,
    avgScore: gpus.length > 0 ? d3.mean(gpus, d => d.g3dmark) : 0,
    avgValue: gpus.length > 0 ? d3.mean(gpus, d => +d.valueScore) : 0
  };
}).filter(d => d.count > 0);

const topByBrand = d3.groups(processedData, d => d.brand)
  .map(([brand, gpus]) => {
    const sorted = gpus.sort((a, b) => +b.valueScore - +a.valueScore);
    return sorted.slice(0, 3);
  }).flat();
```

<div class="hero">
  <h1>Price vs Performance Analysis</h1>
  <p>Find the best value GPUs by comparing G3Dmark benchmark scores against retail pricing</p>
</div>

```js
display(html`<div class="dashboard-layout">
  <div class="sidebar">
    <div class="stat-card">
      <div class="stat-label">Best Value GPU</div>
      <div class="stat-value positive">${bestValue?.gpuName.substring(0, 18)}</div>
      <div class="stat-change">${bestValue?.valueScore} pts/$</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Top Performer</div>
      <div class="stat-value nvidia">${topPerformer?.gpuName.substring(0, 18)}</div>
      <div class="stat-change">${topPerformer?.g3dmark.toLocaleString()} G3D</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Avg Price</div>
      <div class="stat-value">$${Math.round(avgPrice).toLocaleString()}</div>
      <div class="stat-change">${gpuCount} GPUs analyzed</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Avg Value Score</div>
      <div class="stat-value">${avgValue.toFixed(1)}</div>
      <div class="stat-change">G3D points per dollar</div>
    </div>

    <div class="insights">
      <h4>Key Insights</h4>
      <ul>
        <li>Mid-range GPUs ($500-1000) offer best value</li>
        <li>RTX 3060 Ti excels in price/performance</li>
        <li>Workstation cards have premium pricing</li>
        <li>AMD competitive in value segment</li>
      </ul>
    </div>

    <div class="data-sources">
      <strong>Sources:</strong> <a href="https://www.techpowerup.com/gpu-specs/" target="_blank">TechPowerUp</a>, Market Pricing
    </div>
  </div>

  <div class="main-content">
    <div class="chart-container chart-large">
      <h3>Top 15 Best Value GPUs (G3Dmark Points per Dollar)</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        const colorMap = { "NVIDIA": "#76b900", "AMD": "#ed1c24", "Intel": "#0071c5", "Other": "#888888" };
        return Plot.plot({
          width,
          height: isMobile ? 380 : 420,
          marginLeft: isMobile ? 140 : 180,
          marginRight: 60,
          x: { label: "Value Score (G3D pts/$)", grid: true },
          y: { label: null },
          marks: [
            Plot.barX(topValue, {
              x: d => +d.valueScore, y: "gpuName", fill: d => colorMap[d.brand] || "#888888", sort: {y: "-x"}, tip: true,
              title: d => d.gpuName + "\nValue: " + d.valueScore + " pts/$\nPrice: $" + d.price.toFixed(0) + "\nG3D: " + d.g3dmark.toLocaleString() + "\nBrand: " + d.brand
            }),
            Plot.text(topValue, {
              x: d => +d.valueScore, y: "gpuName", text: d => "$" + d.price.toFixed(0), dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
            }),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Performance by Price Range</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 280 : 300,
            marginLeft: isMobile ? 80 : 90,
            marginRight: 40,
            x: { label: "Avg G3Dmark Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(byPriceRange, {
                x: "avgScore", y: "range", fill: "#3b82f6", tip: true,
                title: d => d.range + "\nAvg Score: " + Math.round(d.avgScore).toLocaleString() + "\nGPU Count: " + d.count + "\nAvg Value: " + d.avgValue.toFixed(1) + " pts/$"
              }),
              Plot.text(byPriceRange, {
                x: "avgScore", y: "range", text: d => d.count + " GPUs", dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Value Score by Price Range</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 280 : 300,
            marginLeft: isMobile ? 80 : 90,
            marginRight: 40,
            x: { label: "Avg Value (G3D pts/$)", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(byPriceRange, {
                x: "avgValue", y: "range", fill: "#059669", tip: true,
                title: d => d.range + "\nAvg Value: " + d.avgValue.toFixed(1) + " pts/$\nGPU Count: " + d.count
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Brand Comparison - Average Value</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          const colorMap = { "NVIDIA": "#76b900", "AMD": "#ed1c24", "Intel": "#0071c5", "Other": "#888888" };
          return Plot.plot({
            width,
            height: isMobile ? 200 : 220,
            marginLeft: isMobile ? 70 : 80,
            marginRight: 50,
            x: { label: "Avg Value (G3D pts/$)", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(brandAvgValue, {
                x: "avgValue", y: "brand", fill: d => colorMap[d.brand] || "#888888", sort: {y: "-x"}, tip: true,
                title: d => d.brand + "\nAvg Value: " + d.avgValue.toFixed(1) + " pts/$\nAvg Price: $" + Math.round(d.avgPrice) + "\nGPU Count: " + d.count
              }),
              Plot.text(brandAvgValue, {
                x: "avgValue", y: "brand", text: d => d.count + " GPUs", dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Performance by Category</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 200 : 220,
            marginLeft: isMobile ? 90 : 100,
            marginRight: 40,
            x: { label: "Avg G3Dmark Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(categoryData, {
                x: "avgScore", y: "category", fill: "#8b5cf6", sort: {y: "-x"}, tip: true,
                title: d => d.category + "\nAvg G3D: " + Math.round(d.avgScore).toLocaleString() + "\nAvg Price: $" + Math.round(d.avgPrice).toLocaleString() + "\nCount: " + d.count + " GPUs"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>
  </div>
</div>`)
```
