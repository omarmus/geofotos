<template>
  <div class="users">
    <div class="users-header">
      <h1><i class="icon-user"></i> Participantes</h1>
    </div>
    <div class="container">
      <div class="users-container">
        <div>
          <div class="alert alert-info">Seleccione un participante para ver sus fotografías</div>
          <table class="pure-table">
            <thead>
              <tr>
                <th>Nombre o álias</th>
                <th>Email</th>
                <th>Teléfono</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in users"
                :key="item.id"
                @click="selectUser(item)"
                :class="{ active: user && user.id === item.id }">
                <td>{{ item.first_name }}</td>
                <td>{{ item.email }}</td>
                <td>{{ item.phone }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="users-photos" v-if="user">
          <h2>Fotografías tomadas por {{ user.first_name || user.email }}</h2>
          <div
            class="card card-geo"
            v-for="item in geos"
            :key="item.id">
            <div
              class="card-photo"
              :style="{ 'background-image': `url(${url}/files/${item.photo})` }">
              <div class="card-photo-zoom" @click="openZoom(item.photo)"><i class="icon-zoom"></i></div>
            </div>
            <div class="card-map">
               <div :id="'map-' + item.id" class="card-map-draw"></div>
            </div>
            <div class="card-info">
              <!-- {{ item }} -->
              <ul>
                <li>
                  <strong>Dirección:</strong> {{ item.item29 }} {{ item.road }} {{ item.neighbourhood }}
                </li>
                <li>
                  <strong>Ciudad:</strong> {{ item.city }}
                </li>
                <li>
                  <strong>Coordenadas:</strong>
                  <ul>
                    <li><strong>- Latitud:</strong>{{ item.latitude }}</li>
                    <li><strong>- Longitud:</strong>{{ item.longitude }}</li>
                  </ul>
                </li>
              </ul>
              <div class="card-info-state" v-if="item.state">{{ item.state }}</div>
              <div class="card-info-date" v-if="item.created_at"><strong>Fecha de creación:</strong> {{ formatDate(item.created_at) }}</div>
            </div>
          </div>
        </div>
        <span class="photos-loading" v-if="loading">Cargando...</span>
      </div>
    </div>
    <div class="users-modal" v-if="image">
      <span title="Cerrar ventana" class="close" @click="image = false"><i class="icon-cancel"></i></span>
      <img :src="image" alt="image">
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import icon from '../assets/mark.png'
import 'leaflet'
import '../../node_modules/leaflet/dist/leaflet.css'

const URL = process.env.VUE_APP_BACKEND_API

export default {
  data () {
    return {
      users: [],
      geos: [],
      user: null,
      url: URL,
      loading: false,
      image: false
    }
  },
  mounted () {
    axios.get(`${URL}/model/users`)
      .then(response => {
        if (response.data) {
          this.users = response.data
        }
      })
  },
  methods: {
    openZoom (photo) {
      this.image = `${URL}/files/${photo}`
    },
    formatDate (date) {
      if (typeof date === 'string') {
        date = date.split('T')
        let fecha = date[0].split('-')
        let hora = date[1].split('.')
        return `${fecha[2]}/${fecha[1]}/${fecha[0]} Hrs. ${hora[0]}`
      }
      console.log(typeof date)
      return date
    },
    selectUser (item) {
      this.user = null
      this.loading = true
      axios.get(`${URL}/model/geos?id_user=${item.id}`)
        .then(response => {
          if (response.data) {
            this.loading = false
            this.user = item
            this.geos = response.data
            setTimeout(() => {
              this.renderMaps(this.geos)
            }, 100)
          }
        })
    },
    renderMaps (items) {
      let redIcon = window.L.icon({
        iconUrl: icon,
        iconSize: [22, 34] // size of the icon
        // shadowUrl: 'leaf-shadow.png',
        // shadowSize:   [50, 64], // size of the shadow
        // iconAnchor:   [22, 94], // point of the icon which will correspond to marker's location
        // shadowAnchor: [4, 62],  // the same for the shadow
        // popupAnchor:  [-3, -76] // point from which the popup should open relative to the iconAnchor
      })
      items.map(item => {
        const coord = [item.latitude, item.longitude]
        let map = window.L.map(`map-${item.id}`).setView(coord, 16)

        window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }).addTo(map)

        window.L.marker(coord, { icon: redIcon }).addTo(map)
        // .bindPopup('A pretty CSS3 popup.<br> Easily customizable.')
        // .openPopup()
      })
    }
  }
}
</script>

<style lang="scss">
$primary: #1f8dd6;
body {
  background-color: #f0f0f0;
}
.users-modal {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: 1000;
  background-color: rgba(0, 0, 0, 0.85);
  display: flex;
  justify-content: center;
  align-items: center;

  img {
    border: 10px solid #f0f0f0;
    border-radius: 3px;
  }

  .close {
    position: absolute;
    top: 30px;
    right: 30px;
    cursor: pointer;

    i {
      color: #dddddd;
      font-size: 1.8rem;
    }
  }
}
.photos-loading {
  padding: 15px;
  font-size: .9rem;
}
.container {
  padding: 15px 15px;

  p {
    margin: 0 0 10px;
  }
}
.card-map-draw {
  height: 100%;
}
.users-header {
  background-color: $primary;
  padding: 15px 15px;

  h1 {
    font-size: 1.3rem;
    margin: 0;
    color: white;
  }
}
.users {
  .pure-table {
    background-color: #ffffff;
    width: 100%;

    tbody {
      tr {
        color: #333333;

        &:hover, &.active {
          background-color: $primary;
          color: white;
          cursor: pointer;
        }
      }
      td {
        border: none;
      }
    }
  }
  .users-container {
    display: grid;
    grid-template-columns: 450px 1fr;
  }
  .users-photos {
    padding: 0 0 10px 10px;

    h2 {
      font-size: 1.2rem;
      margin: 16px 0 10px;
      color: $primary;
    }
  }
}
.card {
  border: 1px solid #dddddd;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 3px;
  background-color: #ffffff;
}
.card-geo {
  display: grid;
  grid-template-columns: 250px 250px 1fr;
  grid-column-gap: 10px;

  .card-photo {
    position: relative;
    height: 200px;
    background-size: cover;

    &:hover {
      .card-photo-zoom {
        display: flex;
      }
    }

    .card-photo-zoom {
      display: none;
      position: absolute;
      top: 0;
      right: 0;
      bottom: 0;
      left: 0;
      background-color: rgba(0, 0, 0, .3);
      cursor: pointer;
      align-items: center;
      justify-content: center;
      transition: all ease .4s;

      i {
        font-size: 1.8rem;
        color: white;
      }
    }
  }

  .card-info {
    position: relative;

    ul {
      list-style: none;
      margin: 0;
      padding: 0;
    }

    .card-info-state {
      position: absolute;
      background-color: $primary;
      color: white;
      padding: 10px 15px;
      top: -10px;
      right: -10px;
    }
    .card-info-date {
      position: absolute;
      font-size: .85rem;
      bottom: 0;
      right: 5px;
      font-style: italic;
      color: #777777;
    }
  }
}
.alert {
  padding: 10px 15px;
  margin-bottom: 10px;
  border-radius: 3px;
  font-size: .9rem;
}
.alert-info {
  background-color: lighten($primary, 40%);
  border: 1px solid lighten($primary, 35%);
  color: darken($primary, 15%);
}
</style>
