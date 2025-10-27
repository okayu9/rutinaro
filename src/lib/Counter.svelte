<script lang="ts">
  export let count = 0;

  const increment = () => {
    count += 1;
  };

  const decrement = () => {
    count -= 1;
  };
</script>

<div class="card">
  <button on:click={increment}>
    count is {count}
  </button>
  <button class="secondary" on:click={decrement}>
    decrement
  </button>
  <p>
    Edit <code>src/lib/Counter.svelte</code> to test HMR
  </p>
</div>

<style>
  .card {
    padding: 2em;
  }

  button {
    margin-right: 0.5rem;
  }

  .secondary {
    background-color: transparent;
    color: inherit;
    border: 1px solid currentColor;
  }
</style>
