<script>
  // import { $state, $derived, $effect } from "svelte/internal"; // Use 'svelte/reactivity' in future Svelte versions

  // --- Component State ---
  import OpenAI from "openai";
  let prompt = $state("");
  let model = $state("dall-e-3"); // Default model (updated to latest)
  let n = $state(1); // Number of images
  let size = $state("1024x1024"); // Default size
  let quality = $state("standard"); // Only for dall-e-3
  let style = $state("vivid"); // Only for dall-e-3
  let responseFormat = $state("url"); // 'url' or 'b64_json'

  let isLoading = $state(false);
  let images = $state([]); // Array to store image URLs or b64 data
  let error = $state(null); // Store API errors

  const client = new OpenAI({
    apiKey: import.meta.env.VITE_OPENAI_API_KEY,
    dangerouslyAllowBrowser: true, // Acknowledge security risk
  });

  // --- Derived State ---
  const isDallE3 = $derived(model === "dall-e-3");

  // Available sizes based on the selected model
  const availableSizes = $derived(() => {
    if (isDallE3) {
      return ["1024x1024", "1792x1024", "1024x1792"];
    } else {
      // dall-e-2
      return ["256x256", "512x512", "1024x1024"];
    }
  });

  // Max number of images allowed
  const maxN = $derived(isDallE3 ? 1 : 10);

  // Max prompt length
  const maxPromptLength = $derived(isDallE3 ? 4000 : 1000);

  // --- Effects for Automatic Adjustments ---

  // Effect to reset 'n' if model changes to dall-e-3 and n > 1
  $effect(() => {
    if (isDallE3 && n !== 1) {
      n = 1; // DALL-E 3 only supports n=1
    }
  });

  // Effect to reset size if the current size is invalid for the new model
  // $effect(() => {
  //   if (!availableSizes.includes(size)) {
  //     // Reset to default if current size is not supported by the selected model
  //     size = "1024x1024";
  //   }
  // });

  // Effect to clamp 'n' to the allowed maximum when not DALL-E 3
  $effect(() => {
    if (!isDallE3) {
      if (n > maxN) {
        n = maxN;
      }
      if (n < 1) {
        n = 1;
      }
    }
  });

  // --- API Call Function ---
  async function generateImages() {
    if (!prompt.trim()) {
      error = "Please enter a prompt.";
      return;
    }
    // No need for API key check here, handled by client instantiation

    isLoading = true;
    error = null;
    images = [];

    // Construct parameters for the OpenAI client library
    const params = {
      prompt: prompt,
      model: model,
      n: n,
      size: size,
      response_format: responseFormat,
      // user: 'your-unique-user-id' // Optional
    };

    // Add parameters only applicable to dall-e-3
    if (isDallE3) {
      params.quality = quality;
      params.style = style;
    }
    // No need to explicitly delete for other models,
    // the library won't send them if not in the params object.

    console.log("Sending parameters:", params); // For debugging

    try {
      // Use the OpenAI client library
      const response = await client.images.generate(params);

      console.log("API Success Response:", response); // For debugging
      images = response.data; // The library puts the image array in response.data
    } catch (err) {
      console.error("Failed to generate images:", err);
      if (err instanceof APIError) {
        // Handle specific OpenAI API errors
        error = `API Error (${err.status}): ${err.message}`;
        if (err.code === "invalid_api_key") {
          error =
            "Invalid API Key. Please check your VITE_OPENAI_API_KEY in the .env file.";
        } else if (
          err.message.includes("model does not exist") ||
          err.message.includes("Invalid model")
        ) {
          error = `Model '${model}' might not be supported or available. (Error: ${err.message})`;
        }
      } else if (err instanceof Error) {
        // Handle generic errors
        error = `Error: ${err.message}`;
      } else {
        error = "An unknown error occurred.";
      }
      images = [];
    } finally {
      isLoading = false;
    }
  }
</script>

