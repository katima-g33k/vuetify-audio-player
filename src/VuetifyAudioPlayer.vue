<template>
  <v-card flat>
    <v-card-text>
      <v-row>
        <v-col
            cols="4"
            md="4"
            offset="4"
            offset-md="4"
            style="text-align: center"
        >
          <v-btn icon @click="handleRewind">
            <v-icon>mdi-rewind</v-icon>
          </v-btn>
          <v-btn v-if="!isPlaying" icon @click="play">
            <v-icon>mdi-play</v-icon>
          </v-btn>
          <v-btn v-if="isPlaying" icon @click="pause">
            <v-icon>mdi-pause</v-icon>
          </v-btn>
          <v-btn icon @click="handleFastForward">
            <v-icon>mdi-fast-forward</v-icon>
          </v-btn>
        </v-col>
        <v-col
            cols="4"
            md="4"
            @mouseenter="volumeBarHidden = false"
            @mouseleave="volumeBarHidden = true"
        >
          <v-row justify="end">
            <v-col
                :class="volumeBarHidden ? 'volume hidden' : 'volume'"
                cols="9"
                md="9"
                style="margin-right: 0; padding-right: 0"
            >
              <v-slider
                  min="0"
                  max="1"
                  step="0.01"
                  :value="volume"
                  style="height: 30px; margin-top: 3px"
                  @change="handleValueChange"
              />
            </v-col>
            <v-col cols="3" md="3">
              <v-btn icon @click="toggleMute(storedVolume)">
                <v-icon>{{ volumeIcon }}</v-icon>
              </v-btn>
            </v-col>
          </v-row>
        </v-col>
        <v-col cols="12" md="12" style="margin-top: -15px">
          <v-slider
              v-model="currentTime"
              min="0"
              :max="duration"
              @change="handleSliderChange"
              @mouseup="handleSliderMouseUp"
              @mousedown="handleSliderMouseDown"
          />
        </v-col>
        <v-col cols="12" md="12" style="text-align: center; margin-top: -35px">
          {{ formattedCurrentTime }} / {{ formattedDuration }}
        </v-col>
      </v-row>
    </v-card-text>
    <audio :id="audioPlayerId" ref="audio" :src="src" style="display: none" />
  </v-card>
</template>

<script>
function formatTime(second) {
  return new Date(second * 1000).toISOString().substr(11, 8);
}

export default {
  name: 'AudioPlayer',
  props: {
    audioPlayerId: {
      type: String,
      default: null,
    },
    src: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      audio: null,
      currentTime: 0,
      duration: 0,
      isPlaying: false,
      isSeeking: false,
      storedVolume: 0,
      volumeBarHidden: true,
      volume: 1,
    };
  },
  computed: {
    formattedCurrentTime() {
      return formatTime(this.currentTime);
    },
    formattedDuration() {
      return formatTime(this.duration);
    },
    volumeIcon() {
      if (this.volume > 0.66) {
        return 'mdi-volume-high';
      }

      if (this.volume > 0.33) {
        return 'mdi-volume-medium';
      }

      if (this.volume > 0) {
        return 'mdi-volume-low';
      }

      return 'mdi-volume-off';
    },
  },
  mounted() {
    this.audio = this.$refs.audio;
    this.audio.volume = this.volume;
    this.audio.addEventListener('loadeddata', this._handleLoaded);
    this.audio.addEventListener('ended', this._handleEnded);
    this.audio.addEventListener('pause', this._handlePause);
    this.audio.addEventListener('play', this._handlePlay);
    this.audio.addEventListener('timeupdate', this._handleTimeUpdate);
  },
  methods: {
    _handleLoaded() {
      if (this.audio.readyState < 2) {
        throw new Error('Failed to load sound file');
      }

      if (this.audio.duration !== Infinity) {
        this.duration = +this.audio.duration;
        return;
      }

      // Fix duration for streamed audio source or blob based
      // https://stackoverflow.com/questions/38443084/how-can-i-add-predefined-length-to-audio-recorded-from-mediarecorder-in-chrome/39971175#39971175
      this.audio.currentTime = 1e101;
      this.audio.ontimeupdate = () => {
        this.audio.ontimeupdate = () => {};
        this.audio.currentTime = 0;
        this.duration = +this.audio.duration;
      };
    },
    _handleEnded() {
      this.isPlaying = false;
    },
    _handlePause() {
      this.isPlaying = false;
    },
    _handlePlay() {
      this.isPlaying = true;
    },
    _handleTimeUpdate() {
      if (!this.isSeeking) {
        this.currentTime = this.audio.currentTime;
      }
    },
    handleSliderMouseDown(e) {
      this.isSeeking = true;
    },
    handleSliderMouseUp() {
      this.isSeeking = false;
    },
    handleSliderChange(value) {
      this.audio.currentTime = value;
    },
    handleRewind() {
      this.audio.currentTime -= 15;
    },
    handleFastForward() {
      this.audio.currentTime += 15;
    },
    toggleMute(volume) {
      [this.volume, this.storedVolume] = [volume, this.volume];
      this.audio.muted = volume === 0;
    },
    handleValueChange(value) {
      if (value === 0 || this.volume === 0) {
        return this.toggleMute(value);
      }

      this.audio.volume = this.volume = value;
    },
    pause() {
      this.audio && this.audio.pause();
    },
    play() {
      this.audio && this.audio.play();
    },
  },
};
</script>

<style>
.volume {
  visibility: visible;
  opacity: 1;
  transition: opacity 800ms, visibility 800ms;
}

.volume.hidden {
  opacity: 0;
  visibility: hidden;
}
</style>
