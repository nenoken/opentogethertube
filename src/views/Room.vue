<template>
  <div>
    <v-container fluid :class="{ room: true, fullscreen: $store.state.fullscreen }" v-if="!showJoinFailOverlay">
      <v-col v-if="!$store.state.fullscreen">
        <h1>{{ $store.state.room.title != "" ? $store.state.room.title : ($store.state.room.isTemporary ? "Temporary Room" : $store.state.room.name) }}</h1>
        <span id="connectStatus">{{ connectionStatus }}</span>
      </v-col>
      <v-col :style="{ padding: ($store.state.fullscreen ? 0 : 'inherit') }">
        <v-row no-gutters class="video-container">
          <div class="video-subcontainer" cols="12" :xl="$store.state.fullscreen ? 9 : 7" md="8" :style="{ padding: ($store.state.fullscreen ? 0 : 'inherit') }">
            <v-responsive :aspect-ratio="16/9" class="player-container" :key="currentSource.service">
              <OmniPlayer
                ref="player"
                :source="currentSource"
                :class="{ player: true, 'no-video': !currentSource.service }"
                @playing="onPlaybackChange(true)"
                @paused="onPlaybackChange(false)"
                @ready="onPlayerReady"
                @buffering="onVideoBuffer"
                @error="onVideoError"
              />
            </v-responsive>
            <v-col class="video-controls">
              <vue-slider id="videoSlider" v-model="sliderPosition" @change="sliderChange" :max="$store.state.room.currentSource.length" :tooltip-formatter="sliderTooltipFormatter" :disabled="currentSource.length == null"/>
              <v-row no-gutters align="center">
                <v-tooltip bottom>
                  <template v-slot:activator="{ on, attrs }">
                    <v-btn @click="seekVideo(-10)" v-bind="attrs" v-on="on">
                      <v-icon>fas fa-angle-left</v-icon>
                    </v-btn>
                  </template>
                  <span>Rewind 10s</span>
                </v-tooltip>
                <v-btn @click="togglePlayback()">
                  <v-icon v-if="$store.state.room.isPlaying">fas fa-pause</v-icon>
                  <v-icon v-else>fas fa-play</v-icon>
                </v-btn>
                <v-tooltip bottom>
                  <template v-slot:activator="{ on, attrs }">
                    <v-btn @click="seekVideo(10)" v-bind="attrs" v-on="on">
                      <v-icon>fas fa-angle-right</v-icon>
                    </v-btn>
                  </template>
                  <span>Skip 10s</span>
                </v-tooltip>
                <v-btn @click="skipVideo()">
                  <v-icon>fas fa-fast-forward</v-icon>
                </v-btn>
                <vue-slider v-model="volume" style="width: 150px; margin-left: 10px"/>
                <div style="margin-left: 20px" class="timestamp">
                  {{ timestampDisplay }}
                </div>
                <v-btn @click="toggleFullscreen()" style="margin-left: 10px">
                  <v-icon>fas fa-compress</v-icon>
                </v-btn>
              </v-row>
            </v-col>
          </div>
          <div cols="12" :xl="$store.state.fullscreen ? 3 : 5" md="4" class="chat-container">
            <Chat class="chat" />
          </div>
        </v-row>
        <v-row no-gutters>
          <v-col cols="12" md="8" sm="12">
            <v-tabs grow v-model="queueTab" @change="onTabChange">
              <v-tab>
                Queue
                <span class="bubble">{{ $store.state.room.queue.length <= 99 ? $store.state.room.queue.length : "99+" }}</span>
              </v-tab>
              <v-tab>Add</v-tab>
              <v-tab>Settings</v-tab>
            </v-tabs>
            <v-tabs-items v-model="queueTab" class="queue-tab-content">
              <v-tab-item>
                <div class="video-queue">
                  <draggable v-model="$store.state.room.queue" @end="onQueueDragDrop" handle=".drag-handle">
                    <VideoQueueItem v-for="(itemdata, index) in $store.state.room.queue" :key="index" :item="itemdata"/>
                  </draggable>
                </div>
              </v-tab-item>
              <v-tab-item>
                <div class="video-add">
                  <div>
                    <v-text-field clearable placeholder="Type to search YouTube or enter a Video URL to add to the queue" v-model="inputAddPreview" @keydown="onInputAddPreviewKeyDown" @focus="onFocusHighlightText" :loading="isLoadingAddPreview" />
                    <v-btn v-if="!production" @click="postTestVideo(0)">Add test youtube 0</v-btn>
                    <v-btn v-if="!production" @click="postTestVideo(1)">Add test youtube 1</v-btn>
                    <v-btn v-if="!production" @click="postTestVideo(2)">Add test vimeo 2</v-btn>
                    <v-btn v-if="!production" @click="postTestVideo(3)">Add test vimeo 3</v-btn>
                    <v-btn v-if="!production" @click="postTestVideo(4)">Add test dailymotion 4</v-btn>
                    <v-btn v-if="addPreview.length > 1" @click="addAllToQueue()" :loading="isLoadingAddAll" :disabled="isLoadingAddAll">Add All</v-btn>
                  </div>
                  <v-row v-if="isLoadingAddPreview" justify="center">
                    <v-progress-circular indeterminate/>
                  </v-row>
                  <div v-if="!isLoadingAddPreview">
                    <v-row justify="center">
                      <div v-if="hasAddPreviewFailed">
                        {{ addPreviewLoadFailureText }}
                      </div>
                      <v-container fill-height v-if="addPreview.length == 0 && inputAddPreview.length > 0 && !hasAddPreviewFailed && !isAddPreviewInputUrl">
                        <v-row justify="center" align="center">
                          <v-col cols="12">
                            Search YouTube for "{{ inputAddPreview }}" by pressing enter, or by clicking search.<br>
                            <v-btn @click="requestAddPreviewExplicit">Search</v-btn>
                          </v-col>
                        </v-row>
                      </v-container>
                    </v-row>
                    <div v-if="highlightedAddPreviewItem">
                      <VideoQueueItem :item="highlightedAddPreviewItem" is-preview style="margin-bottom: 20px"/>
                      <h4>Playlist</h4>
                    </div>
                    <VideoQueueItem v-for="(itemdata, index) in addPreview" :key="index" :item="itemdata" is-preview/>
                  </div>
                </div>
              </v-tab-item>
              <v-tab-item>
                <div class="room-settings">
                  <v-form @submit="submitRoomSettings">
                    <v-text-field label="Title" v-model="inputRoomSettingsTitle" :loading="isLoadingRoomSettings" />
                    <v-text-field label="Description" v-model="inputRoomSettingsDescription" :loading="isLoadingRoomSettings" />
                    <v-select label="Visibility" :items="[{ text: 'public' }, { text: 'unlisted' }]" v-model="inputRoomSettingsVisibility" :loading="isLoadingRoomSettings" />
                    <v-select label="Queue Mode" :items="[{ text: 'manual' }, { text: 'vote' }]" v-model="inputRoomSettingsQueueMode" :loading="isLoadingRoomSettings" />
                    <v-btn @click="submitRoomSettings" role="submit" :loading="isLoadingRoomSettings">Save</v-btn>
                  </v-form>
                  <v-btn v-if="!$store.state.room.isTemporary && $store.state.user && !$store.state.room.hasOwner" role="submit" @click="claimOwnership">Claim Room</v-btn>
                </div>
              </v-tab-item>
            </v-tabs-items>
          </v-col>
          <v-col col="4" md="4" sm="12" class="user-invite-container">
            <div class="user-list" v-if="$store.state.room.users">
              <v-card>
                <v-subheader>
                  Users
                  <v-btn icon x-small @click="openEditName"><v-icon>fas fa-cog</v-icon></v-btn>
                </v-subheader>
                <v-list-item v-if="showEditName">
                  <v-text-field @change="onEditNameChange" placeholder="Set your name" v-model="username" :loading="setUsernameLoading" :error-messages="setUsernameFailureText"/>
                </v-list-item>
                <v-list-item v-for="(user, index) in $store.state.room.users" :key="index" :class="user.isLoggedIn ? 'user registered' : 'user'">
                  <span class="name">{{ user.name }}</span>
                  <span v-if="user.isYou" class="is-you">You</span>
                  <v-icon class="player-status" v-if="user.status === 'buffering'">fas fa-spinner</v-icon>
                  <v-icon class="player-status" v-else-if="user.status === 'ready'">fas fa-check</v-icon>
                  <v-icon class="player-status" v-else-if="user.status === 'error'">fas fa-exclamation</v-icon>
                </v-list-item>
                <v-list-item class="nobody-here" v-if="$store.state.room.users.length === 1">
                  There seems to be nobody else here. Invite some friends!
                </v-list-item>
              </v-card>
            </div>
            <div class="share-invite">
              <v-card>
                <v-subheader>
                  Share Invite
                </v-subheader>
                <v-card-text>
                  Copy this link and share it with your friends!
                  <v-text-field outlined ref="inviteLinkText" :value="inviteLink" append-outer-icon="fa-clipboard" :success-messages="copyInviteLinkSuccessText" @focus="onFocusHighlightText" @click:append-outer="copyInviteLink" />
                </v-card-text>
              </v-card>
            </div>
          </v-col>
        </v-row>
      </v-col>
    </v-container>
    <v-footer>
      <v-container pa-0>
        <v-row no-gutters align="center" justify="center">
          <router-link to="/privacypolicy">Privacy Policy</router-link>
        </v-row>
      </v-container>
    </v-footer>
    <v-overlay :value="showJoinFailOverlay">
      <v-layout column>
        <h1>Failed to join room</h1>
        <span>{{ joinFailReason }}</span>
        <v-btn to="/rooms">Find Another Room</v-btn>
      </v-layout>
    </v-overlay>
    <v-snackbar v-if="$store.state.room.events.length > 0" :key="$store.state.room.events.length" v-model="$store.state.room.events[$store.state.room.events.length - 1].isVisible" :timeout="$store.state.room.events[$store.state.room.events.length - 1].timeout">
      {{ snackbarText }}
      <v-btn @click="undoEvent($store.state.room.events[$store.state.room.events.length - 1], $store.state.room.events.length - 1)" v-if="$store.state.room.events[$store.state.room.events.length - 1].isUndoable">Undo</v-btn>
    </v-snackbar>
  </div>
