<script setup lang="ts">
import { ref, Ref, computed } from 'vue'
import { times, cloneDeep } from 'lodash-es'
import { useSound } from '@vueuse/sound'
import { Vex } from 'vexflow'
import { onMounted } from 'vue'

const { Factory, EasyScore, System } = Vex.Flow

onMounted(() => {
  const vf = new Factory({
    renderer: { elementId: 'output', width: 500, height: 200 },
  })

  const score = vf.EasyScore()
  const system = vf.System()

  system
    .addStave({
      voices: [
        score.voice(score.notes('C#5/q, B4, A4, G#4', { stem: 'up' })),
        score.voice(score.notes('C#4/h, C#4', { stem: 'down' })),
      ],
    })
    .addClef('treble')
    .addTimeSignature('4/4')

  vf.draw()
})

interface PKey {
  note: string
  sharp: boolean
  noteNum: number
  octave: number
}

const keySounds = times(24, num => {
  const { play } = useSound(`/public/keys/key${num + 1}.ogg`)
  return play
})
// const { play } = useSound('/public/keys/key01.ogg')

const playNote = (index: number) => {
  keySounds[index]()
}
const octaves = ref(2)
const spacings = computed(() => {
  return {
    key: 100 / octaves.value / 7,
    sharpKey: 100 / octaves.value / 10,
    sharpKeyMargin: 9 / octaves.value,
  }
})

const octave: PKey[] = [
  { note: 'Do', sharp: false, noteNum: 0, octave: 0 },
  { note: 'Do', sharp: true, noteNum: 0, octave: 0 },
  { note: 'Re', sharp: false, noteNum: 1, octave: 0 },
  { note: 'Re', sharp: true, noteNum: 1, octave: 0 },
  { note: 'Mi', sharp: false, noteNum: 2, octave: 0 },
  { note: 'Fa', sharp: false, noteNum: 3, octave: 0 },
  { note: 'Fa', sharp: true, noteNum: 3, octave: 0 },
  { note: 'Sol', sharp: false, noteNum: 4, octave: 0 },
  { note: 'Sol', sharp: true, noteNum: 4, octave: 0 },
  { note: 'La', sharp: false, noteNum: 5, octave: 0 },
  { note: 'La', sharp: true, noteNum: 5, octave: 0 },
  { note: 'Si', sharp: false, noteNum: 6, octave: 0 },
]

const pkeys: Ref<PKey[]> = computed(() => {
  return times(octaves.value, num =>
    cloneDeep(octave).map(pk => {
      pk.octave = num
      return pk
    }),
  ).flat()
})
console.log(pkeys)
</script>

<template>
  <div>
    <v-container>
      <div class="keys">
        <div v-for="(pkey, index) in pkeys" style="display: contents" :key="pkey.note">
          <div
            :class="{ key: true, sharp: pkey.sharp }"
            :style="{
              width: `${pkey.sharp ? spacings.sharpKey : spacings.key}%`,
              marginLeft: pkey.sharp
                ? `${pkey.noteNum * spacings.key + pkey.octave * 7 * spacings.key + spacings.sharpKeyMargin}%`
                : '',
            }"
            @click="playNote(index)"
          ></div>
        </div>
      </div>
      <div id="output"></div>
      <!-- <v-card :width="500" class="settings mt-5" elevation="0">
        <v-text-field v-model="octaves" label="Октавы" type="number"></v-text-field>
      </v-card> -->
    </v-container>
  </div>
</template>

<style lang="scss">
.keys {
  display: flex;
  height: 350px;
  position: relative;
  margin: 40px auto 0;

  .key {
    position: relative;
    border: 4px solid black;
    border-radius: 0.5rem;
    transition: all 0.07s ease;
    display: block;
    box-sizing: border-box;
    z-index: 2;
  }

  .key:not(.sharp) {
    float: left;
    height: 100%;
    background: rgba(255, 255, 255, 0.8);
  }

  .key.sharp {
    position: absolute;
    width: 6%;
    height: 60%;
    background: #000;
    color: #eee;
    top: 0;
    z-index: 3;
  }
}

.settings {
}
</style>
