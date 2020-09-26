<template>
  <video
    id="face-detector"
    autoplay
    width="100%"
    style="
      border-radius: 8px;
      box-shadow: 0 3px 5px -1px rgba(0, 0, 0, 0.2),
        0 5px 8px 0 rgba(0, 0, 0, 0.14), 0 1px 14px 0 rgba(0, 0, 0, 0.12) !important;
    "
    @loadeddata="loading = false"
  />
</template>

<script>
export default {
  name: 'FaceDetector',

  data: () => ({
    video: null,
    found: true,
    loading: false,
  }),

  mounted() {
    this.video = document.getElementById('face-detector')
    this.startCamera()
  },
  destroyed() {
    this.stopCamera()
  },

  methods: {
    async startCamera() {
      this.loading = true

      try {
        this.video.srcObject = await navigator.mediaDevices.getUserMedia({
          audio: false,
          video: true,
        })
      } catch (e) {
        this.loading = false
        this.found = false
      }
    },

    stopCamera() {
      this.video.srcObject?.getTracks()?.forEach((track) => track.stop())
    },
  },
}
</script>
