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
          <li v-on:click="changeCamera"><i class="material-icons">flip_camera_ios</i></li>
          <li v-on:click="changeMode('pixel')"><i class="material-icons">gradient</i></li>
          <li v-on:click="changeMode('mask')"><i class="material-icons">android</i></li>
          <li v-on:click="changeMode('pose')"><i class="material-icons">emoji_people</i></li>
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
import * as posenet from '@tensorflow-models/posenet';
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
      mode: "pixel",
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
      if (this.changingCamera) {
        return
      }

      this.changingCamera = true
      this.cameraIndex++
      if (this.cameraIndex >= this.cameras.length) {
        this.cameraIndex = 0
      }
      await this.loadVideo()
    },
    changeMode (mode) {
      this.mode = mode
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
          deviceId: this.cameras[this.cameraIndex].deviceId
        }
      })
      window.console.log("using camera: " + this.cameras[this.cameraIndex].label)

      video.srcObject = stream;
      return new Promise((resolve) => {
        video.onloadedmetadata = () => {
          let width = video.videoWidth
          let height = video.videoHeight
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

          switch(self.mode) {
            case 'pixel': {
              const partSegmentation = await self.estimatePartSegmentation()
              self.drawPixelatedMask(partSegmentation, canvas, self.video)
              break
            }
            case 'mask': {
              const segmentation = await self.estimateSegmentation()
              self.drawMask(segmentation, canvas, self.video, {r: 61, g: 220, b: 132, a: 255})
              break
            }
            case 'pose': {
              const segmentation = await self.estimateSegmentation()
              self.drawMask(segmentation, canvas, self.video, {r: 0, g: 0, b: 0, a: 0})
              self.drawPose(segmentation, canvas)
              break
            }
          }

        } catch (e) {
          window.console.log("Retrying...")
        } finally {
          requestAnimationFrame(updateFrame)
        }
      }
      updateFrame()
    },
    drawPixelatedMask(segmentation, canvas, video) {
      const ctx = canvas.getContext('2d')
      const coloredPart = bodyPix.toColoredPartMask(segmentation, self.colorScale)
      if (coloredPart != null) {
        bodyPix.drawPixelatedMask(
          canvas,
          video,
          coloredPart,
          1.0,
          0,
          true,
          10.0
        )
      } else {
        ctx.clearRect(0, 0, canvas.width, canvas.height)
      }
    },
    drawMask(segmentation, canvas, video, color) {
      const mask = bodyPix.toMask(
        segmentation,
        color,
        {r: 0, g: 0, b: 0, a: 0 },
        false
      )
      bodyPix.drawMask(
        canvas,
        video,
        mask,
        1.0,
        0,
        true
      )
    },
    drawPose(segmentation, canvas) {
      const ctx = canvas.getContext('2d')
      const scale = 1
      const size = 10
      const color = 'aqua'
      const minScore = 0.1

      segmentation.forEach(personSegmentation => {
        let pose = personSegmentation.pose;
        pose = bodyPix.flipPoseHorizontal(pose, personSegmentation.width)
        for (let i = 0; i < pose.keypoints.length; i++) {
          const keypoint = pose.keypoints[i]
          if(keypoint.score < minScore) {
            continue
          }

          const {y, x} = keypoint.position;
          this.drawPoint(ctx, y * scale, x * scale, size, color)
        }

        const adjacentKeyPoints = posenet.getAdjacentKeyPoints(pose.keypoints, minScore)
        adjacentKeyPoints.forEach(keypoints => {
          this.drawSegment(
            this.toTuple(keypoints[0].position),
            this.toTuple(keypoints[1].position),
            scale,
            size/2,
            color,
            ctx
          )
        })
      })
    },
    drawPoint(ctx, y, x, r, color) {
      ctx.beginPath();
      ctx.arc(x, y, r, 0, 2 * Math.PI);
      ctx.fillStyle = color;
      ctx.fill();
    },
    drawSegment([ay, ax], [by, bx], scale, size, color, ctx) {
      ctx.beginPath()
      ctx.moveTo(ax * scale, ay * scale)
      ctx.lineTo(bx * scale, by * scale)
      ctx.lineWidth = 5
      ctx.strokeStyle = color
      ctx.stroke()
    },
    toTuple({y, x}) {
      return [y, x]
    },
    async estimateSegmentation() {
      return await this.net.segmentMultiPerson(this.video, {
        internalResolution: 'medium',
        segmentationThreshold: 0.7,
        maxDetections: 5,
        scoreThreshold: 0.2,
        nmsRadius: 20,
        numKeypointForMatching: 17,
        refineSteps: 10
      })
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
    bottom: 3em;
    left: 0;
    right: 0;

    ul {
      border-radius: 2px;
      background-color: rgba(255, 255, 255, .3)
    }

    li {
      cursor: pointer;
      user-select: none;
      &:first-child {
        margin-right: 2em;
      }
    }

    .material-icons{
      font-size: 30px;
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
