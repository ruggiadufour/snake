<script setup lang="ts">
import { ref, computed } from "vue";

type DirectionType = "left" | "right" | "up" | "down";
type CoordinateType = { x: number; y: number };
type GameStatusType = "playing" | "stop" | "lose" | "win" | "pause";
type BodyDirectionType =
  | "vertical"
  | "horizontal"
  | "up-right"
  | "up-left"
  | "down-left"
  | "down-right";

type DirectionBodyMap = {
  [key in `${DirectionType}-${DirectionType}`]?: BodyDirectionType;
};

const directionBodyMap: DirectionBodyMap = {
  "left-left": "horizontal",
  "right-right": "horizontal",
  "down-down": "vertical",
  "up-up": "vertical",

  "up-left": "up-left",
  "right-down": "up-left",

  "up-right": "up-right",
  "left-down": "up-right",

  "down-left": "down-left",
  "right-up": "down-left",

  "down-right": "down-right",
  "left-up": "down-right",
};

const directionHeadMap: Record<DirectionType, BodyDirectionType> = {
  down: "vertical",
  up: "vertical",
  left: "horizontal",
  right: "horizontal",
};

const mapGradient: Record<BodyDirectionType, string> = {
  horizontal:
    "linear-gradient(0deg, rgba(255,255,255,0) 0%, rgba(5,185,36,1) 50%, rgba(255,255,255,0) 100%)",
  vertical:
    "linear-gradient(90deg, rgba(255,255,255,0) 0%, rgba(5,185,36,1) 50%, rgba(255,255,255,0) 100%)",
  "up-left":
    "linear-gradient(to bottom left,  rgba(255,255,255,0) 0%,  rgba(255,255,255,0) 50%, #05b924 50%,rgba(255,255,255,0) 100%)",
  "up-right":
    "linear-gradient(to bottom right,  rgba(255,255,255,0) 0%,  rgba(255,255,255,0) 50%, #05b924 50%, rgba(255,255,255,0) 100%)",
  "down-left":
    "linear-gradient(to top left,  rgba(255,255,255,0) 0%,  rgba(255,255,255,0) 50%, #05b924 50%, rgba(255,255,255,0) 100%)",
  "down-right":
    "linear-gradient(to top right,  rgba(255,255,255,0) 0%,  rgba(255,255,255,0) 50%, #05b924 50%, rgba(255,255,255,0) 100%)",
};

type SnakeBodyType = {
  currentPos: CoordinateType & {
    direction: DirectionType;
  };
  previousPos: CoordinateType & {
    direction: DirectionType;
  };
};

const HEAD_START: SnakeBodyType = {
  currentPos: { x: 0, y: 0, direction: "left" },
  previousPos: { x: 0, y: 0, direction: "left" },
};
const DIFF_WIN = 0;
const showControls = ref(false);
const velocity = ref(200);
const rows = ref(10);
const cols = ref(7);
const snake = ref<SnakeBodyType[]>([]);
const direction = ref<DirectionType>("left");
const cellRefs = ref<HTMLDivElement[]>([]);
const intervalId = ref();
const fruitsStart = ref(10);
const fruits = ref<CoordinateType[]>([]);
const currentGame = ref<GameStatusType>("stop");
const ms = ref(0);

const board = computed(() =>
  Array.from({ length: cols.value }, () => Array.from({ length: rows.value }))
);

/* Movements */
const move = {
  _base(snakeBody: SnakeBodyType, cb: () => void) {
    cleanCell(snakeBody.currentPos);
    snakeBody.currentPos.direction = direction.value;
    snakeBody.previousPos = { ...snakeBody.currentPos };
    cb();
    paintHead(snakeBody);
  },
  _right(snakeBody: SnakeBodyType) {
    this._base(snakeBody, () => {
      if (snakeBody.currentPos.x === cols.value - 1) snakeBody.currentPos.x = 0;
      else snakeBody.currentPos.x++;
    });
  },
  _left(snakeBody: SnakeBodyType) {
    this._base(snakeBody, () => {
      if (snakeBody.currentPos.x === 0) snakeBody.currentPos.x = cols.value - 1;
      else snakeBody.currentPos.x--;
    });
  },
  _down(snakeBody: SnakeBodyType) {
    this._base(snakeBody, () => {
      if (snakeBody.currentPos.y === rows.value - 1) snakeBody.currentPos.y = 0;
      else snakeBody.currentPos.y++;
    });
  },
  _up(snakeBody: SnakeBodyType) {
    this._base(snakeBody, () => {
      if (snakeBody.currentPos.y === 0) snakeBody.currentPos.y = rows.value - 1;
      else snakeBody.currentPos.y--;
    });
  },
  head(snakeBody: SnakeBodyType) {
    if (direction.value === "right") this._right(snakeBody);
    if (direction.value === "left") this._left(snakeBody);
    if (direction.value === "up") this._up(snakeBody);
    if (direction.value === "down") this._down(snakeBody);
  },
  body(snakeBody: SnakeBodyType, nextBodyPart: SnakeBodyType) {
    snakeBody.previousPos = snakeBody.currentPos;
    snakeBody.currentPos = nextBodyPart.previousPos;
    cleanCell(snakeBody.previousPos);
    paintBodyPart(snakeBody);
  },
};

