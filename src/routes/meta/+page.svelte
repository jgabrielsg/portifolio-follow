<script>
    import * as d3 from "d3";
    import StackedBar from '$lib/StackedBar.svelte';
    import FileLines from '$lib/FileLines.svelte';
    import Scrolly from "svelte-scrolly";

    const colorScale = d3.scaleOrdinal(d3.schemeTableau10);

    import {
        computePosition,
        autoPlacement,
        offset,
    } from '@floating-ui/dom';

    import { onMount, onDestroy } from "svelte";

    $: minDate = d3.min(commits.map(d => d.date));
    $: maxDate = d3.max(commits.map(d => d.date));
    $: maxDatePlusOne = new Date(maxDate);
    $: maxDatePlusOne.setDate(maxDatePlusOne.getDate() + 1);

    let commitProgress = 100;
    $: timeScale = d3.scaleTime().domain([minDate,maxDate]).range([0,100]);
    $: commitMaxTime = timeScale.invert(commitProgress);

    $: filteredCommits = commits
  .filter(commit => commit.datetime <= commitMaxTime)
  .sort((a, b) => a.datetime - b.datetime);

    $: filteredMinDate = d3.min(filteredCommits.map(d => d.date));
    $: filteredMaxDate = d3.max(filteredCommits.map(d => d.date));
    $: filteredMaxDatePlusOne = new Date(filteredMaxDate);
    $: filteredMaxDatePlusOne.setDate(filteredMaxDatePlusOne.getDate() + 1);

    $: xScale = d3.scaleTime()
                .domain([filteredMinDate, filteredMaxDatePlusOne])
                .range([usableArea.left, usableArea.right])
                .nice();

    $: yScale = d3.scaleLinear()
                .domain([24, 0])
                .range([usableArea.top, usableArea.bottom]);

    let data = [];
    let commits = [];


    onMount(async () => {
        data = await d3.csv("./loc.csv", row => ({
            ...row,
            line: Number(row.line), // or just +row.line
            depth: Number(row.depth),
            length: Number(row.length),
            date: new Date(row.date + "T00:00" + row.timezone),
            datetime: new Date(row.datetime)
        }));
        
        commits = d3.groups(data, d => d.commit).map(([commit, lines]) => {
        let first = lines[0];
        let {author, date, time, timezone, datetime} = first;
        let ret = {
            id: commit,
            url: "https://github.com/jgabrielsg/portifolio-follow/commit/" + commit,
            author, date, time, timezone, datetime,
            hourFrac: datetime.getHours() + datetime.getMinutes() / 60,
            totalLines: lines.length
        };
        commits = d3.sort(commits, d => -d.totalLines);

        // Like ret.lines = lines
        // but non-enumerable so it doesn’t show up in JSON.stringify
        Object.defineProperty(ret, "lines", {
            value: lines,
            configurable: true,
            writable: true,
            enumerable: false,
        });

        return ret;
    });
    });

    let width = 800, height = 600;
    let margin = {top: 10, right: 10, bottom: 30, left: 20};

    let usableArea = {
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left
    };

    usableArea.width = usableArea.right - usableArea.left;
    usableArea.height = usableArea.bottom - usableArea.top;

    let xAxis, yAxis;
    let yAxisGridlines;
    

    $: {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d => String(d % 24).padStart(2, "0") + ":00"));
        d3.select(yAxisGridlines).call(
        d3.axisLeft(yScale)
          .tickFormat("")
          .tickSize(-usableArea.width)
    );
    }

    let hoveredIndex = -1;
    $: hoveredCommit = filteredCommits[hoveredIndex] ?? hoveredCommit ?? {};

    //let cursor = {x: 0, y: 0};

    let commitTooltip
    let tooltipPosition = {x: 0, y: 0};
    let clickedCommits = [];

    async function dotInteraction (index, evt) {
        let hoveredDot = evt.target;
        if (evt.type === "mouseenter") {
            hoveredIndex = index;
            tooltipPosition = {x: evt.x, y: evt.y};
            tooltipPosition = await computePosition(hoveredDot, commitTooltip, {
                strategy: "fixed", // because we use position: fixed
                middleware: [
                    offset(5), // spacing from tooltip to dot
                    autoPlacement() // see https://floating-ui.com/docs/autoplacement
                ],
            });        }
        else if (evt.type === "mouseleave") {
            hoveredIndex = -1
        }

        else if (evt.type === "click") {
            let commit = filteredCommits[index]
            if (!clickedCommits.includes(commit)) {
                // Add the commit to the clickedCommits array
                clickedCommits = [...clickedCommits, commit];
            }
            else {
                    // Remove the commit from the array
                    clickedCommits = clickedCommits.filter(c => c !== commit);
            }
        }
    }

    $: rScale = d3.scaleSqrt()
                .domain(d3.extent(commits.map(d=>d.totalLines)))
                .range([2, 30]);
    
    $: filteredLines = data.filter(d => d.datetime <= commitMaxTime)
    
    $: allTypes = Array.from(new Set(filteredLines.map(d => d.type)));
    $: selectedLines = (filteredCommits.length > 0 ? filteredCommits : commits).flatMap(d => d.lines);

    $: selectedCounts = d3.rollup(
        selectedLines,
        v => v.length,
        d => d.type
    );

    $: languageBreakdown = allTypes.map(type => [type, selectedCounts.get(type) || 0]);

    // 'languageBreakdown' no formato [ [label, count], ... ]
    $: dataObject = Object.fromEntries(languageBreakdown);

    $: keys = Object.keys(dataObject);

    $: dataForStack = [dataObject]; // Array com 1 objeto

    $: stackedData = d3.stack().keys(keys)(dataForStack);

    $: total = d3.max(stackedData, series => d3.max(series, d => d[1])) || 1;

    $: xScaleStack = d3.scaleLinear()
                    .domain([0, total])
                    .range([0, width - margin.left - margin.right]);

    $: numCommits = filteredCommits.length || 1;
    $: step = 100 / numCommits;

    // Para mostrar até qual commit o texto está "ativo"
    $: currentCommitIndex = Math.min(Math.floor(commitProgress / step), numCommits - 1);

    const svgWidth = 0.7 * width;

    onMount(() => {
    document.body.classList.add('meta-page');
    });

    onDestroy(() => {
    document.body.classList.remove('meta-page');
    });
