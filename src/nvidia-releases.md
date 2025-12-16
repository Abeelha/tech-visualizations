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
const releases = FileAttachment("data/nvidia_releases.csv").csv({typed: true});
const benchmarks = FileAttachment("data/gpu_benchmarks.csv").csv({typed: true});
```

```js
const releaseData = (await releases)
  .map(d => ({
    year: +d.release_year,
    count: +d.release
  }))
  .sort((a, b) => a.year - b.year);

const nvidiaPerf = (await benchmarks)
  .filter(d => d.brand === "NVIDIA" && d.testDate && d.G3Dmark)
  .map(d => ({
    gpuName: d.gpuName,
    year: +d.testDate,
    g3dmark: +d.G3Dmark,
    price: +d.price || 0,
    category: d.category || "Unknown"
  }))
  .filter(d => d.year >= 2012 && d.year <= 2022);

const perfByYear = d3.rollups(
  nvidiaPerf,
  v => ({
    avgScore: d3.mean(v, d => d.g3dmark),
    maxScore: d3.max(v, d => d.g3dmark),
    minScore: d3.min(v, d => d.g3dmark),
    count: v.length,
    topGpu: v.sort((a, b) => b.g3dmark - a.g3dmark)[0]?.gpuName
  }),
  d => d.year
).map(([year, stats]) => ({year, ...stats}))
 .sort((a, b) => a.year - b.year);

const cumulativeData = releaseData.map((d, i, arr) => ({
  year: d.year,
  count: d.count,
  cumulative: arr.slice(0, i + 1).reduce((sum, x) => sum + x.count, 0)
}));

const totalReleases = releaseData.reduce((sum, d) => sum + d.count, 0);
const peakYear = releaseData.reduce((max, d) => d.count > max.count ? d : max, {count: 0});
const years = releaseData.map(d => d.year);
const minYear = d3.min(years);
const maxYear = d3.max(years);
const avgPerYear = (totalReleases / releaseData.length).toFixed(1);

