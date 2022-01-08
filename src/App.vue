<template>
  <div id='app'>
    <SpotifyAuth @completeLogin="completeLogin"/>
    <template v-if='token!=null'>
      <div id='playlist-grid' v-if='selectedPlaylist==null'>
        <Playlist v-for='list in playlists' @clicked="getTracksInPlaylist" :key="list.id" :data="list"/>
      </div>
      <Graph v-else 
        :tracks="tracks"
        :token="token"
        :spotifyUid="spotifyUid"
        :tracksCache="tracksCache"
        :tracksLoaded="tracksLoaded"
        :selectedPlaylist="selectedPlaylist"
      />
    </template>
  </div>
</template>

<script>
import axios from 'axios';

import Playlist from './components/Playlist.vue';
import SpotifyAuth from './components/SpotifyAuth.vue';
import Graph from './components/Graph.vue';

export default {
  name: 'App',
  components: {
    Playlist,
    SpotifyAuth,
    Graph,
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
@media screen and (max-width: 800px) {
  #playlist-grid{
    grid-template-columns: auto auto auto;
  }
}
@media screen and (max-width: 600px){
  #playlist-grid{
    grid-template-columns: auto auto;
  }
}
@media screen and (max-width: 400px){
  #playlist-grid{
    grid-template-columns: auto;
  }
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #ffffff;
  padding: 30px;
  height: calc(100vh - 60px);
  overflow-x: hidden;
  overflow-y: scroll;
}
#app::-webkit-scrollbar {
  width: 10px;
}
#app::-webkit-scrollbar-track {
  background: #333333;
  border-radius: 5px;
}
#app::-webkit-scrollbar-thumb {
  background: #666666; 
  border-radius: 5px;
}
#app::-webkit-scrollbar-thumb:hover {
  background: #888888;
}
</style>
