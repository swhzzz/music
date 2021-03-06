<template>
  <div class="player" v-show="playList.length">
    <div class="normal-player" v-show="fullScreen">
      <div class="header">
        <div class="wrap">
          <i class="iconfont icon-back" @click="toggleFullScreen"></i>
          <h3 class="song-name" v-html="currentSong.name"></h3>
        </div>
        <h5 class="singer-name" v-html="currentSong.singer"></h5>
      </div>
      <div class="main" @click="toggleLyric">
        <transition-group name="fade" tag="div">
          <div class="main-left" v-show="isImgShow" key="pic">
            <img :class="rotateCls" :src="currentSong.image" alt="" class="img">
          </div>
          <div class="main-right" v-show="!isImgShow" key="lyric">
            <scroll ref="scroll" :data="mixData" class="scroll">
              <div class="lyric-wrap">
                <p v-for="(item,index) in lyricLines" v-html="item" class="lyric-line"
                   :class="{'active': index === currentLineIndex}" ref="lyricLines">
                </p>
              </div>
            </scroll>
          </div>
        </transition-group>
      </div>
      <div class="footer">
        <div class="time">
          <div class="time-left"><span>{{format(currentTime)}}</span></div>
          <progress-bar class="progress-wrap" :rate="rate" @jumpProgress="jumpProgress"></progress-bar>
          <div class="time-right"><span>{{format(currentSong.duration)}}</span></div>
        </div>
        <div class="icon-wrap">
          <span @click="changeMode"><i :class="modeCls"></i></span>
          <span @click="toPrevSong"><i class="iconfont icon-prev"></i></span>
          <span v-if="isPlaying" @click="togglePlaying"><i class="iconfont icon-pause"></i></span>
          <span v-if="!isPlaying" @click="togglePlaying"><i class="iconfont icon-play"></i></span>
          <span @click="toNextSong"><i class="iconfont icon-next"></i></span>
          <span v-if="!currentSong.isHeart" @click="toggleHeart"><i class="iconfont icon-heart1"></i></span>
          <span v-if="currentSong.isHeart" @click="toggleHeart"><i class="iconfont icon-heart2"></i></span>
        </div>
      </div>
      <div class="bg" :style="bgCls"></div>
      <div class="layer"></div>
    </div>
    <div class="mini-player" v-show="!fullScreen" @click="toggleFullScreen">
      <div class="left">
        <div>
          <img :class="rotateCls" :src="currentSong.image" alt="">
        </div>
        <div class="content">
          <h5 v-html="currentSong.name"></h5>
          <p v-html="currentSong.singer"></p>
        </div>
      </div>
      <div class="right">
        <span v-if="isPlaying" @click.stop="togglePlaying"><i class="iconfont icon-pause"></i></span>
        <span v-if="!isPlaying" @click.stop="togglePlaying"><i class="iconfont icon-play-mini"></i></span>
      </div>
    </div>
    <audio ref="audio" :src="currentSong.url" @timeupdate="updateTime" @ended="end"></audio>
  </div>
</template>

