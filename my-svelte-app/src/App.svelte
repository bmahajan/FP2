<script>
  import svelteLogo from './assets/svelte.svg'
  import viteLogo from '/vite.svg'
  
  import {onMount} from "svelte";
  import * as d3 from "d3";

  let data = [];
  let chartContainers;

  const BUCKETS = ["Low", "Medium", "High"];
  onMount(async () => {
    // load CSV
    const data_2010 = await d3.csv("./public/aggregated2010.csv");

    data_2010.forEach(d => {
      const mhiVal = +d.mhi;
      if (mhiVal < 60000) {
        d.income_bucket = 'Low';
      } else if (mhiVal <= 100000) {
        d.income_bucket = 'Medium';
      } else {
        d.income_bucket = 'High';
      }
    });

    // 3) GROUP by Flip_ind_sum & income_bucket to get FREQUENCY (count)
    // d3.rollups returns nested arrays, so we do:
    //   - First key: d => d.Flip_ind_sum
    //   - Second key: d => d.income_bucket
    //   - Reducer: groupRows => groupRows.length
    const grouped_data = d3.rollups(
      data_2010,
      groupRows => groupRows.length,    // aggregator => the count
      d => d.flip_ind_sum,             // group by Flip_ind_sum
      d => d.income_bucket             // then group by income_bucket
    );

    data = grouped_data.map(([flipValue, arr]) => {
       // arr is something like [ ['Low', count], ['Medium', count], ... ]
       const mapBuckets = new Map(arr);
      return {
        Flip_ind_sum: flipValue,
        Low: mapBuckets.get('Low') || 0,
        Medium: mapBuckets.get('Medium') || 0,
        High: mapBuckets.get('High') || 0
      };
    });
    
    
    createStackedBarChart();

  });

  function createStackedBarChart() {
    // Clear any existing content (helpful on hot reload)
    if (chartContainers) chartContainers.innerHTML = '';

    const margin = { top: 30, right: 30, bottom: 40, left: 60 };
    const width = 600;
    const height = 400;

    // 1) Create SVG
    const svg = d3
      .select(chartContainers)
      .append('svg')
      .attr('width', width)
      .attr('height', height);

    // 2) Setup scales
    // X scale: band scale by Flip_ind_sum
    const x = d3
      .scaleBand()
      .domain(data.map(d => d.Flip_ind_sum))
      .range([margin.left, width - margin.right])
      .padding(0.2);

    // Create a stack generator
    const stack = d3.stack().keys(BUCKETS);

    // Build the stacked series
    const series = stack(data);

    // The highest stacked value for y-domain
    const yMax = d3.max(series[series.length - 1], d => d[1]);

    // Y scale
    const y = d3
      .scaleLinear()
      .domain([0, yMax])
      .range([height - margin.bottom, margin.top])
      .nice();

    // Color scale for the 3 buckets
    const color = d3
      .scaleOrdinal()
      .domain(BUCKETS)
      .range(['steelblue', 'orange', 'green']); // choose your own palette

    // 3) Axes
    svg
      .append('g')
      .attr('transform', `translate(0, ${height - margin.bottom})`)
      .call(d3.axisBottom(x));

    svg
      .append('g')
      .attr('transform', `translate(${margin.left}, 0)`)
      .call(d3.axisLeft(y));

    // 4) Tooltip (basic HTML div)
    const tooltip = d3
      .select(chartContainers)
      .append('div')
      .style('position', 'absolute')
      .style('visibility', 'hidden')
      .style('background', '#f0f0f0')
      .style('padding', '4px')
      .style('border', '1px solid #999')
      .style('border-radius', '4px')
      .style('font-size', '12px');

    // 5) Draw stacked bars
    svg
      .selectAll('g.layer')
      .data(series)
      .join('g')
      .attr('class', 'layer')
      .attr('fill', d => color(d.key)) // color by bucket
      .selectAll('rect')
      .data(d => d)
      .join('rect')
      .attr('x', d => x(d.data.Flip_ind_sum))
      .attr('y', d => y(d[1]))    // top of the segment
      .attr('height', d => y(d[0]) - y(d[1]))
      .attr('width', x.bandwidth())
      .on('mouseover', function (event, d) {
        // Show tooltip with row details
        const flip = d.data.Flip_ind_sum;
        // d.data = {Flip_ind_sum: '5', Low: 2, Medium: 1, High: 1}
        let html = `<strong>Flip_ind_sum: ${flip}</strong><br/>`;
        html += `Low: ${d.data.Low}<br/>`;
        html += `Medium: ${d.data.Medium}<br/>`;
        html += `High: ${d.data.High}`;
        tooltip.html(html).style('visibility', 'visible');
      })
      .on('mousemove', function (event) {
        // position tooltip near mouse
        tooltip
          .style('top', event.pageY + 10 + 'px')
          .style('left', event.pageX + 10 + 'px');
      })
      .on('mouseout', function () {
        tooltip.style('visibility', 'hidden');
      });

    // Optional chart title
    svg
      .append('text')
      .attr('x', width / 2)
      .attr('y', margin.top / 2)
      .attr('text-anchor', 'middle')
      .style('font-weight', 'bold')
      .text('Stacked Bar by Income Bucket');

    // X-axis label
    svg
      .append('text')
      .attr('x', width / 2)
      .attr('y', height - 5)
      .attr('text-anchor', 'middle')
      .style('font-size', '10px')
      .text('Flip_ind_sum');

    // Y-axis label
    svg
      .append('text')
      .attr('transform', 'rotate(-90)')
      .attr('y', margin.left - 50)
      .attr('x', -(height / 2))
      .attr('text-anchor', 'middle')
      .style('font-size', '12px')
      .text('Frequency (count)');

      svg
    .append("g")
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .selectAll("text")        // select all x-axis text
    .attr("text-anchor", "end")
    .attr("transform", "rotate(-45)") // rotate 45° or 90°
    .attr("dx", "-0.5em")    // adjust x-position
    .attr("dy", "-0.1em");   // adjust y-position
  }



</script>

<!-- The container where the chart will go -->
<div bind:this={chartContainers} style="width: 100%; overflow-x: auto;">

</div>

<!-- Display a loading message if data is still empty -->
{#if !data || data.length === 0}
<p>Loading CSV data…</p>
{/if}


<style>
  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: filter 300ms;
  }
  .logo:hover {
    filter: drop-shadow(0 0 2em #646cffaa);
  }
  .logo.svelte:hover {
    filter: drop-shadow(0 0 2em #ff3e00aa);
  }
  .read-the-docs {
    color: #888;
  }
</style>
