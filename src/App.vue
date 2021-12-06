<template>
  <div class="w-full text-center" style="font-family: 'Ubi';">
    <img class="h-auto w-screen h-screen bg-black top-0 left-0 opacity-50 absolute" src="@/assets/background.jpg">
    <div class="relative text-gray-100">
      <div class="mt-6 p-2 border-t-2 border-b-2 border-gray-300">
        <div style="font-size: 50px;" class="font-semibold text-gray-800">Mixerinho</div>
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
                     :cors-anonym="true"
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

        let inputNode = this.audioCtxEffect.createGain();
        let outputNode = this.audioCtxEffect.createGain();
        let inputFeedbackNode = this.audioCtxEffect.createGain();
        let wetGainNode = this.audioCtxEffect.createGain();
        let dryGainNode = this.audioCtxEffect.createGain();
        let delayNode = this.audioCtxEffect.createDelay();
        let gainNode = this.audioCtxEffect.createGain();
        let feedbackNode = this.audioCtxEffect.createGain();

        inputNode.connect(inputFeedbackNode);
        inputNode.connect(dryGainNode);

        inputFeedbackNode.connect(delayNode);
        inputFeedbackNode.connect(wetGainNode);

        delayNode.connect(wetGainNode);
        delayNode.connect(feedbackNode);

        feedbackNode.connect(inputFeedbackNode);

        this.soundSourceWithEffect.connect(gainNode);
        gainNode.connect(delayNode).connect(streamNode);

        delayNode.delayTime.value = 0.5
        gainNode.gain.value = 0.7
        dryGainNode.gain.value = 0.3
        wetGainNode.gain.value = 0.5
        feedbackNode.gain.value = 0.7

        dryGainNode.connect(outputNode);
        wetGainNode.connect(outputNode);

        console.log("flanger prepared")

        let audioElem = this.$el.querySelector("#audio2");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    prepareLowpassFilter() {
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxEffect = new AudioContext();
      this.audioCtxEffect.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceWithEffect = this.audioCtxEffect.createBufferSource();
        this.soundSourceWithEffect.buffer = buffer;

        let biquadFilter = this.audioCtxEffect.createBiquadFilter();

        const streamNode = this.audioCtxEffect.createMediaStreamDestination();

        this.soundSourceWithEffect
            .connect(biquadFilter)
            .connect(streamNode);

        biquadFilter.type = "lowpass";
        biquadFilter.frequency.setValueAtTime(700, this.audioCtxEffect.currentTime);
        biquadFilter.gain.setValueAtTime(2000, this.audioCtxEffect.currentTime);

        let audioElem = this.$el.querySelector("#audio2");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    prepareConvolver() {
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxEffect = new AudioContext();
      this.audioCtxEffect.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceWithEffect = this.audioCtxEffect.createBufferSource();
        this.soundSourceWithEffect.buffer = buffer;

        const streamNode = this.audioCtxEffect.createMediaStreamDestination();

        let convolver = this.audioCtxEffect.createConvolver();
        let noiseBuffer = this.audioCtxEffect.createBuffer(2, 0.5 * this.audioCtxEffect.sampleRate, this.audioCtxEffect.sampleRate);
        let left = noiseBuffer.getChannelData(0);
        let right = noiseBuffer.getChannelData(1);
        for (let i = 0; i < noiseBuffer.length; i++) {
          left[i] = Math.random() * 2 - 1;
          right[i] = Math.random() * 2 - 1;
        }

        convolver.buffer = noiseBuffer;
        console.log("Convolver prepared")

        this.soundSourceWithEffect
            .connect(convolver)
            .connect(streamNode)

        let audioElem = this.$el.querySelector("#audio2");
        audioElem.srcObject = streamNode.stream;
        this.showCharts = true;
      });
    },
    prepareBitcrusher(){
      let fileCopy = this.copy(this.fileArrayBuffer);
      this.audioCtxEffect = new AudioContext();
      this.audioCtxEffect.decodeAudioData(fileCopy).then((buffer) => {
        this.soundSourceWithEffect = this.audioCtxEffect.createBufferSource();
        this.soundSourceWithEffect.buffer = buffer;

        const streamNode = this.audioCtxEffect.createMediaStreamDestination();

        console.log("Bitcrusher prepared")
        const bufferSize = 4096;
        let bitcrusher = this.audioCtxEffect.createScriptProcessor(bufferSize, 1, 1);
        bitcrusher.bits = 4; // between 1 and 16
        bitcrusher.normfreq = 1; // between 0.0 and 1.0
        let step = Math.pow(1/2, bitcrusher.bits);
        let phaser = 0;
        let last = 0;
        bitcrusher.onaudioprocess = function(e) {
          var input = e.inputBuffer.getChannelData(0);
          var output = e.outputBuffer.getChannelData(0);
          for (var i = 0; i < bufferSize; i++) {
            phaser += bitcrusher.normfreq;
            if (phaser >= 1.0) {
              phaser -= 1.0;
              last = step * Math.floor(input[i] / step + 0.5);
            }
            output[i] = last;
          }
        };
        console.log("bitcrusher prepared")

        this.soundSourceWithEffect
            .connect(bitcrusher)
            .connect(streamNode)

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
      } else if (val === "lowpass_filter"){
        this.prepareNormal();
        this.prepareLowpassFilter();
      } else if (val === 'convolver'){
        this.prepareNormal();
        this.prepareConvolver();
      } else if (val === 'bitcrusher'){
        this.prepareNormal();
        this.prepareBitcrusher();
      }
    },

  }
}
</script>
