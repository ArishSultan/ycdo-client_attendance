<template>
  <div>
    <v-app-bar fixed app color="primary" dark dense>
      <v-toolbar-title>YCDO Attendance System</v-toolbar-title>
      <v-btn @click="createNewSheet">Create New Sheet</v-btn>
    </v-app-bar>
    <v-main>
      <v-container fluid>
        <div class="main-container">
          <v-card outlined elevation="4" style="border-radius: 8px">
            <v-data-table
              dense
              sort-by="arrivedAt"
              sort-desc
              :headers="[
                { text: 'Id', value: 'user._id' },
                { text: 'Name', value: 'user.name' },
                { text: 'Username', value: 'user.username' },
                {
                  text: 'Status',
                  value: 'isPresent',
                  align: 'center',
                  filterable: false,
                  sortable: false,
                },
                { text: 'Arrival Time', value: 'arrivedAt' },
              ]"
              :items="attendance.users"
            >
              <template #item.isPresent="{ item }">
                <span
                  v-if="item.isPresent"
                  class="status-card status-card--active"
                  >Present</span
                >
                <span v-else class="status-card status-card--in-active"
                  >Absent</span
                >
              </template>

              <template #item.arrivedAt="{ item }">
                {{ item.arrivedAt | time }}
              </template>
            </v-data-table>
          </v-card>

          <div class="main-container__input">
            <!--            <loading @started="loadFaces" @face="matchFace" />-->
            <!--            <loading @started="loadFaces" @face="matchFace" />-->

            <v-card outlined style="border-radius: 8px">
              <digital-persona-fingerprint @finger="sendFinger" />
            </v-card>
            <v-card outlined elevation="4" style="border-radius: 8px"></v-card>
          </div>
        </div>
      <!--        <div style="display: flex; height: calc(100vh - 88px);">-->
      <!--          <v-card style="flex: 4; margin-right: 15px;">-->
      <!--            <v-data-table-->
      <!--              dense-->
      <!--              :headers="[-->
      <!--                { text: 'Id', value: 'user._id' },-->
      <!--                { text: 'Name', value: 'user.name' },-->
      <!--                { text: 'Username', value: 'user.username' },-->
      <!--                {-->
      <!--                  text: 'Status',-->
      <!--                  value: 'isPresent',-->
      <!--                  align: 'center',-->
      <!--                  filterable: false,-->
      <!--                  sortable: false,-->
      <!--                },-->
      <!--                { text: 'Arrival Time', value: 'arrivedAt' },-->
      <!--              ]"-->
      <!--              -->
      <!--            >-->

      <!--            </v-data-table>-->
      <!--          </v-card>-->

      <!--          <div style="flex: 2; display: flex; flex-direction: column;">-->
      <!--            <loading @started="loadFaces" @face="matchFace" />-->

      <!--            <v-card elevation="5" style="height: 300px; margin-bottom: 10px; padding: 20px">-->
      <!--              <digital-persona-fingerprint @finger="sendFinger" />-->
      <!--            </v-card>-->
      <!--            <v-card style="padding: 10px; height: 100px;">-->
      <!--              <v-text-field v-model="test" />-->
      <!--              <v-btn @click="markAttendance">Mark Attendance</v-btn>-->
      <!--            </v-card>-->
      <!--          </div>-->
      <!--        </div>-->
      <!--        <v-dialog v-model="loading">-->
      <!--          <v-progress-circular indeterminate />-->
      <!--          <p>-->
      <!--            Seems like a whole day has passed since last Attendance Sheet,-->
      <!--            Creating New Sheet ...-->
      <!--          </p>-->
      <!--        </v-dialog>-->
      </v-container>

      <!--      <v-snackbar v-model="matcherState.state" />-->
    </v-main>
  </div>
</template>

<script>
import moment from 'moment'
import Loading from '../components/loading'
import {
  fetchImage,
  detectSingleFace,
  LabeledFaceDescriptors, FaceMatcher
} from 'face-api.js'
import DigitalPersonaFingerprint from '../components/DigitalPersonaFingerprint'

