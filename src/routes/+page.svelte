<script>
  import projects from "$lib/projects.json";
  import Project from "$lib/Project.svelte";

  // let profileData = fetch("https://api.github.com/users/juan1t0");

  import { onMount } from "svelte";

  let githubData = null;
  let loading = true;
  let error = null;

  onMount(async () => {
      try {
          const response = await fetch("https://api.github.com/users/jgabrielsg");
          githubData = await response.json();
      } catch (err) {
          error = err;
      }
      loading = false;
  });
</script>

<svelte:head>
  <title>João Gabriel: Personal site and portfolio</title>
</svelte:head>
<h1>João Gabriel</h1>
   
<img src="./images/mid.jpg" alt="Eu" width="500px">

<p>I am João Gabriel. I live in Rio. I am alive.
</p>
{#if loading}
    <p>Loading...</p>
{:else if error}
    <p class="error">Something went wrong: {error.message}</p>
{:else}
    <section>
        <h2>My GitHub Stats</h2>
        <dl>
            <dt>Followers</dt>
            <dd>{githubData.followers}</dd>
            <dt>Following</dt>
            <dd>{githubData.following}</dd>
            <dt>Public Repositories</dt>
            <dd>{githubData.public_repos}</dd>
        </dl>
    </section>
{/if}
<h2>
  Latest Projects
</h2>
<div class="projects">
{#each projects.slice(0, 3) as p}
  <Project data={p} hLevel="3"/>
{/each}
</div>

