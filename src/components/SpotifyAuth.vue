<template>
    <div>
        <template v-if='token!=null'>
        <h1>Welcome {{ spotifyDisplayName }}</h1>
        </template>
        <div v-else class='center main' style=''>
            <div class='background'></div>
            <div class='center body'>
                <h1>Spotlight</h1>
                <button @click='login'><span><img src='@/assets/spotify.svg'>Login with Spotify</span></button>
            </div>
            
        </div>
        
    </div>
</template>

<script>
import axios from 'axios';
import crypto from "crypto";
import base64url from "base64url";

const clientId = "6180cb2c2ce74ec99e0e648ff2c3d79f";

export default {
    name: "SpotifyAuth",
    
    data (){
        return {
            token: null,
            spotifyDisplayName: "",
            spotifyUid: "",
        }
    },
    created: async function () {
        await this.checkForToken();
        if(this.token!=null){
            const me = (await this.getSpotifyEndpoint("/me"));
            this.spotifyDisplayName = me.display_name;
            this.spotifyUid = me.id;
            this.$emit("completeLogin",{
                token: this.token,
                spotifyDisplayName: me.display_name,
                spotifyUid: me.id
            })
        }
    },
    methods: {
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
    }
}
</script>

<style scoped>
.main{
    width: 100vw;
    max-width: 400px;
    height: 100vw;
    max-height: 400px;
    background-color: rgb(150, 150, 0) !important;
    border-width: 200vh 200vw;
    border-style: solid;
    border-color: rgb(150, 150, 0);
}
button{
    width: 80vw;
    max-width: 225px;
    border: none;
    background-color: #1DB954;
    color: white;
    font-size: 1em;
    font-weight: bold;
    border-radius: 50px;
    padding: 10px 5px;
    cursor: pointer;
}
button img{
    width: 1em;
    height: 1em;
    color: white;
    object-fit: contain;
    filter: invert(1);
    margin-right: 10px;
    margin-bottom: 5px;
    vertical-align:middle;
}
.center{
    margin: 0;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
}
.center.body{
    transform: translate(-50%, calc(-50% - 20px));
}
@keyframes slideInFromLeft {
  0% {
    left: calc(-250vw - 225px);
  }
  100% {
    left: -200vw;
  }
}
.background{
    pointer-events: none;
    height: 100%;
    width: 100%;
    /* background-color: #ffff00; */
    position: absolute;
    z-index: 100;
    border-radius: 100%;
    border-width: 200vh 200vw;
    border-style: solid;
    border-color: #000;
    top: -200vh;
    left: -200vw;
    /* opacity: 0.4; */
    animation: 3s ease-out 0s 1 slideInFromLeft;
    background: radial-gradient(ellipse at center,  rgba(255,0,0,0) 0%,rgba(0,0,0,0) 50%,rgba(0,0,0,1) 60%,rgb(0, 0, 0,1) 100%);
}
.background-spotlight{
    z-index: -1;
    height: 100vh;
    width: 100vw;
    background-color: #ffff00;
    opacity: 0.4;
    border-radius: 100%;

}
</style>