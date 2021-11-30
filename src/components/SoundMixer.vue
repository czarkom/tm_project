<template>
  <div class="flex justify-between">
    <div class="flex p-4 justify-center w-full border-t-2 border-b-2 border-gray-200">
      <div class="px-4">
        Choose your effect:
      </div>
      <div class="flex justify-center">
        <div class="px-2 mr-4">
          <input type="radio" id="normal" value="normal" class="mr-2 cursor-pointer" v-model="chosenEffect">
          <label for="normal" class="cursor-pointer">Without effect</label>
        </div>
        <div class="px-2 mr-4">
          <input type="radio" id="bass_boosted" value="bass_boosted" class="mr-2" v-model="chosenEffect">
          <label for="bass_boosted" class="cursor-pointer">Bass boosted</label>
        </div>
        <div class="px-2 mr-4">
          <input type="radio" id="flanger" value="flanger" class="mr-2 cursor-pointer" v-model="chosenEffect">
          <label for="flanger" class="cursor-pointer">Flanger</label>
        </div>
        <div class="px-2 mr-4">
          <input type="radio" id="convolver" value="convolver" class="mr-2 cursor-pointer" v-model="chosenEffect">
          <label for="convolver" class="cursor-pointer">Convolver</label>
        </div>
        <div class="px-2 mr-4">
          <input type="radio" id="bitcrusher" value="bitcrusher" class="mr-2 cursor-pointer" v-model="chosenEffect">
          <label for="bitcrusher" class="cursor-pointer">Bitcrusher</label>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SoundMixer',
  props: {
    modelValue: String
  },
  emits:['update:modelValue'],
  data() {
    return {
      chosenEffect: "normal"
    }
  },
  watch: {
    chosenEffect() {
      this.$emit('update:modelValue', this.chosenEffect);
    }
  }
}
</script>

