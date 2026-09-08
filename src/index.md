---
title: 100 GW Media Tracking Dashboard
toc: false
---

<style>
  main {
    max-width: 1600px !important;
  }
</style>

```js
// BLOCK 1: FETCH THE DATA
const url = "https://docs.google.com/spreadsheets/d/e/2PACX-1vTwKbjAkYhQ-VaVdt3WBWjP4x1RkGESINxQPgowiUtUWqkk-9mE2Pse0PqfgTgYjdTf5af5d_r-520g/pub?gid=0&single=true&output=csv";
const data = await d3.csv(url, d3.autoType);
```

```js
// BLOCK: EXPORT HELPER FUNCTIONS
function exportCSV(data, filename) {
  const csv = d3.csvFormat(data);
  const blob = new Blob([csv], { type: "text/csv" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename + ".csv";
  a.click();
  URL.revokeObjectURL(url);
}

function exportJSON(data, filename) {
  const json = JSON.stringify(data, null, 2);
  const blob = new Blob([json], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename + ".json";
  a.click();
  URL.revokeObjectURL(url);
}

function exportSVG(plotNode, filename) {
  // Find the SVG element (Plot sometimes wraps it in a <figure>)
  const svg = plotNode.tagName.toLowerCase() === "svg" ? plotNode : plotNode.querySelector("svg");
  if (!svg) return alert("SVG not found");
  
  // Ensure XML namespace is present for standard SVG image viewers
  if (!svg.getAttribute("xmlns")) svg.setAttribute("xmlns", "http://www.w3.org/2000/svg");
  
  const serializer = new XMLSerializer();
  const svgString = serializer.serializeToString(svg);
  const blob = new Blob([svgString], { type: "image/svg+xml;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename + ".svg";
  a.click();
  URL.revokeObjectURL(url);
}
```

```js
// BLOCK 2: SEARCH + TAG FILTERS
const searchResults = view(Inputs.search(data, {
  placeholder: "Cari kata kunci di judul...",
  columns: ["ARTICLE_TITLE"],
  width: "100%"
}));
```

```js
const { selectedTag, filterTarget } = filterState;
```

```js
// COMBINED FILTER INPUTS WITH DYNAMIC COUNTS
const filterState = view((function(prev) {
  // Preserve previous selections so they don't reset when using the search bar
  let currentTag = prev ? prev.value.selectedTag : "Semua";
  let currentTarget = prev ? prev.value.filterTarget : "Judul";
  
  // Setup the main container styling
  const container = document.createElement("div");
  container.style.display = "flex";
  container.style.flexDirection = "column";
  container.style.gap = "12px";
  container.style.fontFamily = "var(--sans-serif, system-ui, sans-serif)";
  container.style.fontSize = "13px";
  
  const tags = ["Semua", "Prabowo", "Bahlil", "Koperasi", "Mentari", "Danantara", "PLN", "Pertamina", "Satgas", "PLTS"];
  const targets = ["Judul", "Konten", "Judul + Konten"];
  
  // Build the layout skeleton
container.style.maxWidth = "100%"; 
container.style.boxSizing = "border-box";

container.innerHTML = `
  <div style="display: flex; gap: 12px; align-items: flex-start;">
    <div style="width: 150px; flex-shrink: 0; font-weight: 500; padding-top: 2px;">
      Filter berdasarkan entitas
    </div>
    <div id="tags-container" style="display: flex; flex-wrap: wrap; gap: 10px 16px; flex: 1; min-width: 0;"></div>
  </div>
  
  <div style="display: flex; gap: 12px; align-items: flex-start;">
    <div style="width: 150px; flex-shrink: 0; font-weight: 500; padding-top: 2px;">
      Pencarian entitas pada:
    </div>
    <div id="targets-container" style="display: flex; flex-wrap: wrap; gap: 10px 16px; flex: 1; min-width: 0;"></div>
  </div>