</template>

<script>
import { API } from "@/common-http.js";
import VideoQueueItem from "@/components/VideoQueueItem.vue";
import { secondsToTimestamp, calculateCurrentPosition } from "@/timestamp.js";
import _ from "lodash";
import draggable from 'vuedraggable';
import VueSlider from 'vue-slider-component';
import OmniPlayer from "@/components/OmniPlayer.vue";
import Chat from "@/components/Chat.vue";

export default {
  name: 'room',
  components: {
    draggable,
    VideoQueueItem,
    VueSlider,
    OmniPlayer,
    Chat,
  },
  data() {
    return {
      sliderPosition: 0,
      sliderTooltipFormatter: secondsToTimestamp,
      volume: 100,
      addPreview: [],

      username: "", // refers to the local user's username

      showEditName: false,
      queueTab: 0,
      isLoadingAddPreview: false,
      hasAddPreviewFailed: false,
      addPreviewLoadFailureText: "",
      inputAddPreview: "",
      isLoadingRoomSettings: false,
      inputRoomSettingsTitle: "",
      inputRoomSettingsDescription: "",
      inputRoomSettingsVisibility: "",
      inputRoomSettingsQueueMode: "",
      setUsernameLoading: false,
      setUsernameFailureText: "",
      isLoadingAddAll: false,

      showJoinFailOverlay: false,
      joinFailReason: "",
      snackbarActive: false,
      snackbarText: "",

      timestampDisplay: "",
      i_timestampUpdater: null,

      copyInviteLinkSuccessText: "",
    };
  },
  computed: {
    connectionStatus() {
      if (this.$store.state.socket.isConnected) {
        return "Connected";
      }
      else {
        return "Connecting...";
      }
    },
    currentSource() {
      return this.$store.state.room.currentSource;
    },
    playbackPosition() {
      return this.$store.state.room.playbackPosition;
    },
    playbackPercentage() {
      if (!this.$store.state.room.currentSource) {
        return 0;
      }
      if (this.$store.state.room.currentSource.length == 0) {
        return 0;
      }
      return this.$store.state.room.playbackPosition / this.$store.state.room.currentSource.length;
    },
    production() {
      return this.$store.state.production;
    },
    isAddPreviewInputUrl() {
      try {
        return !!(new URL(this.inputAddPreview).host);
      }
      catch (e) {
        return false;
      }
    },
    highlightedAddPreviewItem() {
      return _.find(this.addPreview, { highlight: true });
    },
    inviteLink() {
      return window.location.href.split('?')[0].toLowerCase();
    },
  },
  async created() {
    this.$events.on("onRoomEvent", event => {
      if (event.eventType === "play") {
        this.snackbarText = `${event.userName} played the video`;
      }
      else if (event.eventType === "pause") {
        this.snackbarText = `${event.userName} paused the video`;
      }
      else if (event.eventType === "skip") {
        this.snackbarText = `${event.userName} skipped ${event.parameters.video.title}`;
      }
      else if (event.eventType === "seek") {
        this.snackbarText = `${event.userName} seeked to ${secondsToTimestamp(event.parameters.position)}`;
      }
      else if (event.eventType === "joinRoom") {
        this.snackbarText = `${event.userName} joined the room`;
      }
      else if (event.eventType === "leaveRoom") {
        this.snackbarText = `${event.userName} left the room`;
      }
      else if (event.eventType === "addToQueue") {
        if (event.parameters.count) {
          this.snackbarText = `${event.userName} added ${event.parameters.count} videos`;
        }
        else {
          this.snackbarText = `${event.userName} added ${event.parameters.video.title}`;
        }
      }
      else if (event.eventType === "removeFromQueue") {
        this.snackbarText = `${event.userName} removed ${event.parameters.video.title}`;
      }
      else {
        this.snackbarText = `${event.userName} triggered event ${event.eventType}`;
      }
      this.snackbarActive = true;
    });

    this.$events.on("onRoomCreated", () => {
      if (this.$store.state.socket.isConnected) {
        this.$disconnect();
      }
      setTimeout(() => {
        if (!this.$store.state.socket.isConnected) {
          this.$connect(`${window.location.protocol.startsWith("https") ? "wss" : "ws"}://${window.location.host}/api/room/${this.$route.params.roomId}`);
        }
      }, 100);
    });

    this.$events.on("onChatLinkClick", link => {
      this.inputAddPreview = link;
      this.queueTab = 1;
    });

    window.removeEventListener('keydown', this.onKeyDown);
    window.addEventListener('keydown', this.onKeyDown);

    if (!this.$store.state.socket.isConnected) {
      // This check prevents the client from connecting multiple times,
      // caused by hot reloading in the dev environment.
      this.$connect(`${window.location.protocol.startsWith("https") ? "wss" : "ws"}://${window.location.host}/api/room/${this.$route.params.roomId}`);
    }

    this.i_timestampUpdater = setInterval(() => {
      this.sliderPosition = this.$store.state.room.isPlaying ? calculateCurrentPosition(this.$store.state.room.playbackStartTime, new Date(), this.$store.state.room.playbackPosition) : this.$store.state.room.playbackPosition;
      this.sliderPosition = _.clamp(this.sliderPosition, 0, this.$store.state.room.currentSource.length);
      const position = secondsToTimestamp(this.sliderPosition);
      const duration = secondsToTimestamp(this.$store.state.room.currentSource.length || 0);
      this.timestampDisplay = `${position} / ${duration}`;
    }, 1000);

    this.$events.on("onSync", this.rewriteUrlToRoomName);
  },
  destroyed() {
    clearInterval(this.i_timestampUpdater);
    this.$disconnect();
    this.$events.remove("onSync", this.rewriteUrlToRoomName);
  },
  methods: {
    postTestVideo(v) {
      let videos = [
        "https://www.youtube.com/watch?v=WC66l5tPIF4",
        "https://www.youtube.com/watch?v=aI67KDJRnvQ",
        "https://vimeo.com/94338566",
        "https://vimeo.com/239423699",
        "https://www.dailymotion.com/video/x6hkywd",
      ];
      API.post(`/room/${this.$route.params.roomId}/queue`, {
        url: videos[v],
      });
    },
    togglePlayback() {
      if (this.$store.state.room.isPlaying) {
        this.$socket.sendObj({ action: "pause" });
      }
      else {
        this.$socket.sendObj({ action: "play" });
      }
    },
    skipVideo() {
      this.$socket.sendObj({ action: "skip" });
    },
    sliderChange() {
      this.$socket.sendObj({ action: "seek", position: this.sliderPosition });
    },
    addAllToQueue() {
      this.isLoadingAddAll = true;
      API.post(`/room/${this.$route.params.roomId}/queue`, { videos: this.addPreview }).then(() => {
        this.isLoadingAddAll = false;
      });
    },
    openEditName() {
      this.username = this.$store.state.user ? this.$store.state.user.username : this.$store.state.username;
      this.showEditName = !this.showEditName;
    },
    play() {
      this.$refs.player.play();
    },
    pause() {
      this.$refs.player.pause();
    },
    updateVolume() {
      this.$refs.player.setVolume(this.volume);
    },
    requestAddPreview() {
      API.get(`/data/previewAdd?input=${encodeURIComponent(this.inputAddPreview)}`, { validateStatus: status => status < 500 }).then(res => {
        this.isLoadingAddPreview = false;
        if (res.status === 200) {
          this.hasAddPreviewFailed = false;
          this.addPreview = res.data;
          console.log(`Got add preview with ${this.addPreview.length}`);
        }
        else if (res.status === 400) {
          this.hasAddPreviewFailed = true;
          this.addPreviewLoadFailureText = res.data.error.message;
          if (res.data.error.name === "FeatureDisabledException" && !this.isAddPreviewInputUrl) {
            window.open(`https://www.youtube.com/results?search_query=${encodeURIComponent(this.inputAddPreview)}`, "_blank");
          }
        }
        else {
          console.warn("Unknown status for add preview response:", res.status);
        }
      }).catch(err => {
        this.isLoadingAddPreview = false;
        this.hasAddPreviewFailed = true;
        this.addPreviewLoadFailureText = "An unknown error occurred when getting add preview. Try again later.";
        console.error("Failed to get add preview", err);
      });
    },
    requestAddPreviewDebounced: _.debounce(function() {
      // HACK: can't use an arrow function here because it will make `this` undefined
      this.requestAddPreview();
    }, 500),
    /**
     * Request an add preview regardless of the current input.
     */
    requestAddPreviewExplicit() {
      this.isLoadingAddPreview = true;
      this.hasAddPreviewFailed = false;
      this.addPreview = [];
      this.requestAddPreview();
    },
    onEditNameChange() {
      this.setUsernameLoading = true;
      API.post("/user", { username: this.username }).then(() => {
        this.showEditName = false;
        this.setUsernameLoading = false;
        this.setUsernameFailureText = "";
      }).catch(err => {
        this.setUsernameLoading = false;
        this.setUsernameFailureText = err.response ? err.response.data.error.message : err.message;
      });
    },
    onPlaybackChange(changeTo) {
      if (this.currentSource.service === "youtube" || this.currentSource.service === "dailymotion") {
        this.$store.commit("PLAYBACK_STATUS", "ready");
      }
      this.updateVolume();
      if (changeTo == this.$store.state.room.isPlaying) {
        return;
      }

      if (this.$store.state.room.isPlaying) {
        this.play();
      }
      else {
        this.pause();
      }
    },
    onInputAddPreviewChange() {
      this.isLoadingAddPreview = true;
      this.hasAddPreviewFailed = false;
      if (_.trim(this.inputAddPreview).length == 0) {
        this.addPreview = [];
        this.isLoadingAddPreview = false;
        return;
      }
      if (!this.isAddPreviewInputUrl) {
        this.addPreview = [];
        this.isLoadingAddPreview = false;
        // Don't send API requests for non URL inputs without the user's explicit input to do so.
        // This is to help conserve youtube API quota.
        return;
      }
      this.requestAddPreviewDebounced();
    },
    onInputAddPreviewKeyDown(e) {
      if (_.trim(this.inputAddPreview).length == 0 || this.isAddPreviewInputUrl) {
        return;
      }

      if (e.keyCode === 13 && this.addPreview.length == 0) {
        this.requestAddPreviewExplicit();
      }
    },
    onFocusHighlightText(e) {
      e.target.select();
    },
    onPlayerReady() {
      this.$store.commit("PLAYBACK_STATUS", "ready");

      if (this.currentSource.service === "vimeo") {
        this.onPlayerReady_Vimeo();
      }
    },
    onPlayerReady_Vimeo() {
      if (this.$store.state.room.isPlaying) {
        this.play();
      }
      else {
        this.pause();
      }
    },
    onKeyDown(e) {
      if (e.target.nodeName === "INPUT") {
        return;
      }

      if (e.code === "Space" || e.code === "k") {
        this.togglePlayback();
        e.preventDefault();
      }
      else if (e.code === "Home") {
        this.$socket.sendObj({ action: "seek", position: 0 });
        e.preventDefault();
      }
      else if (e.code === "End") {
        this.$socket.sendObj({ action: "skip" });
        e.preventDefault();
      }
      else if (e.code === "KeyF") {
        this.toggleFullscreen();
      }
      else if (e.code === "ArrowLeft" || e.code === "ArrowRight" || e.code === "KeyJ" || e.code === "KeyL") {
        let seekIncrement = 5;
        if (e.ctrlKey || e.code === "KeyJ" || e.code === "KeyL") {
          seekIncrement = 10;
        }
        if (e.code === "ArrowLeft" || e.code === "KeyJ") {
          seekIncrement *= -1;
        }

        this.seekVideo(seekIncrement);
        e.preventDefault();
      }
      else if (e.code === "ArrowUp" || e.code === "ArrowDown") {
        this.volume = _.clamp(this.volume + 5 * (e.code === "ArrowDown" ? -1 : 1), 0, 100);
        e.preventDefault();
      }
    },
    toggleFullscreen() {
      if (document.fullscreenElement) {
        document.exitFullscreen();
      }
      else {
        document.documentElement.requestFullscreen();
      }
    },
    onQueueDragDrop(e) {
      this.$socket.sendObj({
        action: "queue-move",
        currentIdx: e.oldIndex,
        targetIdx: e.newIndex,
      });
    },
    undoEvent(event, idx) {
      this.$socket.sendObj({
        action: "undo",
        event,
      });
      this.$store.state.room.events.splice(idx, 1);
    },
    onTabChange() {
      if (this.queueTab === 2) {
        // FIXME: we have to make an API request becuase visibility is not sent in sync messages.
        this.isLoadingRoomSettings = true;
        API.get(`/room/${this.$route.params.roomId}`).then(res => {
          this.isLoadingRoomSettings = false;
          this.inputRoomSettingsTitle = res.data.title;
          this.inputRoomSettingsDescription = res.data.description;
          this.inputRoomSettingsVisibility = res.data.visibility;
          this.inputRoomSettingsQueueMode = res.data.queueMode;
        });
      }
    },
    submitRoomSettings() {
      this.isLoadingRoomSettings = true;
      API.patch(`/room/${this.$route.params.roomId}`, {
        title: this.inputRoomSettingsTitle,
        description: this.inputRoomSettingsDescription,
        visibility: this.inputRoomSettingsVisibility,
        queueMode: this.inputRoomSettingsQueueMode,
      }).then(() => {
        this.isLoadingRoomSettings = false;
      });
    },
    onVideoBuffer() {
      this.$store.commit("PLAYBACK_STATUS", "buffering");
    },
    onVideoError() {
      this.$store.commit("PLAYBACK_STATUS", "error");
    },
    hideVideoControls: _.debounce(() => {
      let controlsDiv = document.getElementsByClassName("video-controls");
      if (controlsDiv.length) {
        controlsDiv = controlsDiv[0];
        controlsDiv.classList.add("hide");
      }
    }, 3000),
    claimOwnership() {
      this.isLoadingRoomSettings = true;
      API.patch(`/room/${this.$route.params.roomId}`, {
        claim: true,
      }).then(() => {
        this.isLoadingRoomSettings = false;
      }).catch(err => {
        console.log(err);
        this.isLoadingRoomSettings = false;
      });
    },
    copyInviteLink() {
      let textfield = this.$refs.inviteLinkText.$el.querySelector('input');
      textfield.select();
      document.execCommand("copy");
      this.copyInviteLinkSuccessText = "Copied!";
      setTimeout(() => {
        this.copyInviteLinkSuccessText = "";
        textfield.blur();
      }, 3000);
    },
    rewriteUrlToRoomName() {
      if (this.$route.params.roomId !== this.$store.state.room.name) {
        this.$router.replace({ name: "room", params: { roomId: this.$store.state.room.name } });
      }
    },
    seekVideo(delta) {
        let currentPosition = this.$store.state.room.isPlaying ? calculateCurrentPosition(this.$store.state.room.playbackStartTime, new Date(), this.$store.state.room.playbackPosition) : this.$store.state.room.playbackPosition;
        this.$socket.sendObj({
          action: "seek",
          position: _.clamp(currentPosition + delta, 0, this.$store.state.room.currentSource.length),
        });
    },
  },
  mounted() {
    this.$events.on("playVideo", () => {
      this.play();
    });
    this.$events.on("pauseVideo", () => {
      this.pause();
    });
    this.$events.on("roomJoinFailure", eventData => {
      this.showJoinFailOverlay = true;
      this.joinFailReason = eventData.reason;
    });

    document.onmousemove = () => {
      let controlsDiv = document.getElementsByClassName("video-controls");
      if (controlsDiv.length) {
        controlsDiv = controlsDiv[0];
        controlsDiv.classList.remove("hide");
      }
      this.hideVideoControls();
    };

    if (this.$store.state.quickAdd.length > 0) {
      for (let video of this.$store.state.quickAdd) {
        API.post(`/room/${this.$route.params.roomId}/queue`, _.pick(video, [
          "service",
          "id",
          "url",
        ]));
      }
    }
  },
  watch: {
    // username(newValue) {
    //   if (newValue != null) {
    //     // window.localStorage.setItem("username", newValue);
    //   }
    // },
    volume() {
      this.updateVolume();
    },
    async sliderPosition(newPosition) {
      let currentTime = await this.$refs.player.getPosition();

      if (Math.abs(newPosition - currentTime) > 1) {
        this.$refs.player.setPosition(newPosition);
      }
    },
    inputAddPreview() {
      // HACK: The @change event only triggers when the text field is defocused.
      // This ensures that onInputAddPreviewChange() runs everytime the text field's value changes.
      this.onInputAddPreviewChange();
    },
  },
};
</script>

