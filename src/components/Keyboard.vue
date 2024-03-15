<script setup lang="ts">
import { ref, Ref, computed, onMounted } from 'vue'
import { cloneDeep, times } from 'lodash-es'
import { useSound } from '@vueuse/sound'
import { Accidental, Beam, Formatter, RenderContext, Renderer, Stave, StaveNote, Tuplet, Voice } from 'vexflow'

let context: RenderContext
let stave: Stave
const staveContainer = ref()
const notes: Ref<StaveNote[]> = ref([])
const durations = ['1', '2', '4', '8', '16']
const currentDuration = ref('1')

onMounted(() => {
  const renderer = new Renderer(staveContainer.value, Renderer.Backends.SVG)
  renderer.resize(1200, 500)
  context = renderer.getContext()
  redrawStave()
})

// const clearStave = () => {
//   redrawStave()
//   notes = []
// }

const redrawStave = () => {
  context.clear()
  stave = new Stave(10, 40, notes.value.length * 50 + 100)
  stave.addClef('treble').addTimeSignature('4/4')
  stave.setContext(context).draw()

  // trioles
  notes.value.push(new StaveNote({ keys: ['c/4'], duration: currentDuration.value }))
  notes.value.push(new StaveNote({ keys: ['c/4'], duration: currentDuration.value }))
  notes.value.push(new StaveNote({ keys: ['c/4'], duration: currentDuration.value }))
  const tupl = new Tuplet(notes.value)
  Formatter.FormatAndDraw(context, stave, notes.value, { align_rests: true, auto_beam: true })
  tupl.setContext(context).draw()
}

const drawNote = (pkey: PKey) => {
  const note = new StaveNote({ keys: [pkey.note.replace('_', '/')], duration: currentDuration.value })
  if (pkey.sharp) {
    note.addModifier(new Accidental('#'))
  }
  notes.value.push(note)
  redrawStave()
  Formatter.FormatAndDraw(context, stave, notes.value, { align_rests: true, auto_beam: true })
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
  2: ['c_3', 'd_3', 'e_3', 'f_3', 'g_3', 'a_3', 'b_3'],
  3: ['c_4', 'd_4', 'e_4', 'f_4', 'g_4', 'a_4', 'b_4'],
  4: ['c_5', 'd_5', 'e_5', 'f_5', 'g_5', 'a_5', 'b_5'],
  5: ['c3', 'd3', 'e3', 'f3', 'g3', 'a3', 'b3'],
  6: ['c4', 'd4', 'e4', 'f4', 'g4', 'a4', 'b4'],
}

let keyboard: PKey[] = []

Object.entries(notations).forEach(oct => {
  oct[1].forEach((key, index) => {
    const sound = useSound(`/keys/${key}.mp3`)
    keyboard.push({
      note: key,
      octave: Number(oct[0]),
      noteNum: index,
      sharp: false,
      sound: sound,
      active: false,
    })
    if ([0, 1, 3, 4, 5].includes(index)) {
      const sharpSound = useSound(`/keys/${key}s.mp3`)
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
    <v-btn
      @click="
        () => {
          notes = []
          redrawStave()
        }
      "
      >Clear</v-btn
    >
    <v-btn
      @click="
        () => {
          notes.pop()
          redrawStave()
          Formatter.FormatAndDraw(context, stave, notes, { align_rests: true, auto_beam: true })
        }
      "
      >Back</v-btn
    >
    <div class="d-flex mt-3">
      <v-btn
        v-for="duration in durations"
        :key="duration"
        @click="currentDuration = duration"
        :active="currentDuration === duration"
      >
        {{ duration }}
      </v-btn>
    </div>
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
                  drawNote(pkey)
                }
              }
            "
            @mousedown="
              () => {
                pkey.active = true
                isMouseDown = true
                pkey.sound.play()
                drawNote(pkey)
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
      <div id="output" ref="staveContainer"></div>
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
