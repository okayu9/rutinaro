<script lang="ts">
  let count = 0;
</script>

<main>
  <h1>Welcome to Rutinaro</h1>
  <p>Start building your flexible routine manager.</p>
  <button on:click={() => count += 1}>
    count is {count}
  </button>
</main>

<style>
  main {
    text-align: center;
    padding: 1rem;
  }

  button {
    margin-top: 1rem;
  }
</style>
