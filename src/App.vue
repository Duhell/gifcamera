<script setup>
import { ref } from "vue";
import StackableGif from "./components/StackableGif.vue";

const stacks = ref(1);
const orientation = ref("vertical");
const isRecording = ref(false);
const activeIndex = ref(-1);
const countdown = ref(null);
const selectedFilter = ref("none");
const isDownloading = ref(false);
const shouldReset = ref(false);

// store blobs
const recordedBlobs = ref([]);

const startRecording = async () => {
  isRecording.value = true;
  activeIndex.value = -1;
  recordedBlobs.value = [];
  shouldReset.value = true;
  // Reset the shouldReset flag after a brief delay
  setTimeout(() => {
    shouldReset.value = false;
  }, 100);
  await runStacks();
};

const runStacks = async () => {
  for (let i = 0; i < stacks.value; i++) {
    countdown.value = 5;
    while (countdown.value > 0) {
      await new Promise(r => setTimeout(r, 1000));
      countdown.value--;
    }
    activeIndex.value = i;
    await new Promise(r => setTimeout(r, 8000)); // wait for recording
  }
  activeIndex.value = -1;
  countdown.value = null;
  isRecording.value = false;
};

const handleRecorded = ({ blob, index }) => {
  recordedBlobs.value[index] = blob;
};

// ---- Download stitched video via Canvas ----
const downloadAll = async () => {
  isDownloading.value = true;
  try {
    const videos = await Promise.all(
      recordedBlobs.value.map(blob => {
        return new Promise((resolve) => {
          const vid = document.createElement("video");
          vid.src = URL.createObjectURL(blob);
          vid.muted = true;
          vid.loop = true;
          vid.playsInLine = true;
          vid.onloadeddata = () => resolve(vid);
        });
      })
    );

    const size = 300; // each stack square size
    const spacing = 0; // spacing between stacks
    let canvas, ctx;

    const canvasSizes = {
      vertical: {
        height: (size * stacks.value) + (spacing * (stacks.value - 1)),
        width: size
      },
      horizontal: {
        height: size,
        width: (size * stacks.value) + (spacing * (stacks.value - 1))
      }
    }

    if (orientation.value === "vertical") {
      canvas = document.createElement("canvas");
      canvas.width = canvasSizes.vertical.width;
      canvas.height = canvasSizes.vertical.height;
    } else {
      canvas = document.createElement("canvas");
      canvas.width = canvasSizes.horizontal.width;
      canvas.height = canvasSizes.horizontal.height;
    }

    ctx = canvas.getContext("2d");
    ctx.fillStyle = "#18181b"; // bg color matching the app's theme
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const stream = canvas.captureStream(30); // 30fps
    const recorder = new MediaRecorder(stream, { mimeType: "video/webm" });
    const chunks = [];

    recorder.ondataavailable = (e) => chunks.push(e.data);
    recorder.onstop = () => {
      const blob = new Blob(chunks, { type: "video/webm" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "stacked-output.webm";
      a.click();
      URL.revokeObjectURL(url);
    };

    recorder.start();

    // draw videos onto canvas repeatedly
    const draw = () => {
      ctx.fillStyle = "#18181b";
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      videos.forEach((vid, i) => {
        // Create a temporary canvas for each video to apply rounded corners
        const tempCanvas = document.createElement("canvas");
        tempCanvas.width = size;
        tempCanvas.height = size;
        const tempCtx = tempCanvas.getContext("2d");

        // Draw rounded rectangle
        tempCtx.beginPath();
        tempCtx.roundRect(0, 0, size, size, 4);
        tempCtx.clip();

        // Draw the video frame
        tempCtx.drawImage(vid, 0, 0, size, size);

        // Apply selected filter
        if (selectedFilter.value !== "none") {
          tempCtx.filter = {
            grayscale: "grayscale(100%)",
            sepia: "sepia(100%)",
            invert: "invert(100%)",
            blur: "blur(4px)",
            saturate: "saturate(200%)"
          }[selectedFilter.value];

          // Redraw with filter
          const filteredFrame = tempCtx.getImageData(0, 0, size, size);
          tempCtx.putImageData(filteredFrame, 0, 0);
        }

        // Calculate position with spacing
        const pos = i * (size + spacing);

        // Draw to main canvas
        if (orientation.value === "vertical") {
          ctx.drawImage(tempCanvas, 0, pos, size, size);
        } else {
          ctx.drawImage(tempCanvas, pos, 0, size, size);
        }
      });

      requestAnimationFrame(draw);
    };
    draw();

    // play all videos together
    videos.forEach(v => v.play());

    // stop recording after 6s (same duration)
    setTimeout(() => {
      recorder.stop();
      isDownloading.value = false;
    }, 15000);
  } catch (error) {
    console.error("Error creating video:", error);
    isDownloading.value = false;
  }
};
</script>

<template>
  <div class="w-full bg-zinc-900 min-h-screen p-6 flex flex-col text-white">
    <h1 class="text-3xl font-bold mb-4">GIF Camera</h1>

    <!-- Controls -->
    <div class="flex gap-4 mb-6">
      <div>
        <label class="block text-sm mb-1">Select Stack</label>
        <select v-model="stacks" class="bg-zinc-800 text-white px-3 py-2 rounded-md">
          <option v-for="n in 4" :key="n" :value="n">{{ n }} stack</option>
        </select>
      </div>
      <div>
        <label class="block text-sm mb-1">Orientation</label>
        <select v-model="orientation" class="bg-zinc-800 text-white px-3 py-2 rounded-md">
          <option value="vertical">Vertical</option>
          <option value="horizontal">Horizontal</option>
        </select>
      </div>
      <div>
        <label class="block text-sm mb-1">Filter</label>
        <select v-model="selectedFilter" class="bg-zinc-800 text-white px-3 py-2 rounded-md">
          <option value="none">None</option>
          <option value="grayscale">Black & White</option>
          <option value="sepia">Sepia</option>
          <option value="invert">Invert</option>
          <option value="blur">Blur</option>
          <option value="saturate">Saturate</option>
        </select>
      </div>
      <div>
        <label class="block text-sm mb-1">Actions</label>
        <div class="flex gap-3">
          <button @click="startRecording" :disabled="isRecording"
            class="flex-1 bg-zinc-800 text-white hover:bg-zinc-700 disabled:opacity-50 px-4 py-2 rounded-md text-sm font-medium">
            {{ isRecording ? "Recording..." : "Start" }}
          </button>

          <button v-if="recordedBlobs.length > 0 && !isRecording" @click="downloadAll" :disabled="isDownloading"
            class="flex-1 bg-zinc-800 text-white hover:bg-zinc-700 disabled:opacity-50 px-4 py-2 rounded-md text-sm font-medium flex items-center justify-center gap-2">
            <span v-if="isDownloading" class="inline-block animate-spin">↻</span>
            {{ isDownloading ? "Processing..." : "Download" }}
          </button>
        </div>
      </div>

    </div>
    <div class="flex-1 flex justify-center items-center">
      <div :class="[
        'grid gap-4 w-full max-w-5xl',
        orientation === 'vertical'
          ? `grid-rows-${stacks}`
          : `grid-cols-${stacks}`
      ]">
        <StackableGif v-for="i in stacks" :key="i" :index="i - 1" :active="activeIndex === i - 1"
          :countdown="activeIndex === i - 1 ? countdown : null" :selected-filter="selectedFilter"
          :should-reset="shouldReset" @recorded="handleRecorded" class="aspect-square w-full max-w-sm mx-auto" />
      </div>
    </div>
  </div>
</template>
