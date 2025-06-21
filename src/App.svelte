<script>
  import { onMount } from "svelte";
  import * as faceapi from "@vladmandic/face-api";

  import Cop from "./Cop.svelte";

  let faceDescriptors;
  let combinedData;
  let uploadedImage;

  let status = $state();
  let faceApiLoaded = $state(false);
  let databaseLoaded = $state(false);
  let uploadedImageSrc = $state();
  let matches = $state([]);

  let loaded = $derived(faceApiLoaded && databaseLoaded);

  async function loadMatrix(filename) {
    try {
      const response = await fetch(filename);
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const buffer = await response.arrayBuffer();
      const view = new DataView(buffer);

      // Read header length (first 4 bytes, little endian)
      const headerLen = view.getUint32(0, true);

      // Read header JSON
      const headerBytes = new Uint8Array(buffer, 4, headerLen);
      const headerStr = new TextDecoder().decode(headerBytes);
      const header = JSON.parse(headerStr);

      // Read data
      const dataOffset = 4 + headerLen;
      const dataBytes = new Uint8Array(buffer, dataOffset);

      // Convert to Float32Array
      const data = new Float32Array(
        dataBytes.buffer,
        dataBytes.byteOffset,
        dataBytes.byteLength / 4,
      );

      return {
        data: data,
        shape: header.shape,
        dtype: header.dtype,
      };
    } catch (error) {
      console.error("Error loading matrix:", error);
      throw error;
    }
  }

  // Load all models and data in parallel
  async function loadFaceApiModels() {
    status = "Loading...";

    try {
      // Load everything in parallel using Promise.all
      const [
        ssdResult,
        landmarkResult,
        recognitionResult,
        descriptorMatrix,
        metadataResponse,
      ] = await Promise.all([
        faceapi.nets.ssdMobilenetv1.loadFromUri("./models"),
        faceapi.nets.faceLandmark68Net.loadFromUri("./models"),
        faceapi.nets.faceRecognitionNet.loadFromUri("./models"),
        loadMatrix("analysis/lapd_descriptors.bin"),
        fetch("analysis/lapd_metadata.json"),
      ]);

      // Check metadata response and parse JSON
      if (!metadataResponse.ok) {
        throw new Error(`HTTP error! status: ${metadataResponse.status}`);
      }
      const metadata = await metadataResponse.json();

      // Set global variables
      faceDescriptors = descriptorMatrix;
      combinedData = metadata;
      faceApiLoaded = true;
      databaseLoaded = true;

      // Debug logging
      console.log("Face descriptors shape:", faceDescriptors.shape);
      console.log("Combined data length:", combinedData.length);
      console.log("First few combined data entries:", combinedData.slice(0, 3));

      // status = `All models and database loaded successfully! ${faceDescriptors.shape[0]} face descriptors and ${combinedData.length} metadata records available.`;
      status = "";
    } catch (error) {
      console.error("Error loading models and database:", error);

      status =
        "Error loading face API models and database. Please make sure all model files and database files are available.";
    }
  }

  function handleImageFile(e) {
    const reader = new FileReader();
    reader.onload = function (e) {
      uploadedImageSrc = e.target.result;
    };
    reader.readAsDataURL(e.target.files[0]);
  }

  async function recognize() {
    try {
      const detections = await faceapi
        .detectAllFaces(uploadedImage)
        .withFaceLandmarks()
        .withFaceDescriptors();

      if (detections.length === 0) {
        status =
          "No faces detected in image. Please try a different image with clearly visible faces.";
        matches = [];
        return;
      }

      // Use the largest detected face
      const detection = detections.reduce((largest, current) => {
        const currentArea =
          current.detection.box.width * current.detection.box.height;
        const largestArea =
          largest.detection.box.width * largest.detection.box.height;
        return currentArea > largestArea ? current : largest;
      });

      const uploadDescriptor = detection.descriptor;
      const box = detection.detection.box;

      // Create a canvas to extract the face region
      const canvas = document.createElement("canvas");
      const ctx = canvas.getContext("2d");

      // Set canvas size to match image
      canvas.width = uploadedImage.naturalWidth;
      canvas.height = uploadedImage.naturalHeight;

      // Draw the uploaded image to canvas
      ctx.drawImage(uploadedImage, 0, 0);

      // Extract face region for display
      const faceCanvas = document.createElement("canvas");
      const faceCtx = faceCanvas.getContext("2d");

      const padding = 20;
      const faceWidth = box.width + padding * 2;
      const faceHeight = box.height + padding * 2;
      const faceX = Math.max(0, box.x - padding);
      const faceY = Math.max(0, box.y - padding);

      faceCanvas.width = faceWidth;
      faceCanvas.height = faceHeight;

      faceCtx.drawImage(
        canvas,
        faceX,
        faceY,
        faceWidth,
        faceHeight,
        0,
        0,
        faceWidth,
        faceHeight,
      );

      status = "Comparing with database...";

      // Calculate distances to all faces in database
      const similarities = [];
      const numFaces = faceDescriptors.shape[0];
      const descriptorLength = faceDescriptors.shape[1];

      for (let i = 0; i < numFaces; i++) {
        const startIdx = i * descriptorLength;
        const dbDescriptor = faceDescriptors.data.slice(
          startIdx,
          startIdx + descriptorLength,
        );
        const distance = calculateEuclideanDistance(
          uploadDescriptor,
          dbDescriptor,
        );

        similarities.push({
          index: i,
          distance: distance,
          data: combinedData[i],
        });
      }

      // Sort by distance (lower is more similar) and take top 5
      similarities.sort((a, b) => a.distance - b.distance);
      matches = similarities.slice(0, 9);

      faceCanvas.remove();
      canvas.remove();

      status = ``;
    } catch (error) {
      console.error("Error in face recognition:", error);
      // updateStatus('Error processing uploaded image for face recognition.', 'red');
    }
  }

  function calculateEuclideanDistance(desc1, desc2) {
    let sum = 0;
    for (let i = 0; i < desc1.length; i++) {
      const diff = desc1[i] - desc2[i];
      sum += diff * diff;
    }
    return sum; // No square root needed for sorting
  }
  onMount(() => {
    loadFaceApiModels();
  });
