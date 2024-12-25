<script setup lang="ts">
import { onMounted, onUnmounted, reactive, ref, unref } from 'vue';

const RUN_TIME = 30;

const time = ref(RUN_TIME);

import text from './assets/parapraph.txt?raw';
const raw_words = text.toLocaleLowerCase().split(" ");

const enum LetterStatus {
  n = -1,
  c = 0,
  w = 1,
}
type tLetter = {
  status: LetterStatus,
  l: string,
  active: boolean,
}
type tWord = {
  active: boolean;
  letters: tLetter[];
}

const words = reactive<tWord[]>([]);

let wordTotal = 0;
let wordIndex = 0;

function keyDown(e: KeyboardEvent) {
  const currentWord = words[wordIndex];
  if (!currentWord)
    return;

  const _input = $input.value;
  // const value = _input.value.trim().split('');
  const value = _input.value.trim();

  switch (e.key) {
    case 'Backspace': {
      if (value.length === 0) {
        if (!words[wordIndex - 1]?.letters.some(l => l.status !== LetterStatus.c))
          break;
        
        e.preventDefault();

        currentWord.active = false;
        currentWord.letters[0].active = false;
        
        const nextWord = words[--wordIndex];
        
        const vs = {
          [LetterStatus.w] : '*',
          [LetterStatus.n] : '',
          [LetterStatus.c] : null,
        };

        const newValue = nextWord.letters.map(l => vs[l.status] ?? l.l).join('');
        nextWord.active = true;
        
        if (newValue.length < nextWord.letters.length)
          nextWord.letters[newValue.length].active = true;
        
        _input.maxLength = nextWord.letters.length;
        _input.value = newValue;
        break;
      }

      if (e.ctrlKey) {
        for (const letter of currentWord.letters) {
          letter.active = false;
          letter.status = LetterStatus.n;
        }
        
        currentWord.letters[0].active = true;
        break;
      }

      
      if (value.length !== currentWord.letters.length) 
        currentWord.letters[value.length].active = false;
      
      const letter = currentWord.letters[value.length - 1];
      
      letter.status = LetterStatus.n;
      letter.active = true;      
    } 
    break;
    case ' ':{
      e.preventDefault();

      if (wordTotal - 1 === wordIndex || value.length === 0 || currentWord.letters.every(l => l.status === LetterStatus.n))
        break;

      currentWord.active = false;
      currentWord.letters.forEach(l => l.active = false);

      achieve();

      $input.value.value = '';

      setWord(words[++wordIndex]);
    }
    break;
    default:

      if (e.key.length !== 1) {
        e.preventDefault();
        break;
      }

      const vsize = value.length + 1
      const nvalue = value.concat(e.key);
  
      for (const [i, letter] of currentWord.letters.entries()) {
        letter.active = false;
        letter.status = i >= vsize ? LetterStatus.n : letter.l === nvalue[i] ? LetterStatus.c : LetterStatus.w;
      }
        
      if (vsize < currentWord.letters.length)
        currentWord.letters[vsize].active = true;
    break;
  }
}

function setWord(word: tWord) {
  word.active = true;
  word.letters[0].active = true;
  $input.value.maxLength = word.letters.length;
}

function achieve() {
  if (wordTotal - wordIndex > 10) return

  const l = raw_words.length;
  words.push(...Array.from(Array(12), () => genWord(raw_words[Math.ceil(Math.random() * l)])));

  wordTotal += 12;
}


function gameEnd() {
  gameover.value = true;

  document.onkeydown = null;
  $input.value.onkeydown = null;
  $input.value.onkeyup = null;

  const [correct_words, correct_letters, written_letters] = words.reduce((p, w) => {
    let c = 0, t = 0;

    for (const l of w.letters)
      if (l.status === LetterStatus.n)
        break;
      else {
        t += 1;
        c += Number(l.status === LetterStatus.c);
      }

    p[0] += Number(c === w.letters.length)
    p[1] += c;
    p[2] += t;

    return p;
  }, [0, 0, 0] as any[]);

  wpm.value = correct_words * (60 / RUN_TIME);
  accuracy.value = written_letters ? correct_letters / written_letters : 0;
}

function genWord(w: string) {
  return {
    active: false,
    letters: Array.from(w, e => ({
      status: LetterStatus.n,
      l: e,
      active: false,
    }))
  };
}

