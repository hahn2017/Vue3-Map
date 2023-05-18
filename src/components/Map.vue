<template>
  <div class="d-flex mt-5 main-block">
    <div class="container locations-block">
      <h3 class="search-title">Search your location:</h3>
      <div class="search-bar">
        <input
          type="text"
          v-model="searchQuery"
          @keyup.enter="searchLocation"
          placeholder="Enter Location Name"
        />
        <button @click="searchLocation">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="16"
            height="16"
            fill="currentColor"
            class="bi bi-search"
            viewBox="0 0 16 16"
          >
            <path
              d="M11.742 10.344a6.5 6.5 0 1 0-1.397 1.398h-.001c.03.04.062.078.098.115l3.85 3.85a1 1 0 0 0 1.415-1.414l-3.85-3.85a1.007 1.007 0 0 0-.115-.1zM12 6.5a5.5 5.5 0 1 1-11 0 5.5 5.5 0 0 1 11 0z"
            />
          </svg>
        </button>
      </div>

      <!-- Table with pagination -->
      <table class="mt-3 table w-100">
        <thead>
          <tr>
            <th class="col-2">#</th>
            <th class="col-8">Name</th>
            <th class="col-2">
              <button
                :disabled="selectedPlaces.length === 0"
                class="btn btn-dark rounded-pill"
                @click="deleteSelectedPlaces"
              >
                Delete
              </button>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="place in paginatedPlaces" :key="place.id">
            <td>
              <input type="checkbox" v-model="selectedPlaces" :value="place" />
            </td>
            <td>{{ place.name }}</td>
            <td></td>
          </tr>
        </tbody>
      </table>
      <div>
        <button
          class="rounded"
          :disabled="currentPage === 1"
          @click="currentPage--"
        >
          {{ '<' }}
        </button>
        <span class="page-item"
          ><span class="mx-2">{{ currentPage }}</span></span
        >
        <button
          class="rounded"
          :disabled="places.length === 0 || currentPage === totalPages"
          @click="currentPage++"
        >
          {{ '>' }}
        </button>
      </div>

      <div class="mt-5" v-if="latestLocation">
        <p>Time Zone: {{ latestLocation.timeZone }}</p>
        <p>Local Time: {{ latestLocation.localTime }}</p>
      </div>
    </div>

    <div class="map-block">
      <button
        class="btn bg-dark text-white rounded-pill mb-2"
        @click="getCurrentLocation"
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="16"
          height="16"
          fill="currentColor"
          class="bi bi-geo-alt"
          viewBox="0 0 16 16"
        >
          <path
            d="M12.166 8.94c-.524 1.062-1.234 2.12-1.96 3.07A31.493 31.493 0 0 1 8 14.58a31.481 31.481 0 0 1-2.206-2.57c-.726-.95-1.436-2.008-1.96-3.07C3.304 7.867 3 6.862 3 6a5 5 0 0 1 10 0c0 .862-.305 1.867-.834 2.94zM8 16s6-5.686 6-10A6 6 0 0 0 2 6c0 4.314 6 10 6 10z"
          />
          <path
            d="M8 8a2 2 0 1 1 0-4 2 2 0 0 1 0 4zm0 1a3 3 0 1 0 0-6 3 3 0 0 0 0 6z"
          />
        </svg>
        Get Current Location
      </button>

      <!-- Display the map -->
      <div id="map"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, computed, watch } from 'vue'
import { Loader } from '@googlemaps/js-api-loader'

interface Place {
  id: string
  name: string
  marker?: google.maps.Marker
}

interface TimeZoneInfo {
  timeZone: string
  localTime: string
}

const apiKey = import.meta.env.VITE_MAP_API_KEY

const loader = new Loader({
  apiKey,
  version: 'weekly',
  libraries: ['places'],
})

const mapOptions = {
  zoom: 8,
  center: { lat: 44, lng: -79.5 },
}

let map: google.maps.Map

