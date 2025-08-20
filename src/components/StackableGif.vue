<script setup>
import { ref, watch } from "vue";

const props = defineProps({
  index: Number,
  active: Boolean,
  countdown: Number, // pass countdown down
  selectedFilter: {
    type: String,
    default: 'none'
  },
  shouldReset: {
    type: Boolean,
    default: false
  }
});

const getFilterStyle = (filter) => {
  const filters = {
    grayscale: 'grayscale(100%)',
    sepia: 'sepia(100%)',
    invert: 'invert(100%)',
    blur: 'blur(4px)',
    saturate: 'saturate(200%)'
  };
  return filters[filter] || '';
};

const emit = defineEmits(["recorded"]);

const videoRef = ref(null);
const gifUrl = ref(null);

let mediaRecorder;
let chunks = [];

// Start recording for 6 seconds
const startRecording = async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    videoRef.value.srcObject = stream;

    mediaRecorder = new MediaRecorder(stream, { mimeType: "video/webm" });
    chunks = [];

    mediaRecorder.ondataavailable = e => chunks.push(e.data);
    mediaRecorder.onstop = () => {
      const blob = new Blob(chunks, { type: "video/webm" });
      gifUrl.value = URL.createObjectURL(blob);

      emit("recorded", { blob, index: props.index });

      // Stop camera stream
      stream.getTracks().forEach(track => track.stop());
    };

    mediaRecorder.start();
    setTimeout(() => mediaRecorder.stop(), 8000); // 8s
  } catch (err) {
    console.error("Camera error:", err);
  }
};

watch(
  () => props.active,
  (newVal) => {
    if (newVal) startRecording();
  }
);

watch(
  () => props.shouldReset,
  (newVal) => {
    if (newVal) {
      gifUrl.value = null;
      if (videoRef.value && videoRef.value.srcObject) {
        videoRef.value.srcObject.getTracks().forEach(track => track.stop());
        videoRef.value.srcObject = null;
      }
    }
  }
);
</script>

<template>
  <div
    class="bg-zinc-800 rounded-2xl shadow-lg flex items-center justify-center overflow-hidden relative w-full aspect-square">
    <video 
      v-if="!gifUrl" 
      ref="videoRef" 
      autoplay 
      muted 
      playsinline 
      :style="{ filter: selectedFilter !== 'none' ? getFilterStyle(selectedFilter) : '' }"
      class="object-cover w-full h-full transform transition-all duration-300"></video>

    <video 
      v-else 
      :src="gifUrl" 
      autoplay 
      muted 
      loop 
      playsinline 
      :style="{ filter: selectedFilter !== 'none' ? getFilterStyle(selectedFilter) : '' }"
      class="object-cover w-full h-full transform transition-all duration-300"></video>

    <!-- Countdown overlay -->
    <div v-if="countdown !== null && countdown > 0"
      class="absolute inset-0 flex items-center justify-center bg-black/50 text-4xl font-bold">
      {{ countdown }}
    </div>

    <span class="absolute bottom-2 right-2 text-xs text-white bg-black/50 px-2 py-1 rounded-lg">
      Stack {{ index + 1 }}
    </span>
  </div>
</template>