const moveSnake = () => {
  snake.value.forEach((bodyPart, i) => {
    if (i === 0) move.head(bodyPart);
    else {
      const nextBodyPart = snake.value[i - 1];
      move.body(bodyPart, nextBodyPart);
    }
  });

  checkSnakeColision();
};

/* Board painting */
const cleanBoard = () => cellRefs.value.forEach(cleanCell);

const setCellStyle = (
  elCoor: CoordinateType | HTMLDivElement,
  style: Partial<Pick<CSSStyleDeclaration, "background" | "borderRadius">>
) => {
  const isCoor = isElementOrCoor(elCoor);
  const element = isCoor ? getCellElement(elCoor) : elCoor;

  element.style.background = style.background || "";
  element.style.borderRadius = style.borderRadius || "";
};

const isElementOrCoor = (
  elCoor: CoordinateType | HTMLDivElement
): elCoor is CoordinateType => "x" in elCoor;

const cleanCell = (elCoor: CoordinateType | HTMLDivElement) =>
  setCellStyle(elCoor, { background: "gray" });

const paintBodyPart = (body: SnakeBodyType) => {
  const directionBody =
    directionBodyMap[
      `${body.previousPos.direction}-${body.currentPos.direction}`
    ];

  if (directionBody) {
    setCellStyle(body.currentPos, { background: mapGradient[directionBody] });
  }
};

const paintHead = (body: SnakeBodyType) => setCellStyle(body.currentPos, {
    background: mapGradient[directionHeadMap[body.currentPos.direction]],
  })

const paintFruit = (elCoor: CoordinateType | HTMLDivElement) =>
  setCellStyle(elCoor, { background: "red" });

/* Colision */
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
  snake.value.some((bodyPart) => checkColision(pos, bodyPart.currentPos)) ||
  fruits.value.some((fruit) => checkColision(pos, fruit));

/* Snake */
const growSnake = () => {
  const lastBodyPart = snake.value.at(-1);
  if (!lastBodyPart)
    return console.error("There is no last body part (growSnake)");

  const currentPos = { ...lastBodyPart.currentPos };

  snake.value.push({ currentPos, previousPos: { ...currentPos } });
};

const generateRandomSnakeStart = () => {
  const head = structuredClone(HEAD_START);
  const randomCoor = getRandomCoordinate();
  head.currentPos.x = randomCoor.x;
  head.currentPos.y = randomCoor.y;
  head.currentPos.direction = "left";
  head.previousPos = structuredClone(head.currentPos);
  snake.value = [head];
};

/* Fruit */
const fruitEaten = (index: number) => {
  fruits.value.splice(index, 1);
  growSnake();
  const hasWon = checkWin();
  if (!hasWon) showFruit();
};

const addFruit = (coor: CoordinateType) => {
  fruits.value.push(coor);
  paintFruit(coor);
};

const showFruit = () => {
  let x = null,
    y = null;
  let hasColision;

  do {
    const coordinate = getRandomCoordinate();
    x = coordinate.x;
    y = coordinate.y;

    hasColision = checkFruitColision({ x, y });
    if (hasColision) {
      x = null;
      y = null;
    }
  } while (
    hasColision &&
    snake.value.length + fruits.value.length < rows.value * cols.value
  );

  if (x !== null && y !== null) {
    addFruit({ x, y });
  }
};

