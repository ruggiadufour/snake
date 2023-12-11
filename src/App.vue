<script setup lang="ts">
import { ref, computed } from "vue";

type DirectionType = "left" | "right" | "up" | "down";
type CoordinateType = { x: number; y: number };
type GameStatusType = "playing" | "stop" | "lose" | "win" | "pause";
type BodyDirectionType = "│" | "─" | "┌" | "┐" | "┘" | "└";
type EndsType = "^" | "v" | "<" | "o" | ">";
type SnakeBodyType = {
  currentPos: CoordinateType & {
    direction: DirectionType;
  };
  previousPos: CoordinateType & {
    direction: DirectionType;
  };
};

type DirectionBodyMap = {
  [key in `${DirectionType}-${DirectionType}`]?: BodyDirectionType;
};

const headMap: Record<DirectionType | "single", EndsType> = {
  down: "v",
  up: "^",
  left: "<",
  right: ">",
  single: "o",
};

const tailMap: Record<DirectionType, EndsType> = {
  up: "v",
  down: "^",
  right: "<",
  left: ">",
};

const SNAKE_COLOR = "#00CF37"; //00CF37
const SNAKE_SHADOW_COLOR = "#242424"; //242424
const CELL_COLOR = "gray";
const FRUIT_COLOR =`radial-gradient(ellipse farthest-corner at center center, red 0%, ${CELL_COLOR} 100%, ${CELL_COLOR} 100%)`;

const endsGradientMap: Record<EndsType, string> = {
  "^": `radial-gradient(ellipse farthest-corner at bottom center, ${SNAKE_COLOR} 0%, ${SNAKE_SHADOW_COLOR} 64%, ${SNAKE_SHADOW_COLOR} 100%)`,
  v: `radial-gradient(ellipse farthest-corner at top center, ${SNAKE_COLOR} 0%, ${SNAKE_SHADOW_COLOR} 64%, ${SNAKE_SHADOW_COLOR} 100%)`,
  "<": `radial-gradient(ellipse farthest-corner at right center, ${SNAKE_COLOR} 0%, ${SNAKE_SHADOW_COLOR} 64%, ${SNAKE_SHADOW_COLOR} 100%)`,
  ">": `radial-gradient(ellipse farthest-corner at left center, ${SNAKE_COLOR} 0%, ${SNAKE_SHADOW_COLOR} 64%, ${SNAKE_SHADOW_COLOR} 100%)`,
  o: `radial-gradient(ellipse farthest-corner at center center, ${SNAKE_COLOR} 0%, ${SNAKE_SHADOW_COLOR} 64%, ${SNAKE_SHADOW_COLOR} 100%)`,
};

const directionBodyMap: DirectionBodyMap = {
  "left-left": "─",
  "right-right": "─",
  "down-down": "│",
  "up-up": "│",
  "up-left": "┐",
  "right-down": "┐",
  "up-right": "┌",
  "left-down": "┌",
  "down-left": "┘",
  "right-up": "┘",
  "down-right": "└",
  "left-up": "└",
};

const bodyGradientMap: Record<
  BodyDirectionType,
  { background: string; clipPath: string }
> = {
  "─": {
    background: `linear-gradient(0deg, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "",
  },
  "│": {
    background: `linear-gradient(90deg, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "",
  },
  "┐": {
    background: `radial-gradient(ellipse farthest-side at bottom left, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "circle(100.0% at 0 100%)",
  },
  "┌": {
    background: `radial-gradient(ellipse farthest-side at bottom right, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "circle(100.0% at 100% 100%)",
  },
  "┘": {
    background: `radial-gradient(ellipse farthest-side at top left, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "circle(100.0% at 0 0)",
  },
  "└": {
    background: `radial-gradient(ellipse farthest-side at top right, ${SNAKE_SHADOW_COLOR} 0%, ${SNAKE_COLOR} 50%, ${SNAKE_SHADOW_COLOR} 100%)`,
    clipPath: "circle(100.0% at 100% 0)",
  },
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
    snakeBody.currentPos.direction = direction.value;
    this[`_${direction.value}`](snakeBody);
  },
  body(snakeBody: SnakeBodyType, nextBodyPart: SnakeBodyType, isTail: boolean) {
    snakeBody.previousPos = snakeBody.currentPos;
    snakeBody.currentPos = nextBodyPart.previousPos;
    cleanCell(snakeBody.previousPos);
    paintBodyPart(snakeBody, isTail);
  },
};

const moveSnake = () => {
  snake.value.forEach((bodyPart, i) => {
    if (i === 0) move.head(bodyPart);
    else {
      const nextBodyPart = snake.value[i - 1];
      move.body(bodyPart, nextBodyPart, i === snake.value.length - 1);
    }
  });

  checkSnakeColision();
};

/* Board painting */
const cleanBoard = () => cellRefs.value.forEach(cleanCell);

const setCellStyle = (
  elCoor: CoordinateType | HTMLDivElement,
  style: Partial<
    Pick<CSSStyleDeclaration, "background" | "borderRadius" | "clipPath">
  >
) => {
  const isCoor = isElementOrCoor(elCoor);
  const element = isCoor ? getCellElement(elCoor) : elCoor;

  element.style.background = style.background || "";
  element.style.borderRadius = style.borderRadius || "";
  element.style.clipPath = style.clipPath || "";
};

const isElementOrCoor = (
  elCoor: CoordinateType | HTMLDivElement
): elCoor is CoordinateType => "x" in elCoor;

const cleanCell = (elCoor: CoordinateType | HTMLDivElement) =>
  setCellStyle(elCoor, { background: CELL_COLOR });

const paintBodyPart = (body: SnakeBodyType, isTail: boolean) => {
  const directionBody =
    directionBodyMap[
      `${body.previousPos.direction}-${body.currentPos.direction}`
    ];

  if (directionBody) {
    const { background, clipPath } = bodyGradientMap[directionBody];
    const tailBackground = endsGradientMap[tailMap[body.currentPos.direction]];
    setCellStyle(body.currentPos, {
      background: isTail ? tailBackground : background,
      clipPath,
    });
  }
};

const paintHead = (body: SnakeBodyType) =>
  setCellStyle(body.currentPos, {
    background:
      endsGradientMap[
        headMap[snake.value.length === 1 ? "single" : body.currentPos.direction]
      ],
  });

const paintFruit = (elCoor: CoordinateType | HTMLDivElement) =>
  setCellStyle(elCoor, { background: FRUIT_COLOR });

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
  gap: 3px;
}

.cols {
  display: flex;
  flex-direction: column;
  gap: 3px;
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
