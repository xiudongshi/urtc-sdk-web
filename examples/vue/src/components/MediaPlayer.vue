<template>
  <div v-bind:class="classes" v-on:click="handleClick">
    <div style="overflow: 'hidden'; text-overflow: 'ellipsis';">用户ID: {{stream.uid}}</div>
    <div style="overflow: 'hidden'; text-overflow: 'ellipsis';">流ID: {{stream.sid}}</div>
    <div v-show="stream.mediaStream" style="overflow: 'hidden'; text-overflow: 'ellipsis';">音量: {{volume}} % &nbsp;&nbsp;&nbsp;&nbsp;音频丢包率: {{stats.audioLost}} %</div>
    <div v-show="stream.mediaStream" style="overflow: 'hidden'; text-overflow: 'ellipsis';">视频丢包率: {{stats.videoLost}} % &nbsp;&nbsp;&nbsp;&nbsp;网络延时: {{stats.rtt}} ms</div>
    <div class="video-container" v-show="stream.mediaStream">
      <video
        ref="video"
        webkit-playsinline
        autoplay
        playsinline>
      </video>
      <div class="play-mask" v-show="showPlayMask">
        <div class="mask-content">
          <div class="hint">由于当前浏览器不支持视频自动播放，请点击下面的按钮来播放</div>
          <button @click.stop="handlePlay">播放</button>
        </div>
      </div>
    </div>
    <p v-show="!stream.mediaStream">unsubscribe</p>
  </div>
</template>

<script>
import classnames from 'unique-classnames';
import canAutoplay from 'can-autoplay';

export default {
  name: 'MediaPlayer',
  data: function () {
    const classes = classnames('media-player', this.className);

    return {
      classes: classes,
      volume: 0,
      stats: {
        audioLost: 0,
        biggestAudioLost: 0,
        videoLost: 0,
        biggestVideoLost: 0,
        rtt: 0,
        biggestRTT: 0
      },
      canAutoplay: true,
      paused: false
    };
  },
  props: {
    className: {
      type: String,
      default: ''
    },
    stream: {
      type: Object,
      default: function () {
        return {};
      }
    },
    client: {
      type: Object,
      default: function () {
        return null;
      }
    },
    onClick: {
      type: Function,
      default: function () {}
    }
  },
  created: function () {
    this.volumeTimer = 0;
    this.stateTimer = 0;
  },
  mounted: function () {
    this.isComponentDestroyed = false;
    if (this.stream.mediaStream) {
      this.play(this.stream.mediaStream);
    }
    this.$refs.video.onplay = (e) => {
      this.paused = e.target.paused;
    };
    this.paused = this.$refs.video.paused;
    canAutoplay.video().then(res => {
      this.canAutoplay = res.result;
    });
  },
  beforeDestroy: function () {
    this.stop();
  },
  destroyed: function () {
    this.isComponentDestroyed = true;
  },
  watch: {
    'stream.mediaStream': function (val, oldVal) {
      console.log('media stream changed: ', val, oldVal);
      if (val) {
        this.play(val);
      } else {
        this.stop();
      }
    }
  },
  methods: {
    play: function (mediaStream) {
      this.$refs.video.srcObject = mediaStream;
      this.startGetVolume();
      this.startGetState();
    },
    stop: function () {
      this.stopGetVolume();
      this.stopGetState();
      this.$refs.video.srcObject = null;
    },
    startGetVolume: function () {
      const { client, stream } = this;
      if (!client || !stream || !stream.audio) {
        return;
      }
      if (this.volumeTimer) {
        clearInterval(this.volumeTimer);
      }
      this.volumeTimer = setInterval(() => {
        this.volume = client.getAudioVolume(stream.sid);
      }, 1000);
    },
    stopGetVolume: function () {
      clearInterval(this.volumeTimer);
    },
    startGetState: function () {
         const { client, stream } = this;
        if (!client || !stream || !stream.video) {
          return;
        }
        if (this.stateTimer) {
          clearInterval(this.stateTimer);
        }
        this.stateTimer = setInterval(() => {
          client.getAudioStats(stream.sid, (_stats) => {
            if (this.isComponentDestroyed) return;
            const { stats } = this;
            stats.audioLost = _stats.lostpre;
            if (stats.biggestAudioLost < _stats.lostpre) {
              stats.biggestAudioLost = _stats.lostpre;
            }
          }, (e) => {
            console.error('get video stats ', stream.sid);
          });
          client.getVideoStats(stream.sid, (_stats) => {
            if (this.isComponentDestroyed) return;
            const { stats } = this;
            stats.videoLost = _stats.lostpre;
            if (stats.biggestVideoLost < _stats.lostpre) {
              stats.biggestVideoLost = _stats.lostpre;
            }
          }, (e) => {
            console.error('get video stats ', stream.sid);
          });
          client.getNetworkStats(stream.sid, (_stats) => {
            if (this.isComponentDestroyed) return;
            const { stats } = this;
            stats.rtt = _stats.rtt;
            if (stats.biggestRTT < _stats.rtt) {
              stats.biggestRTT = _stats.rtt;
            }
          }, (e) => {
            console.error('get network stats ', stream.sid);
          });
        }, 1000);
    },
    stopGetState: function () {
      clearInterval(this.stateTimer);
    },
    handleClick: function () {
      const { stream, onClick } = this;
      onClick && onClick(stream);
    },
    handlePlay: function () {
      if (this.$refs.video && this.paused) {
        this.$refs.video.play();
      }
    }
  },
  computed: {
    showPlayMask: function () {
      if (this.stream && this.stream.type !== 'subscribe') {
        return false;
      }
      if (!this.canAutoplay && this.paused) {
        return true;
      }
      return false;
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.media-player {
  display: inline-block;
  margin: 2px;
  width: 300px;
  text-align: left;
  white-space: nowrap;
  cursor: pointer;
}

.media-player .video-container {
  position: relative;
}

.media-player .muted-mask,
.media-player .play-mask {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: 9;
  opacity: 0.6;
  background-color: #fff;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.muted-mask .mask-content,
.play-mask .mask-content {
  max-width: 200px;
}

.mask-content .hint {
  margin-bottom: 12px;
  white-space: break-spaces;
  font-size: 14px;
}

.mask-content button {
  min-height: 50px;
}

.media-player video {
  width: 100%;
  height: 100%;
}
</style>