`;
  
  const tagsContainer = container.querySelector("#tags-container");
  const targetsContainer = container.querySelector("#targets-container");
  
  // Core logic to count matching articles
  function getMatchCount(tag, target) {
    if (tag === "Semua") return searchResults.length;
    const regex = new RegExp(tag, "i");
    
    return searchResults.filter(d => {
      const t = regex.test(d.ARTICLE_TITLE ?? "");
      const c = regex.test(d.CONTENT ?? "");
      
      if (target === "Judul") return t;
      if (target === "Konten") return c;
      if (target === "Judul + Konten") return t || c;
      return false;
    }).length;
  }
  
  // Render and attach events
  function render() {
    // 1. Render Entity Tags
    tagsContainer.innerHTML = "";
    tags.forEach(tag => {
      const count = getMatchCount(tag, currentTarget);
      const label = document.createElement("label");
      label.style.display = "flex";
      label.style.alignItems = "center";
      label.style.gap = "6px";
      label.style.cursor = "pointer";
      
      label.innerHTML = `
        <input type="radio" name="tag-group" value="${tag}" ${tag === currentTag ? "checked" : ""}>
        <span>${tag} <span style="opacity: 0.65;">(${count})</span></span>
      `;
      
      label.querySelector("input").addEventListener("change", (e) => {
        currentTag = e.target.value;
        dispatch();
        render(); // Re-render to update the target counts below
      });
      
      tagsContainer.appendChild(label);
    });
    
    // 2. Render Search Targets
    targetsContainer.innerHTML = "";
    targets.forEach(target => {
      const count = getMatchCount(currentTag, target);
      const label = document.createElement("label");
      label.style.display = "flex";
      label.style.alignItems = "center";
      label.style.gap = "6px";
      label.style.cursor = "pointer";
      
      label.innerHTML = `
        <input type="radio" name="target-group" value="${target}" ${target === currentTarget ? "checked" : ""}>
        <span>${target} <span style="opacity: 0.65;">(${count})</span></span>
      `;
      
      label.querySelector("input").addEventListener("change", (e) => {
        currentTarget = e.target.value;
        dispatch();
        render(); // Re-render to update the entity counts above
      });
      
      targetsContainer.appendChild(label);
    });
  }
  
  // Push values to Observable's dataflow
  function dispatch() {
    container.value = { selectedTag: currentTag, filterTarget: currentTarget };
    container.dispatchEvent(new CustomEvent("input"));
  }
  
  // Initial setup
  render();
  container.value = { selectedTag: currentTag, filterTarget: currentTarget };
  
  return container;
})(this));
```

```js
// BLOCK 3: COMBINE FILTERS
const finalFilteredData = selectedTag === "Semua"
  ? searchResults
  : searchResults.filter(d => {
      // Create a case-insensitive regex pattern from the selected tag
      const regex = new RegExp(selectedTag, "i");
      
      // Safely grab the text, defaulting to empty string if null/undefined
      const titleText = d.ARTICLE_TITLE ?? "";
      const contentText = d.CONTENT ?? ""; 

      // Use regex.test() instead of .includes()
      const titleMatches = regex.test(titleText);
      const contentMatches = regex.test(contentText);

      // Return true based on the selected target
      if (filterTarget === "Judul") return titleMatches;
      if (filterTarget === "Konten") return contentMatches;
      if (filterTarget === "Judul + Konten") return titleMatches || contentMatches;
      
      return titleMatches; // Default fallback
    });
```

```js
// NEW: Slider to adjust the number of rows displayed in the table
const rowCount = view(Inputs.range([5, 50], {
  label: "Jumlah baris per halaman",
  step: 5,
  value: 20
}));
```

