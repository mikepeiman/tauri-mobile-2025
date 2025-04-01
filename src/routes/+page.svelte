<script>
  import { invoke } from "@tauri-apps/api/core";
  import { onMount } from "svelte";

  import OpenAI from "openai";

  let browser = false;

  const client = new OpenAI({
    apiKey: import.meta.env.VITE_OPENAI_API_KEY,
    dangerouslyAllowBrowser: true,
  });
  let promptInput = $state("");
  let numImages = $state(2);
  let greetMsg = $state("");
  let contentType = $state("image");
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
    browser = true;
    console.log(`env key: ${import.meta.env.VITE_OPENAI_API_KEY}`);
    const response = await client.responses.create({
      model: "gpt-4o",
      input:
        promptInput ||
        "Hit me with some random wisdom quote. Make it profound.",
    });
    console.log(response);
    console.log(response.output_text);

    // fetchEnvVar("VITE_OPENAI_API_KEY").then((key) => {
    //   console.log("API KEY:", key);
    // });
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
    console.log(`PromptInput: ${promptInput}, Event: `, event);
    promptLLM(promptInput);
    event.preventDefault();
    // Learn more about Tauri commands at https://tauri.app/develop/calling-rust/
    if (browser) {
      greetMsg = await invoke("greet", { promptInput });
      console.log(greetMsg);
      imageGen(promptInput);
    }
  }

  async function imageGen(msg) {
    const image = await client.images.generate({
      model: "dall-e-3",
      prompt: msg,
    });

    console.log(image, image.data);
  }
</script>

<main class="container min-w-full">
  <div class="row">
    <h1 class="header">Metabrain</h1>
  </div>
  <h2>Welcome to your operating system for life.</h2>
  <h3>Track, plan and execute like a fucking champion.</h3>
  <!-- Parent element with gradient background and padding -->
  <div
    class="p-4 bg-gradient-to-r from-cyan-400 via-purple-200 to-green-300 w-full my-8">
    <!-- Child element with solid background -->
    <div class="w-full bg-white p-6 rounded-md">
      <div class="flex w-full justify-between">
        <h2 class="flex items-center w-[20ch]">
          Generate: <span class="p-2 bg-cyan-200 rounded-lg mx-2">
            {contentType}</span>
        </h2>
        <div class="flex justify-center items-center w-full">
          <div class="flex items-center space-x-4">
            <div>
              <input
                type="radio"
                id="text-option"
                name="content-type"
                value="text"
                class="sr-only peer"
                bind:group={contentType} />
              <label
                for="text-option"
                class="
                  px-4
                  py-2
                  bg-gray-200
                  rounded-md
                  text-gray-700
                  cursor-pointer
                  peer-checked:bg-blue-500
                  peer-checked:text-white
                  peer-checked:shadow-md
                  transition-colors
                ">
                Text
              </label>
            </div>

            <div>
              <input
                type="radio"
                id="image-option"
                name="content-type"
                value="image"
                class="sr-only peer"
                bind:group={contentType} />
              <label
                for="image-option"
                class="
                  px-4
                  py-2
                  bg-gray-200
                  rounded-md
                  text-gray-700
                  cursor-pointer
                  peer-checked:bg-blue-500
                  peer-checked:text-white
                  peer-checked:shadow-md
                  transition-colors
                ">
                Image
              </label>
            </div>
          </div>
        </div>
      </div>
      <h2 class="text-lg font-bold text-gray-900 bg-cyan-200 p-6 my-12">
        Image Generation Options
      </h2>
      <div class="grid h-full">
        <div class="grid [grid-template-columns:_1fr_4fr]">
          <div class="flex justify-between">
            <h3 class="mr-4">Number of images:</h3>
            <h3
              class="bg-cyan-300 w-[1.75rem] h-[1.75rem] flex items-center justify-center">
              {numImages}
            </h3>
          </div>
          <div class="">
            <input
              type="range"
              min="1"
              max="4"
              step="1"
              bind:value={numImages}
              class="
                appearance-none
                w-full
                h-2
                bg-gradient-to-r
                from-cyan-400
                via-purple-200
                to-green-300
                rounded-full
                outline-none
                shadow-md
                cursor-pointer
                relative
                z-10
              "
              style="
                --thumb-size: 1.75rem;
              " />
          </div>
        </div>
        <div class=" flex w-full justify-between items-center">
          <h3 class="text-md w-auto">Image Output Size:</h3>

          <div class="flex w-full justify-center items-center">
            <div class="grid grid-cols-2 gap-4">
              <div>
                <label for="width" class="block font-bold p-2 text-gray-700"
                  >Width</label>
                <select
                  id="width"
                  name="width"
                  class="
                      shadow
                      appearance-none
                      border-2 border-emerald-700
                      bg-emerald-100
                      rounded
                      w-full
                      py-2
                      px-3
                      text-gray-700
                      leading-tight
                      focus:outline-none focus:shadow-outline
                    ">
                  <option value="512">512</option>
                  <option value="768">768</option>
                  <option value="1024" selected>1024</option>
                  <option value="2048">2048</option>
                  <option value="4096">4096</option>
                </select>
              </div>
              <div>
                <label for="height" class="block font-bold p-2 text-gray-700"
                  >Height</label>
                <select
                  id="height"
                  name="height"
                  class="
                      shadow
                      appearance-none
                      border-2 border-emerald-700
                      bg-emerald-100
                      rounded
                      w-full
                      py-2
                      px-3
                      text-gray-700
                      leading-tight
                      focus:outline-none focus:shadow-outline
                    ">
                  <option value="512">512</option>
                  <option value="768">768</option>
                  <option value="1024" selected>1024</option>
                  <option value="2048">2048</option>
                  <option value="4096">4096</option>
                </select>
              </div>
            </div>
          </div>
        </div>
      </div>
      <form class="row p-12 rounded-2xl m-4" onsubmit={greet}>
        <textarea
          id="greet-input"
          class="mr-4 p-6 border-2 border-gray-300 rounded-md w-full"
          placeholder="Enter a prompt to create..."
          bind:value={promptInput} />
        <button
          type="submit"
          class="bg-emerald-300 p-6 cursor-pointer rounded-lg hover:bg-emerald-400 transition-all"
          >Submit</button>
      </form>

      {#await greetMsg}
        <p>Loading...</p>
      {:then value}
        <p>{value}</p>
      {:catch error}
        <p style="color: red">{error}</p>
      {/await}
    </div>
  </div>
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

  input[type="range"]::-webkit-slider-thumb {
    appearance: none;
    width: var(--thumb-size);
    height: var(--thumb-size);
    background-color: white;
    border: 2px solid black;
    border-radius: 0.375rem;
    box-shadow:
      0 1px 3px 0 rgba(0, 0, 0, 0.1),
      0 1px 2px 0 rgba(0, 0, 0, 0.06);
    margin-top: calc(
      - (var(--thumb-size) - 0.5rem) / 2
    ); /* Center the thumb vertically */
  }

  input[type="range"]::-moz-range-thumb {
    appearance: none;
    width: var(--thumb-size);
    height: var(--thumb-size);
    background-color: white;
    border: 2px solid black;
    border-radius: 0.375rem;
    box-shadow:
      0 1px 3px 0 rgba(0, 0, 0, 0.1),
      0 1px 2px 0 rgba(0, 0, 0, 0.06);
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
