<template>
  <div class="w-full text-center" style="font-family: 'Ubi';">
    <img class="h-auto w-screen h-screen bg-black top-0 left-0 opacity-50 absolute" src="@/assets/background.jpg">
    <div class="relative text-gray-100">
      <div class="mt-10 p-2 border-t-2 border-b-2 border-gray-300">
        <div style="font-size: 50px;" class="font-semibold text-gray-800">Mixerinho</div>
      </div>
      <div class="my-10 text-xl">
        <div class="pb-2">
          Add file
        </div>
        <div class="flex justify-center text-sm">
          <div class="px-4 py-2 border-2 border-red rounded-xl mx-2 cursor-pointer" @click="fireUpload">
            Browse your files
          </div>
          <div class="flex items-center mx-2" v-if="filename">
            Chosen file: {{filename}}
          </div>
        </div>
        <div>
          <input id="upload" class="text-sm hidden" type="file" @change="handleFileUpload" accept="audio/*">
        </div>

      </div>
      <div v-if="file">
        <SoundMixer/>
        <div class="flex justify-center w-full my-4">
          <div class="border-2 border-red px-2 mx-2 cursor-pointer rounded border-gray-800 bg-gray-800"
               :class="{button: playing}"
               @click="playAudio">
            <span>Play</span>
          </div>
          <div class="border-2 border-red px-2 mx-2 cursor-pointer rounded border-gray-800 bg-gray-800"
               :class="{button: !playing}"
               @click="pauseAudio">
            <span>Pause</span>
          </div>
        </div>

        <div class="flex flex-col items-center">
          <div class="flex justify-center w-full my-4">
            <audio ref="foo" id="audio" :src="source"></audio>
            <audio ref="foo2" id="audio2" class="hidden" controls :src="source" :muted="true"></audio>
          </div>
          <div class="mb-6">
            <av-bars v-if="source"
                     :bar-color="['#f00', '#ff0', '#0f0']"
                     ref-link="foo"
                     :audio-controls="false"
                     :canv-width="1000"
                     :fft-size="4096"
                     :caps-drop-speed="10"
            />
          </div>
          <div class="mt-6">
            <av-waveform
                v-if="source"
                ref-link="foo2"
                :audio-controls="true"
                :canv-width="1000"
                :playtime-clickable="false"

            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import SoundMixer from './components/SoundMixer.vue'

export default {
  name: 'App',
  components: {
    SoundMixer,
  },

  data() {
    return {
      file: null,
      source: null,
      sourceCopy: null,
      filename: null,
      playing: false,
      audioCtx: new AudioContext(),
      reader: new FileReader(),
      soundSource: undefined,
    }
  },

  methods: {
    handleFileUpload(e) {
      var files = e.target.files || e.dataTransfer.files;
      if (!files.length) return;
      let file = files[0];
      this.file = file;
      this.source = URL.createObjectURL(file);
      this.filename = file.name;
      this.processFile(file);
      // document.getElementById("audio").load();
      // document.getElementById("audio2").load();
    },
    fireUpload() {
      // this.$el.querySelector("#upload").click();
      document.getElementById("upload").click();
    },

    processFile(file){
      this.reader.onload = (e) => {
        this.file = e.target.result;
      };
      this.reader.readAsArrayBuffer(file);
    },
    playAudio() {
      this.play();
      // this.play();
      // this.$el.querySelector("#audio2").play();
      // this.$el.querySelector("#audio").play();
      this.playing = true;
    },
    pauseAudio() {
      console.log(this.file)
      this.soundSource.stop(0);
      // this.$el.querySelector("#audio2").pause();
      // this.$el.querySelector("#audio").pause();
      this.playing = false;
    },
    play() {
      console.log(this.file)
      let fileCopy = this.copy(this.file);
      this.audioCtx.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSource = this.audioCtx.createBufferSource();
        this.soundSource.buffer = buffer;

        var compressor = this.audioCtx.createDynamicsCompressor();

        compressor.threshold.setValueAtTime(-20, this.audioCtx.currentTime);
        compressor.knee.setValueAtTime(-30, this.audioCtx.currentTime);
        compressor.ratio.setValueAtTime(5, this.audioCtx.currentTime);
        compressor.attack.setValueAtTime(.05, this.audioCtx.currentTime);
        compressor.release.setValueAtTime(.25, this.audioCtx.currentTime);

        this.soundSource.playbackRate.value = 0.5;
        this.soundSource.connect(compressor);

        compressor.connect(this.audioCtx.destination);
        this.soundSource.start(0);
      });
    },
    copy(src)  {
      let dst = new ArrayBuffer(src.byteLength);
      new Uint8Array(dst).set(new Uint8Array(src));
      return dst;
    }
  }
}
</script>
