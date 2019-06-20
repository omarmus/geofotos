<template>
  <div class="foto">
    <div
      v-show="loading"
      class="foto-loading">
      <div v-html="message"></div>
    </div>
    <div
      v-show="error"
      v-html="error"
      class="foto-error">
    </div>
    <div
      v-show="success"
      v-html="success"
      class="foto-success">
    </div>
    <div class="foto-container">
      <video
        id="video"
        class="foto-video"
        autoplay></video>
      <div
        class="btn-container"
        :class="orientation">
        <button
          id="snap"
          @click="drawPhoto"
          class="btn"><i class="icon-camera"></i></button>
        <button
          type="button"
          v-show="devices.length > 1"
          class="btn btn-small"
          @click="selectCamera()"><i class="icon-loop"></i></button>
      </div>
      <div class="foto-draw" v-show="showAddress">
        <div class="foto-address">
          <h2><i class="icon-location"></i> Ubicación</h2>
          <ul>
            <li>
              <strong>Dirección:</strong> {{ address.address29 }} {{ address.road }} {{ address.neighbourhood }}
            </li>
            <li>
              <strong>Ciudad:</strong> {{ address.city }}
            </li>
            <li>
              <strong>Coordenadas:</strong>
              <ul v-if="address.coordinates">
                <li><strong>- Latitud:</strong>{{ address.coordinates.latitude }}</li>
                <li><strong>- Longitud:</strong>{{ address.coordinates.longitude }}</li>
                <!-- <li><strong>- Altitud:</strong>{{ address.coordinates.altitude }}</li> -->
              </ul>
            </li>
          </ul>
          <div class="foto-data-user" v-if="existUser">
            <hr>
            <h2><i class="icon-user"></i> Mis datos</h2>
            <ul>
              <li><strong>Nombre o álias:</strong> {{ form.first_name }}</li>
              <li><strong>Email:</strong> {{ form.email }}</li>
              <li><strong>Teléfono:</strong> {{ form.phone }}</li>
            </ul>
            <button
              type="button"
              class="pure-button button-small"
              @click="next">
              <i class="icon-edit"></i> Cambiar datos
            </button>
          </div>
        </div>
        <div class="foto-container-canvas">
          <canvas
            id="canvas"
            class="foto-canvas"></canvas>
          <img
            src=""
            class="foto-img"
            id="foto-img">
        </div>

        <div class="foto-options">
          <button
            type="button"
            class="pure-button button-xlarge"
            @click="showAddress = false"
            ><i class="icon-arrow-left"></i> Atrás</button>
          <button
            type="button"
            class="pure-button button-xlarge pure-button-primary"
            v-if="!existUser"
            @click="next"
            >
              {{ existUser ? 'Cambiar datos' : 'Continuar' }} <i class="icon-arrow-right"></i>
            </button>
          <button
            v-if="existUser"
            type="button"
            @click="send"
            class="pure-button button-xlarge pure-button-primary"
            :disabled="sending"
            ><i class="icon-check"></i> Enviar foto</button>
        </div>
      </div>
    </div>
    <div
      class="foto-user"
      v-if="nextStep">
      <form
        class="foto-form pure-form pure-form-stacked"
        @submit.prevent="send"
      >
        <fieldset>
          <legend>Complete los siguientes datos:</legend>
          <div class="foto-input">
            <label for="">Nombre o Álias</label>
            <input
              type="text"
              required
              placeholder="Ejemplo: Maria Flores, juan"
              maxlength="30"
              v-model="form.first_name">
          </div>
          <div class="foto-input">
            <label for="">Correo electrónico</label>
            <input
              type="email"
              placeholder="ejemplo@correo.com"
              maxlength="100"
              v-model="form.email">
          </div>
          <div class="foto-input">
            <label for="">Número de teléfono</label>
            <input
              type="tel"
              maxlength="15"
              placeholder="70070000"
              v-model="form.phone">
          </div>
          <div class="foto-options">
            <button
              type="button"
              class="pure-button button-xlarge"
              @click="cancel"
              ><i class="icon-cancel"></i> Cancelar</button>
            <button
              type="submit"
              class="pure-button button-xlarge pure-button-primary"
              :disabled="sending"
              ><i class="icon-check"></i> Enviar foto</button>
          </div>
        </fieldset>
      </form>
    </div>
  </div>
</template>

<script>
import * as locationService from './location'
import axios from 'axios'

const URL = process.env.VUE_APP_BACKEND_API