<div class="image-generator">
  <h1>OpenAI Image Generator</h1>

  <div class="settings">
    <div class="form-group">
      <label for="model">Model:</label>
      <select id="model" bind:value={model}>
        <option value="gpt-4o">GPT-4o</option>
        <option value="dall-e-3">DALL-E 3</option>
        <option value="dall-e-2">DALL-E 2</option>
      </select>
    </div>

    <div class="form-group">
      <label for="size">Size:</label>
      <select id="size" bind:value={size}>
        {#each availableSizes as s}
          <option value={s}>{s}</option>
        {/each}
      </select>
    </div>

    <div class="form-group">
      <label for="n">Number of Images (Max {maxN}):</label>
      <input
        type="number"
        id="n"
        bind:value={n}
        min="1"
        max={maxN}
        disabled={isDallE3} />
      {#if isDallE3}
        <small>(DALL-E 3 only supports 1 image)</small>
      {/if}
    </div>

    <!-- DALL-E 3 Specific Options -->
    {#if isDallE3}
      <div class="form-group">
        <label for="quality">Quality:</label>
        <select id="quality" bind:value={quality}>
          <option value="standard">Standard</option>
          <option value="hd">HD</option>
        </select>
      </div>

      <div class="form-group">
        <label for="style">Style:</label>
        <select id="style" bind:value={style}>
          <option value="vivid">Vivid</option>
          <option value="natural">Natural</option>
        </select>
      </div>
    {/if}

    <div class="form-group">
      <label for="responseFormat">Response Format:</label>
      <select id="responseFormat" bind:value={responseFormat}>
        <option value="url">URL</option>
        <option value="b64_json">Base64 JSON</option>
      </select>
      {#if responseFormat === "url"}
        <small>(URLs expire after 60 minutes)</small>
      {/if}
    </div>

    <!-- Prompt needs to be last in grid for layout -->
    <div class="form-group">
      <label for="prompt">Prompt (Max {maxPromptLength} chars):</label>
      <textarea
        id="prompt"
        bind:value={prompt}
        rows="5"
        maxlength={maxPromptLength}
        placeholder="e.g., A photorealistic image of an astronaut playing chess with a cat on the moon"
      ></textarea>
    </div>
  </div>

  <button onclick={generateImages} disabled={isLoading || !prompt}>
    {#if isLoading}
      Generating...
    {:else}
      Generate Images
    {/if}
  </button>

  {#if error}
    <div class="error">{error}</div>
  {/if}

  {#if isLoading && images.length === 0}
    <div class="loading">Generating image(s), please wait...</div>
  {/if}

  {#if images.length > 0}
    <div class="image-results">
      {#each images as image}
        {#if responseFormat === "url"}
          <img
            src={image.url}
            alt={prompt}
            title={image.revised_prompt ?? prompt} />
        {:else if responseFormat === "b64_json"}
          <img
            src={`data:image/png;base64,${image.b64_json}`}
            alt={prompt}
            title={image.revised_prompt ?? prompt} />
        {/if}
      {/each}
      {#if isDallE3 && images[0]?.revised_prompt}
        <p
          style="grid-column: 1 / -1; text-align: left; font-size: 0.9em; color: #333;">
          <strong>Revised Prompt (DALL-E 3):</strong>
          {images[0].revised_prompt}
        </p>
      {/if}
    </div>
  {/if}
</div>

<style>
  .image-generator {
    font-family: sans-serif;
    max-width: 800px;
    margin: 2em auto;
    padding: 1em;
    border: 1px solid #ccc;
    border-radius: 8px;
    background-color: #f9f9f9;
  }
  h1 {
    text-align: center;
    color: #333;
  }
  .settings {
    display: grid;
    grid-template-columns: 1fr;
    gap: 1.5em;
    margin-bottom: 1.5em;
  }
  .form-group {
    display: flex;
    flex-direction: column;
  }
  .form-group label {
    margin-bottom: 0.5em;
    font-weight: bold;
    color: #555;
  }
  .form-group input,
  .form-group select,
  .form-group textarea {
    padding: 0.8em;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 1em;
  }
  .form-group textarea {
    resize: vertical;
    min-height: 80px;
  }
  .form-group input[type="number"] {
    max-width: 100px;
  }
  .form-group small {
    font-size: 0.8em;
    color: #777;
    margin-top: 0.3em;
  }
  .api-key-group small {
    color: #c0392b; /* Warning color */
  }
  button {
    padding: 0.8em 1.5em;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 1.1em;
    cursor: pointer;
    transition: background-color 0.2s ease;
    width: 100%;
  }
  button:hover:not(:disabled) {
    background-color: #0056b3;
  }
  button:disabled {
    background-color: #ccc;
    cursor: not-allowed;
  }
  .loading,
  .error {
    text-align: center;
    margin-top: 1em;
    padding: 1em;
    border-radius: 4px;
  }
  .loading {
    color: #007bff;
  }
  .error {
    color: #dc3545;
    background-color: #f8d7da;
    border: 1px solid #f5c6cb;
  }
  .image-results {
    margin-top: 2em;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1em;
  }
  .image-results img {
    max-width: 100%;
    height: auto;
    border: 1px solid #eee;
    border-radius: 4px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
  .image-results p {
    text-align: center;
    font-style: italic;
    color: #555;
  }

  /* Responsive adjustments */
  @media (min-width: 600px) {
    .settings {
      grid-template-columns: 1fr 1fr; /* Two columns on wider screens */
    }
    /* Make prompt span full width */
    .form-group:has(textarea) {
      grid-column: 1 / -1;
    }
    /* Make API key span full width */
    .api-key-group {
      grid-column: 1 / -1;
    }
  }
</style>