export default {
  components: { Loading, DigitalPersonaFingerprint },

  filters: {
    time(input) {
      if (input) {
        return moment(input).format('HH:mm')
      } else {
        return 'None'
      }
    },
  },

  async asyncData({ $axios }) {
    return {
      faces: await $axios.$get('/attendance/load-faces') || [],
      attendance: await $axios.$get('/attendance/latest')
    }
  },

  data: () => ({
    state: {
      error: null,
      loading: false,
    },

    handle: {
      1: null,
      2: null
    },

    fingerMatcher: {
      user: null,
      state: 1,
      error: '',
      matcher: false
    },

    handle1: null,
    handle2: null,

    matcherState: {
      state: 1,
      error: '',
      matcher: false
    }
  }),

  mounted() {
    this.checkDate()
    this.checkCurrentShift()
    // if (this.handle1) clearInterval(this.handle1)
    // if (this.handle2) clearInterval(this.handle2)
    //
    // this.checkDate()
    // this.checkCurrentShift()
    // this.handle1 = setInterval(this.checkDate, 60000)
    // this.handle2 = setInterval(this.checkCurrentShift, 60000)
  },

  methods: {
    matchFace(data) {
      const result = this.faceMatcher?.findBestMatch(data.descriptor)

      if (result && result.label !== 'unknown') {
        for (const user of this.attendance.users) {
          console.log(result.label)

          if (result.label === user.user._id && !user.isPresent) {
              this.markAttendance(result.label)
          }
        }
      }
    },

    async loadFaces() {
      if (this.faceMatcher == null) {
        const result = await Promise.all(
          this.faces.map(async face => {
            const descriptions = []

            for (const image of face.images) {
              const faceDesc = await detectSingleFace(
                await fetchImage(this.$axios.defaults.baseURL + '/users/face-image/' + face.label + '/' + image)
              ).withFaceLandmarks().withFaceDescriptor()

              if (faceDesc) {
                descriptions.push(faceDesc.descriptor)
              }
            }

            return new LabeledFaceDescriptors(face.label, descriptions)
          })
        )

        if (result && result.length > 0) {
          this.faceMatcher = new FaceMatcher(result, .6)
        }
      }
    },

    async checkDate() {
      if (moment().dayOfYear() !== moment(this.attendance.date).dayOfYear()) {
        this.loading = true
        this.attendance = await this.$axios.$post('/attendance/create-sheet', {
          shift: 1,
        })
        this.loading = false
      }
    },

    async checkCurrentShift() {
      const shift = await this.$axios.$get('/shifts/current')

      if (shift.index != this.attendance.shift) {
        this.attendance = await this.$axios.$post('/attendance/create-sheet', {
          shift,
        })
      }
    },

    async sendFinger(finger) {
      this.matcherState.state = 2

      try {
        const formData = new FormData()

        formData.append('finger', finger)
        const data = await this.$axios.$post(
          'http://localhost:7070/match-user/',
          formData
        )

        if (data) {
          this.patient = await this.$axios.$get(
            '/patients/' + data
          )

          await this.markAttendance(data)
          this.matcherState.matcher = false
        } else {
          this.matcherState.error = 'Not found, Close dialog and retry.'
          this.matcherState.state = 3
        }
      } catch (e) {
        this.matcherState.error = e
        this.matcherState.state = 3
      }
    },

    // toggleMatcher() {
    //   if (this.matcherState.matcher) {
    //     this.matcherState.matcher = false
    //   } else {
    //     this.matcherState.state = 1
    //     this.matcherState.matcher = true
    //   }
    // },

    createNewSheet() {
      this.attendance = this.$axios.$get('attendance/reset')
    },

    async markAttendance(id) {
      console.log(id)
      await this.$axios.$patch(
        `attendance/mark-attendance?user=${id}&attendance-sheet=${this.attendance._id}`
      )

      this.attendance = await this.$axios.$get('/attendance/latest')
    }
  }
}
</script>

<style lang="sass" scoped>
.main-container
  display: grid
  margin-right: 12px
  grid-column-gap: 12px
  height: calc(100vh - 72px)
  grid-template-columns: 70% 30%

  &__input
    display: grid
    grid-row-gap: 12px
    grid-template-rows: auto 2fr 1fr

.status-card
  color: white
  padding: 2px 15px
  font-weight: bold
  border-radius: 4px
  letter-spacing: 1.5px
  display: inline-block
  text-transform: uppercase

  &--active
    background: rgba(green, .7)

  &--in-active
    background: red
</style>
