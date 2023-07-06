<script setup lang="ts">
import {ref, onMounted, computed, watch} from 'vue';
import {
  generateDefaultGoogleMeetSegmentationParams,
  generateGoogleMeetSegmentationDefaultConfig,
  GoogleMeetSegmentationWorkerManager,
} from "@dannadori/googlemeet-segmentation-worker-js";

const inputVideoRef = ref(null);
const canvasRef = ref(null);
const contextRef = ref(null);
const mediaStream = ref(null)
const test = ref(null)

const config = (() => {
  const c = generateGoogleMeetSegmentationDefaultConfig();
  c.modelKey = "160x96"; // option: "160x96", "128x128", "256x144", "256x256"
  c.processOnLocal = true; // if you want to run prediction on webworker, set false.
  c.useSimd = true; // if you want to use simd, set true.
  return c;
})();

const params = ((width, height) => {
  const p = generateDefaultGoogleMeetSegmentationParams();
  p.processWidth = width; // processing image width,  should be under 1000.
  p.processHeight = height; // processing image height, should be under 1000.
  return p;
})

onMounted(async () => {
  const sendToMediaPipe = async () => {
    if (!input.videoWidth) {
      requestAnimationFrame(sendToMediaPipe);
    } else {
      onProcced()
      requestAnimationFrame(sendToMediaPipe);
    }
  };

  const manager = new GoogleMeetSegmentationWorkerManager();
  const input = document.createElement("video") as HTMLVideoElement;
  const output = document.createElement("canvas") as HTMLCanvasElement;
  const tmpCanvas = document.createElement("canvas") as HTMLCanvasElement;
  await manager.init(config)

  const constraints = {
    video: { width: { ideal: 640 }, height: { ideal: 360 } },
  };
  navigator.mediaDevices.getUserMedia(constraints).then((stream) => {
    input.autoplay = true;
    input.playsInline = true;
    input.srcObject = stream;
    sendToMediaPipe();
  });



  const onProcced = () => {
    tmpCanvas.width = input.videoWidth;
    tmpCanvas.height = input.videoHeight;
    tmpCanvas
        .getContext("2d")!
        .drawImage(input, 0, 0, tmpCanvas.width, tmpCanvas.height);
    const paramsWithNewWidth = params(tmpCanvas.width, tmpCanvas.height)
    manager.predict(paramsWithNewWidth, tmpCanvas).then((prediction) => {
      if (!prediction) {
        return;
      }
      output.width = paramsWithNewWidth.processWidth;
      output.height = paramsWithNewWidth.processHeight;
      const mask = new ImageData(
          prediction,
          paramsWithNewWidth.processWidth,
          paramsWithNewWidth.processHeight
      );
      const outputCtx = output.getContext("2d")!;

      outputCtx.save();
      outputCtx.clearRect(
          0,
          0,
          output.width,
          output.height
      );

      outputCtx.putImageData(mask, 0, 0);
      outputCtx.globalCompositeOperation = "source-atop";
      outputCtx.drawImage(tmpCanvas, 0, 0, output.width, output.height);

      outputCtx.globalCompositeOperation = 'source-in';
      outputCtx.drawImage(
          tmpCanvas, 0, 0, output.width, output.height);

      // Only overwrite missing pixels.
      outputCtx.globalCompositeOperation = 'destination-atop';
      outputCtx.filter = "blur(8px)"

      outputCtx.drawImage(
          tmpCanvas, 0, 0, output.width, output.height);
      outputCtx.restore();
    });
  }

  mediaStream.value = output.captureStream()
  test.value.srcObject = mediaStream.value
})



</script>

<template>
  <div class="App">
    <video ref="test" playsinline autoplay muted/>
  </div>
</template>

<style scoped>
video {
  width: 640px;
  height: 360px;
}
</style>
