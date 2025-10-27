<script lang="ts">
  import Counter from '$lib/Counter.svelte';
</script>

<main>
  <h1>rutinaro</h1>
  <p>Flexible routine planner PWA scaffolded with Vite + Svelte.</p>
  <Counter />
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
    align-items: center;
  }

  h1 {
    margin: 0;
  }

  p {
    margin: 0;
    color: rgba(255, 255, 255, 0.75);
  }

  @media (prefers-color-scheme: light) {
    p {
      color: #4a5568;
    }
  }
</style>
