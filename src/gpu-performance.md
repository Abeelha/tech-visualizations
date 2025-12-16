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
const apiScores = FileAttachment("data/gpu_api_scores.csv").csv({typed: true});
const benchmarks = FileAttachment("data/gpu_benchmarks.csv").csv({typed: true});
```

```js
const processedData = (await apiScores)
  .map(d => ({
    manufacturer: d.Manufacturer,
    device: d.Device,
    cuda: +d.CUDA || 0,
    metal: +d.Metal || 0,
    opencl: +d.OpenCL || 0,
    vulkan: +d.Vulkan || 0
  }))
  .filter(d => d.cuda > 0 || d.opencl > 0 || d.vulkan > 0);

const topCuda = [...processedData].filter(d => d.cuda > 0).sort((a, b) => b.cuda - a.cuda).slice(0, 15);
const topOpenCL = [...processedData].filter(d => d.opencl > 0).sort((a, b) => b.opencl - a.opencl).slice(0, 15);
const topVulkan = [...processedData].filter(d => d.vulkan > 0).sort((a, b) => b.vulkan - a.vulkan).slice(0, 15);

const manufacturers = [...new Set(processedData.map(d => d.manufacturer))];

const topCudaScore = topCuda[0]?.cuda || 0;
const topOpenCLScore = topOpenCL[0]?.opencl || 0;
const topVulkanScore = topVulkan[0]?.vulkan || 0;
const gpuCount = processedData.length;

const benchmarkData = (await benchmarks)
  .filter(d => d.G3Dmark && d.brand)
  .map(d => ({
    gpuName: d.gpuName,
    g3dmark: +d.G3Dmark,
    brand: d.brand,
    category: d.category || "Unknown",
    tdp: +d.TDP || 0,
    year: +d.testDate || 0
  }));

const topG3D = [...benchmarkData].sort((a, b) => b.g3dmark - a.g3dmark).slice(0, 15);

const brandPerformance = d3.rollups(
  benchmarkData.filter(d => d.brand && d.brand !== "Other"),
  v => ({
    avgScore: d3.mean(v, d => d.g3dmark),
    maxScore: d3.max(v, d => d.g3dmark),
    count: v.length
  }),
  d => d.brand
).map(([brand, stats]) => ({brand, ...stats}))
 .sort((a, b) => b.avgScore - a.avgScore);

const categoryPerformance = d3.rollups(
  benchmarkData.filter(d => d.category && d.category !== "Unknown"),
  v => ({
    avgScore: d3.mean(v, d => d.g3dmark),
    count: v.length
  }),
  d => d.category
).map(([category, stats]) => ({category, ...stats}))
 .sort((a, b) => b.avgScore - a.avgScore);
