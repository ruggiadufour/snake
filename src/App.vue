<script setup lang="ts">
import { ref, onBeforeMount, onMounted, computed } from 'vue';

type DirectionType = 'left' | 'right' | 'up' | 'down';
type CoordinateType = { x: number; y: number };
type SnakeBodyType = {
  currentPos: CoordinateType;
  previousPos: CoordinateType;
};

const HEAD_START = { currentPos: { x: 0, y: 0 }, previousPos: { x: 0, y: 0 } };
const INTERVAL = 200;
const rows = ref(10);
const cols = ref(7);
const snake = ref<SnakeBodyType[]>([]);
const direction = ref<DirectionType>('left');
const cellRefs = ref([]);
const intervalId = ref();
const fruit = ref<CoordinateType | null>(null);
const currentGame = ref<'playing' | 'stop' | 'lose' | 'win' | 'pause'>('stop');
const points = ref(0);
const ms = ref(0);

const board = computed(() =>
  Array.from({ length: cols.value }, () => Array.from({ length: rows.value }))
);

const setColorCell = (elCoor: CoordinateType | HTMLElement, color: string) => {
  const isCoor = isElementOrCoor(elCoor);
  const element = isCoor ? getCellElement(elCoor.x, elCoor.y) : elCoor;
  element.style.backgroundColor = color;
};

const isElementOrCoor = (
  elCoor: CoordinateType | HTMLElement
): elCoor is CoordinateType => 'x' in elCoor;

const cleanCell = (elCoor: CoordinateType | HTMLElement) =>
  setColorCell(elCoor, 'gray');

const paintBodyPart = (elCoor: CoordinateType | HTMLElement) =>
  setColorCell(elCoor, 'lightgreen');

const paintHead = (elCoor: CoordinateType | HTMLElement) =>
  setColorCell(elCoor, 'green');

const paintFruit = (elCoor: CoordinateType | HTMLElement) =>
  setColorCell(elCoor, 'red');

const move = (snakeBody: SnakeBodyType, cb: () => void) => {
  cleanCell(snakeBody.currentPos);
  snakeBody.previousPos = { ...snakeBody.currentPos };
  cb();
  paintHead(snakeBody.currentPos);
};

const moveRight = (snakeBody: SnakeBodyType) => {
  move(snakeBody, () => {
    if (snakeBody.currentPos.x === cols.value - 1) snakeBody.currentPos.x = 0;
    else snakeBody.currentPos.x++;
  });
};

const moveLeft = (snakeBody: SnakeBodyType) => {
  move(snakeBody, () => {
    if (snakeBody.currentPos.x === 0) snakeBody.currentPos.x = cols.value - 1;
    else snakeBody.currentPos.x--;
  });
};

const moveDown = (snakeBody: SnakeBodyType) => {
  move(snakeBody, () => {
    if (snakeBody.currentPos.y === rows.value - 1) snakeBody.currentPos.y = 0;
    else snakeBody.currentPos.y++;
  });
};

const moveUp = (snakeBody: SnakeBodyType) => {
  move(snakeBody, () => {
    if (snakeBody.currentPos.y === 0) snakeBody.currentPos.y = rows.value - 1;
    else snakeBody.currentPos.y--;
  });
};

const moveHead = (bodyPart: SnakeBodyType) => {
  if (direction.value === 'right') moveRight(bodyPart);
  if (direction.value === 'left') moveLeft(bodyPart);
  if (direction.value === 'up') moveUp(bodyPart);
  if (direction.value === 'down') moveDown(bodyPart);
};

const moveBody = (bodyPart: SnakeBodyType, nextBodyPart: SnakeBodyType) => {
  bodyPart.previousPos = bodyPart.currentPos;
  bodyPart.currentPos = nextBodyPart.previousPos;
  cleanCell(bodyPart.previousPos);
  paintBodyPart(bodyPart.currentPos);
};

const moveSnake = () => {
  snake.value.forEach((bodyPart, i) => {
    if (i === 0) moveHead(bodyPart);
    else {
      const nextBodyPart = snake.value[i - 1];
      moveBody(bodyPart, nextBodyPart);
    }
  });

  checkSnakeColision();
};

const checkColision = (pos1: CoordinateType, pos2: CoordinateType) =>
  pos1.x === pos2.x && pos1.y === pos2.y;

const checkSnakeColision = () => {
  const [head] = snake.value;
  const hasColision = snake.value.some(
    (bodyPart, i) =>
      i !== 0 && checkColision(head.currentPos, bodyPart.currentPos)
  );
  if (hasColision) lose();
};

