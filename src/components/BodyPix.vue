<template>
  <div id="bodypix">
    <div id="loading" ref="loading" v-show="loading" :style="{ height: windowHeight + 'px' }">
      <div class="sk-spinner sk-spinner-pulse"></div>
    </div>
    <div id="main" :style="{ height: windowHeight + 'px' }" v-show="!loading">
      <video id="video" ref="video" playsinline style="
          -moz-transform: scaleX(-1);
          -o-transform: scaleX(-1);
          -webkit-transform: scaleX(-1);
          transform: scaleX(-1);
          display: none;
          "></video>
      <canvas id="output" ref="output"/>
      <div id="menu">
        <ul>
          <li id="flip-camera" v-on:click="changeCamera"><i class="material-icons">flip_camera_ios</i></li>
          <li id="pixel"><i class="material-icons">gradient</i></li>
          <li id="mask"><i class="material-icons">android</i></li>
          <li id="pose"><i class="material-icons">emoji_people</i></li>
        </ul>
      </div>
    </div>
    <div id="footer">
      <img src="../assets/qr.png">
      <div>
        <ul>
          <li><a href="https://github.com/dforest/bodypix-sample" target="_blank">Github</a></li>
          <li><a href="https://twitter.com/d_forest" target="_blank">Twitter</a></li>
          <li><a href="https://note.com/d_forest" target="_blank">Blog</a></li>
        </ul>
      </div>
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
      cameraIndex: 0,
      changingCamera: false,
      loading: true,
      windowHeight: 0,
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
    async changeCamera () {
      this.changingCamera = true
      this.cameraIndex++
      if (this.cameraIndex >= this.cameras.length) {
        this.cameraIndex = 0
      }
      await this.loadVideo()
    },
    async loadBodyPix() {
      this.net = await bodyPix.load()
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

      this.stopExistingVideoCapture()

      const stream = await navigator.mediaDevices.getUserMedia({
        'audio': false,
        'video': {
          facingMode: this.isMobile() ? 'user' : null,
          deviceId: this.cameras[this.cameraIndex].deviceId
        }
      })
      video.srcObject = stream;
      return new Promise((resolve) => {
        video.onloadedmetadata = () => {
          let width = video.videoWidth
          let height = video.videoHeight
          if (this.isMobile()) {
            width = width/2
            height = height/2
          }
          video.width = width
          video.height = height
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
        try {
          if (self.changingCamera) {
            await self.loadBodyPix()
            self.changingCamera = false
          }
          const ctx = canvas.getContext('2d')
          const segmentation = await self.estimatePartSegmentation()
          const coloredPart = bodyPix.toColoredPartMask(segmentation, self.colorScale)
          if (coloredPart != null) {
            bodyPix.drawPixelatedMask(
              canvas,
              self.video,
              coloredPart,
              1.0,
              0,
              true,
              10.0
            )
          } else {
            ctx.clearRect(0, 0, canvas.width, canvas.height)
          }
        } catch (e) {
          window.console.error("Retrying...", e)
        } finally {
          requestAnimationFrame(updateFrame)
        }
      }
      updateFrame()
    },
    async estimatePartSegmentation() {
      return await this.net.segmentMultiPersonParts(this.video, {
        flipHorizontal: true,
        internalResolution: 'medium',
        segmentationThreshold: 0.7,
        maxDetections: 5,
        scoreThreshold: 0.3,
        nmsRadius: 20,
        numKeypointForMatching: 17,
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
    this.windowHeight = window.innerHeight
    this.loading = true

    await this.loadBodyPix()
    this.cameras = await this.getVideoInputs()
    await this.loadVideo()

    this.loading = false
    this.doBodyPix()
  }
}
</script>

<style lang="scss">
  #bodypix {
    width: 100%;
    overflow: hidden;
  }

  #loading, #main {
    display: flex;
    justify-content: center;
    align-items: center;
  }

  #menu {
    position: absolute;
    text-align: center;
    bottom: 6em;
    left: 0;
    right: 0;

    li {
      cursor: pointer;
    }

    #flip-camera {
      margin-right: 5em;
    }

    .material-icons{
      font-size: 60px;
    }
  }

  #footer {
    margin-top: 6em;
    text-align: center;
  }

  ul {
    display: inline-block;
    margin: 0;
    padding: 0;

    li {
      float: left;
      list-style: none;
      margin: 1em;
    }
  }

  a {
    color: #0366d6;
    text-decoration: none;
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
