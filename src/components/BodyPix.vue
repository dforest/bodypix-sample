<template>
  <div id="bodypix">
    <div id="loading" v-show="loading">
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
      <canvas id="output"/>
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
      height: window.innerHeight
    }
  },
  methods: {
    async loadBodyPix() {
      this.loading = true;
      this.net = bodyPix.load()
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
    await this.loadBodyPix()

    this.cameras = await this.getVideoInputs()

    await this.loadVideo()
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