```js
// BLOCK 4: RESULTS TABLE WITH RIGHT-SIDE BUTTONS
const resultsTable = Inputs.table(finalFilteredData, {
  columns: ["ARTICLE_TITLE", "MEDIA_OUTLET", "ARTICLE_PUBLISHDATE", "ARTICLE_LINK"],
  header: {
    ARTICLE_TITLE: "Judul",
    MEDIA_OUTLET: "Sumber",
    ARTICLE_PUBLISHDATE: "Tanggal Publikasi",
    ARTICLE_LINK: "Tautan"
  },
  format: {
    ARTICLE_LINK: (link) => htl.html`<a href="${link}" target="_blank" rel="noopener noreferrer">Read Article</a>`
  },
  rows: rowCount,
  layout: "auto"
});

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0;">
    ${dailyTable}
  </div>

<div class="wide-table-container" style="display: flex; align-items: flex-start; gap: 5px; width: 100%;">
  <div style="flex: 1 1 auto; min-width: 0; width: 100%;">
    ${resultsTable}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex: 0 0 auto;">
    ${Inputs.button("Download CSV", { reduce: () => exportCSV(finalFilteredData, "Hasil_Pencarian_Artikel") })}
    ${Inputs.button("Download JSON", { reduce: () => exportJSON(finalFilteredData, "Hasil_Pencarian_Artikel") })}
  </div>
</div>
`);
```

```js
import d3Cloud from "npm:d3-cloud";
```

```js
const downloadedStopwords = await FileAttachment("stopwords.json").json();
```

```js
// BLOCK 5: WORD CLOUD DATA PREPARATION
const topWords = (() => {
  const combinedText = finalFilteredData.map(d => {
    let text = "";
    if (filterTarget === "Judul" || filterTarget === "Judul + Konten") text += " " + (d.ARTICLE_TITLE ?? "");
    if (filterTarget === "Konten" || filterTarget === "Judul + Konten") text += " " + (d.CONTENT ?? "");
    return text;
  }).join(" ");
  
  // Combine your downloaded list + any specific dataset jargon (like "rp" or "pt")
  const stopWords = new Set([...downloadedStopwords, "rp", "pt"]);

  // Extract words (min 3 letters)
  const words = combinedText.toLowerCase().match(/\b[a-z]{3,}\b/g) || [];
  
  // Count word frequencies while filtering out the stopwords
  const wordCounts = d3.rollup(
    words.filter(w => !stopWords.has(w)),
    v => v.length,
    w => w
  );

  // Return the top 100 words formatted for the word cloud
  return Array.from(wordCounts, ([text, size]) => ({ text, size }))
    .sort((a, b) => b.size - a.size)
    .slice(0, 100); 
})();
```

```js
// BLOCK 6: RENDER THE WORD CLOUD WITH EXPORT BUTTON
const wordCloudNode = (() => {
  if (topWords.length === 0) return document.createTextNode("Tidak ada kata untuk ditampilkan.");

  const width = 800;
  const height = 400;

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", [0, 0, width, height])
    .attr("style", "max-width: 100%; height: auto; font-family: 'Archivo', sans-serif; background-color: transparent; padding: 10px; display: block; margin: 20px auto;");

  const g = svg.append("g").attr("transform", `translate(${width / 2},${height / 2})`);
  const sizeScale = d3.scaleSqrt().domain(d3.extent(topWords, d => d.size)).range([12, 70]);
  const colorScale = d3.scaleOrdinal(d3.schemeObservable10);

  const layout = d3Cloud()
    .size([width, height])
    .words(topWords.map(d => Object.create(d))) 
    .padding(3)
    .rotate(0) 
    .font("Archivo")
    .fontSize(d => sizeScale(d.size))
    .on("end", words => {
      g.selectAll("text")
        .data(words)
        .join("text")
        .style("font-size", d => `${d.size}px`)
        .style("font-family", "'Archivo', sans-serif")
        .style("font-weight", "500")
        .style("fill", (d, i) => colorScale(i))
        .attr("text-anchor", "middle")
        .attr("transform", d => `translate(${d.x},${d.y}) rotate(${d.rotate})`)
        .text(d => d.text);
    });

  layout.start();
  return svg.node();
})();

// Replace the bottom of Block 6 with this:
display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0; display: flex; justify-content: center;">
    ${wordCloudNode}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0; margin-top: 100px;">
    ${Inputs.button("Download SVG", { reduce: () => exportSVG(wordCloudNode, "Word_Cloud") })}
  </div>
</div>`);
```

```js
// TABEL 1: FREKUENSI HARIAN DENGAN RIGHT-SIDE BUTTONS
const dailyData = d3.flatRollup(
  finalFilteredData,
  v => v.length,
  d => {
    const date = d.ARTICLE_PUBLISHDATE;
    return date instanceof Date && !isNaN(date) ? d3.timeFormat("%Y-%m-%d")(date) : "Unknown";
  },
  d => d.MEDIA_OUTLET
)
.map(([Tanggal, Sumber, Jumlah]) => ({ Tanggal, Sumber, Jumlah }))
.sort((a, b) => d3.descending(a.Tanggal, b.Tanggal) || d3.descending(a.Jumlah, b.Jumlah));

const dailyTable = Inputs.table(dailyData, {
  header: { Tanggal: "Tanggal", Sumber: "Sumber Media", Jumlah: "Jumlah Artikel" },
  rows: 15,
  maxWidth: "200%",
  layout: "auto"
});

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0;">
    ${dailyTable}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0;">
    ${Inputs.button("Download CSV", { reduce: () => exportCSV(dailyData, "Data_Harian") })}
    ${Inputs.button("Download JSON", { reduce: () => exportJSON(dailyData, "Data_Harian") })}
  </div>
</div>`);
```

```js
// CUSTOM COLOR DOMAIN & RANGE
const MediaColors = {
  "bisnis.com": "#4269d0",
  "bloombergtechnoz.com": "#efb118",
  "cnbcindonesia.com": "#ff725c",
  "cnnindonesia.com": "#6cc5b0",
  "detik.com": "#3ca951",
  "investor.id": "#ff8ab7",
  "kompas.com": "#a463f2",
  "kontan.co.id": "#97bbf5",
  "metrotvnews.com": "#9c6b4e",
  "tvonenews.com": "#9498a0"
};

