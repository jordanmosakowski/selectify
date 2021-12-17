<template>
  <div id='app'>
    <template v-if='token!=null'>
      <h1>Welcome {{ spotifyDisplayName }}</h1>
      <div id='playlist-grid'>
        <Playlist v-for='list in playlists' @clicked="getTracksInPlaylist" :key="list.id" :data="list"/>
      </div>
    </template>
    <button v-else @click='login'>Login</button>
  </div>
</template>

<script>
import axios from 'axios';
import crypto from "crypto";
import base64url from "base64url";

import Playlist from './components/Playlist.vue';
const clientId = "6180cb2c2ce74ec99e0e648ff2c3d79f";

export default {
  name: 'App',
  components: {
    Playlist
  },
  data (){
    return {
      token: null,
      spotifyDisplayName: "",
      spotifyUid: "",
      playlists: [],
      // tracksCache: [],
    }
  },
  methods: {
    async getTracksInPlaylist(playlist){
      console.log(playlist.id);
      let arr = [];
      let promises = [];
      for(let i=0; i<playlist.tracks.total; i+=20){
        promises.push(await this.getTrackFeatures(arr,playlist.id,i,20));
      }
      await Promise.all(promises);
      console.log(arr);
      console.log(this.tracksCache);
    },
    async getTrackFeatures(arr,playlistId,offset,count){
      let tracks = await this.getSpotifyEndpoint("/playlists/"+playlistId+"/tracks?limit="+count.toString()+"&offset="+offset.toString());
      let filteredTracks = (tracks?.items ?? []).filter(t => t.is_local == false);
      if(filteredTracks.length>0){
        // this.tracksCache.push(...filteredTracks);
        let trackIds = filteredTracks.map(t => t.track.id) ?? [];
        let features = await this.getSpotifyEndpoint("/audio-features?ids="+encodeURIComponent(trackIds.join(",")));
        arr.push(...(features.audio_features));
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

    async checkForToken(){
      if(this.token!=null){
        return;
      }
      //Check if the token is saved in local storage
      let dataStr = localStorage.getItem("token-info");
      if(dataStr == null){
        //check if the user has an auth code that needs to be converted into a token
        await this.checkForAuthCode();
        return;
      }
      let oldData = JSON.parse(dataStr);
      let now = new Date();
      //Check if the token in local storage is still valid
      if(oldData.expires>now.getTime()){
        this.token = oldData.access;
        return;
      }
      //If the token has expired, request a new token
      const formData = {
        client_id: clientId,
        grant_type: "refresh_token",
        refresh_token: oldData.refresh
      };
      await this.fetchToken(formData);
    },

    async checkForAuthCode(){
      if(this.token !=null){
        return;
      }
      const urlParams = new URLSearchParams(window.location.search);
      let code = urlParams.get("code");
      console.log(code);
      if(code == null){
        return;
      }
      //Convert the auth code into an access token
      const formData = {
        client_id: clientId,
        grant_type: "authorization_code",
        code: code,
        redirect_uri: window.location.origin,
        code_verifier: localStorage.getItem("verifier")
      };
      await this.fetchToken(formData);
      //clean up url
      const uri = window.location.href;
      if (uri.indexOf("?") > 0) {
        var clean_uri = uri.substring(0, uri.indexOf("?"));
        window.history.replaceState({}, document.title, clean_uri);
      }
    },

    async fetchToken(formData){
      //request an access token using the form data
      let body = Object.keys(formData).map((a) => `${a}=${encodeURIComponent(formData[a])}`).join("&");
      const response = await axios.post('https://accounts.spotify.com/api/token', body, {
        headers: {
          "Content-Type": "application/x-www-form-urlencoded"
        },
      });
      //save the access token, refresh token, and expiration time in localstorage
      let json = response.data;
      var t = new Date();
      t.setSeconds(t.getSeconds() + json.expires_in);
      let data = {
        access: json.access_token,
        refresh: json.refresh_token,
        expires: t.getTime()
      }
      this.token = json.access_token;
      localStorage.setItem("token-info",JSON.stringify(data));
    },

    login(){
      localStorage.removeItem("token-info");
      localStorage.removeItem("verifier");
      const buffer  = crypto.randomBytes(48);
      const verifier = buffer.toString('hex');
      localStorage.setItem("verifier",verifier);
      let challenge = crypto.createHash("sha256").update(verifier).digest("base64");
      challenge = base64url.fromBase64(challenge);
      const scopes = ["playlist-modify-private","playlist-read-private","playlist-read-collaborative","playlist-modify-public"];
      const redirectUrl = window.location.origin;
      //redirect to the spotify auth website with the correct scopes and client id.
      window.location.href = "https://accounts.spotify.com/authorize?response_type=code&client_id=" + clientId +
          "&redirect_uri="+encodeURIComponent(redirectUrl)+"&scope=" + scopes.join("%20") +
          "&code_challenge=" + challenge + "&code_challenge_method=S256";
    },
    //helper function for any GET endpoint in the Spotify API
    async getSpotifyEndpoint(endpoint){
      return (await axios.get("https://api.spotify.com/v1" + endpoint,{headers: {"Authorization": "Bearer " + this.token}})).data;
    }
  },
  created: async function () {
    await this.checkForToken();
    if(this.token!=null){
      const me = (await this.getSpotifyEndpoint("/me"));
      this.spotifyDisplayName = me.display_name;
      this.spotifyUid = me.id;
      this.getPlaylists();
    }
  }
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
</style>
