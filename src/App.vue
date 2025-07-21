<script setup lang="ts">
import axios from 'axios';
import { ref } from 'vue';

// Define API response interfaces
interface LocationResult {
  id: number;
  name: string;
  latitude: number;
  longitude: number;
  country: string;
  admin1: string;
}

interface WeatherData {
  time: string[];
  temperature_2m_max: number[];
  temperature_2m_min: number[];
  precipitation_probability_max: number[];
  weathercode: number[];
}

interface LocationApiResponse {
  results: LocationResult[];
}

interface WeatherApiResponse {
  daily: WeatherData;
}

const searchQuery = ref<string>('');
const locations = ref<LocationResult[]>([]);
const weatherData = ref<WeatherData | null>(null);
const loading = ref<boolean>(false);
const error = ref<string | null>(null);

const searchLocations = async (): Promise<void> => {
  try {
    const response = await axios.get<LocationApiResponse>(
      'https://geocoding-api.open-meteo.com/v1/search',
      {
        params: {
          name: searchQuery.value,
          count: 10,
          language: 'en',
          format: 'json',
        },
      },
    );
    locations.value = response.data.results;
    error.value = null;
  } catch (err) {
    error.value = 'Failed to fetch locations. Check your search query.';
    console.error('Search error:', err);
  }
};

// Fetch weather data for selected location
const selectLocation = async (loc: LocationResult): Promise<void> => {
  loading.value = true;
  error.value = null;
  try {
    const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
    const response = await axios.get<WeatherApiResponse>(
      'https://api.open-meteo.com/v1/forecast',
      {
        params: {
          latitude: loc.latitude,
          longitude: loc.longitude,
          daily:
            'weathercode,temperature_2m_max,temperature_2m_min,precipitation_probability_max',
          timezone,
        },
      },
    );
    weatherData.value = response.data.daily;
    loading.value = false;
  } catch (err) {
    error.value = 'Failed to fetch weather data. Try another location.';
    console.error('Weather fetch error:', err);
    loading.value = false;
  }
};

// Format date to user's local timezone
const formatDate = (dateString: string): string => {
  const date = new Date(dateString);
  return date.toLocaleDateString();
};
</script>

<template>
  <div class="weather-app">
    <div class="search-container">
      <input
        type="text"
        v-model="searchQuery"
        class="search-input"
        @keyup.enter="searchLocations"
        placeholder="Search location..." />
      <button
        @click="searchLocations"
        class="search-button">
        Search
      </button>
    </div>
    <h2 v-if="locations.length">Choose a city</h2>
    <ul v-if="locations.length">
      <li
        v-for="loc in locations"
        :key="loc.id"
        @click="selectLocation(loc)"
        class="location-item">
        {{ loc.name }}, {{ loc.admin1 ? `${loc.admin1}, ` : ''
        }}{{ loc.country }}
      </li>
    </ul>
    <div v-if="loading">Loading weather data...</div>
    <div
      v-if="error"
      class="error">
      {{ error }}
    </div>
    <div
      v-if="weatherData"
      class="weather-details">
      <h2>Current Conditions</h2>
      <p>Max Temp: {{ weatherData.temperature_2m_max[0] }}°C</p>
      <p>Min Temp: {{ weatherData.temperature_2m_min[0] }}°C</p>
      <h3>Daily Forecast</h3>
      <ul>
        <li
          v-for="(date, index) in weatherData.time.slice(1, 6)"
          :key="index"
          class="forecast-item">
          {{ formatDate(date) }}:
          {{ weatherData.temperature_2m_max[index + 1] }}°C /
          {{ weatherData.temperature_2m_min[index + 1] }}°C
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.search-container {
  display: flex;
  gap: 0.75rem;
  margin-bottom: 1.25rem;
}

.search-input {
  flex: 1;
  padding: 0.75rem;
  outline: none;
  border: 2px solid transparent;
  border-bottom: 2px solid hsl(210, 45%, 75%);

  &:focus {
    border: solid 2px hsl(210, 30%, 30%);
    background-color: hsl(210, 50%, 90%);
  }
}

.search-button {
  padding: 0.75em 1.25em;
  background-color: transparent;
  border: none;
  cursor: pointer;

  &:hover {
    background-color: hsl(210, 30%, 30%);
    color: white;
  }
}

.weather-app {
  max-width: 40rem;
  margin: 0 auto;
  padding: 1rem;
}

.location-item {
  cursor: pointer;
  padding: 0.75em 1.25em;
  border: 2px dashed hsl(210, 45%, 75%);
  margin: 0.25rem 0;

  &:hover {
    border: 2px solid hsl(210, 50%, 50%);
    background-color: hsl(210, 50%, 90%);
  }
}

.error {
  color: hsl(0, 100%, 65%);
  margin-top: 0.6rem;
}

.forecast-item {
  padding: 0.5em;
  background: hsl(210, 45%, 90%);
  margin: 0.25rem 0;
}
</style>