```

<div class="hero">
  <h1>GPU Performance Comparison</h1>
  <p>Compare GPU performance across CUDA, OpenCL, Vulkan APIs and G3Dmark benchmarks</p>
</div>

```js
display(html`<div class="dashboard-layout">
  <div class="sidebar">
    <div class="stat-card">
      <div class="stat-label">Top CUDA Score</div>
      <div class="stat-value nvidia">${topCudaScore.toLocaleString()}</div>
      <div class="stat-change">${topCuda[0]?.device}</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Top OpenCL Score</div>
      <div class="stat-value" style="color: #3b82f6;">${topOpenCLScore.toLocaleString()}</div>
      <div class="stat-change">${topOpenCL[0]?.device}</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Top Vulkan Score</div>
      <div class="stat-value" style="color: #8b5cf6;">${topVulkanScore.toLocaleString()}</div>
      <div class="stat-change">${topVulkan[0]?.device}</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">GPUs Tracked</div>
      <div class="stat-value">${gpuCount}</div>
      <div class="stat-change">${manufacturers.length} manufacturers</div>
    </div>

    <div class="insights">
      <h4>Key Insights</h4>
      <ul>
        <li>RTX 3090 Ti leads consumer GPUs</li>
        <li>A100 dominates data center workloads</li>
        <li>Vulkan excels for gaming performance</li>
        <li>OpenCL best for cross-platform compute</li>
      </ul>
    </div>

    <div class="data-sources">
      <strong>Sources:</strong> <a href="https://www.techpowerup.com/gpu-specs/" target="_blank">TechPowerUp</a>, <a href="https://www.passmark.com/" target="_blank">PassMark</a>
    </div>
  </div>

  <div class="main-content">
    <div class="chart-container chart-large">
      <h3>Top 15 GPUs by G3Dmark Score</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        const colorMap = { "NVIDIA": "#76b900", "AMD": "#ed1c24", "Intel": "#0071c5", "Other": "#888888" };
        return Plot.plot({
          width,
          height: isMobile ? 380 : 420,
          marginLeft: isMobile ? 150 : 190,
          marginRight: 50,
          x: { label: "G3Dmark Score", grid: true },
          y: { label: null },
          marks: [
            Plot.barX(topG3D, {
              x: "g3dmark", y: "gpuName", fill: d => colorMap[d.brand] || "#888888", sort: {y: "-x"}, tip: true,
              title: d => d.gpuName + "\nG3Dmark: " + d.g3dmark.toLocaleString() + "\nBrand: " + d.brand + "\nCategory: " + d.category
            }),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Top 15 CUDA Performance</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 340 : 380,
            marginLeft: isMobile ? 130 : 170,
            marginRight: 30,
            x: { label: "CUDA Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(topCuda, {
                x: "cuda", y: "device", fill: "#76b900", sort: {y: "-x"}, tip: true,
                title: d => d.device + "\nCUDA: " + d.cuda.toLocaleString()
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Top 15 OpenCL Performance</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 340 : 380,
            marginLeft: isMobile ? 130 : 170,
            marginRight: 30,
            x: { label: "OpenCL Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(topOpenCL, {
                x: "opencl", y: "device", fill: "#3b82f6", sort: {y: "-x"}, tip: true,
                title: d => d.device + "\nOpenCL: " + d.opencl.toLocaleString()
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Top 15 Vulkan Performance</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 340 : 380,
            marginLeft: isMobile ? 130 : 170,
            marginRight: 30,
            x: { label: "Vulkan Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(topVulkan, {
                x: "vulkan", y: "device", fill: "#8b5cf6", sort: {y: "-x"}, tip: true,
                title: d => d.device + "\nVulkan: " + d.vulkan.toLocaleString()
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Average G3Dmark by Brand</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          const colorMap = { "NVIDIA": "#76b900", "AMD": "#ed1c24", "Intel": "#0071c5", "Other": "#888888" };
          return Plot.plot({
            width,
            height: isMobile ? 200 : 220,
            marginLeft: isMobile ? 70 : 80,
            marginRight: 50,
            x: { label: "Avg G3Dmark Score", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(brandPerformance, {
                x: "avgScore", y: "brand", fill: d => colorMap[d.brand] || "#888888", sort: {y: "-x"}, tip: true,
                title: d => d.brand + "\nAvg: " + Math.round(d.avgScore).toLocaleString() + "\nMax: " + d.maxScore.toLocaleString() + "\nGPUs: " + d.count
              }),
              Plot.text(brandPerformance, {
                x: "avgScore", y: "brand", text: d => d.count + " GPUs", dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>

    <div class="chart-container chart-large">
      <h3>Average Performance by Category</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 200 : 220,
          marginLeft: isMobile ? 100 : 120,
          marginRight: 50,
          x: { label: "Avg G3Dmark Score", grid: true },
          y: { label: null },
          marks: [
            Plot.barX(categoryPerformance, {
              x: "avgScore", y: "category", fill: "#f59e0b", sort: {y: "-x"}, tip: true,
              title: d => d.category + "\nAvg Score: " + Math.round(d.avgScore).toLocaleString() + "\nGPUs: " + d.count
            }),
            Plot.text(categoryPerformance, {
              x: "avgScore", y: "category", text: d => d.count + " GPUs", dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
            }),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>
  </div>
</div>`)
```
