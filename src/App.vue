<template>
  <div id='app'>
    <SpotifyAuth @completeLogin="completeLogin"/>
    <template v-if='token!=null'>
      <div id='playlist-grid' v-if='selectedPlaylist==null'>
        <Playlist v-for='list in playlists' @clicked="getTracksInPlaylist" :key="list.id" :data="list"/>
      </div>
      <template v-else>
        <h3>Loaded {{ tracksLoaded }} / {{selectedPlaylist.tracks.total}}</h3>
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

export default {
  name: 'App',
  components: {
    Playlist,
    SpotifyAuth,
  },
  data (){
    return {
      token: null,
      spotifyDisplayName: "",
      spotifyUid: "",
      playlists: [],
      selectedPlaylist: null,
      tracksLoaded: 0,
      tracks: null,
      tracksCache: {},
    }
  },
  methods: {
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
          scales: {
            x: {min: 0, max: 1},
            y: {min: 0, max: 1},
          },
          plugins: {
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
</style>