export default {
  data () {
    return {
      message: null,
      error: null,
      success: null,
      sending: false,
      nextStep: false,
      showAddress: false,
      devices: [],
      existUser: false,
      currentStream: null,
      video: null,
      address: {},
      // Make it possible to conditionally render
      // elements based on if the geolocation API
      // is availabel or not.
      geolocationSupported: 'geolocation' in navigator,
      loading: false,
      width: 0,
      height: 0,
      camera: '',
      form: {
        neighbourhood: '',
        city: '',
        county: '',
        state: '',
        postcode: '',
        country: '',
        country_code: '',
        latitude: '',
        longitude: '',
        altitude: '',
        photo: '',
        username: '',
        email: '',
        phone: '',
        first_name: '',
        last_name: ''
      },
      time: 7000,
      orientation: ''
    }
  },
  created () {
    this.setUser()
  },
  mounted () {
    this.video = document.getElementById('video')

    // Get access to the camera!
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      this.selectCamera(true)
    } else {
      alert('Su navegador no tiene soporte para el acceso a la cámara')
    }

    window.addEventListener('orientationchange', () => {
      // iOS doesn't have screen.orientation, so fallback to window.orientation.
      // screen.orientation will
      let angle
      if (screen.orientation) {
        angle = screen.orientation.angle
      } else {
        angle = window.orientation
      }

      if (angle === 270 || angle === -90 || angle === 90) {
        this.orientation = 'left'
      } else {
        this.orientation = ''
      }

      // 0   portrait-primary
      // 180 portrait-secondary device is down under
      // 90  landscape-primary  buttons at the right
      // 270 landscape-secondary buttons at the left
    }, false)
  },
  methods: {
    cancel (answer = true) {
      if (!answer || confirm('¿Está seguro de cancelar el envío de su fotografía?')) {
        this.showAddress = false
        this.nextStep = false
      }
    },
    empty (value) {
      return value === undefined || value === null || value.length === 0 || /^\s+$/.test(value)
    },
    next () {
      console.log('next')
      let img = document.getElementById('foto-img')
      if (this.empty(img.src) || img.src === 'data:,') {
        this.showError('⚠️ Debe sacar una foto primero, habilite la camara')
        return false
      }
      this.setUser()
      this.nextStep = true
    },
    setUser () {
      if (window.localStorage.getItem('app-geofoto-user')) {
        let user = JSON.parse(window.localStorage.getItem('app-geofoto-user'))
        this.form.first_name = user.first_name
        this.form.email = user.email
        this.form.phone = user.phone

        this.existUser = true
      }
    },
    async send () {
      if (this.empty(this.form.email) && this.empty(this.form.phone)) {
        this.showError('⚠️ Debe llenar al menos un correo electrónico o un número de teléfono')
        return false
      }
      let img = document.getElementById('foto-img')
      let data = Object.assign({}, this.address)
      if (this.empty(img.src) || img.src === 'data:,') {
        this.showError('⚠️ Debe sacar una foto antes de envíarla')
        return false
      }
      if (!data.coordinates) {
        this.showError('⚠️ Debe envíar la ubicación de la fotografía')
        return false
      }
      this.sending = true
      data.latitude = data.coordinates.latitude
      data.longitude = data.coordinates.longitude
      data.altitude = data.coordinates.altitude
      data.photo = img.src
      data.first_name = this.form.first_name
      data.email = this.form.email
      data.phone = this.form.phone

      delete data.coordinates

      let user = {
        first_name: this.form.first_name,
        email: this.form.email,
        phone: this.form.phone
      }
      try {
        this.showLoading('Enviando su fotografía, <br> espere por favor...')
        let response = await axios.post(URL + '/api/register', data)
        this.sending = false
        if (response.data.id) {
          this.hideLoading()
          this.showSuccess('✅ ¡Su foto se envió correctamente!')
          this.cancel(false)
          window.localStorage.setItem('app-geofoto-user', JSON.stringify(user))
          this.existUser = true
        } else {
          this.showError('⚠️ No se pudo crear la ubicación')
        }
      } catch (error) {
        this.sending = false
        this.showError(error)
      }
    },
    drawPhoto () {
      this.fetchAddress()
      this.showAddress = true

      // Elements for taking the snapshot
      let canvas = document.getElementById('canvas')
      let img = document.getElementById('foto-img')
      let context = canvas.getContext('2d')
      let video = document.querySelector('#video')
      canvas.height = video.videoHeight
      canvas.width = video.videoWidth
      window.video = video

      context.drawImage(video, 0, 0)
      img.src = canvas.toDataURL('image/webp')
    },
    selectCamera (init) {
      try {
        if (this.currentStream) {
          this.stopMediaTracks(this.currentStream)
        }
        let videoConstraints = {
          // width: { ideal: 1280 }, // No funciona con Android's antiguos :(
          // height: { ideal: 720 }
        }
        if (init) {
          this.camera = 'environment'
        } else {
          this.camera = this.camera === 'environment' ? 'user' : 'environment'
          // this.camera = this.devices[0].deviceId === this.camera ? this.devices[1].deviceId : this.devices[0].deviceId
          // videoConstraints.deviceId = { exact: this.camera }
        }
        videoConstraints.facingMode = this.camera
        const constraints = {
          video: videoConstraints,
          audio: false
        }

        navigator.mediaDevices
          .getUserMedia(constraints)
          .then(stream => {
            this.currentStream = stream
            this.video.srcObject = stream
            return navigator.mediaDevices.enumerateDevices()
          })
          .then(mediaDevices => {
            if (this.devices.length === 0) {
              this.gotDevices(mediaDevices)
            }
          })
          .catch(error => {
            alert('ERROR navigator.mediaDevices: ' + (error.message || ''))
          })
      } catch (error) {
        alert('ERROR selectCamera: ' + (error.message || ''))
      }
    },
    stopMediaTracks (stream) {
      try {
        stream.getTracks().forEach(track => {
          track.stop()
        })
      } catch (error) {
        alert('ERROR stopMediaTracks: ' + (error.message || ''))
      }
    },
    gotDevices (mediaDevices) {
      try {
        let devices = []
        mediaDevices.forEach(device => {
          if (device.kind === 'videoinput') {
            let media = {
              deviceId: device.deviceId,
              label: device.label.split(' ').reverse()[0]
            }
            devices.push(media)
          }
        })
        this.devices = devices
        // if (this.devices[1]) {
        //   this.camera = this.devices[1].deviceId
        // }
      } catch (error) {
        console.error('ERROR gotDevices: ' + (error.message || ''))
      }
    },
    async fetchAddress () {
      this.address = {}
      try {
        this.showLoading('Obteniendo ubicación...')
        this.address = await locationService.currentAddress()
        this.hideLoading()
      } catch (error) {
        this.showError(error)
      }
    },
    showError (error) {
      this.error = error.message || error
      this.success = null
      setTimeout(this.hideLoading, this.time)
    },
    showSuccess (success) {
      this.error = null
      this.success = success
      setTimeout(this.hideLoading, this.time)
    },
    showLoading (message = 'Cargando...') {
      this.error = null
      this.success = null
      this.message = message
      this.loading = true
    },
    hideLoading () {
      this.loading = false
      this.error = null
      this.success = null
      this.message = null
    }
  },
  render () {
    return this.$scopedSlots.default({
      // Data
      address: this.address,
      error: this.error,
      geolocationSupported: this.geolocationSupported,
      loading: this.loading,
      // Methods
      fetchAddress: this.fetchAddress
    })
  }
}
</script>

