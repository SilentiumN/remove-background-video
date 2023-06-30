<script setup lang="ts">
import {ref, onMounted, computed, watch} from 'vue';
import { SelfieSegmentation } from "@mediapipe/selfie_segmentation";

const inputVideoRef = ref(null);
const canvasRef = ref(null);
const contextRef = ref(null);
const mediaStream = ref(null)
const test = ref(null)

onMounted(() => {
  contextRef.value = canvasRef.value.getContext("2d");
  const constraints = {
    video: { width: { min: 1280 }, height: { min: 720 } },
  };
  navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
    inputVideoRef.value.srcObject = stream;
    sendToMediaPipe();
  });

  const selfieSegmentation = new SelfieSegmentation({
    locateFile: (file) =>
        `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`,
  });

  selfieSegmentation.setOptions({
    modelSelection: 1,
    selfieMode: false,
  });

  selfieSegmentation.onResults(onResults);

  const sendToMediaPipe = async () => {
    if (!inputVideoRef.value.videoWidth) {
      console.log(inputVideoRef.value.videoWidth);
      requestAnimationFrame(sendToMediaPipe);
    } else {
      await selfieSegmentation.send({ image: inputVideoRef.value });
      requestAnimationFrame(sendToMediaPipe);
    }
  };
})

const onResults = (results) => {

  contextRef.value.save();
  contextRef.value.clearRect(
      0,
      0,
      canvasRef.value.width,
      canvasRef.value.height
  );
  contextRef.value.filter = "blur(0)"

  contextRef.value.drawImage(
      results.segmentationMask,
      0,
      0,
      canvasRef.value.width,
      canvasRef.value.height
  );
  // Only overwrite existing pixels.
  contextRef.value.globalCompositeOperation = 'source-in';
  contextRef.value.drawImage(
      results.image, 0, 0, canvasRef.value.width, canvasRef.value.height);

  // Only overwrite missing pixels.
  contextRef.value.globalCompositeOperation = 'destination-atop';
  contextRef.value.filter = "blur(8px)"

  contextRef.value.drawImage(
      results.image, 0, 0, canvasRef.value.width, canvasRef.value.height);
  contextRef.value.restore();
};

watch(canvasRef, (newVal, oldVal) => {
  console.log(newVal, oldVal, "newVal", "oldVal")
  if (newVal && !oldVal) {
    mediaStream.value = canvasRef.value.captureStream(25)
    console.log(mediaStream.value)
    test.value.srcObject = mediaStream.value
  }
})

</script>

<template>
  <div class="App">
    <video autoPlay ref="inputVideoRef" />
    <canvas ref="canvasRef" width="640" height="360" />
    <video ref="test" autoplay/>
  </div>
</template>

<style scoped>
video {
  width: 640px;
  height: 360px;
}

canvas {
  display: none;
}
</style>
