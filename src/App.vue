<template>
  <div id='app'>
    <SpotifyAuth @completeLogin="completeLogin"/>
    <template v-if='token!=null'>
      <div id='playlist-grid' v-if='selectedPlaylist==null'>
        <Playlist v-for='list in playlists' @clicked="getTracksInPlaylist" :key="list.id" :data="list"/>
      </div>
      <template v-else>
        <div v-if='tracks!=null'>
          <h3>Selected {{countSelected}} Tracks</h3>
          <h4>X:</h4>
          <div class='range-slider'>
            <input type="range" min="0" max="1" step="0.001" v-model="sliderXMin" @change="updateGraphBox">
            <input type="range" min="0" max="1" step="0.001" v-model="sliderXMax" @change="updateGraphBox">
          </div>
          <h4>Y:</h4>
          <div class='range-slider'>
            <input type="range" min="0" max="1" step="0.001" v-model="sliderYMin" @change="updateGraphBox">
            <input type="range" min="0" max="1" step="0.001" v-model="sliderYMax" @change="updateGraphBox">
          </div>
          <input v-model="newPlaylistName" type='text'><button v-on:click='createPlaylist'>Create</button>
        </div>
        <h3 v-else>Loaded {{ tracksLoaded }} / {{selectedPlaylist.tracks.total}}</h3>
        <div id='chart-holder' v-if='selectedPlaylist!=null'>
          <canvas width="500px" height="500px" id="chart"></canvas>
        </div>
      </template>
    </template>
  </div>
</template>

<script>
import axios from 'axios';

import Playlist from './components/Playlist.vue';
import SpotifyAuth from './components/SpotifyAuth.vue';
import Chart from 'chart.js/auto';
import zoomPlugin from 'chartjs-plugin-zoom';
import annotationPlugin from 'chartjs-plugin-annotation';

