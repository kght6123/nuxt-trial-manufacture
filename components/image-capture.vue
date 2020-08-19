<template>
  <div class="p-4">
    <!-- https://googlechrome.github.io/samples/image-capture/ -->

    <h3 class="text-4xl block">ImageCapture Web APIのサンプル</h3>

    <video autoplay @play="onPlay" ref="video"></video>

    <input type="range" :min="imageWidthMin" :max="imageWidthMax" :step="imageWidthStep" v-model="imageWidth">

    <button ref="takePhotoButton" disabled @click="onTakePhotoButtonClick">Take Photo</button>
    <button @click="onStart">Start</button>
    <button @click="onStop">Stop</button>

    <canvas ref="canvas"></canvas>

    <h4 class="text-3xl block">getSupportedConstraints</h4>
    <ul class="mb-4">
      <li v-for="c in supportedConstraints" :key="c">
        <span class="p-2">
          ● Constraint：{{ c }}
        </span>
      </li>
    </ul>

    <h4 class="text-3xl block">enumerateDevices</h4>
    <ul class="mb-4">
      <li v-for="(d, i) in devices" :key="i">
        <span class="p-2">
        ● DeviceID：{{ d.deviceId }}<br>
        Kind：{{ d.kind }}<br>
        Label：{{ d.label }}
        </span>
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  data() {
    return {
      imageWidthMin: 0,
      imageWidthMax: 0,
      imageWidthStep: 0,
      imageWidthValue: 0,
      imageCapture: null,
      devices: [],
      supportedConstraints: []
    }
  },
  computed: {
    imageWidth: {
      get: function() {
        return this.imageWidthValue
      },
      set: function(newValue) {
        this.imageWidthValue = newValue
      }
    }
  },
  mounted() {
    if (!navigator.mediaDevices) {
      alert('この端末は非対応です。')
      return
    }
    // https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/getSupportedConstraints
    const supportedConstraints = navigator.mediaDevices.getSupportedConstraints()
    for (let constraint in supportedConstraints) {
      if (supportedConstraints.hasOwnProperty(constraint)) {
        console.log(`constraint: ${constraint}`)
        this.supportedConstraints.push(constraint)
      }
    }
    // https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/enumerateDevices
    navigator.mediaDevices.enumerateDevices()
      .then(devices => {
        devices.forEach(device => {
          console.log(`${device.kind} : ${device.label} id = ${device.deviceId}`)
        })
        this.devices = devices
      })
      .catch(error => {
        console.log(`${error.name}:${error.message}`)
        alert(`${error.name}:${error.message}`)
      })

    this.onStart()
  },
  methods: {
    onTakePhotoButtonClick() {
      this.imageCapture.takePhoto({ imageWidth: this.imageWidthValue })
      .then(blob => createImageBitmap(blob))
      .then(imageBitmap => {
        this.drawCanvas(imageBitmap)
        console.log(`Photo size is ${imageBitmap.width}x${imageBitmap.height}`)
      })
      .catch(error => {
        console.log(error)
        alert(`${error.name}:${error.message}`)
      })
    },
    onPlay() {
      this.$refs.takePhotoButton.disabled = false;
    },
    drawCanvas(img) {
      const { canvas } = this.$refs
      canvas.width = getComputedStyle(canvas).width.split('px')[0]
      canvas.height = getComputedStyle(canvas).height.split('px')[0]
      let ratio  = Math.min(canvas.width / img.width, canvas.height / img.height)
      let x = (canvas.width - img.width * ratio) / 2
      let y = (canvas.height - img.height * ratio) / 2
      canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height)
      canvas.getContext('2d').drawImage(img, 0, 0, img.width, img.height,
          x, y, img.width * ratio, img.height * ratio)
    },
    onStop() {
      if (this.$refs.video.srcObject && this.$refs.video.srcObject.getVideoTracks)
        this.$refs.video.srcObject.getVideoTracks()[0].stop()
    },
    onStart() {
      // https://developer.mozilla.org/ja/docs/Web/API/MediaDevices/getUserMedia
      const front = false
      // const constraints = { video: true, audio: false }
      const constraints = { video: { facingMode: (front ? 'user' : 'environment') }, audio: false }
      // { video: { facingMode: (front ? 'user' : { exact: 'environment' }) }, audio: false }
      // { video: { deviceId: { exact: myExactCameraOrBustDeviceId } }, audio: false }
      // { video: true, audio: false }
      // { video: { frameRate: { ideal: 10, max: 15 } }, audio: false }
      navigator.mediaDevices.getUserMedia(constraints)
        .then(mediaStream => {
          console.log('mediaStream', mediaStream, this.$refs)
          this.$refs.video.srcObject = mediaStream

          // Once crbug.com/711524 is fixed, we won't need to wait anymore. This is
          // currently needed because capabilities can only be retrieved after the
          // device starts streaming. This happens after and asynchronously w.r.t.
          // getUserMedia() returns.
          // await this.sleep(1000);

          const track = mediaStream.getVideoTracks()[0]
          this.imageCapture = new ImageCapture(track)

          return this.imageCapture.getPhotoCapabilities()
        })
        .then(photoCapabilities => {
          const settings = this.imageCapture.track.getSettings()

          this.imageWidthMin = photoCapabilities.imageWidth.min
          this.imageWidthMax = photoCapabilities.imageWidth.max
          this.imageWidthStep = photoCapabilities.imageWidth.step

          return this.imageCapture.getPhotoSettings()
        })
        .then(photoSettings => {
          this.imageWidthValue = photoSettings.imageWidth
        })
        .catch(error => {
          console.log('Argh!', error)
          alert(`${error.name}:${error.message}`)
        })
    },
    sleep(ms = 0) {
      return new Promise(r => setTimeout(r, ms));
    }
  }
}
</script>

<style scoped>
video {
  @apply w-full h-64 bg-black;
}

input {
  @apply w-full block;
}

button {
  @apply bg-gray-400 rounded-lg p-4 m-2;
}

canvas {
  @apply block my-4 mx-auto w-full h-64;
}
</style>
