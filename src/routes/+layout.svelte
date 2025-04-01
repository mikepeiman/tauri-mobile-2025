<script>
  /** @type {{ data: import('./$types').LayoutData, children: import('svelte').Snippet }} */
  let { data, children } = $props();
  import "../app.css";
</script>

{@render children()}