export default {
  name: 'App',
  components: {
    Playlist,
    SpotifyAuth,
  },
  data (){
    return {
      token: null,
      xMin: 0.0,
      xMax: 1.0,
      yMin: 0.0,
      yMax: 1.0,
      spotifyDisplayName: "",
      spotifyUid: "",
      playlists: [],
      selectedPlaylist: null,
      tracksLoaded: 0,
      tracks: null,
      tracksCache: {},
      newPlaylistName: "My new playlist"
    }
  },
  computed: {
    countSelected: function(){
      if(this.tracks==null){
        return 0;
      }
      return this.getFilteredTracks().length;
    },
    sliderXMin: {
      get: function() {
        var val = Number(this.xMin);
        return val;
      },
      set: function(val) {
        val = Number(val);
        if (val > this.xMax) {
          this.xMax = val;
        }
        this.xMin = val;
        const graphBox = this.chart.options.plugins.annotation.annotations.box1;
        graphBox.xMin = this.xMin;
        graphBox.xMax = this.xMax;
        this.chart.update();
      }
    },
    sliderXMax: {
      get: function() {
        var val = Number(this.xMax);
        return val;
      },
      set: function(val) {
        val = Number(val);
        if (val < this.xMin) {
          this.xMin = val;
        }
        this.xMax = val;
        const graphBox = this.chart.options.plugins.annotation.annotations.box1;
        graphBox.xMin = this.xMin;
        graphBox.xMax = this.xMax;
        this.chart.update();
      }
    },
    sliderYMin: {
      get: function() {
        var val = Number(this.yMin);
        return val;
      },
      set: function(val) {
        val = Number(val);
        if (val > this.yMax) {
          this.yMax = val;
        }
        this.yMin = val;
        const graphBox = this.chart.options.plugins.annotation.annotations.box1;
        graphBox.yMin = this.yMin;
        graphBox.yMax = this.yMax;
        this.chart.update();
      }
    },
    sliderYMax: {
      get: function() {
        var val = Number(this.yMax);
        return val;
      },
      set: function(val) {
        val = Number(val);
        if (val < this.yMin) {
          this.yMin = val;
        }
        this.yMax = val;

        const graphBox = this.chart.options.plugins.annotation.annotations.box1;
        graphBox.yMin = this.yMin;
        graphBox.yMax = this.yMax;
        this.chart.update();
      }
    },
  },
  methods: {
    getFilteredTracks(){
      return this.tracks.filter(t => t.energy>=this.xMin && t.energy<=this.xMax && t.valence>=this.yMin && t.valence<=this.yMax);
    },
    async createPlaylist(){
      const newPlaylist = (await axios.post("https://api.spotify.com/v1/users/"+this.spotifyUid+"/playlists", {
        name:this.newPlaylistName,
        description: "Playlist generated from selectify"
      },{headers: {"Authorization": "Bearer " + this.token}})).data;
      let tracks = this.getFilteredTracks();
      while(tracks.length>0){
        let ids = tracks.slice(0,100).map(t => "spotify:track:"+(t?.id ?? ""));
        console.log((await axios.post("https://api.spotify.com/v1/playlists/"+newPlaylist.id+"/tracks", {
          uris: ids
        },{headers: {"Authorization": "Bearer " + this.token}})).data);
        tracks = tracks.slice(100);
      }

    },
    updateGraphBox(){
      const graphBox = this.chart.options.plugins.annotation.annotations.box1;
      graphBox.xMin = this.xMin;
      graphBox.xMax = this.xMax;
      graphBox.yMin = this.yMin;
      graphBox.yMax = this.yMax;
      this.chart.update();

    },
    async getTracksInPlaylist(playlist){
      console.log(playlist.id);
      this.selectedPlaylist = playlist;
      let arr = [];
      let promises = [];
      this.tracksLoaded = 0;
      for(let i=0; i<playlist.tracks.total; i+=50){
        promises.push(await this.getTrackFeatures(arr,playlist.id,i,50));
      }
      await Promise.all(promises);
      console.log(arr);
      console.log(this.tracksCache);
      this.tracks = arr;
      Chart.register(zoomPlugin);
      Chart.register(annotationPlugin);
      const data = {
        datasets: [{
          labels: this.tracks.map(t => t?.id ?? ""),
          label: "Tracks in "+playlist.name,
          data: this.tracks.map(t => {
              if(t==null || t.energy==null || t.danceability==null){
                  console.log(t);
              }
              return {
                  x: t?.energy ?? 0,
                  y: t?.valence ?? 0
              }
          }),
          backgroundColor: 'rgb(255, 0, 0)'
        }],
      };
      const config = {
        type: 'scatter',
        data: data,
        options: {
          responsive:true,
          scales: {
            x: {min: 0, max: 1},
            y: {min: 0, max: 1},
          },
          plugins: {
            annotation: {
              annotations: {
                box1: {
                  type: 'box',
                  xMin: 0.0,
                  xMax: 1.0,
                  yMin: 0.0,
                  yMax: 1.0,
                  backgroundColor: 'rgba(66, 130, 255, 0.25)'
                }
              }
            },
            tooltip: {
              callbacks: {
                  label: (data) => {
                    // console.log(this.tracksCache);
                    // console.log(data);
                    const track = this.tracksCache[data.dataset.labels[data.dataIndex]];
                    // console.log(track);
                    return "\""+track.name+"\" by "+track.artists.map(a => a.name).join(", ");
                  }
              }
            },
            zoom: {
              limits: {
                x: {min: 0, max: 1},
                y: {min: 0, max: 1},
              },
              pan: {
                enabled: true
              },
              zoom: {
                wheel: {
                  enabled: true,
                },
                pinch: {
                  enabled: true
                },
              }
            }
          }
        }
      };
      console.log(document.getElementById('chart'));
      this.chart = new Chart(
        document.getElementById('chart'),
        config
      );
    },
    async getTrackFeatures(arr,playlistId,offset,count){
      let tracks = await this.getSpotifyEndpoint("/playlists/"+playlistId+"/tracks?limit="+count.toString()+"&offset="+offset.toString());
      let filteredTracks = (tracks?.items ?? []).filter(t => t.is_local == false);
      if(filteredTracks.length>0){
        for(let t of filteredTracks){
          this.tracksCache[t.track.id] = t.track;
        }
        let trackIds = filteredTracks.map(t => t.track.id) ?? [];
        let features = await this.getSpotifyEndpoint("/audio-features?ids="+encodeURIComponent(trackIds.join(",")));
        arr.push(...(features.audio_features.filter(t => t!=null)));
        this.tracksLoaded = arr.length;
      }
    },

    async getPlaylists(){
      let newList = [];
      let nextUrl = "https://api.spotify.com/v1/me/playlists?limit=50";
      while(nextUrl!=null){
        let newSet = (await axios.get(nextUrl, {headers: {"Authorization": "Bearer " + this.token}})).data;
        nextUrl = newSet.next;
        newList.push(...(newSet.items ?? []));
      }
      newList.sort((a,b) => {
        if((a.owner.id === this.spotifyUid && b.owner.id === this.spotifyUid)
            || (a.owner.id !== this.spotifyUid && b.owner.id !== this.spotifyUid)){
          return b.tracks.total - a.tracks.total;
        }
        if(a.owner.id === this.spotifyUid && b.owner.id !== this.spotifyUid){
          return -1;
        }
        return 1;
      });
      this.playlists = newList;
      console.log(this.playlists[0]);
    },
    //helper function for any GET endpoint in the Spotify API
    async getSpotifyEndpoint(endpoint){
      return (await axios.get("https://api.spotify.com/v1" + endpoint,{headers: {"Authorization": "Bearer " + this.token}})).data;
    },
    async completeLogin(data){
      this.token = data.token;
      this.spotifyUid = data.spotifyUid;
      this.spotifyDisplayName = data.spotifyDisplayName;
      await this.getPlaylists();
    }
  },
}
</script>

<style>
#playlist-grid{
  margin: 30px;
  display: grid;
  grid-template-columns: auto auto auto auto;
  gap: 30px;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#chart-holder{
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

.range-slider input[type=range] {
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
}
.range-slider input[type=range]::-webkit-slider-runnable-track {
  width: 100%;
}
.range-slider input[type=range]::-webkit-slider-thumb {
  z-index: 2;
  position: relative;
}
</style>
