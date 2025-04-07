<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";
  // stores.js
import { writable } from 'svelte/store';

  let chartContainer2010;
  let chartContainer2020;

  // Income buckets for stacking
  const BUCKETS = ["Low", "Medium", "High"];
  const selectedBin = writable(null);

  onMount(async () => {
    // ---------------------------
    // 1) LOAD & PREP 2010 DATA
    // ---------------------------
    const data_2010 = await d3.csv("/aggregated2010.csv");
    // Convert to numeric + classify by MHI => income_bucket
    data_2010.forEach(d => {
      d.flip_ind_sum = +d.flip_ind_sum; // ensure numeric
      const mhiVal = +d.mhi;
      if (mhiVal < 60000) {
        d.income_bucket = "Low";
      } else if (mhiVal <= 100000) {
        d.income_bucket = "Medium";
      } else {
        d.income_bucket = "High";
      }
    });

    // Create stacked histogram
    createStackedHistogram({
      container: chartContainer2010,
      rawData: data_2010,
      title: "2010 Data (Stacked Histogram)"
    });

    // ---------------------------
    // 2) LOAD & PREP 2020 DATA
    // ---------------------------
    const data_2020 = await d3.csv("/aggregated2020.csv");
    // Convert to numeric + classify
    data_2020.forEach(d => {
      d.flip_ind_sum = +d.flip_ind_sum;
      const mhiVal = +d.mhi;
      if (mhiVal < 60000) {
        d.income_bucket = "Low";
      } else if (mhiVal <= 100000) {
        d.income_bucket = "Medium";
      } else {
        d.income_bucket = "High";
      }
    });

    // Create stacked histogram
    createStackedHistogram({
      container: chartContainer2020,
      rawData: data_2020,
      title: "2020 Data (Stacked Histogram)"
    });
  });

  /**
   * createStackedHistogram
   * ----------------------
   * Expects:
   *   rawData: array of rows { flip_ind_sum, income_bucket, ...}
   *   container: a DOM element where we append <svg>
   *   title: optional chart title
   */
  function createStackedHistogram({ container, rawData, title }) {
    // Clear container
    if (container) container.innerHTML = "";

    const margin = { top: 30, right: 30, bottom: 40, left: 60 };
    const width = 600;
    const height = 400;

    // 1) Create SVG
    const svg = d3
      .select(container)
      .append("svg")
      .attr("width", width)
      .attr("height", height);

    // 2) Determine xMin & xMax from rawData
    const flipVals = rawData.map(d => d.flip_ind_sum);
    const xMin = d3.min(flipVals);
    const xMax = d3.max(flipVals);

    // 3) Build a bin generator with fixed step = 5
    //    If xMin > 0, you could do d3.range(xMin, xMax+5, 5)
    //    If there's negative or large data, adjust as needed.
    const binGen = d3
      .bin()
      .value(d => d.flip_ind_sum)
      .domain([xMin, xMax])
      .thresholds(d3.range(0, xMax + 5, 5)); // [0, 5, 10, 15, ...] up to xMax

    // 4) Generate bins (each bin is an array of rows)
    //    bin.x0 = lower edge, bin.x1 = upper edge
    const bins = binGen(rawData);

    // 5) For each bin, group by income_bucket => count
    //    We'll pivot that into { x0, x1, Low, Medium, High }
    const binnedData = bins.map(bin => {
      const roll = d3.rollups(
        bin,
        v => v.length,
        d => d.income_bucket
      );
      // roll is e.g. [ ["Low", 3], ["Medium", 5], ["High", 1] ]

      const mapB = new Map(roll);
      return {
        x0: bin.x0,
        x1: bin.x1,
        Low: mapB.get("Low") || 0,
        Medium: mapB.get("Medium") || 0,
        High: mapB.get("High") || 0
      };
    });

    // 6) Use d3.stack() to create stacked layers for [Low, Medium, High]
    const stack = d3.stack().keys(BUCKETS);
    const series = stack(binnedData);
    // Example of series shape:
    // series[0] => array of [y0, y1, dataObj] for 'Low'
    // series[1] => array of [y0, y1, dataObj] for 'Medium'
    // series[2] => array of [y0, y1, dataObj] for 'High'

    // 7) Y domain (max stacked height)
    const yMax = d3.max(series[series.length - 1], d => d[1]);
    // Build y scale
    const y = d3
      .scaleLinear()
      .domain([0, yMax])
      .range([height - margin.bottom, margin.top])
      .nice();

    // 8) X scale is linear from xMin to xMax
    const x = d3
      .scaleLinear()
      .domain([xMin, xMax])
      .range([margin.left, width - margin.right]);

    // Axes
    svg
      .append("g")
      .attr("transform", `translate(0, ${height - margin.bottom})`)
      .call(d3.axisBottom(x));

    svg
      .append("g")
      .attr("transform", `translate(${margin.left}, 0)`)
      .call(d3.axisLeft(y));

    // 9) Color scale
    const color = d3
      .scaleOrdinal()
      .domain(BUCKETS)
      .range(["steelblue", "orange", "green"]);

    // 10) Tooltip
    const tooltip = d3
      .select(container)
      .append("div")
      .style("position", "absolute")
      .style("visibility", "hidden")
      .style("background", "#f0f0f0")
      .style("padding", "4px")
      .style("border", "1px solid #999")
      .style("border-radius", "4px")
      .style("font-size", "12px");

    // 11) Draw stacked rectangles
    const rects = svg
      .selectAll("g.layer")
      .data(series) // 3 layers
      .join("g")
      .attr("class", "layer")
      .attr("fill", d => color(d.key))
      .selectAll("rect")
      .data(d => d) // each layer's bins
      .join("rect")
      .attr("x", d => x(d.data.x0))
      .attr("width", d => x(d.data.x1) - x(d.data.x0)) // bin width
      .attr("y", d => y(d[1]))
      .attr("height", d => y(d[0]) - y(d[1]))
      .on("mouseover", function (event, d) {
        const binRange = `[${d.data.x0}, ${d.data.x1})`;
        const html = `
          <strong>Flip_ind_sum range:</strong> ${binRange}<br/>
          Low: ${d.data.Low}<br/>
          Medium: ${d.data.Medium}<br/>
          High: ${d.data.High}<br/>
        `;
        tooltip.html(html).style("visibility", "visible");
        // Also highlight this bin in the other chart
        // We'll store the [x0, x1) in `selectedBin`.
        selectedBin.set({ x0: d.data.x0, x1: d.data.x1 });
      })
      .on("mousemove", function (event) {
        tooltip
          .style("top", event.pageY + 10 + "px")
          .style("left", event.pageX + 10 + "px");
      })
      .on("mouseout", function () {
        tooltip.style("visibility", "hidden");
        selectedBin.set(null);
      });

    // 12) Title
    svg
      .append("text")
      .attr("x", width / 2)
      .attr("y", margin.top / 2)
      .attr("text-anchor", "middle")
      .style("font-weight", "bold")
      .text(title || "Stacked Histogram (Bin = 5)");

    // X-axis label
    svg
      .append("text")
      .attr("x", width / 2)
      .attr("y", height - 5)
      .attr("text-anchor", "middle")
      .style("font-size", "10px")
      .text("flip_ind_sum (binned)");

    // Y-axis label
    svg
      .append("text")
      .attr("transform", "rotate(-90)")
      .attr("y", margin.left - 50)
      .attr("x", -(height / 2))
      .attr("text-anchor", "middle")
      .style("font-size", "12px")
      .text("Frequency (Count)");

     // SUBSCRIBE to store so we can highlight or un-highlight bins in response
     selectedBin.subscribe(bin => {
      if (bin == null) {
        // If store is null, remove highlight from all rects
        rects.attr("stroke", null).attr("stroke-width", null);
        tooltip.style("visibility", "hidden");
      } else {
        // We have a bin range => highlight matching bin
        rects.each(function (d) {
          if (d.data.x0 === bin.x0 && d.data.x1 === bin.x1) {
            d3.select(this).attr("stroke", "black").attr("stroke-width", 2);
            const binRange = `[${d.data.x0}, ${d.data.x1})`;
            const html = `
              <strong>Flip_ind_sum range:</strong> ${binRange}<br/>
              Low: ${d.data.Low}<br/>
              Medium: ${d.data.Medium}<br/>
              High: ${d.data.High}<br/>
            `;
            tooltip.html(html).style("visibility", "visible");
          } else {
            d3.select(this).attr("stroke", null).attr("stroke-width", null);
          }
        });
      }
    });
  }
</script>

<!-- The container where the chart will go -->


<div style="display: flex; gap: 2rem;">
  <!-- Each container for a chart -->
  <div bind:this={chartContainer2010}></div>
  <div bind:this={chartContainer2020}></div>
</div>



<style>
 
</style>
