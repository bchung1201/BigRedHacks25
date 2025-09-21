<script lang="ts">
  import { Loader, Upload } from "lucide-svelte";
  import { onMount } from "svelte";
  import paper from "paper";
  import { throttle, debounce } from "throttle-debounce";
  import { GoogleGenAI } from "@google/genai";

  import Paint from "$lib/Paint.svelte";
  import valleyImg from "$lib/assets/valley.png";
  import PreviewImages from "$lib/PreviewImages.svelte";
  import ImageOutput from "$lib/ImageOutput.svelte";
  import Tools from "$lib/Tools.svelte";

  const promptOptionsByImage: Record<string, string[]> = {
    abstract: [
      "seaside village, illustration, studio ghibli style",
      "a scene from the novel Dune, surreal, desert, futuristic",
      "lunar landing in the style of a van gogh painting",
    ],
    puppy: [
      "cute dog, drawing, bright, happy",
      "intense cybernetic dog, realistic",
      "3d render of a dog, whimsical, cartoonish",
    ],
    car: [
      "vintage, image of a car, classic theme",
      "Tesla driving on the Moon, planets in the background",
      "futuristic car in cyberpunk cityscape, photorealistic",
    ],
    valley: [
      "cityscape, studio ghibli, illustration",
      "open field, animals, friends",
      "sunset, majestic, pencil drawing",
    ],
    pasta: [
      "delicious food, photorealistic",
      "cartoon, MARVEL-style, colorful",
      "painting, still life, artistic",
    ],
  };
  let value: string = "";
  $: currentImageName = "valley";
  $: promptOptions = promptOptionsByImage[currentImageName];

  let imgInput: HTMLImageElement;
  let imgOutput: HTMLImageElement;
  let canvasDrawLayer: HTMLCanvasElement;
  let inputElement: HTMLInputElement;
  let fileInput: HTMLInputElement;

  let isImageUploaded = false;
  let isFirstImageGenerated = false;
  let isMobile = false;
  let showEnhanceModal = false;
  let enhancementPrompt = "";
  $: numIterations = 0;

  // we track lastUpdatedAt so that expired requests don't overwrite the latest
  let lastUpdatedAt = 0;

  // used for undo/redo functionality
  let outputImageHistory: string[] = [];
  $: currentOutputImageIndex = -1;

  $: isLoading = false;
  let isInputImageLoading = false;

  let brushSize = "sm";
  let paint = "#000000";
  const radiusByBrushSize: Record<string, number> = {
    xs: 1,
    sm: 2,
    md: 3,
    lg: 4,
  };

  function setPaint(e: CustomEvent<string>) {
    paint = e.detail;
  }
  function setBrushSize(e: CustomEvent<string>) {
    brushSize = e.detail;
  }

  function checkBreakpoint() {
    isMobile = window.innerWidth < 640;
  }

  onMount(() => {
    // Manual mobile check is needed because of binding issues with <ImageOutput />
    checkBreakpoint();
    window.addEventListener("resize", checkBreakpoint);

    /* 
      Setup paper.js for canvas which is a layer above our input image.
      Paper is used for drawing/paint functionality.
    */
    paper.setup(canvasDrawLayer);
    const tool = new paper.Tool();

    let path: paper.Path;

    tool.onMouseDown = (event: paper.ToolEvent) => {
      path = new paper.Path();
      path.strokeColor = new paper.Color(paint);
      path.strokeWidth = radiusByBrushSize[brushSize] * 4;
      path.add(event.point);

      throttledgenerateOutputImage();
    };

    tool.onMouseDrag = (event: paper.ToolEvent) => {
      path.add(event.point);

      throttledgenerateOutputImage();
    };

    if (inputElement) {
      inputElement.focus();
    }

    setImage(valleyImg);
    return () => window.removeEventListener("resize", checkBreakpoint);
  });

  function onLoadInputImg(event: Event) {
    resizeImage(event);

    isImageUploaded = true;
    isInputImageLoading = false;

    // kick off an inference on first image load so output image is populated as well
    // otherwise it will be empty
    if (!isFirstImageGenerated) {
      generateOutputImage();
      isFirstImageGenerated = true;
    }
  }

  function setImage(src: string) {
    isInputImageLoading = true;
    imgInput.src = src;

    outputImageHistory.unshift(src);
    currentOutputImageIndex = 0;

    function loopGenerate() {
      if (isInputImageLoading) {
        // wait for onload before generating an image
        setTimeout(loopGenerate, 100);
        return;
      }

      generateOutputImage();
    }

    loopGenerate();
  }

  function setPrompt(prompt: string) {
    value = prompt;
    generateOutputImage();
  }

  // Our images need to be sized 320x320 for both input and output
  // This is important because we combine the canvas layer with the image layer
  // so the pixels need to matchup. We use object-fit: cover to crop images.
  function resizeImage(event: Event) {
    const target = event.target as HTMLImageElement;

    // Ensure dimensions are set correctly (though they should already be set via HTML attributes)
    target.style.width = `320px`;
    target.style.height = `320px`;
  }

  function loadImage(e: Event) {
    const target = e.target as HTMLInputElement;
    if (!target || !target.files) return;
    const file = target.files[0];

    if (file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        if (e?.target?.result && typeof e.target.result === "string") {
          isImageUploaded = true;
          setImage(e?.target.result);
        }
      };

      reader.readAsDataURL(file);
    }
  }

  function getImageData(useOutputImage: boolean = false): Promise<Blob> {
    return new Promise((resolve, reject) => {
      const tempCanvas = document.createElement("canvas");
      tempCanvas.width = 320;
      tempCanvas.height = 320;
      const tempCtx = tempCanvas.getContext("2d");
      if (!tempCtx) {
        reject("no context");
        return;
      }

      if (useOutputImage) {
        tempCtx.drawImage(imgOutput, 0, 0, 320, 320);
      } else {
        // combines the canvas with the input image so that the
        // generated image contains edits made by paint brush
        tempCtx.drawImage(imgInput, 0, 0, 320, 320);
        tempCtx.drawImage(canvasDrawLayer, 0, 0, 320, 320);
      }

      tempCanvas.toBlob((blob) => {
        if (blob) {
          resolve(blob);
        } else {
          reject("blob creation failed");
        }
      }, "image/jpeg");
    });
  }

  const throttledgenerateOutputImage = throttle(
    250,
    () => generateOutputImage(),
    { noLeading: false, noTrailing: false }
  );

  const debouncedgenerateOutputImage = debounce(
    100,
    () => generateOutputImage(),
    { atBegin: false }
  );

  function movetoCanvas() {
    imgInput.src = imgOutput.src;
  }

  function downloadImage() {
    let a = document.createElement("a");
    a.href = imgOutput.src;
    a.download = "ai-generated-image.jpeg";
    a.click();
  }

  async function enhanceWithGemini() {
    console.log("🚀 Enhance button clicked!");

    if (!imgOutput.src || !isFirstImageGenerated) {
      console.error("❌ No output image to enhance");
      return;
    }

    console.log("✅ Starting Gemini enhancement...");
    console.log("📝 Current prompt:", enhancementPrompt);
    isLoading = true;

    try {
      console.log("🖼️ Converting output image to base64...");
      // Convert current output image to base64
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = 320;
      canvas.height = 320;

      if (!ctx) {
        throw new Error("Could not get canvas context");
      }

      ctx.drawImage(imgOutput, 0, 0, 320, 320);
      const base64Image = canvas.toDataURL('image/png').split(',')[1];
      console.log("✅ Image converted to base64, length:", base64Image.length);

      // Initialize Gemini AI
      console.log("🔑 Initializing Gemini AI...");
      console.log("🔍 API Key check:", import.meta.env.VITE_GEMINI_API_KEY ? "Found" : "Missing");
      const ai = new GoogleGenAI({
        apiKey: import.meta.env.VITE_GEMINI_API_KEY
      });
      console.log("✅ Gemini AI initialized");

      const prompt = [
        {
          text: `Create a picture of "${enhancementPrompt}" based on the provided image. Enhance and improve the quality while maintaining the core elements.`
        },
        {
          inlineData: {
            mimeType: "image/png",
            data: base64Image,
          },
        },
      ];

      console.log("📡 Sending request to Gemini API...");
      const sentAt = new Date().getTime();
      const response = await ai.models.generateContent({
        model: "gemini-2.5-flash-image-preview",
        contents: prompt,
      });
      console.log("✅ Received response from Gemini API");
      console.log("📋 Response structure:", response);

      if (sentAt > lastUpdatedAt) {
        console.log("🔍 Looking for image in response...");
        // Find the image in the response
        let foundImage = false;
        for (const part of response.candidates[0].content.parts) {
          if (part.inlineData) {
            console.log("🖼️ Found image in response!");
            const imageData = part.inlineData.data;
            console.log("📊 Image data length:", imageData.length);

            // Convert base64 to blob (browser equivalent of Buffer.from in Node.js)
            const binaryString = atob(imageData);
            const bytes = new Uint8Array(binaryString.length);
            for (let i = 0; i < binaryString.length; i++) {
              bytes[i] = binaryString.charCodeAt(i);
            }
            const blob = new Blob([bytes], { type: 'image/png' });
            const imageURL = URL.createObjectURL(blob);

            outputImageHistory = [imageURL, ...outputImageHistory];
            if (outputImageHistory.length > 10) {
              outputImageHistory = outputImageHistory.slice(0, 10);
            }
            imgOutput.src = imageURL;
            numIterations += 1;
            lastUpdatedAt = sentAt;
            foundImage = true;
            console.log("✅ Successfully updated output image!");
            break;
          }
        }

        if (!foundImage) {
          console.log("⚠️ No image found in response, checking for text...");
          for (const part of response.candidates[0].content.parts) {
            if (part.text) {
              console.log("📝 Text response:", part.text);
            }
          }
        }
      } else {
        console.log("⏰ Request outdated, ignoring response");
      }

      isFirstImageGenerated = true;
      console.log("🎉 Enhancement process completed!");
    } catch (error) {
      console.error("❌ Gemini enhancement failed:", error);
      console.error("🔍 Error details:", error.message);
      console.error("📊 Full error:", error);
    } finally {
      isLoading = false;
      console.log("🏁 Enhancement function finished");
    }
  }

  function enhance() {
    showEnhanceModal = true;
  }

  function redoOutputImage() {
    if (currentOutputImageIndex > 0 && outputImageHistory.length > 1) {
      currentOutputImageIndex -= 1;
      imgOutput.src = outputImageHistory[currentOutputImageIndex];
    } else {
      generateOutputImage(true);
    }
  }

  function undoOutputImage() {
    if (currentOutputImageIndex < outputImageHistory.length - 1) {
      currentOutputImageIndex += 1;
      imgOutput.src = outputImageHistory[currentOutputImageIndex];
    }
  }

  function resetInput() {
    if (fileInput) {
      fileInput.value = "";
    }
  }

  async function generateOutputImage(
    useOutputImage: boolean = false,
    iterations: number = 2
  ) {
    isLoading = true;
    const data = await getImageData(useOutputImage);

    const formData = new FormData();
    formData.append("image", data, "image.jpg");
    formData.append("prompt", value);
    formData.append("num_iterations", iterations.toString());

    const sentAt = new Date().getTime();
    try {
      const res = await fetch((window as any).INFERENCE_BASE_URL, {
        method: "POST",
        body: formData,
      });
      const blob = await res.blob();

      if (sentAt > lastUpdatedAt) {
        const imageURL = URL.createObjectURL(blob);
        outputImageHistory = [imageURL, ...outputImageHistory];
        if (outputImageHistory.length > 10) {
          outputImageHistory = outputImageHistory.slice(0, 10);
        }
        imgOutput.src = imageURL;
        if (useOutputImage) {
          numIterations += iterations;
        } else {
          numIterations = 1;
        }
        lastUpdatedAt = sentAt;
      }

      isFirstImageGenerated = true;
    } finally {
      isLoading = false;
    }
  }