const topNvidiaGpus = nvidiaPerf.sort((a, b) => b.g3dmark - a.g3dmark).slice(0, 10);
```

<div class="hero">
  <h1>NVIDIA Historical Releases</h1>
  <p>Track NVIDIA GPU release history from ${minYear} to ${maxYear} - product launches and performance evolution</p>
</div>

```js
display(html`<div class="dashboard-layout">
  <div class="sidebar">
    <div class="stat-card">
      <div class="stat-label">Total Releases</div>
      <div class="stat-value nvidia">${totalReleases}</div>
      <div class="stat-change">${minYear}-${maxYear}</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Peak Year</div>
      <div class="stat-value">${peakYear.year}</div>
      <div class="stat-change">${peakYear.count} GPUs released</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Avg Per Year</div>
      <div class="stat-value">${avgPerYear}</div>
      <div class="stat-change">GPUs per year</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Years Tracked</div>
      <div class="stat-value">${years.length}</div>
      <div class="stat-change">${minYear} to ${maxYear}</div>
    </div>

    <div class="insights">
      <h4>Key Insights</h4>
      <ul>
        <li>2013 saw most releases (25 GPUs)</li>
        <li>Release frequency declined after 2016</li>
        <li>Focus shifted to fewer, powerful cards</li>
        <li>RTX series launched in 2018</li>
      </ul>
    </div>

    <div class="data-sources">
      <strong>Sources:</strong> <a href="https://www.nvidia.com/" target="_blank">NVIDIA</a>, <a href="https://www.techpowerup.com/gpu-specs/" target="_blank">TechPowerUp</a>
    </div>
  </div>

  <div class="main-content">
    <div class="chart-container chart-large">
      <h3>GPU Releases by Year</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 280 : 320,
          marginLeft: isMobile ? 45 : 50,
          marginRight: isMobile ? 20 : 30,
          marginBottom: isMobile ? 50 : 45,
          x: { label: "Year", tickFormat: d => d.toString(), ticks: isMobile ? 6 : 11 },
          y: { label: "Number of Releases", grid: true },
          marks: [
            Plot.barY(releaseData, {
              x: "year", y: "count", fill: "#76b900", tip: true,
              title: d => d.year + "\nReleases: " + d.count
            }),
            Plot.ruleY([0])
          ]
        });
      })}
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Cumulative Releases Over Time</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 260 : 300,
            marginLeft: isMobile ? 45 : 50,
            marginRight: isMobile ? 20 : 30,
            marginBottom: isMobile ? 50 : 45,
            x: { label: "Year", tickFormat: d => d.toString(), ticks: isMobile ? 5 : 8 },
            y: { label: "Cumulative Releases", grid: true },
            marks: [
              Plot.areaY(cumulativeData, { x: "year", y: "cumulative", fill: "#76b900", fillOpacity: 0.3, curve: "monotone-x" }),
              Plot.line(cumulativeData, { x: "year", y: "cumulative", stroke: "#76b900", strokeWidth: 2.5, curve: "monotone-x" }),
              Plot.dot(cumulativeData, {
                x: "year", y: "cumulative", fill: "#76b900", r: 4, stroke: "white", strokeWidth: 1, tip: true,
                title: d => d.year + ": " + d.cumulative + " total releases"
              }),
              Plot.ruleY([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Releases Ranked by Year</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          const sortedByCount = [...releaseData].sort((a, b) => b.count - a.count);
          return Plot.plot({
            width,
            height: isMobile ? 260 : 300,
            marginLeft: isMobile ? 50 : 60,
            marginRight: 40,
            x: { label: "Number of Releases", grid: true },
            y: { label: null, tickFormat: d => d.toString() },
            marks: [
              Plot.barX(sortedByCount, {
                x: "count", y: "year", fill: "#76b900", sort: {y: "-x"}, tip: true,
                title: d => d.year + ": " + d.count + " releases"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>

    <div class="chart-container chart-large">
      <h3>Performance Evolution (G3Dmark by Year)</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 280 : 320,
          marginLeft: isMobile ? 60 : 70,
          marginRight: isMobile ? 20 : 30,
          marginBottom: isMobile ? 50 : 45,
          x: { label: "Year", tickFormat: d => d.toString(), ticks: isMobile ? 5 : 8 },
          y: { label: "G3Dmark Score", grid: true },
          marks: [
            Plot.areaY(perfByYear, { x: "year", y: "maxScore", y1: "minScore", fill: "#76b900", fillOpacity: 0.2, curve: "monotone-x" }),
            Plot.line(perfByYear, { x: "year", y: "avgScore", stroke: "#76b900", strokeWidth: 2.5, curve: "monotone-x" }),
            Plot.line(perfByYear, { x: "year", y: "maxScore", stroke: "#059669", strokeWidth: 2, strokeDasharray: "4,4", curve: "monotone-x" }),
            Plot.dot(perfByYear, {
              x: "year", y: "avgScore", fill: "#76b900", r: 5, stroke: "white", strokeWidth: 2, tip: true,
              title: d => d.year + "\nAvg: " + Math.round(d.avgScore).toLocaleString() + "\nMax: " + Math.round(d.maxScore).toLocaleString() + "\nTop: " + d.topGpu
            }),
            Plot.ruleY([0])
          ]
        });
      })}
    </div>

    <div class="chart-container chart-large">
      <h3>Top 10 NVIDIA GPUs by G3Dmark</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 280 : 300,
          marginLeft: isMobile ? 150 : 180,
          marginRight: 50,
          x: { label: "G3Dmark Score", grid: true },
          y: { label: null },
          marks: [
            Plot.barX(topNvidiaGpus, {
              x: "g3dmark", y: "gpuName", fill: "#76b900", sort: {y: "-x"}, tip: true,
              title: d => d.gpuName + "\nG3Dmark: " + d.g3dmark.toLocaleString() + "\nYear: " + d.year
            }),
            Plot.text(topNvidiaGpus, {
              x: "g3dmark", y: "gpuName", text: d => d.year, dx: 5, textAnchor: "start", fontSize: 10, fill: "#666"
            }),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>
  </div>
</div>`)
```