function gameStart() {
  gameover.value = false;

  document.onkeydown = () => $input.value.focus();
  $input.value.onkeydown = keyDown;
  
  // $input.value.focus();
  setTimeout(() => $input.value.focus());

  words.length = 0;
  words.push(...raw_words.slice(0, 24).sort(() => Math.random() - .5).map(genWord));

  wordIndex = 0;
  wordTotal = words.length;

  time.value = RUN_TIME;
  let s = setInterval(() => {
    time.value -= 1;
    if (time.value <= 0) {
      gameEnd();
      clearInterval(s)
    }
  }, 1000);

  wpm.value = 0;
  accuracy.value = 0;

  setWord(words[0]);
}

onMounted(() => {
  document.addEventListener('click', () => {
    interaction.value = true;
    gameStart();
  }, { once : true });
});

onUnmounted(() => {
  $input.value.onkeydown = null;
  $input.value.onkeyup = null;
});

const interaction = ref(false);

const $input = ref<HTMLInputElement>(null as any);

const gameover = ref(false);

const wpm = ref(0);
const accuracy = ref(0);
</script>

<template>
  <main style="max-width: 100ch; margin-inline: auto; margin-bottom: 2rem;">
    <header>
      <hgroup>
        <h1 style="color: gold;">
          MonkeyType Clone!
        </h1>
        <p>Esta es una copia del juego MonkeyType. Si os gusta, no duden en visitar el <a href="https://monkeytype.com"
            style="color: gold;">juego original</a>.</p>
      </hgroup>
      <p :style="{ visibility: interaction ? 'hidden' : 'visible' }"><i style="opacity: .75;">para iniciar el juego, es
        necesario un click en la página</i></p>
      <!-- <pre>{{ words[wordIndex] }}</pre> -->
    </header>
    <section v-show="interaction">
      <p style="color: gold">
        tiempo: <time :date="new Date(`PT${time}S`)" :format="{ second: '2-digit' }">{{ time }}</time>
      </p>
      <p class="board" ref="board">
        <TransitionGroup name="appear">
          <span v-for="w, i of words" :key="i" :class="[w.active && 'active']">
            <span v-for="l, i of w.letters" :key="i" :data-status="l.status" v-text="l.l"
              :class="[l.active && 'active']"></span>
          </span>
        </TransitionGroup>
      </p>
      <input type="text" ref="$input" autofocus style="position: absolute; left: 0; top: -0vh;" />
    </section>

    <aside v-show="gameover">
      <h2>PPM</h2>
      <output :value="wpm.toFixed(2)"></output>
      <h2>precisión</h2>
      <output :value="`${(accuracy * 100).toFixed(2)}%`"></output>

      <button style="display: block; margin-block: 1rem; background-color: #333; padding : .5rem 1rem"
        @click="gameStart">Jugar Otra vez</button>
    </aside>
  </main>
  <footer style="position: fixed; bottom: 0; padding : .5rem; text-align: center; width: 100%; background-color: #111;">
    <small>Proyecto basado en las guia de Midudev.</small>
  </footer>
</template>

<style>
.appear-enter-active,
.appear-leave-active {
  transition: opacity .3s ease-out, translate .3s ease-out;
  /* transition-delay: calc(.15s * (var(--d) - var(--t))); */
}

.appear-enter-from,
.appear-leave-to {
  opacity: 0;
  translate: 0 1ch;
}

main {
  padding: 1rem;
}

.board {
  font-family: Menlo, monospace;
  display: flex;
  flex-wrap: wrap;
  gap: .25ch 1ch;
}

.board>span {
  border-bottom: 2px solid transparent;
  letter-spacing: .25ch;
}

.board>span.active:not(:has(.active))>span:last-child::before {
  content: '|';
  position: absolute;
  translate: +.75ch 0;
  color: gold;
  animation: 1s blink infinite linear;
}

.board>span:has(> span:not([data-status="0"])):has(~ span.active) {
  border-color: red;
}

.board>span>span[data-status="0"] {
  color: lime;
}

.board>span>span[data-status="1"] {
  color: red;
}

.board>span>span.active::before {
  content: '|';
  position: absolute;
  translate: -.75ch 0;
  color: gold;

  animation: 1s blink infinite linear;
}



@keyframes blink {

  0%,
  35% {
    opacity: 1;
  }

  75% {
    opacity: 0;
  }
}
</style>
