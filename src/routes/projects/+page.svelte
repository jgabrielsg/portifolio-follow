<script>
  import * as d3 from 'd3';
  import Pie from '$lib/Pie.svelte';
  import projects from "$lib/projects.json";
  import Project from "$lib/Project.svelte";

  let rolledData = d3.rollups(projects, v => v.length, d => d.year);

  let pieData;
  
  $: {
      pieData = rolledData.map(([year, count]) => {
          return { value: count, label: year };
      });
  }

  let query = "";

  // Filtragem por pesquisa 
  $: filteredProjects = projects.filter(project => {
      let values = Object.values(project).join("\n").toLowerCase();
      return values.includes(query.toLowerCase());
  });

  let selectedYearIndex = -1;
  let selectedYear;
  
  $: selectedYear = selectedYearIndex > -1 ? pieData[selectedYearIndex].label : null;

  // Filtra projetos por ano selecionado
  $: filteredByYear = filteredProjects.filter(project => {
      if (selectedYear) {
          return project.year === selectedYear;
      }
      return true;
  });
</script>

<svelte:head>
  <title>Projects</title>
</svelte:head>

<h1>{ projects.length } Projects</h1>

<Pie data={pieData} bind:selectedIndex={selectedYearIndex} />

<input type="search" bind:value={query} aria-label="Search projects" placeholder="🔍 Search projects..." />

<div class="projects">
    {#each filteredByYear as p}
        <Project data={p}/>
    {/each}
</div>

<style>
  h1 {
      text-align: center;
  }

  input {
      display: block;
      width: 100%;
      max-width: 400px;
      margin: 1rem auto;
      padding: 0.5rem;
      font-size: 1rem;
  }

  .projects {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
      gap: 1rem;
      margin-top: 1rem;
  }
  
</style>
