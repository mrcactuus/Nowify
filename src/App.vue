<template>
  <div id="app">
    <Component
      :is="getCurrentComponent"
      :auth="auth"
      :endpoints="endpoints"
      :player="player"
      @spotifyTrackUpdated="updateCurrentTrack"
      @requestRefreshToken="requestRefreshTokens"
    ></Component>
  </div>
</template>

<script>
import Authorise from '@/components/Authorise'
import NowPlaying from '@/components/NowPlaying'

import { getStoredAuth, setStoredAuth } from '@/utils/utils.js'

export default {
  name: 'App',

  components: {
    Authorise,
    NowPlaying
  },

  props: {},

  data() {
    return {
      storedAuth: '',
      test: 'hello, world',
      auth: {
        status: false,
        clientId: process.env.VUE_APP_SP_CLIENT_ID,
        clientSecret: process.env.VUE_APP_SP_CLIENT_SECRET,
        authCode: '',
        accessToken: '',
        refreshToken: '',
      /**
      * added for two user accounts
      */
        clientId1: process.env.SPOTIFY_CLIENT_ID_USER1,
        refreshToken1: process.env.SPOTIFY_REFRESH_TOKEN_USER1,
        clientId2: process.env.SPOTIFY_CLIENT_ID_USER2,
        refreshToken2: process.env.SPOTIFY_REFRESH_TOKEN_USER2
      },
      endpoints: {
        auth: 'https://accounts.spotify.com/authorize',
        token: 'https://accounts.spotify.com/api/token',
        base: 'https://api.spotify.com/v1',
        nowPlaying: 'me/player/currently-playing'
      },
      player: {
        playing: false,
        trackArtists: [],
        trackTitle: '',
        trackAlbum: []
      },
      storedId: ''
    }
  },

  computed: {
    /**
     * Check for the existence of a stored access token and
     * return the correct Component to be displayed.
     * @return {String}
     */
    getCurrentComponent() {
      return this.auth.status === false ? 'Authorise' : 'NowPlaying'
    }
  },

  created() {
    this.auth = {
      ...this.auth,
      ...getStoredAuth()
    }
  },

  mounted() {
      this.pollNowPlayingWithPriority();
      setInterval(this.pollNowPlayingWithPriority, 10000); // every 10 seconds
  },

  methods: {
    /**
     * Store
     */
    storeAccessToken() {
      this.getAccessToken()
    },

    /**
     * Request a refresh token from Spotify.
     */
    requestRefreshTokens() {
      this.auth.status = false
    },

    /**
     * Update the player object.
     * @param {Object} value - Spotify playr object.
     */
    updateCurrentTrack(value) {
      this.player = value
    },

   /**
     * add two user refresh logic
     * 
     */
 
    
      async refreshAccessTokenFor(user) {
      const clientId = this.auth[`clientId${user}`];
      const refreshToken = this.auth[`refreshToken${user}`];
      const clientSecret = this.auth.clientSecret;

      const encoded = btoa(`${clientId}:${clientSecret}`);

      const res = await fetch(this.endpoints.token, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
          Authorization: `Basic ${encoded}`
        },
        body: new URLSearchParams({
          grant_type: 'refresh_token',
          refresh_token: refreshToken
        })
      });

      const data = await res.json();
      return data.access_token;
    },

    async fetchNowPlaying(accessToken) {
      const res = await fetch(`${this.endpoints.base}/${this.endpoints.nowPlaying}`, {
        headers: {
          Authorization: `Bearer ${accessToken}`
        }
      });

      if (!res.ok) return null;

      const data = await res.json();
      return data;
    },

    async pollNowPlayingWithPriority() {
      try {
        const accessToken1 = await this.refreshAccessTokenFor('1');
        const data1 = await this.fetchNowPlaying(accessToken1);
    
        if (data1?.is_playing) {
          this.player = this.transformPlaybackData(data1);
          return;
        }
    
        const accessToken2 = await this.refreshAccessTokenFor('2');
        const data2 = await this.fetchNowPlaying(accessToken2);
    
        if (data2?.is_playing) {
          this.player = this.transformPlaybackData(data2);
          return;
        }
    
        // Fallback if neither user is playing
        this.player = {
          playing: false,
          trackArtists: [],
          trackTitle: '',
          trackAlbum: []
        };
    
      } catch (err) {
        console.error("Polling failed:", err);
      }
    },

    transformPlaybackData(data) {
      return {
        playing: data.is_playing,
        trackTitle: data.item?.name || '',
        trackArtists: data.item?.artists?.map(a => a.name).join(', ') || '',
        trackAlbum: {
          title: data.item?.album?.name || '',
          image: data.item?.album?.images?.[0]?.url || ''
        }

      };
    }

    /**
     * end of added two user refresh logic
     * 
     */
 
  
  },

  watch: {
    /**
     * Watch the authorisation status.
     */
    'auth.status': function() {
      setStoredAuth(this.auth)
    }
  }
}
</script>
