<template>
  <div class="ui one column stackable grid">
    <div class="column ">
      <form class="ui segment large form" @submit.prevent="searchLocation">
        <div class="ui message" v-show="error">{{ error }}</div>
        <div class="ui segment">
          <div class="field">
            <div class="ui right icon input large" :class="{ loading: spinner }">
              <input
                type="text"
                placeholder="Enter your address"
                v-model="address"
                id="autocomplete"
                @keyup.enter="searchLocation"
              />
              <i class="dot circle link icon" @click="getLocation"></i>
            </div>
          </div>
          <button class="ui button" @click="searchLocation">Search</button>
        </div>
      </form>
      <div id="map"></div>
    </div>
    <h1 style="color: #8ab388; text-align: center; width: 100%;">Search History</h1>
    <div class="column">
      <table class="ui celled table">
        <!-- Table content -->
        <thead>
          <tr>
            <th></th>
            <th>Address</th>
            <th>Time Zone</th>
            <th>Local Time</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(result, index) in paginatedResults" :key="index">
            <td>
              <div class="ui checkbox">
                <input
                  type="checkbox"
                  :id="'record-' + index"
                  :checked="isSelected(result)"
                  @change="toggleRecordSelection(result)"
                />
                <label :for="'record-' + index"></label>
              </div>
            </td>
            <td>{{ result.address }}</td>
            <td>{{ result.timeZone }}</td>
            <td>{{ result.localTime }}</td>
          </tr>
        </tbody>
        <tfoot>
          <tr>
            <th colspan="4">
              <div class="buttons">
                <div>
                  <button class="ui button" @click="deleteSelectedRecords">
                    Delete Selected Records
                  </button>
                </div>   
                <div>
                  <button class="ui button" @click="goToPreviousPage" :disabled="currentPage === 1">Previous</button>
                  <button class="ui button" v-for="page in totalPages" :key="page" @click="goToPage(page)" :class="{ active: currentPage === page }">{{ page }}</button>
                  <button class="ui button" @click="goToNextPage" :disabled="currentPage === totalPages">Next</button>
                </div>
              </div>  
            </th>
          </tr>
        </tfoot>
      </table>
    </div>
  </div>
</template>

<script>

import axios from "axios";

