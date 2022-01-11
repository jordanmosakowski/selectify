<template>
  <div id="wrapper">
    <div v-if="tracks != null">
      <input type='text' class='search' placeholder="Search" v-model='query'>
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
          <input type="range" class='slider-min' min="0" max="1" step="0.001" v-model="sliderXMin" @change="updateGraphBox"/>
          <input type="range" min="0" max="1" step="0.001" v-model="sliderXMax" @change="updateGraphBox"/>
        </div>
        <div class='feature-y'>
          <select v-model='featureY' class='feature-y' @change='updateGraphFeatures()'>
            <option v-for="feature in featureOptions" :key="feature" :value="feature">{{ feature }}</option>
          </select>
        </div>
        <div class="range-slider slider-y">
          <input orient="vertical" class='slider-min' type="range" min="0" max="1" step="0.001" v-model="sliderYMin" @change="updateGraphBox"/>
          <input orient="vertical" type="range" min="0" max="1" step="0.001" v-model="sliderYMax" @change="updateGraphBox"/>
        </div>
      </template>
      <div id="chart-holder" v-if="selectedPlaylist != null">
        <canvas width="500px" height="500px" id="chart"></canvas>
      </div>
    </main>
    <div v-if="tracks != null">
      <h3>Selected {{ countSelected }} Tracks</h3>
      <h4>Tracks to Include:</h4>
      <input class='slider' type="range" min="0" :max='countSelected' step="1" v-model='playlistLength'><span> {{playlistLength}}</span><br>
      <h4>Recommendations to Add:</h4>
      <input class='slider' type="range" min="0" max='100' step="1" v-model='recommendations'><span> {{recommendations}}</span><br>
      <!-- <label for="distribution">Distribution:</label> -->
      <!-- <select name="distribution" id="distribution" v-model="distribution" @change='updateDistributionMethod()'>
        <option value="uniform">Uniform</option>
        <option value="dist">Distance</option>
        <option value="distSq">Distance Squared</option>
        </select><br /> -->
      <template v-if="distribution != 'uniform'">
        <label for="distributionAround">Calculate distance from:</label>
        <select name="distributionAround" id="distributionAround" v-model="distributionAround" @change='updateDistributionMethod()'>
          <option value="center">Box Center</option>
          <option value="point">Selected Point</option>
          <!-- <option value="track">Selected Track</option> -->
        </select>
        <br/>
      </template>
      <br/>
      <!-- <span> {{searchResults.length}} results</span> -->
      <br/>
      <div class='create-playlist'>
        <input v-model="newPlaylistName" type="text" />
        <button v-on:click="createPlaylist">
          Create
        </button>
      </div>
    </div>
  </div>
</template>

<script>
// import axios from "axios";
import zoomPlugin from "chartjs-plugin-zoom";
import annotationPlugin from "chartjs-plugin-annotation";
import 'chartjs-plugin-dragdata'
import {Chart,registerables} from "chart.js";
import axios from 'axios';

