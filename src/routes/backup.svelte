<script>
  import { invoke } from "@tauri-apps/api/core"; // Assuming Tauri context
  import { onMount } from "svelte";
  import OpenAI from "openai";

  // --- Reactive State (Svelte 5 Runes) ---
  let promptInput = $state("");
  let numImages = $state(1); // Default to 1, user can change
  let contentType = $state("image"); // 'image' or 'text'
  let selectedLLM = $state("gpt-4o");
  let generationPromise = $state(null); // Holds the promise for the generation
  let currentPrompt = $state(""); // Store the prompt used for the current generation
  let savedGenerations = $state([]); // Array to hold { id: number, prompt: string, type: string, result: string | string[] }

  let selectedWidth = $state(1024); // Example default value
  let selectedHeight = $state(1024); // Example default value

  // Reactive statement to generate the dimension string
  let dimensionString = $derived(`${selectedWidth}x${selectedHeight}`);

  // --- OpenAI Client Setup ---
  // WARNING: Exposing API keys directly in client-side code is insecure for web deployment.
  // Consider using a backend proxy or Tauri backend command to handle API calls securely.
  const client = new OpenAI({
    apiKey: import.meta.env.VITE_OPENAI_API_KEY,
    dangerouslyAllowBrowser: true, // Acknowledge security risk
  });

  // --- Load Saved Generations on Mount ---
  onMount(() => {
    const storedGenerations = localStorage.getItem("savedGenerations");
    if (storedGenerations) {
      try {
        savedGenerations = JSON.parse(storedGenerations);
      } catch (e) {
        console.error("Failed to parse saved generations from localStorage", e);
        localStorage.removeItem("savedGenerations"); // Clear corrupted data
      }
    }
  });

  // --- Persist Saved Generations to localStorage ---
  $effect(() => {
    // This effect runs whenever savedGenerations changes
    if (savedGenerations.length >= 0) {
      // Check length >= 0 to ensure it runs even when cleared
      localStorage.setItem(
        "savedGenerations",
        JSON.stringify(savedGenerations),
      );
      console.log("Generations saved to localStorage");
    }
  });

  function preventDefault(fn) {
    return function (event) {
      event.preventDefault();
      fn.call(this, event);
    };
  }

  // --- Core Generation Logic ---
  async function generateContent(prompt, count) {
    currentPrompt = prompt; // Store the prompt for potential saving
    console.log(
      `🚀 Generating ${contentType} for prompt: "${prompt}", count: ${count}`,
    );

    if (contentType === "image") {
      console.log(`   Type: Image`);
      try {
        const response = await client.images.generate({
          model: "dall-e-3", // Or other models supporting 'n' > 1 if needed
          prompt: prompt,
          n: numImages, // Use the reactive count
          response_format: "url", // Get URLs
          size: imageSize || "1024x1024", // Add size if needed, ensure model supports it
        });
        // Extract just the URLs
        const imageUrls = response.data.map((item) => item.url);
        console.log("Image URLs received:", imageUrls);
        if (!imageUrls || imageUrls.length === 0) {
          throw new Error("API returned no image URLs.");
        }
        // Return object indicating type and result
        return { type: "image", result: imageUrls };
      } catch (err) {
        console.error("Image generation error:", err);
        throw err; // Propagate error to the await block
      }
    } else {
      console.log(`   Type: Text`);
      try {
        // Using chat completions for text generation
        const response = await client.chat.completions.create({
          model: selectedLLM || "gpt-4o",
          messages: [{ role: "user", content: prompt }],
          n: 1, // Text generation typically returns one primary response
        });
        const textResult = response.choices[0]?.message?.content;
        console.log("Text response received:", textResult);
        if (!textResult) {
          throw new Error("API returned no text content.");
        }
        // Return object indicating type and result
        return { type: "text", result: textResult };
      } catch (err) {
        console.error("Text generation error:", err);
        throw err; // Propagate error
      }
    }
  }

  // --- Form Submission Handler ---
  function handleSubmit() {
    if (!promptInput.trim()) {
      alert("Please enter a prompt.");
      return;
    }
    // Assign the promise to trigger the #await block
    generationPromise = generateContent(promptInput, numImages);
  }

  // --- Save Generation Logic ---
  function saveCurrentGeneration(prompt, generationData) {
    if (!prompt || !generationData) return;

    const newSave = {
      id: Date.now(), // Simple unique ID
      prompt: prompt,
      type: generationData.type,
      result: generationData.result,
    };
    // Add to the beginning of the array for newest first
    savedGenerations = [newSave, ...savedGenerations];
    console.log("Generation saved:", newSave);
  }

  // --- Delete Saved Generation ---
  function deleteSavedGeneration(idToDelete) {
    savedGenerations = savedGenerations.filter((gen) => gen.id !== idToDelete);
    console.log("Deleted generation with ID:", idToDelete);
  }