export default {

  // function returns the initial data properties used in the component
  data() {
    return {
      googleApiKey: process.env.VUE_APP_GOOGLE_KEY,
      map: null,
      address: "",
      error: "",
      spinner: false,
      markers: [], // Array to store markers
      searchResults: [], // Array to store search results
      selectedRecords: [], // Array to store selected records
      currentPage: 1, // Current page of the table
      pageSize: 10, // Number of records to display per page
    };
  },

  //lifecycle hook is triggered after the component is mounted to the DOM. 
  //It initializes the Google Maps API and sets up an autocomplete 
  //feature for the address input field.
  mounted() {
    this.initializeMap();
    //MAKE SURE ADD window BEFORE GOOGLE 
    let autocomplete = new window.google.maps.places.Autocomplete(
    document.getElementById("autocomplete"),
    {
      bounds: new window.google.maps.LatLngBounds(
        new window.google.maps.LatLng(43.7181228, -79.5428659)
      )
    }
  );
    autocomplete.addListener("place_changed", () => {
      let place = autocomplete.getPlace();
      if (place && place.formatted_address) {
        this.address = place.formatted_address;
      }
    });
  },

  //section contains various methods used by the component to perform specific actions
  methods: {

    //Initializes the Google Map using the provided options
    initializeMap() {
      this.map = new window.google.maps.Map(document.getElementById("map"), {
        zoom: 12,
        center: new window.google.maps.LatLng(43.6503656, -79.3865919),
        mapTypeId: window.google.maps.MapTypeId.ROADMAP,
      });
    },
    //Retrieves the user's current location using the Geolocation API
    getLocation() {
      this.spinner = true;

      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          (position) => {
            this.getAddress(
              position.coords.latitude,
              position.coords.longitude
            );
            this.showUserLocation(
              position.coords.latitude,
              position.coords.longitude
            );
          },
          (error) => {
            this.error =
              "We are not able to find your address. Please type your address manually";
            this.spinner = false;
            console.error(error.message);
          }
        );
      } else {
        this.error = "Geolocation is not supported by this browser.";
        this.spinner = false;
      }
    },

    //Updates the map's center with the given latitude and longitude
    updateMap(lat, lng) {
      const center = new window.google.maps.LatLng(lat, lng);
      this.map.setCenter(center);
    },

     // Retrieves the time zone and local time for the given latitude and longitude
     getTimeZoneAndLocalTime(lat, lng, result) {
      axios
        .get(
          "https://maps.googleapis.com/maps/api/timezone/json?location=" +
            lat +
            "," +
            lng +
            "&timestamp=" +
            Math.floor(Date.now() / 1000) +
            "&key=" +
            this.googleApiKey
        )
        .then((response) => {
          if (response.data.timeZoneId) {
            result.timeZone = response.data.timeZoneId;
            result.localTime = this.getLocalTime(
              response.data.timeZoneId,
              new Date().toISOString()
            );
          }
        })
        .catch((error) => {
          console.error(error);
        });
    },

    // Retrieves the local time for the given time zone and timestamp
    getLocalTime(timeZone, timestamp) {
      const options = {
        timeZone: timeZone,
        hour12: false,
        hour: "numeric",
        minute: "numeric",
        second: "numeric",
      };
      return new Date(timestamp).toLocaleTimeString([], options);
    },

    //Performs a search for the entered address using the Google Geocoding API
    searchLocation() {
      this.spinner = true;
      this.error = "";

      if (this.address) {
        axios
          .get(
            "https://maps.googleapis.com/maps/api/geocode/json?address=" +
              encodeURIComponent(this.address) +
              "&key=" +
              this.googleApiKey
          )
          .then((response) => {
            if (response.data.results.length > 0) {
              const result = response.data.results[0];
              const { lat, lng } = result.geometry.location;

              // Update the search result with time zone and local time
              this.getTimeZoneAndLocalTime(lat, lng, result);
              this.updateMap(lat, lng);

              // Check if the address already exists in the searchResults array
              const existingResult = this.searchResults.find(
                (r) => r.address === result.formatted_address
              );

              if (!existingResult) {
                // Add the search result to the searchResults array
                this.searchResults.push({
                  address: result.formatted_address,
                  timeZone: "", // Set the initial value for timeZone
                  localTime: "", // Set the initial value for localTime
                  marker: null, // Set the initial value for marker
                });
              }

              this.showUserLocation(lat, lng);
            } else {
              this.error = "Location not found.";
            }
            this.spinner = false;
          })
          .catch((error) => {
            this.error = "An error occurred while searching for the location.";
            console.error(error);
            this.spinner = false;
          });
      } else {
        this.error = "Please enter an address.";
        this.spinner = false;
      }
    },

    //Retrieves the formatted address for the given latitude and longitude using the Geocoding API.
    getAddress(lat, long) {
      axios
        .get(
          "https://maps.googleapis.com/maps/api/geocode/json?latlng=" +
            lat +
            "," +
            long +
            "&key=" +
            process.env.VUE_APP_GOOGLE_KEY
        )
        .then((response) => {
          if (response.data.error_message) {
            this.error = response.data.error_message;
          } else {
            this.address = response.data.results[0].formatted_address;
          }
          this.spinner = false;
        })
        .catch((error) => {
          this.error = error.message;
          this.spinner = false;
        });
    },

    //Shows the user's location on the map by adding a marker
    showUserLocation(latitude, longitude) {
      // Remove existing markers from the map
      this.clearMarkers();

      // Add marker for the current location and assign it to the record
      const marker = new window.google.maps.Marker({
        position: new window.google.maps.LatLng(latitude, longitude),
        map: this.map,
      });
      this.markers.push(marker);

      // Assign the marker index to each record in the searchResults array
      for (const result of this.searchResults) {
        result.markerIndex = this.markers.length - 1;
      }

       // Update the search result with time zone and local time
      this.getTimeZoneAndLocalTime(latitude, longitude, this.searchResults[0]);
    },

    //Removes all markers from the map
    clearMarkers() {
      for (const marker of this.markers) {
        marker.setMap(null);
      }
      this.markers = [];
    },

    //Toggles the selection of a record in the search results table and adds/removes its marker on the map.
    toggleRecordSelection(record) {
      if (this.selectedRecords.includes(record)) {
    this.selectedRecords = this.selectedRecords.filter((r) => r !== record);
    if (record.markerIndex !== undefined) {
      const marker = this.markers[record.markerIndex];
      if (marker) {
        marker.setMap(null); // Remove the marker from the map
        record.marker = null; // Clear the marker reference in the record
        delete record.markerIndex; // Remove the marker index from the record
      }
    }
  } else {
    this.selectedRecords.push(record);
    if (record.markerIndex === undefined) {
      const marker = new window.google.maps.Marker({
        position: this.getAddressLatLng(record.address),
        map: this.map,
      });
      record.markerIndex = this.markers.length;
      record.marker = marker; // Store the marker reference in the record
      this.markers.push(marker);
    } else {
      const marker = this.markers[record.markerIndex];
      if (marker) {
        marker.setMap(this.map); // Add the marker to the map
        record.marker = marker; // Update the marker reference in the record
      }
    }
  }
  this.clearMarkers(); // Remove all markers on every selection change
    },

    //Checks if a record is currently selected.
    isSelected(record) {
      return this.selectedRecords.includes(record);
      
    },

    //Deletes the selected records from the search results and removes their markers from the map
    deleteSelectedRecords() {
      // Remove selected records from the searchResults array
  const filteredResults = this.searchResults.filter(
    (result) => !this.selectedRecords.includes(result)
  );

  // Remove markers for the selected records from the map and markers array
  for (const record of this.selectedRecords) {
    if (record.markerIndex !== undefined) {
      const marker = this.markers[record.markerIndex];
      if (marker) {
        marker.setMap(null); // Remove the marker from the map
        this.markers.splice(record.markerIndex, 1); // Remove the marker from the markers array
      }
    }
  }

  // Update the markerIndex values for the remaining markers in the searchResults array
  for (let i = 0; i < filteredResults.length; i++) {
    const record = filteredResults[i];
    if (record.markerIndex !== undefined) {
      record.markerIndex = i;
    }
  }

  // Assign the updated searchResults array to the filteredResults array
  this.searchResults = filteredResults;

  // Clear the selectedRecords array
  this.selectedRecords = [];
    },

    //Handle pagination functionality for the search results table.
    goToPreviousPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },

    goToNextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },

    goToPage(page) {
      if (page >= 1 && page <= this.totalPages) {
        this.currentPage = page;
      }
    },
  },

  //section defines computed properties that are derived from the component's data.
  computed: {

    //Returns a subset of search results based on the current page and page size.
    paginatedResults() {
      const startIndex = (this.currentPage - 1) * this.pageSize;
      const endIndex = startIndex + this.pageSize;
      return this.searchResults.slice(startIndex, endIndex);
    },

    //Calculates the total number of pages based on the number of search results and page size.
    totalPages() {
      return Math.ceil(this.searchResults.length / this.pageSize);
    },
  },
};
</script>

<style>
html{
  margin: 0 20em;
}
  .ui.button,
  .dot.circle.icon {
    background-color: #8ab388;
    color: white;
  }

  #map {
    width: 100%;
    height: 400px;
    z-index: -1;
  }

  .ui.form {
    display: inline-block;
    z-index: 1;
    width: 100%;
  }

  .table {
    position: relative;
  }

  .pagination {
    margin-top: 20px;
    display: flex;
    justify-content: center;
    position: relative;
  }

  .pagination button {
    margin: 0 5px;
  }

  .pagination button.active {
    background-color: #8ab388;
    color: white;
  }

  .buttons{
    display: flex;
    flex-direction: row;
    justify-content: space-between;
  }
</style>
