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
const storageData = FileAttachment("data/cloud_storage_pricing.csv").csv({typed: true});
```

```js
const providers = (await storageData).map(d => ({
  provider: d.Provider,
  storageCost: +d["Storage Price ($/TB)"] || 0,
  downloadCost: +d["Download Price ($/TB)"] || 0,
  url: d.URL || null,
  notes: d.Notes || ""
})).filter(d => d.provider);

const byStorage = [...providers].sort((a, b) => a.storageCost - b.storageCost);
const byDownload = [...providers].sort((a, b) => a.downloadCost - b.downloadCost);
const cheapestStorage = byStorage[0];
const cheapestDownload = byDownload[0];

const avgStorage = d3.mean(providers, d => d.storageCost);
const avgDownload = d3.mean(providers, d => d.downloadCost);
const freeEgressCount = providers.filter(d => d.downloadCost === 0).length;
const totalProviders = providers.length;

const usageCosts = providers.map(d => ({
  ...d,
  lightUsage: d.storageCost + d.downloadCost * 0.5,
  moderateUsage: d.storageCost + d.downloadCost * 2,
  heavyUsage: d.storageCost + d.downloadCost * 5
})).sort((a, b) => a.moderateUsage - b.moderateUsage);

const freeEgressProviders = providers.filter(d => d.downloadCost === 0);
```

<div class="hero">
  <h1>Cloud Storage Pricing Comparison</h1>
  <p>Compare storage and egress costs across major cloud providers - prices per TB/month</p>
</div>

```js
display(html`<div class="dashboard-layout">
  <div class="sidebar">
    <div class="stat-card">
      <div class="stat-label">Cheapest Storage</div>
      <div class="stat-value positive">${cheapestStorage?.provider}</div>
      <div class="stat-change">$${cheapestStorage?.storageCost}/TB/mo</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Free Egress</div>
      <div class="stat-value" style="color: #3b82f6;">${freeEgressCount} providers</div>
      <div class="stat-change">$0/TB download cost</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Avg Storage Cost</div>
      <div class="stat-value">$${avgStorage.toFixed(1)}</div>
      <div class="stat-change">per TB/month</div>
    </div>

    <div class="stat-card">
      <div class="stat-label">Avg Download Cost</div>
      <div class="stat-value">$${avgDownload.toFixed(1)}</div>
      <div class="stat-change">per TB egress</div>
    </div>

    <div class="insights">
      <h4>Key Insights</h4>
      <ul>
        <li>Cloudflare R2 offers free egress with competitive storage</li>
        <li>Traditional clouds (AWS, GCP) have high egress fees</li>
        <li>Backblaze, Wasabi offer free egress with ratio limits</li>
        <li>Storj cheapest storage but segment pricing for small files</li>
      </ul>
    </div>

    <div class="data-sources">
      <strong>Sources:</strong> <a href="https://gist.github.com/Manouchehri/733e6235457e60de24fdbb15046fba7f" target="_blank">Provider Analysis</a>
    </div>
  </div>

  <div class="main-content">
    <div class="chart-container chart-large">
      <h3>Storage vs Download Cost ($/TB) - Lower is Better</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 340 : 380,
          marginLeft: isMobile ? 50 : 60,
          marginRight: isMobile ? 20 : 30,
          marginBottom: isMobile ? 50 : 45,
          x: { label: "Storage ($/TB/mo)", grid: true },
          y: { label: "Download ($/TB)", grid: true },
          marks: [
            Plot.dot(providers, {
              x: "storageCost",
              y: "downloadCost",
              r: 8,
              fill: d => d.downloadCost === 0 ? "#059669" : "#3b82f6",
              stroke: "white",
              strokeWidth: 2,
              tip: true,
              title: d => d.provider + "\nStorage: $" + d.storageCost + "/TB\nDownload: $" + d.downloadCost + "/TB\n" + (d.notes ? "Note: " + d.notes.substring(0, 50) + "..." : "")
            }),
            Plot.text(providers, {
              x: "storageCost",
              y: "downloadCost",
              text: "provider",
              dy: -12,
              fontSize: isMobile ? 9 : 10,
              fill: "#666"
            }),
            Plot.ruleY([0]),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Storage Cost by Provider ($/TB/mo)</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 420 : 460,
            marginLeft: isMobile ? 100 : 120,
            marginRight: 50,
            x: { label: "$/TB/month", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(byStorage, {
                x: "storageCost",
                y: "provider",
                fill: d => d.storageCost <= 10 ? "#059669" : d.storageCost <= 20 ? "#f59e0b" : "#dc2626",
                sort: {y: "x"},
                tip: true,
                title: d => d.provider + "\n$" + d.storageCost + "/TB/month"
              }),
              Plot.text(byStorage, {
                x: "storageCost",
                y: "provider",
                text: d => "$" + d.storageCost,
                dx: 5,
                textAnchor: "start",
                fontSize: 10,
                fill: "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>Download/Egress Cost by Provider ($/TB)</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 420 : 460,
            marginLeft: isMobile ? 100 : 120,
            marginRight: 50,
            x: { label: "$/TB egress", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(byDownload, {
                x: "downloadCost",
                y: "provider",
                fill: d => d.downloadCost === 0 ? "#059669" : d.downloadCost <= 20 ? "#f59e0b" : "#dc2626",
                sort: {y: "x"},
                tip: true,
                title: d => d.provider + "\n$" + d.downloadCost + "/TB egress" + (d.notes ? "\nNote: " + d.notes.substring(0, 60) : "")
              }),
              Plot.text(byDownload, {
                x: "downloadCost",
                y: "provider",
                text: d => d.downloadCost === 0 ? "FREE" : "$" + d.downloadCost,
                dx: 5,
                textAnchor: "start",
                fontSize: 10,
                fill: d => d.downloadCost === 0 ? "#059669" : "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>
    </div>

    <div class="chart-container chart-large">
      <h3>Monthly Cost Comparison: 1TB Storage + 2TB Download</h3>
      ${resize((width) => {
        const isMobile = width < 640;
        return Plot.plot({
          width,
          height: isMobile ? 420 : 460,
          marginLeft: isMobile ? 100 : 120,
          marginRight: 60,
          x: { label: "Total Monthly Cost ($)", grid: true },
          y: { label: null },
          marks: [
            Plot.barX(usageCosts, {
              x: "moderateUsage",
              y: "provider",
              fill: d => d.moderateUsage <= 20 ? "#059669" : d.moderateUsage <= 50 ? "#f59e0b" : "#dc2626",
              sort: {y: "x"},
              tip: true,
              title: d => d.provider + "\nTotal: $" + d.moderateUsage.toFixed(2) + "/mo\nStorage: $" + d.storageCost + "\nDownload (2TB): $" + (d.downloadCost * 2).toFixed(2)
            }),
            Plot.text(usageCosts, {
              x: "moderateUsage",
              y: "provider",
              text: d => "$" + d.moderateUsage.toFixed(0),
              dx: 5,
              textAnchor: "start",
              fontSize: 10,
              fill: "#666"
            }),
            Plot.ruleX([0])
          ]
        });
      })}
    </div>

    <div class="chart-grid">
      <div class="chart-container">
        <h3>Free Egress Providers</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          return Plot.plot({
            width,
            height: isMobile ? 200 : 220,
            marginLeft: isMobile ? 100 : 120,
            marginRight: 50,
            x: { label: "Storage Cost ($/TB/mo)", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(freeEgressProviders.sort((a, b) => a.storageCost - b.storageCost), {
                x: "storageCost",
                y: "provider",
                fill: "#059669",
                sort: {y: "x"},
                tip: true,
                title: d => d.provider + "\nStorage: $" + d.storageCost + "/TB\nDownload: FREE\n" + (d.notes ? "Condition: " + d.notes.substring(0, 80) : "")
              }),
              Plot.text(freeEgressProviders, {
                x: "storageCost",
                y: "provider",
                text: d => "$" + d.storageCost,
                dx: 5,
                textAnchor: "start",
                fontSize: 10,
                fill: "#666"
              }),
              Plot.ruleX([0])
            ]
          });
        })}
      </div>

      <div class="chart-container">
        <h3>High Egress Cost Providers (>$50/TB)</h3>
        ${resize((width) => {
          const isMobile = width < 640;
          const highEgress = providers.filter(d => d.downloadCost >= 50).sort((a, b) => b.downloadCost - a.downloadCost);
          return Plot.plot({
            width,
            height: isMobile ? 200 : 220,
            marginLeft: isMobile ? 100 : 120,
            marginRight: 50,
            x: { label: "Download Cost ($/TB)", grid: true },
            y: { label: null },
            marks: [
              Plot.barX(highEgress, {
                x: "downloadCost",
                y: "provider",
                fill: "#dc2626",
                sort: {y: "-x"},
                tip: true,
                title: d => d.provider + "\nDownload: $" + d.downloadCost + "/TB\nStorage: $" + d.storageCost + "/TB"
              }),
              Plot.text(highEgress, {
                x: "downloadCost",
                y: "provider",
                text: d => "$" + d.downloadCost,
                dx: 5,
                textAnchor: "start",
                fontSize: 10,
                fill: "#666"
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
