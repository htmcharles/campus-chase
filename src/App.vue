<template>
  <div class="min-h-full flex items-center justify-center p-6">
    <div class="w-full max-w-3xl">
      <div class="flex items-baseline justify-between gap-4 mb-4">
        <div>
          <h1 class="text-3xl sm:text-4xl font-black tracking-tight text-yellow-300">
            Campus Chase
          </h1>
          <p class="text-slate-300 text-sm sm:text-base">
            Collect books. Avoid prefects. Grab a free period to hide.
          </p>
        </div>

        <div class="text-right font-mono">
          <div class="text-lg sm:text-xl">Score: {{ score }}</div>
          <div class="text-xs sm:text-sm text-slate-300">
            Books left: {{ booksLeft }}
            <span v-if="hideTicksLeft > 0" class="ml-2 text-cyan-300">
              Hide: {{ hideSecondsLeft }}s
            </span>
          </div>
        </div>
      </div>

      <div
          class="mx-auto w-fit bg-slate-950/60 border border-slate-700 rounded-xl p-3 shadow-lg shadow-black/30"
      >
        <div
          class="rounded-lg overflow-hidden"
          :style="{ display: 'grid', gridTemplateColumns: `repeat(${cols}, ${cellPx}px)` }"
        >
          <div
            v-for="(cell, index) in flatBoard"
            :key="index"
            class="relative flex items-center justify-center select-none"
            :style="{ width: `${cellPx}px`, height: `${cellPx}px` }"
            :class="cellClass(cell)"
          >
            <div v-if="cell === CELL.BOOK" class="w-1.5 h-1.5 bg-yellow-300 rounded-full"></div>
            <div v-else-if="cell === CELL.POWER" class="w-3 h-3 bg-cyan-300 rounded-full"></div>

            <div v-if="isStudent(index)" class="text-[18px]" title="Student">🧑‍🎓</div>
            <div v-else-if="teacherAtIndex(index)" class="text-[18px]" title="Prefect">🧑‍🏫</div>
          </div>
        </div>
      </div>

      <!-- Mobile controls -->
      <div class="grid grid-cols-3 gap-2 w-48 mx-auto mt-5 sm:hidden">
        <div class="col-start-2">
          <button
            class="bg-slate-800/70 active:bg-slate-700 p-3 rounded-lg w-full text-2xl border border-slate-600"
            @click="setNextDirFromCode('ArrowUp')"
            aria-label="Move up"
          >
            ⬆️
          </button>
        </div>
        <div class="col-start-1 row-start-2">
          <button
            class="bg-slate-800/70 active:bg-slate-700 p-3 rounded-lg w-full text-2xl border border-slate-600"
            @click="setNextDirFromCode('ArrowLeft')"
            aria-label="Move left"
          >
            ⬅️
          </button>
        </div>
        <div class="col-start-3 row-start-2">
          <button
            class="bg-slate-800/70 active:bg-slate-700 p-3 rounded-lg w-full text-2xl border border-slate-600"
            @click="setNextDirFromCode('ArrowRight')"
            aria-label="Move right"
          >
            ➡️
          </button>
        </div>
        <div class="col-start-2 row-start-3">
          <button
            class="bg-slate-800/70 active:bg-slate-700 p-3 rounded-lg w-full text-2xl border border-slate-600"
            @click="setNextDirFromCode('ArrowDown')"
            aria-label="Move down"
          >
            ⬇️
          </button>
        </div>
      </div>

      <div class="mt-5 flex flex-wrap items-center gap-3">
        <div
          v-if="gameOver"
          class="px-3 py-2 rounded-lg bg-red-500/10 border border-red-400/30 text-red-200"
        >
          Game over: a prefect caught you.
        </div>
        <div
          v-else-if="gameWon"
          class="px-3 py-2 rounded-lg bg-emerald-500/10 border border-emerald-400/30 text-emerald-200"
        >
          You won: you collected all the books!
        </div>
        <div class="text-slate-300 text-sm">
          Controls: Arrow keys. Free period (cyan) gives you hide power.
        </div>

        <button
          class="ml-auto px-4 py-2 rounded-lg bg-slate-800 hover:bg-slate-700 border border-slate-600 text-slate-100 font-semibold"
          @click="resetGame"
        >
          Restart
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref } from "vue";