export default {
  name: "Graph",
  data() {
    return {
      chart: null,
      xMin: 0.4, xMax: 0.6, yMin: 0.4, yMax: 0.6,
      playlistLength: 0,
      newPlaylistName: "My New Playlist",
      distribution: "uniform",
      distributionAround: "center",
      distributionX: 0.5,
      distributionY: 0.5,
      hasCreatedGraph: false,
      featureX: "energy",
      featureY: "danceability",
      featureOptions: ["acousticness", "danceability", "energy", "instrumentalness", "liveness", "popularity", "speechiness", "valence"],
      query: "",
      recommendations: 10,
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
        const newPlaylist = (await axios.post("https://api.spotify.com/v1/users/" + this.spotifyUid + "/playlists", {
            name: this.newPlaylistName,
            description: "Playlist generated from Spotlight",
          },{ headers: { Authorization: "Bearer " + this.token } }
        )).data;
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
          let center = this.chart.data.datasets[1].data[0];
          let totalScore = 0;
          for(let t of tracks){
              let distX = t[this.featureX] - center.x;
              let distY = t[this.featureY] - center.y;
              let distSq = distX * distX + distY * distY;
              if(distSq == 0){
                distSq = 0.0000000000001; // avoid divide by zero
              }
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
        
        if(this.recommendations>0){
          if(this.distribution!="uniform"){
            filteredTracks.sort((a,b) => b.score - a.score);
          }
          console.log(filteredTracks);
          const seedIds = filteredTracks.slice(0,5).map(t => t.id);
          console.log(seedIds);
          const params = {
              seed_tracks: seedIds.join(","),
              limit: 100,
          };
          //set min and max features for X and Y
          params["min_" + this.featureX] = this.xMin;
          params["max_" + this.featureX] = this.xMax;
          params["min_" + this.featureY] = this.yMin;
          params["max_" + this.featureY] = this.yMax;
          //scale by 100 and round if feature is popularity
          if(this.featureX=="popularity"){
            params["min_" + this.featureX] = Math.floor(this.xMin*100);
            params["max_" + this.featureX] = Math.floor(this.xMax*100);
          }
          if(this.featureY=="popularity"){
            params["min_" + this.featureY] = Math.floor(this.yMin*100);
            params["max_" + this.featureY] = Math.floor(this.yMax*100);
          }
          //if distribution isn't uniform, set target feature for x and y to chart center
          if(this.distribution!="uniform"){
            params["target_" + this.featureX] = this.chart.data.datasets[1].data[0].x;
            params["target_" + this.featureY] = this.chart.data.datasets[1].data[0].y;
            //scale by 100 and round if feature is popularity
            if(this.featureX=="popularity"){
              params["target_" + this.featureX] = Math.floor(params["target_" + this.featureX]*100);
            }
            if(this.featureY=="popularity"){
              params["target_" + this.featureY] = Math.floor(params["target_" + this.featureY]*100);
            }
          }
          console.log(params);
          const recommendations = (await axios.get("https://api.spotify.com/v1/recommendations", {
            params: params,
            headers: { Authorization: "Bearer " + this.token }
          })).data.tracks;
          console.log(recommendations);
          let fullList = this.getFilteredTracks();
          for(let i=0; i<this.recommendations; i++){
            if(recommendations.length==0){
              continue;
            }
            //Add the first recommendation that isn't in tracks
            let index = 0;
            while(index<recommendations.length && fullList.map(t => t.id).includes(recommendations[index].id)){
              index++;
            }
            if(index<recommendations.length){
              filteredTracks.push(recommendations[index]);
              recommendations.splice(index, 1);
              continue;
            }
            index = 0;
            while(index<recommendations.length && filteredTracks.map(t => t.id).includes(recommendations[index].id)){
              index++;
            }
            if(index<recommendations.length){
              filteredTracks.push(recommendations[index]);
              recommendations.splice(index, 1);
            }
          }
        }

        console.log(filteredTracks.map(t => {
          //return track name if it is in cache
          if(this.tracksCache[t.id]){
            return this.tracksCache[t.id].name + " by " + this.tracksCache[t.id].artists.map(a => a.name).join(", ");
          }
          //return t.name and t.artists
          else{
            return "(REC) "+t.name + " by " + t.artists.map(a => a.name).join(", ");
          }
        }));

        while (filteredTracks.length > 0) {
            let ids = filteredTracks.slice(0, 100).map((t) => "spotify:track:" + (t?.id ?? ""));
            console.log((await axios.post("https://api.spotify.com/v1/playlists/" +newPlaylist.id +"/tracks",
                { uris: ids,}, { headers: { Authorization: "Bearer " + this.token } }
            )).data);
            filteredTracks = filteredTracks.slice(100);
        }
        window.open(newPlaylist.external_urls.spotify, "_blank");
    },
    updateGraphBox() {
      const graphBox = this.chart.options.plugins.annotation.annotations.box1;
      graphBox.xMin = this.xMin;
      graphBox.xMax = this.xMax;
      graphBox.yMin = this.yMin;
      graphBox.yMax = this.yMax;
      this.chart.update();
      if(this.distribution!= "uniform" && this.distributionAround == "center"){
        this.updateDistributionMethod(); 
      }
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
      this.chart.data.datasets[0].pointBackgroundColor = data.map((d) => d.disabled ? "#888888" : "#ffb300");
      this.chart.update();
    },
    updateDistributionMethod(){
      const dot = this.chart.data.datasets[1];
      if(this.distribution === "uniform"){
        dot.radius = 0;
        dot.hoverRadius = 0;
        dot.dragData = false;
      }
      else{
        dot.radius = 5;
        dot.hoverRadius = 7;
        if(this.distributionAround == 'point'){
            dot.dragData = true;
        }
        else{
          dot.dragData = false;
          dot.data[0].x = (this.xMin + this.xMax)/2;
          dot.data[0].y = (this.yMin+this.yMax)/2;
        }
      }
      this.chart.update();
    },
    createGraph() {
      Chart.register(zoomPlugin);
      Chart.register(annotationPlugin);
      Chart.register(...registerables);
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
            dragData: false,
            pointBackgroundColor: new Array(this.tracks.length).fill("#ffb300")
          },
          {
            data: [
              {
                x: this.distributionX,
                y: this.distributionY,
              },
            ],
            radius: 0,
            hoverRadius: 0,
            dragData: true,
            dragX: true,
            backgroundColor: "#00ff00"
          }
        ],
      };
      const config = {
        type: "scatter",
        data: data,
        options: {
          responsive: true,
          scales: {
            x: { 
              min: 0, max: 1,
              grid: {
                color: "#666666",
              }
            },
            y: { 
              min: 0, max: 1,
              grid: {
                color: "#666666",
              }
            },
          },
          plugins: {
            dragData: {
              round: 3,
              dragX: true
            },
            legend: {
              display: false,
            },
            annotation: {
              annotations: {
                box1: {
                  drawTime: "beforeDatasetsDraw",
                  type: "box",
                  xMin: 0.4, xMax: 0.6,
                  yMin: 0.4, yMax: 0.6,
                  backgroundColor: "rgba(255, 255, 0, 0.3)",
                },
              },
            },
            tooltip: {
              filter: (tooltip) =>{
                return tooltip.datasetIndex === 0;
              },
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
input[type="range"]::-webkit-slider-runnable-track{
  background-color: #333333;
  border-radius: 20px;
}

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
input[type="range"]::-webkit-slider-thumb {
  z-index: 10;
  position: relative;
}
.range-slider input[type="range"][orient="vertical"] {
  /* writing-mode: bt-lr; */
  /* -webkit-appearance: slider-vertical; */
  /* -webkit-transform:rotate(90deg); */
  height: 1.5em;
  width: 500px;
  position: absolute;
  top: 240px;
  left: -250px;
  /* height: 100%; */
}
.range-slider input[type="range"][orient="vertical"]::-webkit-slider-thumb{
  left: 1px;
}

.slider{
    width: 500px;
}

main{
  justify-content: center;
  display: grid;
  grid-template-columns: auto 1em 500px;
  grid-template-rows: 500px 0.25em 3em;
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
    z-index: 100;
}
.slider-y{
  grid-column: 2;
  grid-row: 1;
  height: 100%;
  width: 1em;
  transform: rotate(270deg);
  z-index: 10;
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
  background-color: #ffb300;
  height: 10px;
  width: 0%;
  border-radius: 50px;
}
#chart{
  z-index: 500;
}
.search{
  width: 80vw;
  max-width: 500px;
  background-color: #333333;
  font-size: 1.5em;
  color: #ffffff;
  border-radius: 50px;
  padding: 10px 20px;
  margin: 20px auto;
  border: none;
}
input:focus, select:focus{
  outline: none;
}

.create-playlist{
  width: 80vw;
  max-width: 500px;
  background-color: #333333;
  font-size: 1.5em;
  color: #ffffff;
  border-radius: 50px;
  margin: 20px auto;
  border: none;
}
.create-playlist button{
  background-color: #ffb300;
  color: #ffffff;
  border: none;
  padding: 10px 20px;
  border-radius: 50px;
  width: 100px;
  font-weight: bold;
  font-size: 0.8em;
}

.create-playlist input{
  width: calc(100% - 140px);
  background-color: #333333;
  font-size: 0.7em;
  color: #ffffff;
  border-radius: 50px;
  padding-left: 20px;
  padding-right: 20px;
  border: none;
}


select{
  background-color: #333333;
  font-size: 1em;
  color: #ffffff;
  border-radius: 50px;
  padding: 8px 5px;
  margin: 20px auto;
  border: none;
}

</style>