<template>
  <div id="wrapper">
    <div v-if="tracks != null">
      <h3>Selected {{ countSelected }} Tracks</h3>
      <h4>New Playlist Length:</h4>
      <input class='slider' type="range" min="0" :max='countSelected' step="1" v-model='playlistLength'><span>{{playlistLength}}</span><br>
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
          <!-- <option value="track">Selected Track</option> -->
        </select>
        <template v-if='distributionAround=="point"'>
            <h4>X:</h4>
            <input class='slider' type="range" min="0" max="1" step="0.001" v-model="distributionX"/>
            <h4>Y:</h4>
            <input class='slider' type="range" min="0" max="1" step="0.001" v-model="distributionY"/>
        </template>
        <br/>
      </template>
      <br/>
      <input type='text' v-model='query'><span> {{searchResults.length}} results</span><br/>
      <input v-model="newPlaylistName" type="text" /><button v-on:click="createPlaylist">
        Create
      </button>
    </div>
    <div v-else>
      <h3>Loading tracks</h3>
      <div class='loading-bar'>
        <div class='loading-bar-fill' :style="{width: loadedPercent + '%'}"></div>
      </div>
    </div>
    <main>
      <template v-if='tracks!=null'>
        <div class='feature-x'>
          <select v-model='featureX' @change='updateGraphFeatures()'>
            <option v-for="feature in featureOptions" :key="feature" :value="feature">{{ feature }}</option>
          </select>
        </div>
        <div class="range-slider slider-x">
          <input type="range" min="0" max="1" step="0.001" v-model="sliderXMin" @change="updateGraphBox"/>
          <input type="range" min="0" max="1" step="0.001" v-model="sliderXMax" @change="updateGraphBox"/>
        </div>
        <div class='feature-y'>
          <select v-model='featureY' class='feature-y' @change='updateGraphFeatures()'>
            <option v-for="feature in featureOptions" :key="feature" :value="feature">{{ feature }}</option>
          </select>
        </div>
        <div class="range-slider slider-y">
          <input orient="vertical" type="range" min="0" max="1" step="0.001" v-model="sliderYMin" @change="updateGraphBox"/>
          <input orient="vertical" type="range" min="0" max="1" step="0.001" v-model="sliderYMax" @change="updateGraphBox"/>
        </div>
      </template>
      <div id="chart-holder" v-if="selectedPlaylist != null">
        <canvas width="500px" height="500px" id="chart"></canvas>
      </div>
    </main>
  </div>
</template>

<script>
// import axios from "axios";
import zoomPlugin from "chartjs-plugin-zoom";
import annotationPlugin from "chartjs-plugin-annotation";
import Chart from "chart.js/auto";