</script>

<main class="flex flex-col items-center md:pt-12 text-gray-900">
  <div class="md:max-w-screen-lg lg:w-[1024px] w-full">
    <div
      class="bg-white sm:border sm:border-gray-200 md:rounded-lg px-4 py-6 sm:p-6 flex flex-col gap-6 shadow-sm"
    >
      <div class="flex flex-col gap-3 sm:gap-1">
        <div class="flex items-center justify-between">
          <h1 class="text-4xl font-degular">Diffusion Art</h1>
        </div>
      </div>

      <div class="flex flex-col gap-4">
        <h3 class="heading">Prompt</h3>
        <div class="flex flex-col sm:flex-row gap-2 lg:flex-nowrap flex-wrap">
          {#each promptOptions as item}
            <button
              class="italic flex-shrink-0 text-xs px-4 py-2 border border-gray-300 rounded-full text-gray-600 hover:bg-gray-50"
              class:prompt-active={item === value}
              on:click={() => setPrompt(item)}>{item}</button
            >
          {/each}
        </div>
        <input
          class="rounded-full bg-gray-50 border border-gray-200 py-4 px-6 w-full text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
          bind:value
          bind:this={inputElement}
          on:input={debouncedgenerateOutputImage}
          placeholder="Enter prompt here"
        />
      </div>

      <div class="flex flex-col md:flex-row gap-6 md:gap-0">
        <div
          class="flex flex-col gap-6 md:pr-4 md:border-r md:border-gray-200 w-full"
        >
          <div class="flex flex-col gap-1">
            <div class="heading flex gap-1 items-center">
              Canvas
              {#if isLoading}
                <Loader size={14} class="animate-spin" />
              {/if}
            </div>
            <div class="text-xs">Draw on the image to generate a new one</div>
          </div>

          <div class="flex gap-6 flex-col sm:flex-row">
            <div class="relative">
              <img
                alt="input"
                bind:this={imgInput}
                width={320}
                height={320}
                class="absolute inset-0 bg-[#D9D9D9] pointer-events-none object-cover"
                class:hidden={!isImageUploaded}
                on:load={onLoadInputImg}
              />
              <canvas
                bind:this={canvasDrawLayer}
                width={320}
                height={320}
                class="relative z-10"
              ></canvas>
            </div>
            {#if isMobile}
              <ImageOutput
                bind:imgOutput
                {isFirstImageGenerated}
                {resizeImage}
              />
            {/if}
            <div class="flex gap-4">
              <Paint
                {paint}
                {brushSize}
                on:clearCanvas={() => {
                  paper.project.activeLayer.removeChildren();
                  paper.view.update();
                  generateOutputImage();
                }}
                on:setPaint={setPaint}
                on:setBrushSize={setBrushSize}
              />
              <div class="sm:hidden block">
                <Tools
                  {undoOutputImage}
                  {redoOutputImage}
                  {enhance}
                  {movetoCanvas}
                  {downloadImage}
                />
              </div>
            </div>
          </div>
          <div class="flex gap-2">
            <PreviewImages
              {promptOptionsByImage}
              {imgInput}
              {setImage}
              setCurrentImage={(name) => (currentImageName = name)}
              setPrompt={(v) => (value = v)}
            />
          </div>
          <input
            type="file"
            accept="image/*"
            id="file-upload"
            hidden
            bind:this={fileInput}
            on:change={loadImage}
            on:click={resetInput}
          />
          <label
            for="file-upload"
            class="btns-container flex-col w-fit cursor-pointer w-full sm:w-fit"
          >
            <div class="flex items-center gap-2 font-medium">
              <Upload size={16} />
              Upload Image (JPG, PNG)
            </div>
          </label>
        </div>

        <div class="flex flex-col gap-6 w-full md:pl-6">
          <div class="flex flex-col gap-1 sm:block hidden">
            <div class="flex items-center gap-1 heading">
              Output
              {#if isLoading}
                <Loader size={14} class="animate-spin" />
              {/if}
            </div>
            <div class="text-xs">
              Generated Image (iterations: {numIterations})
            </div>
          </div>

          <div class="flex gap-4 flex-col sm:flex-row md:flex-col lg:flex-row">
            {#if !isMobile}
              <ImageOutput
                bind:imgOutput
                {isFirstImageGenerated}
                {resizeImage}
              />
            {/if}
            <div class="sm:block hidden">
              <Tools
                {undoOutputImage}
                {redoOutputImage}
                {enhance}
                {movetoCanvas}
                {downloadImage}
              />
            </div>
          </div>
        </div>
      </div>
    </div>

    <div
      class="lg:w-full flex mt-6 mb-8 md:mb-12 mx-2 md:mx-6 lg:mx-0 justify-center items-center"
    >
      <div class="text-sm text-gray-500">
        Powered by SDXL Turbo
      </div>
    </div>
  </div>
  {#if showEnhanceModal}
    <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white p-6 rounded-lg shadow-lg">
        <h2 class="text-2xl font-bold mb-4">Enhance Image</h2>
        <p class="mb-4">Enter a new prompt to enhance the image with Gemini.</p>
        <input
          class="rounded-full bg-gray-50 border border-gray-200 py-4 px-6 w-full text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
          bind:value={enhancementPrompt}
          placeholder="Enter enhancement prompt"
        />
        <div class="flex justify-end gap-4 mt-6">
          <button
            class="btns-container"
            on:click={() => {
              showEnhanceModal = false;
              enhancementPrompt = "";
            }}
          >
            Cancel
          </button>
          <button
            class="btns-container"
            on:click={() => {
              if (enhancementPrompt) {
                enhanceWithGemini();
                showEnhanceModal = false;
                enhancementPrompt = "";
              }
            }}
          >
            Enhance
          </button>
        </div>
      </div>
    </div>
  {/if}
</main>

<style lang="postcss">
  .heading {
    @apply text-2xl font-degular;
  }

  .btns-container {
    @apply flex items-center gap-2 py-2 px-6 border rounded-full border-gray-300 text-sm hover:bg-gray-50;
  }


  .prompt-active {
    @apply text-blue-600 border-blue-500 bg-blue-50;
  }
</style>
