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
const apiScores = FileAttachment("data/gpu_api_scores.csv").csv({typed: true});
const releases = FileAttachment("data/nvidia_releases.csv").csv({typed: true});
```

```js
const gpuCount = (await benchmarks).length;
const topGpu = (await benchmarks)[0];
const brands = [...new Set((await benchmarks).map(d => d.brand))].filter(b => b);
const totalReleases = (await releases).reduce((sum, d) => sum + d.release, 0);
```

<div class="hero-large">
  <h1>GPU Performance & Pricing Data Portal</h1>
  <p>Explore comprehensive GPU benchmarks, performance comparisons across graphics APIs, pricing trends, and market forecasts.</p>
</div>

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

<div class="landing-grid">
  <div class="landing-card">
    <a href="./gpu-performance">
      <h3>GPU Performance Comparison</h3>
      <p>Compare GPU performance across CUDA, OpenCL, and Vulkan APIs. Analyze compute scores for deep learning, simulations, and gaming workloads.</p>
    </a>
  </div>
  <div class="landing-card">
    <a href="./price-performance">
      <h3>Price vs Performance</h3>
      <p>Find the best value GPUs with price-to-performance analysis. Compare G3Dmark scores against retail pricing to identify optimal purchases.</p>
    </a>
  </div>
  <div class="landing-card">
    <a href="./nvidia-releases">
      <h3>NVIDIA Historical Releases</h3>
      <p>Track NVIDIA's GPU release history from 2012 to present. Visualize release patterns and product launch frequency by year.</p>
    </a>
  </div>
</div>

---

## Data Overview

```js
const topPerformers = (await benchmarks)
  .filter(d => d.G3Dmark && d.brand)
  .sort((a, b) => b.G3Dmark - a.G3Dmark)
  .slice(0, 10);

const brandCounts = d3.rollups(
  (await benchmarks).filter(d => d.brand),
  v => v.length,
  d => d.brand
).map(([brand, count]) => ({brand, count}))
 .sort((a, b) => b.count - a.count);
```

<div class="grid grid-cols-2" style="grid-auto-rows: auto;">
  <div class="card">
    <h3>Top 10 GPUs by G3Dmark Score</h3>
    ${resize((width) => Plot.plot({
      width,
      height: 300,
      marginLeft: 180,
      marginRight: 40,
      x: {
        label: "G3Dmark Score",
        grid: true
      },
      y: {
        label: null
      },
      color: {
        domain: ["NVIDIA", "AMD", "Intel", "Other"],
        range: ["#76b900", "#ed1c24", "#0071c5", "#888888"]
      },
      marks: [
        Plot.barX(topPerformers, {
          x: "G3Dmark",
          y: "gpuName",
          fill: "brand",
          sort: {y: "-x"},
          tip: true,
          title: d => `${d.gpuName}\nScore: ${d.G3Dmark.toLocaleString()}\nPrice: ${d.price ? '$' + d.price.toFixed(2) : 'N/A'}`
        }),
        Plot.ruleX([0])
      ]
    }))}
  </div>
  <div class="card">
    <h3>GPUs by Manufacturer</h3>
    ${resize((width) => Plot.plot({
      width,
      height: 300,
      marginLeft: 80,
      x: {
        label: "Number of GPUs",
        grid: true
      },
      y: {
        label: null
      },
      color: {
        domain: ["NVIDIA", "AMD", "Intel", "Other"],
        range: ["#76b900", "#ed1c24", "#0071c5", "#888888"]
      },
      marks: [
        Plot.barX(brandCounts, {
          x: "count",
          y: "brand",
          fill: "brand",
          sort: {y: "-x"},
          tip: true
        }),
        Plot.ruleX([0])
      ]
    }))}
  </div>
</div>

---

## Data Sources

- **GPU Benchmarks**: TechPowerUp GPU Database, PassMark
- **Price Data**: Current market pricing from major retailers
- **Forecasts**: Statista market projections
- **Release History**: NVIDIA official product database