const CELL = Object.freeze({
  EMPTY: 0,
  WALL: 1,
  BOOK: 2,
  POWER: 3,
});

const cellPx = 26;

// Legend:
//  # = wall (school block / maize field)
//  . = book
//  o = free period (hide power)
//  S = student start
//  T = prefect start
const level = [
  "###############",
  "#S....#....o..#",
  "#.###.#.###.###",
  "#.....#.....T.#",
  "###.#####.###.#",
  "#...#...#.....#",
  "#.#.#.#.#.###.#",
  "#.#...#...#...#",
  "#.###.###.#.#.#",
  "#.....#.....#.#",
  "#.###.#.###.#.#",
  "#..o..#....T..#",
  "###.###.###.###",
  "#.....#.......#",
  "###############",
];

const rows = level.length;
const cols = level[0].length;

const board = ref([]);
const score = ref(0);
const gameOver = ref(false);
const gameWon = ref(false);

// Hide timer: 10 ticks per second
const hideTicksLeft = ref(0);

const student = ref({ x: 1, y: 1 });
const studentDir = ref({ dx: 0, dy: 0 });
const nextStudentDir = ref({ dx: 0, dy: 0 });
const teachers = ref([]);
const teacherSpawns = ref([]);

let teacherLoopId = null;
let timerLoopId = null;
let studentLoopId = null;

const flatBoard = computed(() => board.value.flat());
const booksLeft = computed(() => flatBoard.value.filter((c) => c === CELL.BOOK).length);
const hideSecondsLeft = computed(() => Math.ceil(hideTicksLeft.value / 10));

function parseLevel() {
  const nextBoard = [];
  const nextTeachers = [];
  const nextSpawns = [];
  let start = { x: 1, y: 1 };

  for (let y = 0; y < rows; y++) {
    const row = [];
    for (let x = 0; x < cols; x++) {
      const ch = level[y][x];
      if (ch === "#") row.push(CELL.WALL);
      else if (ch === ".") row.push(CELL.BOOK);
      else if (ch === "o") row.push(CELL.POWER);
      else row.push(CELL.EMPTY);

      if (ch === "S") start = { x, y };
      if (ch === "T") {
        nextTeachers.push({ x, y });
        nextSpawns.push({ x, y });
      }
    }
    nextBoard.push(row);
  }

  board.value = nextBoard;
  student.value = start;
  teachers.value = nextTeachers;
  teacherSpawns.value = nextSpawns;
  score.value = 0;
  gameOver.value = false;
  gameWon.value = false;
  hideTicksLeft.value = 0;
  studentDir.value = { dx: 0, dy: 0 };
  nextStudentDir.value = { dx: 0, dy: 0 };
}

function idxToXY(index) {
  return { x: index % cols, y: Math.floor(index / cols) };
}

function isStudent(index) {
  const { x, y } = idxToXY(index);
  return student.value.x === x && student.value.y === y;
}

function teacherAtIndex(index) {
  const { x, y } = idxToXY(index);
  return teachers.value.some((t) => t.x === x && t.y === y);
}

function cellClass(cell) {
  if (cell === CELL.WALL) return "bg-emerald-800/55 border border-emerald-700/40";
  return "bg-slate-900/40 border border-slate-800/60";
}

function isWall(x, y) {
  return board.value[y]?.[x] === CELL.WALL;
}

function maybeEat() {
  const { x, y } = student.value;
  const here = board.value[y][x];
  if (here === CELL.BOOK) {
    board.value[y][x] = CELL.EMPTY;
    score.value += 10;
  } else if (here === CELL.POWER) {
    board.value[y][x] = CELL.EMPTY;
    score.value += 50;
    hideTicksLeft.value = 60; // 6 seconds
  }

  if (booksLeft.value === 0 && !gameOver.value) {
    gameWon.value = true;
    stopLoops();
  }
}