export default {
  name: "Graph",
  data() {
    return {
      chart: null,
      xMin: 0.0, xMax: 1.0, yMin: 0.0, yMax: 1.0,
      playlistLength: 0,
      newPlaylistName: "My new playlist",
      distribution: "uniform",
      distributionAround: "center",
      distributionX: 0.5,
      distributionY: 0.5,
      hasCreatedGraph: false,
      featureX: "energy",
      featureY: "valence",
      featureOptions: ["acousticness", "danceability", "energy", "instrumentalness", "liveness", "speechiness", "valence"],
      query: "",

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
    query: function(){
      this.updateGraphFromSearch()
    },
      tracks: function(newVal){
        console.log(newVal);
        if(newVal!=null && !this.hasCreatedGraph){
            this.hasCreatedGraph = true;
            this.createGraph(); 
        }
      }
  },
  computed: {
    searchResults: function(){
      let reslt = this.tracks;
      if(this.query.length > 0){
        reslt = this.tracks.filter((t) => {
          const track = this.tracksCache[t.id];
          return track.name.toLowerCase().includes(this.query.toLowerCase())
          || track.artists.map((a) => a.name).join(" ").toLowerCase()
          .includes(this.query.toLowerCase());
        });
      }
      console.log(reslt.map(t => this.tracksCache[t.id].name));
      return reslt;
    },
    loadedPercent: function(){
      return Math.floor(this.tracksLoaded / this.selectedPlaylist.tracks.total*100);
    },
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
        this.xMin = val;Number(this.xMax);
        this.updateGraphBox();
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
    clamp(num,min,max){
      return Math.min(Math.max(num,min),max);
    },
    getFilteredTracks() {
      return this?.tracks?.filter(
        (t) => t[this.featureX] >= this.xMin && t[this.featureX] <= this.xMax &&
          t[this.featureY] >= this.yMin && t[this.featureY] <= this.yMax
      ) ?? 0;
    },
    async createPlaylist() {
    //   const newPlaylist = (await axios.post("https://api.spotify.com/v1/users/" + this.spotifyUid + "/playlists", {
    //         name: this.newPlaylistName,
    //         description: "Playlist generated from Spotlight",
    //       },{ headers: { Authorization: "Bearer " + this.token } }
    //     )).data;
        let tracks = this.getFilteredTracks();

        let filteredTracks = [];
        if(this.playlistLength>=tracks.length){
            filteredTracks = tracks;
        }
        else if(this.distribution === "uniform"){
          for(let i=0; i<this.playlistLength; i++){
              let index = Math.floor(Math.random() * tracks.length);
              filteredTracks.push(tracks[index]);
              tracks.splice(index, 1);
          }
        }
        else{
          let center = {x:0,y:0}
          if(this.distributionAround === "center"){
              center = {x: (this.xMin + this.xMax)/2, y: (this.yMin+this.yMax)/2};
          }
          let totalScore = 0;
          for(let t of tracks){
              let distX = t[this.featureX] - center.x;
              let distY = t[this.featureY] - center.y;
              let distSq = distX * distX + distY * distY;
              let score = 0;
              if(this.distribution == "dist"){
                  score = 1/(Math.sqrt(distSq));
              }
              else{
                  score = 1/(distSq * 100);
              }
              t.score = score;
              totalScore += score;
          }
          console.log(center,totalScore);
          for(let i=0; i<this.playlistLength; i++){
              let score = Math.random() * totalScore;
              let index = 0;
              while(index<tracks.length && tracks[index].score<score){
                  score -= tracks[index].score;
                  index++;
              }
              filteredTracks.push(tracks[index]);
              totalScore -= tracks[index].score;
              tracks.splice(index, 1);
          }
        }
        console.log(filteredTracks.map(t => this.tracksCache[t.id].name));
        console.log(filteredTracks.map(t => t.score));
        // while (filteredTracks.length > 0) {
        //     let ids = filteredTracks.slice(0, 100).map((t) => "spotify:track:" + (t?.id ?? ""));
        //     console.log((await axios.post("https://api.spotify.com/v1/playlists/" +newPlaylist.id +"/tracks",
        //         { uris: ids,}, { headers: { Authorization: "Bearer " + this.token } }
        //     )).data);
        //     filteredTracks = filteredTracks.slice(100);
        // }
    },
    updateGraphBox() {
      const graphBox = this.chart.options.plugins.annotation.annotations.box1;
      graphBox.xMin = this.xMin;
      graphBox.xMax = this.xMax;
      graphBox.yMin = this.yMin;
      graphBox.yMax = this.yMax;
      this.chart.update();
      if(this.playlistLength>this.countSelected){
          this.playlistLength = this.countSelected;
      }
    },
    updateGraphFeatures(){
      const data = this.chart.data.datasets[0].data;
      for(let i=0; i<this.tracks.length; i++){
          data[i].x = this.clamp(this.tracks[i][this.featureX] ?? 0,0,1);
          data[i].y = this.clamp(this.tracks[i][this.featureY] ?? 0,0,1);
      }
      this.chart.update();
    },
    updateGraphFromSearch(){
      const data = this.chart.data.datasets[0].data;
      for(let i=0; i<data.length; i++){
        const track = this.tracksCache[data[i].id];
        data[i].disabled = !(track.name.toLowerCase().includes(this.query.toLowerCase())
          || track.artists.map((a) => a.name).join(" ").toLowerCase()
          .includes(this.query.toLowerCase()));
      }
      this.chart.data.datasets[0].pointBackgroundColor = data.map((d) => d.disabled ? "#888888" : "#ff0000");
      this.chart.update();
    },
    createGraph() {
      Chart.register(zoomPlugin);
      Chart.register(annotationPlugin);
      const data = {
        datasets: [
          {
            data: this.tracks.map((t) => {
              return {
                x: t[this.featureX] ?? 0,
                y: t[this.featureY] ?? 0,
                id: t.id ?? 0,
                disabled: false,
              };
            }),
            pointBackgroundColor: new Array(this.tracks.length).fill("rgb(255,0,0)")
          }
        ],
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
            legend: {
              display: false,
            },
            annotation: {
              annotations: {
                box1: {
                  type: "box",
                  xMin: 0.0, xMax: 1.0,
                  yMin: 0.0, yMax: 1.0,
                  backgroundColor: "rgba(255, 255, 0, 0.3)",
                },
              },
            },
            tooltip: {
              // filter: (tooltip) =>{
              //   return !tooltip.dataset.data[tooltip.dataIndex].disabled;
              // },
              callbacks: {
                label: (data) => {
                  const track = this.tracksCache[data.dataset.data[data.dataIndex].id];
                  return (
                    '"' + track.name + '" by ' +
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
      this.playlistLength = Math.min(Math.floor(this.tracks.length / 4),50);
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
.range-slider input[type="range"][orient="vertical"] {
  writing-mode: bt-lr; /* IE */
  -webkit-appearance: slider-vertical; /* WebKit */
  width: 1em;
  height: 100%;
}
.slider{
    width: 500px;
}

main{
  justify-content: center;
  display: grid;
  grid-template-columns: 200px 1em 500px;
  grid-template-rows: 500px 1em auto;
  gap: 10px;
  /* width: 100%; */
  /* height: 600px; */
}
#chart-holder{
  grid-column: 3;
  grid-row: 1;
}
.feature-y{
  grid-column: 1;
  grid-row: 1;
}
.feature-y select{
    margin: 0;
    position: relative;
    top: 50%;
    -ms-transform: translateY(-50%);
    transform: translateY(-50%);
}
.slider-y{
  grid-column: 2;
  grid-row: 1;
  height: 100%;
  width: 1em;
}
.feature-x{
  grid-column: 3;
  grid-row: 3;
}
.slider-x{
  grid-column: 3;
  grid-row: 2;
}

.loading-bar{
  background-color: #333333;
  width: 80vw;
  max-width: 500px;
  height: 10px;
  border-radius: 50px;
  margin: auto;
}
.loading-bar-fill{
  background-color: #ff0000;
  height: 10px;
  width: 0%;
  border-radius: 50px;
}
#chart{
  z-index: 500;
}
</style>