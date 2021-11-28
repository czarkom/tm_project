<template>
  <div class="w-full text-center" style="font-family: 'Ubi';">
    <img class="h-auto w-screen h-screen bg-black top-0 left-0 opacity-50 absolute" src="@/assets/background.jpg">
    <div class="relative text-gray-100">
      <div class="mt-6 p-2 border-t-2 border-b-2 border-gray-300">
        <div style="font-size: 50px;" class="font-semibold text-gray-800">Mixerinho {{ chosenEffect }}</div>
      </div>
      <div class="my-4 text-xl">
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
      <div v-if="fileArrayBuffer">
        <SoundMixer @update:model-value="handleChosenEffect($event)" :model-value="chosenEffect"/>
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
          <div class="flex justify-center w-full">
            <audio ref="foo" id="audio"></audio>
            <audio ref="foo2" id="audio2"></audio>
          </div>
          <div class="mb-6" v-if="showCharts">
            <div class="flex justify-center">Frequency chart for sound without effect</div>
            <av-bars
                     :bar-color="['#f00', '#ff0', '#0f0']"
                     ref-link="foo"
                     :audio-controls="true"
                     :canv-width="1000"
                     :fft-size="4096"
                     :caps-drop-speed="10"
            />
          </div>
          <div class="mb-6" v-if="showCharts && chosenEffect !== 'normal'">
            <div class="flex justify-center">Frequency chart for sound with {{chosenEffect}} effect</div>
            <av-bars
                :bar-color="['#f00', '#ff0', '#0f0']"
                ref-link="foo2"
                :audio-controls="true"
                :canv-width="1000"
                :fft-size="4096"
                :caps-drop-speed="10"
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
      fileArrayBuffer: null,
      filename: null,
      playing: false,
      reader: new FileReader(),
      chosenEffect: 'normal',
      showCharts: false,
      soundSourceNormal: null,
      soundSourceWithEffect: null,
      audioCtxNormal: null,
      audioCtxEffect: null
    }
  },

  methods: {
    handleFileUpload(e) {
      var files = e.target.files || e.dataTransfer.files;
      if (!files.length) return;
      let file = files[0];
      this.source = URL.createObjectURL(file);
      this.filename = file.name;
      this.processFile(file);
    },
    fireUpload() {
      this.$el.querySelector("#upload").click();
    },

    processFile(file) {
      this.reader.onload = (e) => {
        this.fileArrayBuffer = e.target.result;
        this.prepareNormal();
      };
      this.reader.readAsArrayBuffer(file);
    },
    playAudio() {
      if(this.audioCtxNormal.state === 'suspended'){
        this.audioCtxNormal.resume();
      } else {
        this.soundSourceNormal.start(0);
      }
      if(this.chosenEffect !== 'normal' && this.audioCtxEffect && this.audioCtxEffect.state === 'suspended'){
        this.audioCtxEffect.resume();
      } else if(this.chosenEffect !== 'normal') {
        this.soundSourceWithEffect.start(0);
      }
      this.$el.querySelector("#audio2").play();
      this.$el.querySelector("#audio").play();
      this.playing = true;
    },
    pauseAudio() {
      this.audioCtxNormal.suspend();
      if(this.chosenEffect !== 'normal') this.audioCtxEffect.suspend();
      this.$el.querySelector("#audio2").pause();
      this.$el.querySelector("#audio").pause();
      this.playing = false;
    },
    copy(src) {
      let dst = new ArrayBuffer(src.byteLength);
      new Uint8Array(dst).set(new Uint8Array(src));
      return dst;
    },
    prepareNormal() {
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxNormal = new AudioContext();
      this.audioCtxNormal.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceNormal = this.audioCtxNormal.createBufferSource();
        this.soundSourceNormal.buffer = buffer;

        const streamNode = this.audioCtxNormal.createMediaStreamDestination();
        this.soundSourceNormal.connect(streamNode);

        const gainNode = this.audioCtxNormal.createGain();
        this.soundSourceNormal
            .connect(gainNode)
            .connect(this.audioCtxNormal.destination)

        if(this.chosenEffect !== 'normal'){
          console.log("Original audio muted")
          gainNode.gain.value = -2;
        }

        let audioElem = this.$el.querySelector("#audio");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    prepareFlanger() {
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxEffect = new AudioContext();
      this.audioCtxEffect.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceWithEffect = this.audioCtxEffect.createBufferSource();
        this.soundSourceWithEffect.buffer = buffer;

        const streamNode = this.audioCtxEffect.createMediaStreamDestination();

        let biquadFilter = this.audioCtxEffect.createBiquadFilter();
        let convolver = this.audioCtxEffect.createConvolver();
        convolver.buffer = buffer;
        console.log("bassfilter prepared")

        this.soundSourceWithEffect
            .connect(biquadFilter)
            .connect(convolver)
            .connect(streamNode)

        biquadFilter.type = "lowshelf";
        biquadFilter.frequency.setValueAtTime(1000, this.audioCtxEffect.currentTime);
        biquadFilter.gain.setValueAtTime(50, this.audioCtxEffect.currentTime);

        let audioElem = this.$el.querySelector("#audio2");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    prepareBassBoosted() {
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxEffect = new AudioContext();
      this.audioCtxEffect.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceWithEffect = this.audioCtxEffect.createBufferSource();
        this.soundSourceWithEffect.buffer = buffer;

        let bassFilter = this.audioCtxEffect.createBiquadFilter();
        bassFilter.type = "lowshelf";
        bassFilter.frequency.value = 1000;  // switches to 400 in UI

        const streamNode = this.audioCtxEffect.createMediaStreamDestination();

        this.soundSourceWithEffect.connect(streamNode);

        let audioElem = this.$el.querySelector("#audio2");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    handleChosenEffect(val) {
      this.chosenEffect = val
      this.$el.querySelector("#audio").pause();
      this.$el.querySelector("#audio2").pause();
      this.audioCtxNormal.suspend();
      if(this.audioCtxEffect) this.audioCtxEffect.suspend();
      this.playing = false;
      if(val === "normal"){
        this.prepareNormal();
      } else if (val === "flanger"){
        this.prepareNormal();
        this.prepareFlanger();
      } else if (val === "bass_boosted"){
        this.prepareNormal();
        this.prepareBassBoosted();
      }
    }
  }
}
</script>