const checkFruit = () => {
  const [head] = snake.value;
  const index = fruits.value.findIndex((fruit) =>
    checkColision(head.currentPos, fruit)
  );
  if (index !== -1) fruitEaten(index);
};

const generateFruits = () => {
  Array.from({ length: fruitsStart.value }).forEach(showFruit);
};

/* Game loop */
const startInterval = () => {
  intervalId.value = setInterval(() => {
    const start = performance.now();
    moveSnake();
    checkFruit();
    const end = performance.now();

    ms.value = Math.round((end - start) * 100) / 100;
  }, velocity.value);
};

const stopInterval = () => {
  clearInterval(intervalId.value);
};

/* Game status handlers */
const checkWin = () => {
  const total = snake.value.length;
  const totalCells = rows.value * cols.value;
  const hasWon = totalCells === total + DIFF_WIN;

  if (hasWon) win();
  return hasWon;
};

const reset = () => {
  generateRandomSnakeStart();
  generateRandomDirection();
  fruits.value = [];
  ms.value = 0;
  cleanBoard();
};

const resetGame = () => {
  stop();
  reset();
  play();
};

const play = () => {
  if (currentGame.value !== "pause") {
    reset();
    generateFruits();
  }
  currentGame.value = "playing";
  startInterval();
  document.addEventListener("keyup", checkKey);
};

const pause = () => {
  stop("pause");
};

const lose = () => {
  stop("lose");
};

const win = () => {
  stop("win");
};

const stop = (status: GameStatusType = "stop") => {
  currentGame.value = status;
  stopInterval();
  document.removeEventListener("keyup", checkKey);
};

/* Extra methods */
const generateRandomDirection = () => {
  const directions: DirectionType[] = ["left", "down", "up", "right"];
  setDirection(directions[getRandom(directions.length)]);
};

const setDirection = (dir: DirectionType) => {
  direction.value = dir;
};

const getRandom = (n: number) => Math.floor(Math.random() * n);

const getRandomCoordinate = (): CoordinateType => ({
  x: getRandom(cols.value),
  y: getRandom(rows.value),
});

const getCellElement = ({ x, y }: CoordinateType) =>
  cellRefs.value[x * rows.value + y];

function checkKey(e: KeyboardEvent) {
  if (["ArrowUp", "w"].includes(e.key)) {
    direction.value !== "down" && setDirection("up");
  } else if (["ArrowDown", "s"].includes(e.key)) {
    direction.value !== "up" && setDirection("down");
  } else if (["ArrowLeft", "a"].includes(e.key)) {
    direction.value !== "right" && setDirection("left");
  } else if (["ArrowRight", "d"].includes(e.key)) {
    direction.value !== "left" && setDirection("right");
  }
}
</script>

<template>
  <div class="container">
    <div class="flex">
      <button @click="currentGame === 'playing' ? pause() : play()">
        {{ currentGame === "playing" ? "Pause" : "Play" }}
      </button>
      <button
        @click="resetGame"
        :disabled="!['playing', 'pause'].includes(currentGame)"
      >
        Reset
      </button>
    </div>
    <br />
    <div class="flex">
      <label><strong>Controls:</strong></label>
      <input type="checkbox" id="showControls" v-model="showControls" />
    </div>
    <br />
    <div v-if="showControls">
      <div class="input-container flex">
        <label>Velocity:</label>
        <input type="text" id="velocity" v-model="velocity" />
      </div>
      <div class="input-container flex">
        <label>Rows:</label>
        <input type="text" id="rows" v-model="rows" />
      </div>
      <div class="input-container flex">
        <label>Cols:</label>
        <input type="text" id="cols" v-model="cols" />
      </div>
      <div class="input-container flex">
        <label>Fruits start:</label>
        <input type="text" id="fruitsStart" v-model="fruitsStart" />
      </div>
    </div>
    <h1 v-if="['win', 'lose'].includes(currentGame)">
      <span v-if="currentGame === 'win'">You Won!</span>
      <span v-if="currentGame === 'lose'">You Lose!</span>
    </h1>

    <br />
    <h2>Score: {{ snake.length }}</h2>
    <p>{{ ms }}ms</p>
    <br />
    <div class="board">
      <div class="cols" v-for="(col, x) of board" :key="x">
        <div class="cell" v-for="(_, y) of col" :key="y" ref="cellRefs"></div>
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

.input-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  text-align: end;
}
</style>
