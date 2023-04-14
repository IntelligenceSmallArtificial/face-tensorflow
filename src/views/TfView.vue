<script setup>
import { ref, onMounted } from 'vue'
import * as bodySegmentation from '@tensorflow-models/body-segmentation'

const video = ref()
const canvas = ref()

const ctx = ref(null)

const loadModel = async () => {
  const model = bodySegmentation.SupportedModels.MediaPipeSelfieSegmentation
  const segmenterConfig = {
    runtime: 'mediapipe', // or 'tfjs'
    solutionPath: 'https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation',
    modelType: 'general'
  }
  const segmenter = await bodySegmentation.createSegmenter(model, segmenterConfig)

  const render = async () => {
    if (video.value.readyState === video.value.HAVE_ENOUGH_DATA) {
      const _canvas = document.createElement('canvas')
      _canvas.height = 300
      _canvas.width = 300
      const _ctx = _canvas.getContext('2d')
      _ctx.drawImage(video.value, 0, 0, 300, 300)
      const originImageData = _ctx.getImageData(0, 0, 300, 300)
      const people = await segmenter.segmentPeople(originImageData)
      const foregroundColor = { r: 0, g: 0, b: 0, a: 0 }
      const backgroundColor = { r: 255, g: 255, b: 255, a: 1 }
      const backgroundMask = await bodySegmentation.toBinaryMask(
        people,
        foregroundColor,
        backgroundColor,
        false,
        0.5
      )
      const handledUint8Array = new Uint8ClampedArray(originImageData.data.length)
      const pixelLength = handledUint8Array.length >> 2
      for (let i = 0; i < pixelLength; i++) {
        const indexStart = i << 2
        if (backgroundMask.data[indexStart + 3]) {
          handledUint8Array[indexStart] = backgroundMask.data[indexStart]
          handledUint8Array[indexStart + 1] = backgroundMask.data[indexStart + 1]
          handledUint8Array[indexStart + 2] = backgroundMask.data[indexStart + 2]
          handledUint8Array[indexStart + 3] = backgroundMask.data[indexStart + 3]
        } else {
          handledUint8Array[indexStart] = originImageData.data[indexStart]
          handledUint8Array[indexStart + 1] = originImageData.data[indexStart + 1]
          handledUint8Array[indexStart + 2] = originImageData.data[indexStart + 2]
          handledUint8Array[indexStart + 3] = originImageData.data[indexStart + 3]
        }
      }
      const handledImageData = new ImageData(handledUint8Array, 300, 300)
      ctx.value.putImageData(handledImageData, 0, 0)
    }
    requestAnimationFrame(render)
  }
  await render()
}

const getMedia = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: false
  })
  video.value.srcObject = stream
  video.value.onloadedmetadata = function () {
    video.value.play()
  }
}

const init = async () => {
  try {
    ctx.value = canvas.value.getContext('2d')
    await getMedia()
    await loadModel()
  } catch (e) {
    console.log(e)
  }
}

onMounted(init)
</script>

<template>
  <div class="tf">
    <video class="video" ref="video"></video>
    <canvas width="300" height="300" ref="canvas" />
  </div>
</template>

<style>
.tf {
  display: flex;
}
.video {
  width: 300px;
  height: 300px;
  margin-right: 20px;
  object-fit: cover;
  display: none;
}
</style>