</script>

<main class="container mx-auto p-4 md:p-8">
  <div class="row">
    <h1 class="header text-4xl md:text-6xl lg:text-8xl">ImageGen</h1>
  </div>
  <h2 class="text-xl md:text-2xl mb-8 text-center text-gray-600">
    Make cool stuff with AI
  </h2>

  <!-- Generation Controls -->
  <div
    class="p-1 bg-gradient-to-r from-cyan-400 via-purple-200 to-green-300 w-full my-8 rounded-lg shadow-lg">
    <div class="w-full bg-white p-4 md:p-6 rounded-md">
      <!-- Content Type Selector -->
      <div
        class="flex flex-col sm:flex-row w-full justify-between items-start sm:items-center mb-6 gap-4">
        <h3 class="flex items-center text-lg font-semibold whitespace-nowrap">
          Generate:
        </h3>
        <div class="flex items-center space-x-2 sm:space-x-4">
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
              class="px-3 py-1.5 md:px-4 md:py-2 bg-gray-200 rounded-md text-sm md:text-base text-gray-700 cursor-pointer peer-checked:bg-blue-500 peer-checked:text-white peer-checked:shadow-md transition-colors hover:bg-gray-300">
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
              class="px-3 py-1.5 md:px-4 md:py-2 bg-gray-200 rounded-md text-sm md:text-base text-gray-700 cursor-pointer peer-checked:bg-blue-500 peer-checked:text-white peer-checked:shadow-md transition-colors hover:bg-gray-300">
              Image
            </label>
          </div>
        </div>
      </div>

      <!-- Image Specific Options (Conditional) -->
      {#if contentType === "image"}
        <div class="border-t border-gray-200 pt-6 mb-6">
          <h3 class="text-lg font-semibold text-gray-800 mb-4">
            Image Generation Options
          </h3>
          <!-- Number of Images Slider -->
          <div
            class="grid grid-cols-1 md:grid-cols-[auto_1fr] items-center gap-x-4 gap-y-2 mb-4">
            <label for="num-images-slider" class="font-medium text-gray-700"
              >Number:</label>
            <div class="flex items-center gap-3">
              <input
                id="num-images-slider"
                type="range"
                min="1"
                max="4"
                step="1"
                bind:value={numImages}
                class="appearance-none w-full h-2 bg-gradient-to-r from-cyan-300 to-purple-300 rounded-full outline-none shadow-sm cursor-pointer flex-grow"
                style="--thumb-size: 1.5rem;" />
              <span
                class="bg-cyan-100 text-cyan-800 font-medium w-8 h-8 flex items-center justify-center rounded-full text-sm">
                {numImages}
              </span>
            </div>
          </div>

          <!-- Image Size Selector (Example - not wired to API call yet) -->
          <div
            class="grid grid-cols-1 md:grid-cols-[auto_1fr] items-center gap-x-4 gap-y-2">
            <span class="font-medium text-gray-700">Size:</span>
            <div class="grid grid-cols-2 gap-3">
              <div>
                <label for="width" class="sr-only">Width</label>
                <!-- <select
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
              </select> -->
                <select
                  id="width"
                  name="width"
                  class="shadow-sm appearance-none border border-gray-300 bg-white rounded w-full py-1.5 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 text-sm">
                  <option value="512">512</option>
                  <option value="768">768</option>
                  <option value="1024" selected>1024</option>
                  <option value="2048">2048</option>
                  <option value="4096">4096</option>
                </select>
              </div>
              <div>
                <label for="height" class="sr-only">Height</label>
                <select
                  id="height"
                  name="height"
                  class="shadow-sm appearance-none border border-gray-300 bg-white rounded w-full py-1.5 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 text-sm">
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
      {/if}

      <!-- Prompt Input Form -->
      <form
        class="border-t border-gray-200 pt-6"
        onsubmit={preventDefault(handleSubmit)}>
        <label
          for="prompt-input"
          class="block text-lg font-semibold text-gray-800 mb-3"
          >Enter your prompt:</label>
        <div class="flex flex-col sm:flex-row gap-3">
          <textarea
            id="prompt-input"
            class="flex-grow p-3 border border-gray-300 rounded-md shadow-sm focus:ring-2 focus:ring-cyan-500 focus:border-cyan-500 resize-none"
            rows="3"
            placeholder="e.g., A cute cat astronaut floating in space"
            bind:value={promptInput}></textarea>
          <button
            type="submit"
            class="bg-emerald-500 text-white font-semibold px-5 py-3 cursor-pointer rounded-lg hover:bg-emerald-600 transition-colors shadow-sm disabled:opacity-50 disabled:cursor-not-allowed"
            disabled={!promptInput.trim() || generationPromise}>
            {#if generationPromise}
              Generating...
            {:else}
              Generate
            {/if}
          </button>
        </div>
      </form>
    </div>
  </div>

  <!-- Generation Result Display -->
  <div class="mt-8 mb-12 min-h-[100px]">
    {#if generationPromise}
      {#await generationPromise}
        <div class="p-4 text-center text-gray-500">
          <svg
            class="animate-spin h-8 w-8 text-cyan-500 mx-auto"
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24">
            <circle
              class="opacity-25"
              cx="12"
              cy="12"
              r="10"
              stroke="currentColor"
              stroke-width="4"></circle>
            <path
              class="opacity-75"
              fill="currentColor"
              d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
            ></path>
          </svg>
          <p class="mt-2">Generating your creation...</p>
        </div>
      {:then data}
        <div class="bg-white p-4 rounded-lg shadow">
          <h3 class="text-lg font-semibold mb-3">
            Result for: "{currentPrompt}"
          </h3>
          {#if data.type === "image"}
            <div
              class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
              {#each data.result as imageUrl, i (imageUrl)}
                <div class="relative group">
                  <img
                    src={imageUrl}
                    alt="Generated image {i + 1} for prompt: {currentPrompt}"
                    class="w-full h-auto object-cover rounded border border-gray-200 shadow-sm"
                    loading="lazy" />
                  <a
                    href={imageUrl}
                    target="_blank"
                    rel="noopener noreferrer"
                    class="absolute inset-0 bg-black bg-opacity-0 group-hover:bg-opacity-50 flex items-center justify-center text-white opacity-0 group-hover:opacity-100 transition-opacity text-xs font-bold"
                    >View Full</a>
                </div>
              {/each}
            </div>
          {:else if data.type === "text"}
            <div
              class="text-result bg-gray-50 p-4 border rounded whitespace-pre-wrap">
              {data.result}
            </div>
          {/if}
          <!-- Save Button -->
          <div class="mt-4 text-right">
            <button
              onclick={() => saveCurrentGeneration(currentPrompt, data)}
              class="bg-blue-500 hover:bg-blue-600 text-white text-sm font-medium py-1.5 px-3 rounded shadow-sm transition-colors">
              Save Result
            </button>
          </div>
        </div>
      {:catch error}
        <div
          class="p-4 text-center text-red-600 bg-red-50 border border-red-200 rounded-lg">
          <p class="font-semibold">Generation failed!</p>
          <p class="text-sm mt-1">{error.message}</p>
        </div>
      {/await}
    {/if}
  </div>

  <!-- Saved Generations Display -->
  {#if savedGenerations.length > 0}
    <div class="mt-12 border-t pt-8">
      <h2 class="text-2xl font-semibold mb-6 text-center">Saved Generations</h2>
      <div class="space-y-6">
        {#each savedGenerations as saved (saved.id)}
          <div class="bg-white p-4 rounded-lg shadow relative group">
            <button
              onclick={() => deleteSavedGeneration(saved.id)}
              class="absolute top-2 right-2 p-1 bg-red-100 text-red-600 hover:bg-red-200 rounded-full opacity-0 group-hover:opacity-100 transition-opacity focus:opacity-100 focus:outline-none focus:ring-2 focus:ring-red-500"
              aria-label="Delete saved generation">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
                stroke-width="2">
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
            <p class="text-sm text-gray-500 mb-2">Prompt:</p>
            <p class="font-medium text-gray-800 mb-3">"{saved.prompt}"</p>
            {#if saved.type === "image"}
              <div
                class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-3">
                {#each saved.result as imageUrl, i (imageUrl)}
                  <div class="relative group/img">
                    <img
                      src={imageUrl}
                      alt="Saved image {i + 1} for prompt: {saved.prompt}"
                      class="w-full h-auto object-cover rounded border border-gray-200"
                      loading="lazy" />
                    <a
                      href={imageUrl}
                      target="_blank"
                      rel="noopener noreferrer"
                      class="absolute inset-0 bg-black bg-opacity-0 group-hover/img:bg-opacity-50 flex items-center justify-center text-white opacity-0 group-hover/img:opacity-100 transition-opacity text-xs font-bold"
                      >View</a>
                  </div>
                {/each}
              </div>
            {:else if saved.type === "text"}
              <div
                class="text-result bg-gray-50 p-3 border rounded text-sm whitespace-pre-wrap">
                {saved.result}
              </div>
            {/if}
          </div>
        {/each}
      </div>
    </div>
  {/if}
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