<style lang="scss">
.foto-data-user {
  position: relative;

  .pure-button {
    position: absolute;
    top: 10px;
    right: 0;
  }
}
.foto-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #333333;
  display: flex;
}
.foto-user {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  z-index: 4;
  background: rgba(255, 255, 255, .95);
}
.foto-video {
  width: 100%;
}
.btn {
  outline: none;
  border: none;
  background-color: #0078e7;
  color: #fff;
  display: inline-block;
  vertical-align: bottom;
  height: 72px;
  width: 72px;
  border-radius: 50%;
  margin: 5px;
  font-size: .9rem;

  .icon-camera {
    font-size: 1.8rem;
  }

  .icon-loop {
    font-size: 1.3rem;
  }

  &.btn-small {
    height: 56px;
    width: 56px;
    position: absolute;
    right: 10px;
    bottom: 10px;
    background-color: lighten(#0078e7, 15%);
  }
}
.btn-container {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  text-align: center;
  padding-bottom: 10px;

  &.left {
    top: 0;
    left: initial;
    top: 0;
    left: initial;
    display: flex;
    align-items: center;
    padding: 0;
  }
}
.foto-canvas {
  display: none;
}

.foto-draw {
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
}
.foto-address {
  position: absolute;
  bottom: 70px;
  left: 0;
  right: 0;
  background-color: rgba(0, 0, 0, .45);
  font-size: 0.9rem;
  color: white;
  padding: 10px;
  z-index: 2;

  h2 {
    margin: 0 0 8px;
  }

  ul {
    list-style: none;
    margin: 0 auto;
    padding-left: 0px;

    strong {
      margin-right: 5px;
    }
  }
  ul ul {
    padding-left: 15px;
  }
}
.foto-options {
  background-color: rgba(255, 255, 255, .8);
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  text-align: right;
  padding: 10px 15px;
  z-index: 2;

  .pure-button {
    margin: 0 1px !important;
  }
}
.foto-container-canvas {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #333333;
  z-index: 1;
}
.foto-img {
  width: 100%;
}
.foto-loading {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  padding: 20px;
  background-color: rgba(255, 255, 255, .95);
  color: #777777;
  font-size: 1.5rem;
  z-index: 99;
  display: flex;
  align-items: center;
  justify-content: center;

  & > div {
    text-align: center;
  }
}
.foto-error {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  padding: 15px 20px;
  background: crimson;
  color: white;
  text-align: center;
  z-index: 100;
  font-size: 1.1rem;
}
.foto-success {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  padding: 15px 20px;
  background-color: rgb(28, 184, 65);
  color: white;
  text-align: center;
  z-index: 100;
  font-size: 1.1rem;
}
.foto-form {
  padding: 20px;
}
.button-xlarge {
  font-size: 1.1rem;
}
.button-small {
  font-size: 0.8rem;
}

@media (max-width: 768px) {
  .foto-input {
    input {
      width: 100%;
    }
  }
}
</style>