<script>
  import {mapGetters} from 'vuex'
  import progressBar from '../base/progress-bar.vue'
  import {playMode} from '../common/js/config'
  import {messList} from '../api/util'
  import {getLyric} from '../api/lyric'
  import Lyric from 'lyric-parser'
  import Scroll from '../base/scroll.vue'

  export default {
    data() {
      return {
        timer: null,
        currentTime: 0,
        lyric: null,
        lyricLines: [],
        currentLineIndex: 0,
        isImgShow: true
      }
    },
    components: {progressBar, Scroll},
    computed: {
      mixData() {
        let temp = []
        temp = this.lyricLines.slice()
        temp.push(this.isImgShow)
        return temp
      },
      ...mapGetters(['playList', 'sequenceList', 'fullScreen', 'currentSong', 'mode', 'isPlaying', 'currentIndex']),
      rotateCls() {
        return this.isPlaying ? 'rotate' : 'rotate pause'
      },
      bgCls() {
        return `background-image: url(${this.currentSong.image})`
      },
      rate() {
        return this.currentTime / this.currentSong.duration
      },
      modeCls() {
        const map = {
          0: 'icon-sequence',
          1: 'icon-loop',
          2: 'icon-random'
        }
        return `iconfont ${map[this.mode]}`
      }
    },
    methods: {
      toggleFullScreen() {
        this.$store.commit('setFullScreen', !this.fullScreen)
      },
      togglePlaying() {
        this.$store.commit('setIsPlaying', !this.isPlaying)
        this.lyric.togglePlay()
      },
      toPrevSong() {
        let prev = this.currentIndex - 1 >= 0 ? this.currentIndex - 1 : this.playList.length - 1
        clearTimeout(this.timer) // 解决频繁点击报错
        setTimeout(() => {
          this.$store.dispatch('toPrevOrNext', {
            newIndex: prev
          })
        }, 300)
      },
      toNextSong() {
        let next = this.currentIndex + 1 === this.playList.length ? 0 : this.currentIndex + 1
        clearTimeout(this.timer)
        setTimeout(() => {
          this.$store.dispatch('toPrevOrNext', {
            newIndex: next
          })
        }, 300)
      },
      updateTime(e) {
        this.currentTime = e.target.currentTime
      },
      format(interval) {
        interval = interval | 0
        let minute = interval / 60 | 0
        let second = interval % 60
        second = second < 10 ? `0${second}` : second
        return `${minute}:${second}`
      },
      jumpProgress(rate) {
        let currentTime = rate * this.currentSong.duration
        this.$refs.audio.currentTime = currentTime
        this.lyric.seek(currentTime * 1000)
      },
      changeMode() {
        let mode = (this.mode + 1) % 3
        this.$store.commit('setMode', mode)
        let list = []
        if (mode === playMode.random) {
          list = messList(this.sequenceList)
        } else {
          list = this.sequenceList
        }
        this.resetCurrentIndex(list)
        this.$store.commit('setPlayList', list)
      },
      resetCurrentIndex(list) {
        let index = list.findIndex((item) => {
          return item.id === this.currentSong.id
        })
        this.$store.commit('setCurrentIndex', index)
      },
      end() {
        if (this.mode === playMode.loop) {
          this.$refs.audio.currentTime = 0
          this.$refs.audio.play()
          this.lyric.seek(0)
        } else {
          this.toNextSong()
        }
      },
      _getLyric() {
        getLyric(this.currentSong.mid).then((lyric) => {
          this.lyric = new Lyric(lyric, this.handleLyric)
          this.lyric.lines.map((item) => {
            this.lyricLines.push(item.txt)
          })
          if (this.isPlaying) {
            this.lyric.play()
          }
        })
      },
      handleLyric({lineNum, txt}) {
        this.currentLineIndex = lineNum
        if (lineNum > 8) {
          let el = this.$refs.lyricLines[lineNum - 8]
          this.$refs.scroll.scrollToElement(el, 1000) // 设置滚动歌词
        } else {
          this.$refs.scroll.scrollToElement(0, 0) // 如果歌词行数小于8，不滚动
        }
      },
      toggleHeart() {
        if (!this.currentSong.isHeart) {
          this.currentSong.isHeart = true
        } else {
          this.currentSong.isHeart = !this.currentSong.isHeart
        }
      },
      toggleLyric() {
        this.isImgShow = !this.isImgShow
      }
    },
    watch: {
      currentSong(newSong, oldSong) {
        if (newSong === oldSong) {
          return
        }
        if (this.lyric) { // 在切换歌曲的时候把当前的lyric停止以防歌词跳动
          this.lyric.stop()
        }

        this.lyricLines = [] // 清空歌词数组
        this.currentLineIndex = 0
        this.$nextTick(() => {
          this.$refs.audio.play()
          this._getLyric()
        })
      },
      isPlaying(newPlaying) {
        let audio = this.$refs.audio
        this.$nextTick(() => {
          newPlaying === true ? audio.play() : audio.pause()
        })
      }
    }
  }
</script>

