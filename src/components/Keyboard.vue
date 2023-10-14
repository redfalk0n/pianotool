<script setup lang="ts">
import { ref, Ref, computed } from 'vue'
import { cloneDeep, times } from 'lodash-es'
import { useSound } from '@vueuse/sound'
import { Vex } from 'vexflow'
import { onMounted } from 'vue'

const { Factory } = Vex.Flow

onMounted(() => {
  drawScore()
})

const drawNote = (pkey: PKey) => {
  voices.push('C#6/q')
  drawScore()
}

const voices: String[] = []
const drawScore = () => {
  const vf = new Factory({
    renderer: { elementId: 'output', width: 500, height: 200 },
  })

  const score = vf.EasyScore()
  const system = vf.System()
  let normVoices: String[] = cloneDeep(voices)
  if (voices.length === 0 || voices.length % 4 !== 0) {
    times(4 - (voices.length % 4), () => normVoices.push('B4/r'))
  }

  system
    .addStave({
      voices: voices.length > 0 ? [score.voice(score.notes(normVoices.join(', '), { stem: 'up' }))] : [],
    })
    .addClef('treble')
    .addTimeSignature('4/4')

  vf.draw()
}

interface Sound {
  play: Function
  stop: Function
  pause: Function
  isPlaying: Ref<boolean>
  duration: Ref<number | null>
}

interface PKey {
  note: string
  sharp: boolean
  noteNum: number
  octave: number
  sound: Sound
  active: boolean
}

const isMouseDown = ref(false)
const octaves = ref(3)
const spacings = computed(() => {
  return {
    key: 100 / octaves.value / 7,
    sharpKey: 100 / octaves.value / 10,
    sharpKeyMargin: 9 / octaves.value,
  }
})

const notations = {
  0: ['C_1', 'D_1', 'E_1', 'F_1', 'G_1', 'A_1', 'B_1'],
  1: ['C', 'D', 'E', 'F', 'G', 'A', 'B'],
  2: ['cc', 'dd', 'ee', 'ff', 'gg', 'aa', 'bb'],
  3: ['c1', 'd1', 'e1', 'f1', 'g1', 'a1', 'b1'],
  4: ['c2', 'd2', 'e2', 'f2', 'g2', 'a2', 'b2'],
  5: ['c3', 'd3', 'e3', 'f3', 'g3', 'a3', 'b3'],
  6: ['c4', 'd4', 'e4', 'f4', 'g4', 'a4', 'b4'],
}

let keyboard: PKey[] = []

Object.entries(notations).forEach(oct => {
  oct[1].forEach((key, index) => {
    const sound = useSound(`/public/keys/${key}.mp3`)
    keyboard.push({
      note: key,
      octave: Number(oct[0]),
      noteNum: index,
      sharp: false,
      sound: sound,
      active: false,
    })
    if ([0, 1, 3, 4, 5].includes(index)) {
      const sharpSound = useSound(`/public/keys/${key}s.mp3`)
      keyboard.push({
        note: `${key}s`,
        octave: Number(oct[0]),
        noteNum: index,
        sharp: true,
        sound: sharpSound,
        active: false,
      })
    }
  })
})

const pkeys: Ref<PKey[]> = ref([])

if (octaves.value === 1) {
  pkeys.value = cloneDeep(keyboard).slice(36, 48)
} else if (octaves.value === 2) {
  pkeys.value = cloneDeep(keyboard).slice(36, 60)
} else if (octaves.value === 3) {
  pkeys.value = cloneDeep(keyboard).slice(24, 60)
} else {
  pkeys.value = keyboard
}
</script>

<template>
  <div>
    <v-btn @click="drawNote(pkeys[0])">asd</v-btn>
    <v-container>
      <div class="keys" @mouseleave="isMouseDown = false">
        <div v-for="(pkey, index) in pkeys" style="display: contents" :key="pkey.note">
          <div
            :class="{ key: true, sharp: pkey.sharp }"
            :style="{
              width: `${pkey.sharp ? spacings.sharpKey : spacings.key}%`,
              marginLeft: pkey.sharp
                ? `${
                    pkey.noteNum * spacings.key + Math.floor(index / 12) * 7 * spacings.key + spacings.sharpKeyMargin
                  }%`
                : '',
              background: pkey.active ? 'gray' : '',
            }"
            @mouseenter="
              () => {
                if (isMouseDown) {
                  pkey.active = true
                  pkey.sound.play()
                }
              }
            "
            @mousedown="
              () => {
                pkey.active = true
                isMouseDown = true
                pkey.sound.play()
              }
            "
            @mouseup="
              () => {
                pkey.active = false
                isMouseDown = false
              }
            "
            @mouseleave="() => (pkey.active = false)"
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