let places = reactive<Place[]>([])
const selectedPlaces = ref<Place[]>([])
const searchQuery = ref('')
const latestLocation = ref<TimeZoneInfo | null>(null)
const currentPage = ref(1)
const itemsPerPage = 10

function getCurrentLocation() {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition((position) => {
      const latLng = new google.maps.LatLng(
        position.coords.latitude,
        position.coords.longitude
      )
      addMarker(latLng)
      map.setCenter(latLng)
    })
  }
}

function searchLocation() {
  if (searchQuery.value) {
    const geocoder = new google.maps.Geocoder()
    geocoder.geocode({ address: searchQuery.value }, (results, status) => {
      if (status === 'OK' && results && results.length > 0) {
        let seen = false
        for (let place of places) {
          if (place.id === results[0].place_id) {
            seen = true
          }
        }
        const latLng = results[0].geometry.location
        if (!seen) {
          const place: Place = {
            id: results[0].place_id,
            name: results[0].formatted_address,
          }
          place.marker = addMarker(latLng)
          places.push(place)
          const lat = latLng.lat()
          const lng = latLng.lng()

          getTimeZoneInfo(lat, lng).then((timeZoneInfo) => {
            latestLocation.value = timeZoneInfo
          })
        }
        map.setCenter(latLng)
      }
    })
  }
  searchQuery.value = ''
}

function addMarker(latLng: google.maps.LatLng): google.maps.Marker {
  return new google.maps.Marker({
    position: latLng,
    map: map,
  })
}

function deleteSelectedPlaces() {
  selectedPlaces.value.forEach((place) => {
    if (place.marker) {
      place.marker.setVisible(false) //this line works
      place.marker.setMap(null)
      place.marker.setPosition(null)
      delete place.marker
    }
    const index = places.indexOf(place)
    places.splice(index, 1)
  })
  selectedPlaces.value = []
}

async function getTimeZoneInfo(
  lat: number,
  lng: number
): Promise<TimeZoneInfo> {
  const timestamp = Math.floor(Date.now() / 1000)
  const response = await fetch(
    `https://maps.googleapis.com/maps/api/timezone/json?location=${lat},${lng}&timestamp=${timestamp}&key=${apiKey}`
  )
  const data = await response.json()

  return {
    localTime: new Date(timestamp * 1000).toLocaleString('en-US', {
      timeZone: data.timeZoneId,
    }),
    timeZone: data.timeZoneId,
  }
}

const paginatedPlaces = computed<Place[]>(() => {
  const startIndex = (currentPage.value - 1) * itemsPerPage
  const endIndex = startIndex + itemsPerPage
  return places.slice(startIndex, endIndex)
})

const totalPages = computed(() => Math.ceil(places.length / itemsPerPage))

watch(totalPages, (totalPages, _) => {
  currentPage.value =
    currentPage.value <= totalPages
      ? currentPage.value
      : totalPages === 0
      ? 1
      : totalPages
})

onMounted(() => {
  loader.load().then(() => {
    const mapElement = document.getElementById('map')
    if (mapElement) {
      map = new google.maps.Map(mapElement, mapOptions)
    }
  })
})
</script>

<style lang="scss" scoped>
.main-block {
  width: 1350px;
  height: 1000px;

  .locations-block {
    .search-title {
      text-align: left;
    }

    .search-bar {
      width: 100%;
      position: relative;
      margin: 5px 0;

      input {
        width: 100%;
        padding: 0 30px;
        border-radius: 50px;
        border: solid;
        font-weight: 600;
        font-size: 16px;
        text-transform: capitalize;
        position: relative;
        color: #333;
        height: 50px;
      }

      button {
        position: absolute;
        right: 6px;
        height: 40px;
        border: none;
        color: #fff;
        background: #000;
        top: 5px;
        border-radius: 50px;
        width: 80px;

        &:hover {
          background: grey;
        }
      }
    }

    button[disabled] {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }

  .map-block {
    min-width: 640px;
    text-align: end;
    #map {
      height: 600px;
      width: 100%;
      border-radius: 20px;
    }
  }
}
</style>