const mediaDomain = Object.keys(MediaColors);
const mediaRange = Object.values(MediaColors);
```

```js
// CHART 2: MINGGUAN (WEEKLY) DENGAN EXPORT BUTTON
const chartMingguan = Plot.plot({
  title: "Frekuensi Publikasi Mingguan",
  x: { label: "Minggu", tickFormat: "%d %b %Y" },
  y: { label: "Jumlah Artikel" },
  color: { legend: true, label: "Sumber", domain: mediaDomain, range: mediaRange },
  marks: [
    Plot.rectY(finalFilteredData, Plot.binX(
      { y: "count" },
      { x: "ARTICLE_PUBLISHDATE", fill: "MEDIA_OUTLET", interval: "week" }
    )),
    Plot.ruleY([0])
  ]
});

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0; overflow-x: auto;">
    ${chartMingguan}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0; margin-top: 32px;">
    ${Inputs.button("Download SVG", { reduce: () => exportSVG(chartMingguan, "Chart_Mingguan") })}
  </div>
</div>`);
```

```js
// CHART 3: DWIMINGGUAN (BIWEEKLY)
const chartDwimingguan = display(Plot.plot({
  title: "Frekuensi Publikasi Dwimingguan",
  x: { label: "Dwiminggu", tickFormat: "%d %b %Y" },
  y: { label: "Jumlah Artikel" },
  color: { legend: true, label: "Sumber", domain: mediaDomain, range: mediaRange },
  marks: [
    Plot.rectY(finalFilteredData, Plot.binX(
      { y: "count" },
      { x: "ARTICLE_PUBLISHDATE", fill: "MEDIA_OUTLET", interval: d3.timeMonday.every(2) }
    )),
    Plot.ruleY([0])
  ]
}));

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0; overflow-x: auto;">
    ${chartDwimingguan}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0; margin-top: 32px;">
    ${Inputs.button("Download SVG", { reduce: () => exportSVG(chartDwimingguan, "Chart_Dwimingguan") })}
  </div>
</div>`);
```

```js
// CHART 4: BULANAN (MONTHLY)
const chartBulanan = display(Plot.plot({
  title: "Frekuensi Publikasi Bulanan",
  x: { label: "Bulan", tickFormat: "%B %Y" },
  y: { label: "Jumlah Artikel" },
  color: { legend: true, label: "Sumber", domain: mediaDomain, range: mediaRange },
  marks: [
    Plot.rectY(finalFilteredData, Plot.binX(
      { y: "count" },
      { x: "ARTICLE_PUBLISHDATE", fill: "MEDIA_OUTLET", interval: "month" }
    )),
    Plot.ruleY([0])
  ]
}));

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0; overflow-x: auto;">
    ${chartBulanan}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0; margin-top: 32px;">
    ${Inputs.button("Download SVG", { reduce: () => exportSVG(chartBulanan, "Chart_Bulanan") })}
  </div>
</div>`);
```

```js
// CHART 5: KESELURUHAN (MEDIA OUTLET)
const chartKeseluruhan = display(Plot.plot({
  title: "Total Publikasi Keseluruhan per Sumber",
  marginLeft: 150,
  x: { label: "Jumlah Artikel" },
  y: { label: "Sumber Media" },
  color: { domain: mediaDomain, range: mediaRange }, 
  marks: [
    Plot.barX(finalFilteredData, Plot.groupY(
      { x: "count" },
      { y: "MEDIA_OUTLET", fill: "MEDIA_OUTLET", sort: { y: "x", reverse: true } }
    )),
    Plot.ruleX([0])
  ]
}));

display(htl.html`<div style="display: flex; align-items: flex-start; gap: 16px;">
  <div style="flex-grow: 1; min-width: 0; overflow-x: auto;">
    ${chartKeseluruhan}
  </div>
  <div style="display: flex; flex-direction: column; gap: 8px; flex-shrink: 0; margin-top: 32px;">
    ${Inputs.button("Download SVG", { reduce: () => exportSVG(chartKeseluruhan, "Chart_Keseluruhan") })}
  </div>
</div>`);
```
