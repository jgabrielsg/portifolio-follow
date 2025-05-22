<script>
    import { computePosition, autoPlacement, offset } from '@floating-ui/dom';
    import { onMount } from "svelte";
    
    export let commits = [];
    export let xScale;
    export let yScale;
    export let rScale;
    export let hoveredIndex = -1;
    export let hoveredCommit = {};
    export let clickedCommits = [];
    export let tooltipPosition = {x: 0, y: 0};
    export let commitTooltip;

    // Criação e manipulação dos eixos com D3
    let xAxis, yAxis, yAxisGridlines;

    // Função de interação do ponto
    async function dotInteraction (index, evt) {
        let hoveredDot = evt.target;
        if (evt.type === "mouseenter") {
            hoveredIndex = index;
            tooltipPosition = {x: evt.x, y: evt.y};
            tooltipPosition = await computePosition(hoveredDot, commitTooltip, {
                strategy: "fixed", // porque usamos position: fixed
                middleware: [
                    offset(5), // espaçamento entre o tooltip e o ponto
                    autoPlacement() // ver https://floating-ui.com/docs/autoplacement
                ],
            });
        }
        else if (evt.type === "mouseleave") {
            hoveredIndex = -1
        }
        else if (evt.type === "click") {
            let commit = commits[index]
            if (!clickedCommits.includes(commit)) {
                clickedCommits = [...clickedCommits, commit];
            }
            else {
                clickedCommits = clickedCommits.filter(c => c !== commit);
            }
        }
    }

    // Lógica para os eixos
    onMount(() => {
        d3.select(xAxis).call(d3.axisBottom(xScale));
        d3.select(yAxis).call(d3.axisLeft(yScale).tickFormat(d => String(d % 24).padStart(2, "0") + ":00"));
        d3.select(yAxisGridlines).call(
            d3.axisLeft(yScale)
            .tickFormat("")
            .tickSize(-1000) 
        );
    });
</script>

<svg viewBox="0 0 1000 600">
    <g transform="translate(0, 600)" bind:this={xAxis} />
    <g transform="translate(20, 0)" bind:this={yAxis} />
    <g class="gridlines" transform="translate(20, 0)" bind:this={yAxisGridlines} />
    <g class="dots">
        {#each commits as commit, index }
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
            />
        {/each}
    </g>
</svg>

<dl class="info tooltip" hidden={hoveredIndex === -1} style="top: {tooltipPosition.y}px; left: {tooltipPosition.x}px" bind:this={commitTooltip}>
    <dt>Commit</dt>
    <dd><a href="{ hoveredCommit.url }" target="_blank">{ hoveredCommit.id }</a></dd>

    <dt>Date</dt>
    <dd>{ hoveredCommit.datetime?.toLocaleString("en", {dateStyle: "full"}) }</dd>

    <dt>Author</dt>
    <dd>{ hoveredCommit.author }</dd>

    <dt>Time</dt>
    <dd>{ hoveredCommit.time }</dd>
</dl>

<style>
    svg {
        overflow: visible;
    }

    .gridlines {
        stroke-opacity: .2;
    }

    circle {
        transition: 200ms;
        transform-origin: center;
        transform-box: fill-box;
        &:hover {
            transform: scale(1.5);
        }
    }

    .selected {
        fill: var(--color-accent);
    }

    .info {
        display: grid;
        margin: 0;
        grid-template-columns: 2;
        background-color: oklch(100% 0% 0 / 80%);
        box-shadow: 1px 1px 3px 3px gray;
        border-radius: 5px;
        backdrop-filter: blur(10px);
        padding: 10px;

        transition-duration: 500ms;
        transition-property: opacity, visibility;

        &[hidden]:not(:hover, :focus-within) {
            opacity: 0;
            visibility: hidden;
        }
    }
    
</style>
