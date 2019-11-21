<template>
  <div id="bodypix">
    <div id="loading" ref="loading" v-show="loading">
      <div class="sk-spinner sk-spinner-pulse"></div>
    </div>
    <div id="main" v-show="!loading">
      <video id="video" ref="video" playsinline style="
          -moz-transform: scaleX(-1);
          -o-transform: scaleX(-1);
          -webkit-transform: scaleX(-1);
          transform: scaleX(-1);
          display: none;
          "></video>
      <canvas id="output" ref="output"/>
    </div>
  </div>
</template>

<script>
import * as bodyPix from '@tensorflow-models/body-pix';

export default {
  name: 'BodyPix',
  data () {
    return {
      net: {},
      video: {},
      cameras: [],
      loading: true,
      width: window.innerWidth,
      height: window.innerHeight,
      colorScale: [
        [110, 64, 170], [143, 61, 178], [178, 60, 178], [210, 62, 167],
        [238, 67, 149], [255, 78, 125], [255, 94, 99],  [255, 115, 75],
        [255, 140, 56], [239, 167, 47], [217, 194, 49], [194, 219, 64],
        [175, 240, 91], [135, 245, 87], [96, 247, 96],  [64, 243, 115],
        [40, 234, 141], [28, 219, 169], [26, 199, 194], [33, 176, 213],
        [47, 150, 224], [65, 125, 224], [84, 101, 214], [99, 81, 195]
      ]
    }
  },
  methods: {
    async loadBodyPix() {
      this.loading = true;
      this.net = await bodyPix.load()
      this.loading = false;
    },
    async loadVideo() {
      this.video = await this.setupCamera()
      this.video.play()
    },
    async setupCamera() {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        throw new Error('Browser API navigator.mediaDevices.getUserMedia not available')
      }

      const video = this.$refs.video
      video.width = this.width
      video.height = this.height

      this.stopExistingVideoCapture()

      const stream = await navigator.mediaDevices.getUserMedia({
        'audio': false,
        'video': {
          facingMode: this.isMobile() ? 'user' : null,
          deviceId: this.cameras[0].deviceId
        }
      })
      video.srcObject = stream;
      return new Promise((resolve) => {
        video.onloadedmetadata = () => {
          resolve(video)
        }
      })
    },
    async getVideoInputs() {
      if (!navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices) {
        window.console.log('enumerateDevices() not supported.')
        return []
      }

      const devices = await navigator.mediaDevices.enumerateDevices()
      const videoDevices = devices.filter(device => device.kind === 'videoinput')
      return videoDevices;
    },
    stopExistingVideoCapture() {
      if (this.video &&  this.video.srcObject) {
        this.video.srcObject.getTracks().forEach(track => {
          track.stop()
        })
        this.video.srcObject = null
      }
    },
    doBodyPix() {
      const canvas = this.$refs.output

      const self = this
      async function updateFrame() {
        const segmentation = await self.estimatePartSegmentation()
        const coloredPart = bodyPix.toColoredPartMask(segmentation, self.colorScale)
        window.console.log(segmentation)
        window.console.log(coloredPart)
        if (coloredPart != null) {
          bodyPix.drawPixelatedMask(
            canvas,
            self.video,
            coloredPart,
            1.0,
            0,
            true,
            20.0
          )
        }
        requestAnimationFrame(updateFrame)
      }

      updateFrame()
    },
    async estimatePartSegmentation() {
      return await this.net.segmentMultiPersonParts(this.video, {
        flipHorizontal: true,
        internalResolution: 'low',
        segmentationThreshold: 0.7,
        maxDetections: 10,
        scoreThreshold: 0.2,
        nmsRadius: 20,
        minKeypointScore: 0.3,
        refineSteps: 10
      })
    },
    isAndroid() {
      return /Android/i.test(navigator.userAgent)
    },
    isiOS() {
      return /iPhone|iPad|iPod/i.test(navigator.userAgent)
    },
    isMobile() {
      return this.isAndroid() || this.isiOS()
    }
  },
  async mounted() {
    const loading = this.$refs.loading
    loading.width = this.width
    loading.height = this.height

    await this.loadBodyPix()

    this.cameras = await this.getVideoInputs()

    await this.loadVideo()
    this.doBodyPix()
  }
}
</script>

<style>
  #bodypix {
    width: 100%;
    overflow: hidden;
  }

  #loading {
    display: flex;
  }

  /*
  *  The following loading spinner CSS is from SpinKit project
  *  https://github.com/tobiasahlin/SpinKit
  */
  .sk-spinner-pulse {
    width: 60px;
    height: 60px;
    margin: 40px auto;
    float: left;
    background-color: #333;
    border-radius: 100%;
    -webkit-animation: sk-pulseScaleOut 1s infinite ease-in-out;
    animation: sk-pulseScaleOut 1s infinite ease-in-out;
  }
  @-webkit-keyframes sk-pulseScaleOut {
    0% {
      -webkit-transform: scale(0);
      transform: scale(0);
    }
    100% {
      -webkit-transform: scale(1.0);
      transform: scale(1.0);
      opacity: 0;
    }
  }
  @keyframes sk-pulseScaleOut {
    0% {
      -webkit-transform: scale(0);
      transform: scale(0);
    }
    100% {
      -webkit-transform: scale(1.0);
      transform: scale(1.0);
      opacity: 0;
    }
  }
</style>
