<template>
  <body class="container">
    <video
      ref="video"
      width="720"
      height="560"
      @play="onVideoPlays"
      autoplay
      muted
      playsinline
    />
  </body>
</template>

<script>
import * as faceapi from 'face-api.js'
import { resizeResults } from 'face-api.js'
export default {
  data() {
    return {
      video: null,
      videoErr: null,
      videoStream: null,
      errorMessage: '',
      // canvas: null,
    }
  },
  methods: {
    getOverlayValues(landmarks) {
      const nose = landmarks.getNose()
      const jawline = landmarks.getJawOutline()

      const jawLeft = jawline[0]
      const jawRight = jawline.splice(-1)[0]
      const adjacent = jawRight.x - jawLeft.x
      const opposite = jawRight.y - jawLeft.y
      const jawLength = Math.sqrt(Math.pow(adjacent, 2) + Math.pow(opposite, 2))

      // Both of these work. The chat believes atan2 is better.
      // I don't know why. (It doesn’t break if we divide by zero.)
      // const angle = Math.round(Math.tan(opposite / adjacent) * 100)
      const angle = Math.atan2(opposite, adjacent) * (180 / Math.PI)
      const width = jawLength * 2.2

      return {
        width,
        angle,
        leftOffset: jawLeft.x - width * 0.27,
        topOffset: nose[0].y - width * 0.47,
      }
    },
    getRandomMask(masks) {
      const index = Math.floor(masks.length * Math.random())
      return masks[index]
    },
    async maskify(masks) {
      console.log('Maskify starting...')

      //  Get a list of DOM item containing <img />
      const items = document.querySelectorAll('.grid-item')
      items.forEach(async (item) => {
        const originalImage = item.querySelector('img')
        const scale = originalImage.width / originalImage.naturalWidth

        const handleImage = (oldImage, newImage) => async () => {
          const detection = await faceapi
            .detectSingleFace(newImage, new faceapi.TinyFaceDetectorOptions())
            .withFaceLandmarks(true)

          if (!detection) {
            return
          }

          const overlayValues = getOverlayValues(detection.landmarks)

          const overlay = document.createElement('img')
          overlay.src = getRandomMask(masks)
          overlay.alt = 'mask overlay.'
          overlay.style.cssText = `
        position: absolute;
        left: ${overlayValues.leftOffset * scale}px;
        top: ${overlayValues.topOffset * scale}px;
        width: ${overlayValues.width * scale}px;
        transform: rotate(${overlayValues.angle}deg);
      `

          item.appendChild(overlay)
        }

        // To avoid CORS issues we create a cross-origin-friendly copy of the image.
        const image = new Image()
        image.crossOrigin = 'Anonymous'
        image.addEventListener('load', handleImage(originalImage, image))
        image.src = originalImage.src
      })
    },
    async initCamera() {
      console.log('init camera')
      this.video = this.$refs.video
      this.videoStream = await navigator.mediaDevices.getUserMedia({
        audio: false,
        video: {
          facingMode: 'user',
        },
      })
      this.video.srcObject = this.videoStream
    },
    async loadModels() {
      //  Load models
      try {
        await Promise.all([
          faceapi.nets.tinyFaceDetector.loadFromUri('/models'),
          faceapi.nets.faceLandmark68Net.loadFromUri('/models'),
          faceapi.nets.faceRecognitionNet.loadFromUri('/models'),
          faceapi.nets.faceExpressionNet.loadFromUri('/models'),
        ])
        console.log('models loaded')
      } catch (error) {
        console.error(error)
      }
    },
    drawMask(detections) {
      if (detections.length == 0) return
      const overlayValues = this.getOverlayValues(detections[0].landmarks)

      const scale = 1.13

      const overlay = document.createElement('img')
      overlay.src = '/media/overlay.png'
      overlay.alt = 'mask overlay.'
      overlay.style.cssText = `
        position: absolute;
        left: ${overlayValues.leftOffset * scale}px;
        top: ${overlayValues.topOffset * scale}px;
        width: ${overlayValues.width * scale}px;
        transform: rotate(${overlayValues.angle}deg);
      `
      document.body.appendChild(overlay)

      if (document.body.querySelector('img')) {
        const oldChild = document.body.querySelector('img')
        document.body.replaceChild(overlay, oldChild)
      }
    },
    onVideoPlays() {
      console.log('video starts playing')

      const canvas = faceapi.createCanvasFromMedia(this.video)
      document.body.append(canvas)
      const displaySize = { width: this.video.width, height: this.video.height }
      faceapi.matchDimensions(canvas, displaySize)

      setInterval(async () => {
        const detections = await faceapi
          .detectAllFaces(
            this.video,
            new faceapi.TinyFaceDetectorOptions({ inputSize: 160 })
          )
          .withFaceLandmarks()

        if (!detections) return

        // const resizedDetection = faceapi.resizeResults(detections, displaySize)
        // canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height)
        // faceapi.draw.drawDetections(canvas, resizedDetection)
        // faceapi.draw.drawFaceLandmarks(canvas, resizedDetection)

        this.drawMask(detections)
      }, 10)
    },
  },
  async mounted() {
    //  Load models
    await this.loadModels()

    //  Init camera
    await this.initCamera()
  },
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  /* display: flex;
  justify-content: center;
  align-items: center; */
}

canvas {
  position: absolute;
}
</style>