function endGame() {
  gameOver.value = true;
  stopLoops();
}

function checkCollision() {
  const hit = teachers.value.find((t) => t.x === student.value.x && t.y === student.value.y);
  if (!hit) return;

  if (hideTicksLeft.value > 0) {
    const i = teachers.value.indexOf(hit);
    const spawn = teacherSpawns.value[i] ?? { x: cols - 2, y: rows - 2 };
    hit.x = spawn.x;
    hit.y = spawn.y;
    score.value += 200;
    return;
  }

  endGame();
}

function setNextDirFromCode(code) {
  let dx = 0;
  let dy = 0;
  if (code === "ArrowUp") dy = -1;
  if (code === "ArrowDown") dy = 1;
  if (code === "ArrowLeft") dx = -1;
  if (code === "ArrowRight") dx = 1;
  nextStudentDir.value = { dx, dy };
}

function handleKeydown(e) {
  if (gameOver.value || gameWon.value) return;
  if (!["ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(e.code)) return;
  e.preventDefault();
  setNextDirFromCode(e.code);
}

function moveStudent() {
  if (gameOver.value || gameWon.value) return;

  const buffered = nextStudentDir.value.dx !== 0 || nextStudentDir.value.dy !== 0;
  if (buffered) {
    const tx = student.value.x + nextStudentDir.value.dx;
    const ty = student.value.y + nextStudentDir.value.dy;
    if (!isWall(tx, ty)) {
      studentDir.value = nextStudentDir.value;
    }
  }

  const moving = studentDir.value.dx !== 0 || studentDir.value.dy !== 0;
  if (!moving) return;

  const nx = student.value.x + studentDir.value.dx;
  const ny = student.value.y + studentDir.value.dy;
  if (isWall(nx, ny)) return;

  student.value = { x: nx, y: ny };
  maybeEat();
  checkCollision();
}

function manhattan(ax, ay, bx, by) {
  return Math.abs(ax - bx) + Math.abs(ay - by);
}

function shuffle(array) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

function moveTeachers() {
  if (gameOver.value || gameWon.value) return;

  const dirs = shuffle([
    { dx: 0, dy: -1 },
    { dx: 0, dy: 1 },
    { dx: -1, dy: 0 },
    { dx: 1, dy: 0 },
  ]);

  for (const t of teachers.value) {
    const options = dirs
      .map((d) => ({ x: t.x + d.dx, y: t.y + d.dy }))
      .filter((p) => !isWall(p.x, p.y));

    if (options.length === 0) continue;

    const target = student.value;
    const hiding = hideTicksLeft.value > 0;

    let best = options[0];
    let bestScore = hiding
      ? -manhattan(best.x, best.y, target.x, target.y)
      : manhattan(best.x, best.y, target.x, target.y);

    for (const p of options) {
      const s = hiding
        ? -manhattan(p.x, p.y, target.x, target.y)
        : manhattan(p.x, p.y, target.x, target.y);
      if (s < bestScore) {
        bestScore = s;
        best = p;
      }
    }

    t.x = best.x;
    t.y = best.y;
  }

  checkCollision();
}

function startLoops() {
  stopLoops();
  timerLoopId = window.setInterval(() => {
    if (hideTicksLeft.value > 0) hideTicksLeft.value -= 1;
  }, 100);
  teacherLoopId = window.setInterval(() => {
    moveTeachers();
  }, 260);
  studentLoopId = window.setInterval(() => {
    moveStudent();
  }, 200);
}

function stopLoops() {
  if (teacherLoopId) window.clearInterval(teacherLoopId);
  if (timerLoopId) window.clearInterval(timerLoopId);
  if (studentLoopId) window.clearInterval(studentLoopId);
  teacherLoopId = null;
  timerLoopId = null;
  studentLoopId = null;
}

function resetGame() {
  parseLevel();
  startLoops();
}

onMounted(() => {
  window.addEventListener("keydown", handleKeydown, { passive: false });
  resetGame();
});

onUnmounted(() => {
  window.removeEventListener("keydown", handleKeydown);
  stopLoops();
});
</script>