</script>

<svelte:head>
  <title>Meta</title>
</svelte:head>

<h1>Meta</h1>
<p>Total lines of code: {data.length}</p>

<Scrolly bind:progress={commitProgress} style="max-width: 600px; margin: auto;">
    {#if filteredCommits.length === 0}
      <p>Loading commits...</p>
    {:else}
      {#each filteredCommits as commit, index (commit.id)}
        <p class="narrative" style="opacity: {commitProgress >= index * step ? 1 : 0.2}; transition: opacity 0.3s;">
          On <strong>{commit.datetime.toLocaleString("en", { dateStyle: "full", timeStyle: "short" })}</strong>,  
          {index === 0
            ? `I made the first commit, starting the project. You can see it <a href="${commit.url}" target="_blank">here</a>.`
            : `I added another commit. Check it in this link: <a href="${commit.url}" target="_blank">here</a>. It was a very important commit to me and definitely helped me become a better programming genius, as I currently am. I praise the existence of this commit and all its reasoning, he is full of love.`}
          This commit changed <strong>{commit.totalLines}</strong> lines in <strong>{d3.groups(commit.lines, d => d.file).length}</strong> files.
        </p>
      {/each}
      {/if}
  
    <svelte:fragment slot="viz">
      <!-- Scatterplot com commits -->
      <svg viewBox="0 0 {width} {height}" style="display: block; margin: 20px auto;">
        <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
        <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
        <g class="gridlines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridlines} />
        <g class="dots">
          {#each filteredCommits as commit, index}
            <circle
              on:mouseenter={evt => dotInteraction(index, evt)}
              on:mouseleave={evt => dotInteraction(index, evt)}
              on:click={evt => dotInteraction(index, evt)}
              class:selected={clickedCommits.includes(commit)}
              cx={xScale(commit.datetime)}
              cy={yScale(commit.hourFrac)}
              r={rScale(commit.totalLines)}
              fill="steelblue"
              fill-opacity="0.5"
              style="cursor: pointer;"
            />
          {/each}
        </g>
      </svg>
  
      <!-- Stacked Bar mostrando tipos -->
      <StackedBar
        {stackedData}
        {keys}
        {width}
        barHeight={40}
        xScale={xScaleStack}
        colorScale={colorScale}
      />
      
    </svelte:fragment>
</Scrolly>

<FileLines lines={filteredLines} width={width} colorScale={colorScale} />

<style>
    .narrative {
    font-size: 1rem;
    line-height: 1.5;
    margin-bottom: 1.5rem;
  }
  .narrative a {
    color: #007acc;
    text-decoration: none;
  }
  .narrative a:hover {
    text-decoration: underline;
  }
  .selected {
    stroke: black;
    stroke-width: 2px;
  }
  
    svg {
        overflow: visible;
    }

    .gridlines {
        stroke-opacity: .2;
    }

    .info{
        display: grid;
        margin:0;
        grid-template-columns: 2;
        background-color: oklch(100% 0% 0 / 80%);
        box-shadow: 1px 1px 3px 3px gray;
        border-radius: 5px;
        backdrop-filter: blur(10px);
        padding:10px;

        transition-duration: 500ms;
        transition-property: opacity, visibility;

        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }

    .selected {
        fill: var(--color-accent);
    }

    .info dt{
        grid-column:1;
        grid-row:auto;
    }

    .info dd{
        grid-column:2;
        grid-row:auto;
        font-weight: 400;
    }

    .tooltip{
        position: fixed;
        top: 1em;
        left: 1em;
    }

    circle {
    transition: 200ms;
    transform-origin: center;
    transform-box: fill-box;
        &:hover {
            transform: scale(1.5);
        }
        
        @starting-style {
            r: 0;}
            transition: all 200ms,
                r calc(var(--r) * 100ms);
    }

    .slider-container{
	display:grid;
    }
    .slider{
        display: flex;
    }
    #slider{
        flex:1;
    }
    .time-label{
        font-size: 0.75em;
        text-align: right;
    }

    :global(body.meta-page) {
        width: 70%;
        max-width: none;
        margin-inline: auto; /* step 1 */
        padding: 1cm; /* step 1 */
    }

</style>

