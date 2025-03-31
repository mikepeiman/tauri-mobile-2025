<script>
  import { invoke } from "@tauri-apps/api/core";
  import { onMount } from "svelte";

  import OpenAI from "openai";
  const client = new OpenAI({
    apiKey: import.meta.env.VITE_OPENAI_API_KEY,
    dangerouslyAllowBrowser: true,
  });
  let name = $state("");
  let greetMsg = $state("");
  $effect(async () => {
    console.log(`env key: ${import.meta.env.VITE_OPENAI_API_KEY}`);
  });

  async function fetchEnvVar(key) {
    const value = await invoke("get_env_var", { key });
    console.log(value);
    return value;
  }

  // Example usage

  onMount(async () => {
    console.log(`env key: ${import.meta.env.VITE_OPENAI_API_KEY}`);
    const response = await client.responses.create({
      model: "gpt-4o",
      input: name || "Write a one-sentence bedtime story about a unicorn.",
    });
    console.log(response);
    console.log(response.output_text);

    fetchEnvVar("VITE_OPENAI_API_KEY").then((key) => {
      console.log("API KEY:", key);
    });
  });

  async function promptLLM(prompt) {
    const response = await client.responses.create({
      model: "gpt-4o",
      input: prompt,
    });
    console.log(response);
    console.log(response.output_text);
  }

  async function greet(event) {
    console.log(`Name: ${name}, Event: `, event);
    promptLLM(name);
    event.preventDefault();
    // Learn more about Tauri commands at https://tauri.app/develop/calling-rust/
    greetMsg = await invoke("greet", { name });
    console.log(greetMsg);
  }
</script>

<main class="container">
  <div class="row">
    <h1 class="header">Metabrain</h1>
  </div>
  <h2>Welcome to your operating system for life.</h2>
  <h3>Track, plan and execute like a fucking champion.</h3>

  <form class="row" onsubmit={greet}>
    <input id="greet-input" placeholder="Enter a name..." bind:value={name} />
    <button type="submit">Greet</button>
  </form>

  {#await greetMsg}
    <p>Loading...</p>
  {:then value}
    <p>{value}</p>
  {:catch error}
    <p style="color: red">{error}</p>
  {/await}
</main>

<style>
  .logo.vite:hover {
    filter: drop-shadow(0 0 2em #747bff);
  }

  .logo.svelte-kit:hover {
    filter: drop-shadow(0 0 2em #ff3e00);
  }

  :root {
    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;

    color: #0f0f0f;
    background-color: #f6f6f6;

    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
  }

  .container {
    margin: 0;
    padding-top: 10vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
  }

  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: 0.75s;
  }

  .logo.tauri:hover {
    filter: drop-shadow(0 0 2em #24c8db);
  }

  .row {
    display: flex;
    justify-content: center;
  }

  a {
    font-weight: 500;
    color: #646cff;
    text-decoration: inherit;
  }

  a:hover {
    color: #535bf2;
  }

  h1 {
    text-align: center;
  }

  .header {
    font-size: 8rem;
    font-weight: 800;
    color: #0f0f0f;
    text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
    margin-bottom: 5rem;
    margin-top: 5rem;
    font-family: "montserrat", "Roboto", sans-serif;
  }

  input,
  button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #0f0f0f;
    background-color: #ffffff;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
  }

  button {
    cursor: pointer;
  }

  button:hover {
    border-color: #396cd8;
  }
  button:active {
    border-color: #396cd8;
    background-color: #e8e8e8;
  }

  input,
  button {
    outline: none;
  }

  #greet-input {
    margin-right: 5px;
  }

  @media (prefers-color-scheme: dark) {
    :root {
      color: #f6f6f6;
      background-color: #2f2f2f;
    }

    a:hover {
      color: #24c8db;
    }

    input,
    button {
      color: #ffffff;
      background-color: #0f0f0f98;
    }
    button:active {
      background-color: #0f0f0f69;
    }
  }
</style>
