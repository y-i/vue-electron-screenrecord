<template>
  <div id="wrapper">
    <main>
      <div class="left-side">
        <video autoplay controls ref="video"></video>
        <button @click="saveFile">Stop &amp; Download</button>
      </div>

      <div class="right-side">
        <button @click="getCaptures">Reload</button>
        <div id="thumbnails">
          <img class="thumb" v-for="(thumb, i) in thumbnails" @click="setToVideoElem(thumb)" :src="thumb.src" :alt="thumb.name" :title="thumb.name" :key="i">
        </div>
      </div>
    </main>
  </div>
</template>

<script>
  import {desktopCapturer} from 'electron'
  const { dialog } = require('electron').remote

  export default {
    name: 'landing-page',
    components: {},
    data() {
      return {
        thumbnails: [],
        chunks: [],
        recorder: null
      }
    },
    mounted() {
      console.log('mounted')
      this.getCaptures()
    },
    methods: {
      getCaptures() {
        desktopCapturer.getSources({types: ['window', 'screen']}, (error, sources) => {
          if (error) throw error
          console.log(sources)
          this.thumbnails = []
          for (let source of sources) {
            this.thumbnails.push({
              id: source.id,
              name: source.name,
              src: source.thumbnail.toDataURL()
            })
          }
        })
      },
      async setToVideoElem(source) {
        console.log('set')
        const stream = await navigator.mediaDevices.getUserMedia({
          // audio: false,
          audio: {
            mandatory: {
              chromeMediaSource: 'desktop'
            }
          },
          /*
          */
          video: {
            mandatory: {
              chromeMediaSourceId: source.id,
              chromeMediaSource: 'desktop'
              /*
              minWidth: 1280,
              maxWidth: 1280,
              minHeight: 720,
              maxHeight: 720
              */
            }
          }
        }).catch(e => {
          console.error(e)
          throw e
        })
        this.$refs.video.srcObject = stream
        this.chunks = []
        this.recorder = new MediaRecorder(stream, {
          mimeType: 'video/webm'
        })
        this.recorder.addEventListener('dataavailable', ({data}) => {
          console.log('add')
          this.chunks.push(data)
        })
        this.recorder.start()
      },
      saveFile() {
        // this.recorder.requestData()
        this.recorder.stop()
        dialog.showSaveDialog({
          defaultPath: `./recorded-screen.webm`
        }, (filename, bookmark) => {
          console.log(filename, bookmark)
          if (!filename) return
          const fs = require('fs')
          const fileReader = new FileReader()
          fileReader.onload = () => {
            fs.writeFileSync(filename, Buffer.from(new Uint8Array(fileReader.result)))
          }
          fileReader.readAsArrayBuffer(new Blob(this.chunks))
        })
      },
      open (link) {
        this.$electron.shell.openExternal(link)
      }
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body { font-family: 'Source Sans Pro', sans-serif; }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }

  main {
    display: flex;
    justify-content: space-between;
  }

  main > div { flex-basis: 50%; }

  .left-side {
    display: flex;
    flex-direction: column;
  }

  #thumbnails {
    display: block;
    overflow: scroll;
    height: calc(100vh - 80px);
  }
  .thumb {
    object-fit: contain;
    width: 320px;
    height: 240px;
  }
  video {
    width: 480px;
    height: auto;
  }
</style>