const checkFruitColision = (pos: CoordinateType) =>
  snake.value.some((bodyPart) => checkColision(pos, bodyPart.currentPos));

const growSnake = () => {
  const lastBodyPart = snake.value.at(-1);
  const currentPos = { ...lastBodyPart.currentPos };
  if (direction.value === 'right') currentPos.x--;
  if (direction.value === 'left') currentPos.x++;
  if (direction.value === 'up') currentPos.y++;
  if (direction.value === 'down') currentPos.y--;

  snake.value.push({ currentPos, previousPos: { ...currentPos } });
};

const checkFruit = () => {
  const [head] = snake.value;
  const hasColision = checkColision(head.currentPos, fruit.value);
  if (hasColision) fruitEaten();
};

const setDirection = (dir: DirectionType) => {
  direction.value = dir;
};

const getRandom = (n: number) => Math.floor(Math.random() * n + 1);

const getCellElement = (x: number, y: number) =>
  cellRefs.value[x * rows.value + y];

const fruitEaten = () => {
  points.value++;
  fruit.value = null;
  growSnake();
  const hasWon = checkWin();
  if (!hasWon) showFruit();
};

const checkWin = () => {
  const DIFF_WIN = 2;
  const total = snake.value.length;
  const hasWon = rows.value * cols.value - DIFF_WIN === total;
  if (hasWon) win();
  return hasWon;
};

const showFruit = () => {
  if (fruit.value) return;

  let x = null,
    y = null;
  let hasColision;

  do {
    x = getRandom(cols.value - 1);
    y = getRandom(rows.value - 1);
    hasColision = checkFruitColision({ x, y });
    if (hasColision) {
      x = null;
      y = null;
    }
    // TODO: fix infinity loop
  } while (hasColision);

  if (x && y) {
    fruit.value = { x, y };
    paintFruit(fruit.value);
  }
};

const startInterval = () => {
  intervalId.value = setInterval(() => {
    const start = performance.now();
    moveSnake();
    checkFruit();
    const end = performance.now();

    ms.value = (end - start).toFixed(2);
  }, INTERVAL);
};

const stopInterval = () => {
  clearInterval(intervalId.value);
};

const cleanBoard = () => cellRefs.value.forEach(cleanCell);

const reset = () => {
  snake.value = [structuredClone(HEAD_START)];
  direction.value = 'right';
  points.value = 0;
  fruit.value = null;
  ms.value = 0;
  cleanBoard();
};

const resetGame = () => {
  stop();
  reset();
  play();
};

const play = () => {
  if (currentGame.value !== 'pause') {
    reset();
    showFruit();
  }
  currentGame.value = 'playing';
  startInterval();
  document.addEventListener('keyup', checkKey);
};

const pause = () => {
  stop();
  currentGame.value = 'pause';
};

const lose = () => {
  stop();
  currentGame.value = 'lose';
};

const win = () => {
  stop();
  currentGame.value = 'win';
};

const stop = () => {
  currentGame.value = 'stop';
  stopInterval();
  document.removeEventListener('keyup', checkKey);
};

function checkKey(e) {
  if ([38, 87].includes(e.keyCode)) {
    direction.value !== 'down' && setDirection('up');
  } else if ([40, 83].includes(e.keyCode)) {
    direction.value !== 'up' && setDirection('down');
  } else if ([37, 65].includes(e.keyCode)) {
    direction.value !== 'right' && setDirection('left');
  } else if ([39, 68].includes(e.keyCode)) {
    direction.value !== 'left' && setDirection('right');
  }
}
</script>

<template>
  <div class="container">
    <div class="flex">
      <button @click="currentGame === 'playing' ? pause() : play()">
        {{ currentGame === 'playing' ? 'Pause' : 'Play' }}
      </button>
      <button
        @click="resetGame"
        :disabled="!['playing', 'pause'].includes(currentGame)"
      >
        Reset
      </button>
    </div>
    <h1 v-if="['win', 'lose'].includes(currentGame)">
      <span v-if="currentGame === 'win'">You Won!</span>
      <span v-if="currentGame === 'lose'">You Lose!</span>
    </h1>

    <br />
    <h2>Points: {{ points }}</h2>
    <p>{{ ms }}ms</p>
    <br />
    <div class="board">
      <div class="cols" v-for="(col, x) of board">
        <div class="cell" v-for="(cell, y) of col" ref="cellRefs"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.board {
  display: flex;
  gap: 5px;
}

.cols {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.cell {
  width: 30px;
  height: 30px;
  background-color: grey;
}

.flex {
  display: flex;
  gap: 5px;
}

h1,
h2 {
  margin: 0;
}
</style>
