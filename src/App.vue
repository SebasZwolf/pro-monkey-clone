<script setup lang="ts">
import { onMounted, onUnmounted, reactive, ref } from 'vue';

const RUN_TIME = 30;

const time = ref(RUN_TIME);

const raw_words = ("Reprehenderit deserunt nisi eiusmod voluptate ea do aute irure nostrud laborum amet sint. Nostrud commodo amet non nostrud tempor minim nisi aliqua sint occaecat. Exercitation id ut aliqua ex elit deserunt do aute aliquip aliqua. Laborum laborum Lorem eu exercitation laboris reprehenderit sunt laboris id qui. Quis mollit commodo duis labore nostrud dolor dolore exercitation dolore consectetur fugiat do. Exercitation sit nostrud id aliquip proident qui cupidatat ut laborum esse excepteur sint exercitation qui.".toLocaleLowerCase().split(" "))

enum LetterStatus {
  n = -1,
  c,
  w,
}
type tLetter = {
  status : LetterStatus,
  l : string,
  active : boolean,
}
type tWord = { 
  active : boolean;
  letters : tLetter[];
}

const words = reactive<tWord[]>([]);

let wordTotal = 0;
let wordIndex = 0;

function keyDown(e : KeyboardEvent) {

  const currentWord = words[wordIndex];
  if (!currentWord) return;

  const value = (e.target as HTMLInputElement).value.trim().split('');

  if (e.key === ' ' || e.key.startsWith('Arrow')) 
    e.preventDefault();

  if (e.key === 'Backspace' && value.length === 0 && wordIndex > 0 && words[wordIndex - 1].letters.some(l => l.status != LetterStatus.c)) {
    e.preventDefault();

    keyUp(e);

    currentWord.active = false;
    currentWord.letters[0].active = false;
    // currentWord.letters.forEach(l => l.active = false);
    
    const nextWord = words[--wordIndex];

    const newValue = nextWord.letters.map(l => l.status === LetterStatus.c ? l.l : l.status === LetterStatus.w ? '*' : '').join('');
    
    nextWord.active = true;
    if (newValue.length < nextWord.letters.length) 
      nextWord.letters[newValue.length].active = true;

    $input.value.value = newValue;
    $input.value.maxLength = nextWord.letters.length;
  }  
}

function achieve() {
  if (wordTotal - wordIndex > 10) return 
  
  const l = raw_words.length;
  words.push(...Array.from(Array(12), () => genWord(raw_words[Math.ceil(Math.random() * l)])));

  wordTotal+= 12;
}

function keyUp(e : KeyboardEvent) {
  const currentWord = words[wordIndex];
  if (!currentWord) return;

  const value = (e.target as HTMLInputElement).value.trim().split('');
  const l = value.length;

  if (e.key === ' ') {
    if (wordTotal - 1 === wordIndex || l === 0 || currentWord.letters.every(l => l.status === LetterStatus.n))
      return;

    currentWord.active = false;
    currentWord.letters.forEach(l => l.active = false);
    // if (l < currentWord.w.length) currentWord.letters[l].active = false;

    achieve();

    $input.value.value = '';
    
    setWord(words[++wordIndex]);
    return;
  }
  
  for (let i = currentWord.letters.length - 1; i >= 0; i--) {
    const letter = currentWord.letters[i];

    letter.active = false;
    letter.status = i >= l ? LetterStatus.n : letter.l === value[i] ? LetterStatus.c : LetterStatus.w;
  }

  if (l < currentWord.letters.length) 
    currentWord.letters[l].active = true;
}

function setWord(word : tWord) {
  word.active = true;
  word.letters[0].active = true;
  $input.value.maxLength = word.letters.length;
}



function gameEnd() {
  gameover.value = true;

  document.onkeydown = null;
  $input.value.onkeydown = null;
  $input.value.onkeyup = null;

  const [correct_words, correct_letters, written_letters] = words.reduce((p, w) => {
    let c = 0, t = 0;

    for ( const l of w.letters)
      if(l.status === LetterStatus.n)
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

function genWord(w : string) {
  return {
    active : false,
    letters : Array.from(w, e => ({
      status : LetterStatus.n,
      l : e,
      active : false,
    }))
  };
}

function gameStart() {
  gameover.value = false;

  document.onkeydown = () => $input.value.focus();
  $input.value.onkeydown = keyDown;
  $input.value.onkeyup = keyUp;

  words.length = 0;
  words.push(...raw_words.slice(0, 24).sort(() => Math.random() - .5).map(genWord));

  wordIndex = 0;
  wordTotal = words.length;

  time.value = RUN_TIME;
  let s = setInterval(() =>{
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
  document.onclick = () => {
    document.onclick = null;
    gameStart();
    interaction.value = true;
  }
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
        <p>Esta es una copia del juego MonkeyType. Si os gusta, no duden en visitar el <a href="https://monkeytype.com" style="color: gold;">juego original</a>.</p>
      </hgroup>
      <p :style="{ visibility : interaction ? 'hidden' : 'visible' }"><i style="opacity: .75;">para iniciar el juego, es necesario un click en la página</i></p>
    </header>
    <section v-show="interaction">
      <p style="color: gold">
        tiempo: <time  :date="new Date(`PT${time}S`)" :format="{ second : '2-digit' }">{{ time }}</time>
      </p>
      <p class="board" ref="board">
        <TransitionGroup name="appear">
          <span v-for="w, i of words" :key="i" :class="[w.active && 'active']">
            <span v-for="l, i of w.letters" :key="i" :data-status="l.status" v-text="l.l" :class="[l.active && 'active']"></span>
          </span>
        </TransitionGroup>
      </p>
      <input type="text" ref="$input" autofocus style="position: absolute; left: 0; top: -100vh;"/>
    </section>

    <aside v-show="gameover">
      <h2>PPM</h2>
      <output :value="wpm.toFixed(2)"></output>
      <h2>precisión</h2>
      <output :value="`${(accuracy * 100).toFixed(2)}%`"></output>

      <button style="display: block; margin-block: 1rem; background-color: #333; padding : .5rem 1rem" @click="gameStart">Jugar Otra vez</button>
    </aside>
  </main>
  <footer style="position: fixed; bottom: 0; padding : .5rem; text-align: center; width: 100%; background-color: #111;">
    <small>Proyecto basado en las guia de Midudev.</small>
  </footer>
</template>

<style >
.appear-enter-active,
.appear-leave-active {
  transition: opacity .3s ease-out, translate .3s ease-out;
  /* transition-delay: calc(.15s * (var(--d) - var(--t))); */
}

.appear-enter-from,
.appear-leave-to {
  opacity: 0;
  translate : 0 1ch;
}

main {
  padding : 1rem;
}
.board {
  font-family: Menlo, monospace;
  display: flex;
  flex-wrap: wrap;
  gap : .25ch 1ch;
}

.board > span {
  border-bottom: 2px solid transparent;
  letter-spacing: .25ch;
}
.board > span.active:not(:has(.active)) > span:last-child::before { 
  content: '|';
  position: absolute;
  translate: +.75ch 0;
  color : gold;
  animation: 1s blink infinite linear;
}
.board > span:has(> span:not([data-status="0"])):has(~ span.active) {
  border-color : red;
}

.board > span > span[data-status="0"] { color: lime; }
.board > span > span[data-status="1"] { color: red; }
.board > span > span.active::before { 
  content: '|';
  position: absolute;
  translate: -.75ch 0;
  color : gold;

  animation: 1s blink infinite linear;
}



@keyframes blink {
  0%,35%{
    opacity: 1;
  }
  75%{
    opacity: 0;
  }
}
</style>