</script>

<main>
  <div class="about-link">
    <a
      href="https://github.com/kylemcdonald/lapd-face-search/"
      target="_blank"
      rel="noopener noreferrer"
      title="About">?</a
    >
  </div>

  <h1>LAPD Face Search</h1>
  <h2>Search over 9,000 LAPD headshots</h2>
  <p>
    Face recognition happens on your device and images are not uploaded.
    <br />Blurry, low-resolution photos will not match.
  </p>

  {#if !loaded}
    <form>
      <label for="image-input" class="loading-label">Loading...</label>
      <input id="image-input" type="file" accept="image/*" hidden="" disabled />
    </form>
  {/if}

  {#if loaded}
    <form>
      <label for="image-input">Select Photo</label>
      <input
        oninput={handleImageFile}
        id="image-input"
        type="file"
        accept="image/*"
        hidden=""
      />
    </form>

    <img
      bind:this={uploadedImage}
      src={uploadedImageSrc}
      onload={recognize}
      alt="User uploaded face"
      class="uploaded-photo"
      style:display={"none"}
    />

    <div class="status">{status}</div>

    {#if uploadedImageSrc}
      <div class="uploaded-face-section">
        <div class="cop uploaded-cop">
          <div class="cop-face">
            <img
              src={uploadedImageSrc}
              alt="Uploaded face"
              class="uploaded-face-img"
            />
          </div>
        </div>
      </div>
    {/if}

    {#if matches.length > 0}
      <hr class="results-divider" />
    {/if}

    <div class="cops">
      {#each matches as cop}
        <div class="cop">
          <Cop cop={cop.data} index={cop.index} />
        </div>
      {/each}
    </div>
  {/if}
</main>

<style>
  main {
    max-width: 960px;
    margin: auto;
    padding: 2rem;
    text-align: center;
    width: 100%;
    position: relative;
  }
  h1 {
    font-size: 4em;
    font-weight: 700;
    margin: 0;
    text-transform: uppercase;
    letter-spacing: 0.1em;
  }
  h2 {
    text-transform: uppercase;
    font-size: 1.5em;
    font-weight: 400;
    margin-bottom: 1em;
    color: rgba(255, 255, 255, 0.7);
  }
  p {
    max-width: 600px;
    margin: 0 auto 2em auto;
    line-height: 1.6;
    color: rgba(255, 255, 255, 0.6);
    font-size: 1.2rem;
  }

  .status {
    min-height: 1.5em;
    margin: 1em 0;
  }
  form {
    margin: 40px 0;
  }
  label {
    font-size: 1.8em;
    background-color: red;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    text-transform: uppercase;
    padding: 0.4em 1em;
    cursor: pointer;
    color: #fff;
    border: 2px solid red;
    transition: all 0.2s ease-in-out;
  }
  label:hover {
    background-color: transparent;
    color: red;
  }
  .uploaded-photo {
    max-width: 250px;
    width: 100%;
    display: inline-block;
    border: 3px solid #333;
    margin-bottom: 2em;
  }
  .cops {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1.5rem;
  }

  .cop {
    background: #111;
    transition: background 0.2s ease;
    border: 2px solid red;
  }

  .cop:hover {
    background: #222;
  }

  .uploaded-face-section {
    margin-bottom: 2rem;
    display: flex;
    justify-content: center;
  }

  .uploaded-cop {
    width: 100%;
    max-width: 300px;
    margin: 0 auto;
    background: #111;
    border: 2px solid red;
    text-transform: uppercase;
    overflow: hidden;
    text-align: center;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
  }
  .cop-info {
    padding: 1em;
  }
  .name {
    font-size: 1.4em;
    margin-bottom: 5px;
    font-weight: bold;
  }
  .serial {
    font-size: 1.2em;
    color: rgba(255, 255, 255, 0.7);
  }

  .uploaded-face-img {
    width: 100%;
    display: block;
  }

  .results-divider {
    border: none;
    height: 2px;
    background: linear-gradient(to right, transparent, red, transparent);
    margin: 2rem 0;
  }

  .results-title {
    margin-top: 2em;
    margin-bottom: 1em;
    border-top: 1px solid #333;
    padding-top: 2em;
  }

  .about-link {
    position: fixed;
    top: 1rem;
    right: 1rem;
    z-index: 1000;
  }

  .about-link a {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 2rem;
    height: 2rem;
    background-color: red;
    color: white;
    text-decoration: none;
    border-radius: 50%;
    font-weight: bold;
    font-size: 1.2rem;
    transition: all 0.2s ease-in-out;
    box-shadow: 0 0 8px rgba(0, 0, 0, 1);
  }

  .about-link a:hover {
    background-color: #cc0000;
    transform: scale(1.1);
  }

  @media (max-width: 768px) {
    main {
      padding: 1rem;
    }
    h1 {
      font-size: 2.2em;
    }
    h2 {
      font-size: 1.1em;
    }
    p {
      font-size: 1.1rem;
    }
    p br {
      display: none;
    }
    label {
      font-size: 1.5em;
    }
    .cops {
      grid-template-columns: 1fr;
    }
  }

  .loading-label {
    background-color: #666 !important;
    border-color: #666 !important;
    color: #999 !important;
    cursor: not-allowed !important;
    animation: pulse 1.5s ease-in-out infinite;
  }

  .loading-label:hover {
    background-color: #666 !important;
    color: #999 !important;
  }

  @keyframes pulse {
    0% {
      opacity: 1;
    }
    50% {
      opacity: 0.5;
    }
    100% {
      opacity: 1;
    }
  }
</style>
