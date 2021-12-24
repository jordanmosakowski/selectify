<template>
  <div id="wrapper">
    <div v-if="tracks != null">
      <h3>Selected {{ countSelected }} Tracks</h3>
      <h4>X:</h4>
      <div class="range-slider">
        <input type="range" min="0" max="1" step="0.001" v-model="sliderXMin" @change="updateGraphBox"/>
        <input type="range" min="0" max="1" step="0.001" v-model="sliderXMax" @change="updateGraphBox"/>
      </div>
      <h4>Y:</h4>
      <div class="range-slider">
        <input type="range" min="0" max="1" step="0.001" v-model="sliderYMin" @change="updateGraphBox"/>
        <input type="range" min="0" max="1" step="0.001" v-model="sliderYMax" @change="updateGraphBox"/>
      </div>
      <label for="distribution">Distribution:</label>
      <select name="distribution" id="distribution" v-model="distribution">
        <option value="uniform">Uniform</option>
        <option value="dist">Distance</option>
        <option value="distSq">Distance Squared</option>
        </select><br />
      <template v-if="distribution != 'uniform'">
        <label for="distributionAround">Calculate distance from:</label>
        <select name="distributionAround" id="distributionAround" v-model="distributionAround">
          <option value="center">Box Center</option>
          <option value="point">Selected Point</option>
          <option value="track">Selected Track</option>
        </select>
        <br />
      </template>
      <br />
      <input v-model="newPlaylistName" type="text" /><button v-on:click="createPlaylist">
        Create
      </button>
    </div>
    <h3 v-else>
      Loaded {{ tracksLoaded }} / {{ selectedPlaylist.tracks.total }}
    </h3>
    <div id="chart-holder" v-if="selectedPlaylist != null">
      <canvas width="500px" height="500px" id="chart"></canvas>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import zoomPlugin from "chartjs-plugin-zoom";
import annotationPlugin from "chartjs-plugin-annotation";
import Chart from "chart.js/auto";

export default {
  name: "Graph",
  data() {
    return {
      chart: null,
      xMin: 0.0, xMax: 1.0, yMin: 0.0, yMax: 1.0,
      newPlaylistName: "My new playlist",
      distribution: "uniform",
      distributionAround: "center",
      distributionX: 0.5,
      distributionY: 0.5,
      hasCreatedGraph: false,
    };
  },
  props: {
    tracks: Array,
    token: String,
    spotifyUid: String,
    tracksCache: Object,
    tracksLoaded: Number,
    selectedPlaylist: Object,
  },
  watch: {
      tracks: function(newVal){
        console.log(newVal);
        if(newVal!=null && !this.hasCreatedGraph){
            this.hasCreatedGraph = true;
            this.createGraph(); 
        }
      }
  },
  computed: {
    countSelected: function () {
      return this.tracks == null ? 0 : this.getFilteredTracks().length;
    },
    sliderXMin: {
      get: function () {
        return Number(this.xMin);
      },
      set: function (val) {
        val = Number(val);
        if (val > this.xMax) {this.xMax = val;}
        this.xMin = val;Number(this.xMax)
      },
    },
    sliderXMax: {
      get: function () {
        return Number(this.xMax);
      },
      set: function (val) {
        val = Number(val);
        if (val < this.xMin) {this.xMin = val;}
        this.xMax = val;
        this.updateGraphBox();
      },
    },
    sliderYMin: {
      get: function () {
        return Number(this.yMin);
      },
      set: function (val) {
        val = Number(val);
        if (val > this.yMax) {this.yMax = val;}
        this.yMin = val;
        this.updateGraphBox();
      },
    },
    sliderYMax: {
      get: function () {
        return Number(this.yMax);
      },
      set: function (val) {
        val = Number(val);
        if (val < this.yMin) { this.yMin = val;}
        this.yMax = val;
        this.updateGraphBox();
      },
    },
  },
  methods: {
    getFilteredTracks() {
      return this.tracks.filter(
        (t) => t.energy >= this.xMin && t.energy <= this.xMax &&
          t.valence >= this.yMin && t.valence <= this.yMax
      );
    },
    async createPlaylist() {
      const newPlaylist = (await axios.post("https://api.spotify.com/v1/users/" + this.spotifyUid + "/playlists",
          {
            name: this.newPlaylistName,
            description: "Playlist generated from selectify",
          },
          { headers: { Authorization: "Bearer " + this.token } }
        )).data;
      let tracks = this.getFilteredTracks();
      while (tracks.length > 0) {
        let ids = tracks.slice(0, 100).map((t) => "spotify:track:" + (t?.id ?? ""));
        console.log((await axios.post("https://api.spotify.com/v1/playlists/" +newPlaylist.id +"/tracks",
              {
                uris: ids,
              },
              { headers: { Authorization: "Bearer " + this.token } }
        )).data);
        tracks = tracks.slice(100);
      }
    },
    updateGraphBox() {
      const graphBox = this.chart.options.plugins.annotation.annotations.box1;
      graphBox.xMin = this.xMin;
      graphBox.xMax = this.xMax;
      graphBox.yMin = this.yMin;
      graphBox.yMax = this.yMax;
      this.chart.update();
    },
    createGraph() {
      Chart.register(zoomPlugin);
      Chart.register(annotationPlugin);
      const data = {
        datasets: [{
            labels: this.tracks.map((t) => t?.id ?? ""),
            label: "Tracks in " + this.selectedPlaylist.name,
            data: this.tracks.map((t) => {
              return {
                x: t?.energy ?? 0,
                y: t?.valence ?? 0,
              };
            }),
            backgroundColor: "rgb(255, 0, 0)",
        }],
      };
      const config = {
        type: "scatter",
        data: data,
        options: {
          responsive: true,
          scales: {
            x: { min: 0, max: 1 },
            y: { min: 0, max: 1 },
          },
          plugins: {
            annotation: {
              annotations: {
                box1: {
                  type: "box",
                  xMin: 0.0, xMax: 1.0,
                  yMin: 0.0, yMax: 1.0,
                  backgroundColor: "rgba(66, 130, 255, 0.25)",
                },
              },
            },
            tooltip: {
              callbacks: {
                label: (data) => {
                  const track = this.tracksCache[data.dataset.labels[data.dataIndex]];
                  return (
                    '"' +
                    track.name +
                    '" by ' +
                    track.artists.map((a) => a.name).join(", ")
                  );
                },
              },
            },
            zoom: {
              limits: {
                x: { min: 0, max: 1 },
                y: { min: 0, max: 1 },
              },
              pan: {
                enabled: true,
              },
              zoom: {
                wheel: {
                  enabled: true,
                },
                pinch: {
                  enabled: true,
                },
              },
            },
          },
        },
      };
      console.log(document.getElementById("chart"));
      this.chart = new Chart(document.getElementById("chart"), config);
    },
  },
};
</script>

<style scoped>
#chart-holder {
  width: 500px;
  height: 500px;
}

.range-slider {
  width: 500px;
  margin: auto;
  text-align: center;
  position: relative;
  height: 1em;
}

.range-slider input[type="range"] {
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
}
.range-slider input[type="range"]::-webkit-slider-runnable-track {
  width: 100%;
}
.range-slider input[type="range"]::-webkit-slider-thumb {
  z-index: 2;
  position: relative;
}
</style>