<style lang="scss" scoped>
@import "../variables.scss";

.video-container {
  margin: 10px;

  .player {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }

  .no-video {
    height: 100%;
    color: #696969;
    border: 1px solid #666;
    border-radius: 3px;
  }

  .video-subcontainer {
    width: calc(100% / 12 * 8);
  }

  .chat-container {
    width: calc(100% / 12 * 4);

    .chat {
      min-height: 400px;
      height: 100%;
    }
  }

  @media (max-width: $md-max) {
    .video-subcontainer, .chat-container {
      width: 100%;
    }

    margin: 0;
  }

  @media (min-width: $xl-min) {
    .video-subcontainer {
      width: calc(100% / 12 * 7);
    }

    .chat-container {
      width: calc(100% / 12 * 5);
    }
  }
}
.video-queue, .video-add {
  margin: 0 10px;
  min-height: 500px;
}
.user-invite-container {
  padding: 0 10px;
  min-height: 500px;

  > * {
    margin-bottom: 10px;
  }
}
.nobody-here {
  font-style: italic;
  opacity: 0.5;
  font-size: 0.9em;
}
.queue-tab-content {
  background: transparent !important;
}
.is-you {
  color: #ffb300;
  border: 1px #ffb300 solid;
  border-radius: 10px;
  margin: 5px;
  padding: 0 5px;
  font-size: 10px;
}
.player-status {
  margin: 0 5px;
  font-size: 12px;
}
.bubble{
  height: 25px;
  width: 25px;
  margin-left: 10px;
  background-color: #3f3838;
  border-radius: 50%;
  display: inline-block;

  font-weight: bold;
  color:#fff;
  text-align: center;
  line-height: 1.8;
}
.chat-container {
  padding: 5px 10px;
}

.flip-list-move {
  transition: transform 0.5s;
}
.no-move {
  transition: transform 0s;
}

.fullscreen {
  padding: 0;

  .video-container {
    margin: 0;
  }

  .video-subcontainer {
    width: calc(100% / 12 * 9);
  }

  .chat-container {
    width: calc(100% / 12 * 3);
  }

  .player-container {
    height: 100vh;

    .player {
      border: none;
      border-right: 1px solid #666;
    }
  }

  .video-controls {
    position: sticky;
    bottom: 0;
    background: $background-color;
    transition: opacity 0.2s;

    &.hide {
      opacity: 0;
    }
  }

  @media only screen and (max-aspect-ratio: 16/9) {
    .video-subcontainer {
      width: 100%;
    }

    .chat-container {
      display: none;
    }
  }
}
.user {
  .name {
    opacity: 0.5;
    font-style: italic;
  }

  &.registered {
    .name {
      opacity: 1;
      font-style: normal;
    }
  }
}

.room {
  @media (max-width: $md-max) {
    padding: 0;
  }
}
</style>