<style lang="scss" scoped>
  @import '../common/sass/index';

  .icon-sequence, .icon-random,
  .icon-loop, .icon-prev,
  .icon-next, .icon-heart1,
  .icon-play-mini, .icon-list {
    font-size: px2rem(32);
    color: $color-theme;
  }

  .icon-heart2 {
    font-size: px2rem(32);
    color: red;
  }

  .icon-play, .icon-pause {
    font-size: px2rem(40);
    color: $color-theme;
  }

  .player {
    .bg {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      z-index: -2;
      background-size: cover;
      background-position: center;
      filter: blur(30px);
    }
    .layer {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba(0, 0, 0, .4);
      z-index: -1;
    }
  }

  .normal-player {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 101;
    background-color: $color-background;
    height: 100%;
    color: #fff;
    .header {
      .wrap {
        position: relative;
        padding: px2rem(8) 0;
        height: px2rem(32);
        .icon-back {
          position: absolute;
          left: px2rem(24);
          transform: rotate(-90deg);
          font-size: px2rem(24);
          color: $color-theme;
        }
        .song-name {
          position: absolute;
          left: 20%;
          width: 60%;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          text-align: center;
        }
      }
      .singer-name {
        padding-top: px2rem(8);
        text-align: center;
      }
    }
    .main {
      position: fixed;
      width: 100%;
      top: px2rem(52);
      bottom: px2rem(144);
      overflow: hidden;
      white-space: nowrap;
      .main-left {
        position: absolute;
        text-align: center;
        width: 100%;
        top: px2rem(82);
        overflow: hidden;
        .img {
          border-radius: 50%;
          width: 75%;
        }
      }
      .main-right {
        position: absolute;
        padding-top: px2rem(16);
        width: 100%;
        height: 100%;
        overflow: hidden;
        .scroll {
          height: 100%;
          overflow: hidden;
          .lyric-wrap {
            width: 100%;
            .lyric-line {
              display: flex;
              justify-content: center;
              align-items: center;
              padding-bottom: px2rem(16);
              font-size: px2rem(14);
              color: rgba(2555, 255, 255, 0.5);
              &.active {
                color: #fff;
              }
            }
          }
        }
      }
    }
    .footer {
      position: absolute;
      top: 80%;
      display: flex;
      flex-direction: column;
      width: 100%;
      justify-content: center;
      .time {
        display: flex;
        align-items: center;
        width: 100%;
        padding: px2rem(16) px2rem(48);
        font-size: px2rem(12);
        .time-left {
          width: px2rem(32);
          box-sizing: border-box;
          padding-right: px2rem(8);
        }
        .progress-wrap {
          width: 80%;
        }
        .time-right {
          width: px2rem(32);
          box-sizing: border-box;
          padding-left: px2rem(8);
        }
      }
      .icon-wrap {
        display: flex;
        justify-content: space-around;
        align-items: center;
        width: 100%;
        padding: 0 px2rem(32);
        text-align: center;
      }
    }
  }

  .mini-player {
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100%;
    padding: px2rem(8) px2rem(16);
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: $color-highlight-background;
    z-index: 101;
    .left {
      display: flex;
      img {
        width: px2rem(40);
        border-radius: 50%;
      }
      .content {
        float: left;
        padding-left: px2rem(12);
        display: flex;
        flex-direction: column;
        justify-content: center;
        h5 {
          color: #fff;
          width: px2rem(200);
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          padding-bottom: px2rem(8);
        }
        p {
          width: px2rem(200);
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          font-size: px2rem(14);
          color: $color-text-l;
        }
      }
    }
    .right {
      color: $color-theme;
      .icon-pause {
        font-size: px2rem(32);
      }
    }
  }

  .rotate {
    animation: rotate 20s infinite linear;
  }

  .pause {
    animation-play-state: paused;
  }

  @keyframes rotate {
    from {
      transform: rotate(0);
    }
    to {
      transform: rotate(360deg);
    }
  }

  .fade-enter-active, .fade-leave-active {
    transition: all 0.75s;
  }

  .fade-enter, .fade-leave-to {
    opacity: 0;
  }

</style>
