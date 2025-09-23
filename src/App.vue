<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import './assets/style.css'


interface Position {
  row: number
  col: number
}


interface Ball {
  position: Position
  color: string
  isNextRound: boolean
}

interface History {
  board: (Ball | null)[][]
  score: number
}

const ROW = 9
const COL = 9
const STREAK = 5

// const board = ref<(Ball | null)[][]>([[]])
const round = ref<number>(0)
const colors = ref<string[]>(['red', 'green', 'blue', 'yellow', 'purple', 'brown', 'gray'])
// const score = ref<number>(0)
const history = ref<History[]>([])

const removingBall = ref<Ball[]>([])
const selectedBall = ref<Ball | null>(null)

function playRound(row: number, col: number) {
  const currentPosition: Position = { row, col }
  const currentBall = getBall(gameboard.value, currentPosition, true)


  if (currentBall !== null && !currentBall.isNextRound) { // click vao ball gans selected = ball
    selectedBall.value = currentBall
    return
  }

  if (selectedBall.value === null) {
    return
  }
  //check co the move ball
  if (!hasPath(selectedBall.value.position, currentPosition)) {
    return
  }
  // end

  const ballToAdd: Ball = { color: selectedBall.value.color, isNextRound: false, position: currentPosition }
  const ballToRemove: Ball = selectedBall.value

  const nextBoard = cloneBoard(gameboard.value)
  // let nextScore = score.value

  const nexRoundBalls = nextBoard.flat().filter(ball => ball !== null && ball.isNextRound) as Ball[]

  setBoardValue(nextBoard, ballToAdd.position, ballToAdd)
  setBoardValue(nextBoard, ballToRemove.position, null)

  let streaks: Ball[] = []
  streaks.push(...checkStreak(nextBoard, ballToAdd))

  if (streaks.length === 0) {
    for (const ball of nexRoundBalls) {
      if (isTheSamePosition(ball.position, ballToAdd.position)) {
        const [newRandomBall] = randomFutureBall(nextBoard, 1, false)
        if (newRandomBall) {
          newRandomBall.color = ball.color
          streaks.push(...checkStreak(nextBoard, newRandomBall))
        }
      } else {
        ball.isNextRound = false
        streaks.push(...checkStreak(nextBoard, ball))
      }
    }
    randomFutureBall(nextBoard, 3, true)
  }

  removingBall.value = streaks

  // board.value = nextBoard
  history.value = history.value.slice(0, round.value + 1).concat({ board: nextBoard, score: score.value + streaks.length })
  // history.value.push({board: nextBoard, score : nextScore})
  selectedBall.value = null


  round.value++
}

function randomInt(min: number, max: number) {
  return Math.floor(Math.random() * (max - min) + min)
}
function randomPosition(currentBoard: (Ball | null)[][]) {
  let position = null
  while (!isFull(currentBoard)) {
    const row = randomInt(0, ROW)
    const col = randomInt(0, COL)
    if (getBall(currentBoard, { row, col }, true) !== null) {
      continue
    }
    return position = { row, col }
  }
  return null
}

function randomColor() {
  return colors.value[randomInt(0, colors.value.length)]
}
function getBall(currentBoard: (Ball | null)[][], position: Position, isGetNextRoundBall = false) {
  const { row, col } = position
  if (!isValidPosotion({ row, col })) {
    throw new Error("Parameter is wrong");
  }
  const ball = currentBoard[row][col]
  if (ball === undefined) {
    console.log(row, col);

  }
  if (ball === null || isGetNextRoundBall) {
    return ball
  }
  if (ball.isNextRound) {
    return null
  }
  return ball

}

function setBoardValue(currentBoard: (Ball | null)[][], position: Position, ball: Ball | null) {
  const { row, col } = position
  if (!isValidPosotion({ row, col })) {
    throw new Error("Parameter is wrong");
  }
  currentBoard[row][col] = ball
}
function isValidPosotion(position: Position) {
  const { row, col } = position
  if (row < 0 || row >= ROW || col < 0 || col >= COL) {
    return false
  }
  return true
}

function cloneBall(ball: Ball): Ball {
  return { ...ball, position: { ...ball.position } }
  // return {...ball}
}

function randomFutureBall(currentBoard: (Ball | null)[][], numberOfBalls: number, isNextRound: boolean) {
  const balls: Ball[] = []
  while (balls.length < numberOfBalls) {
    const position: Position | null = randomPosition(currentBoard)
    if (position === null) {
      break
    }
    const color = randomColor()
    const ball = { color, isNextRound, position }
    setBoardValue(currentBoard, position, ball)
    balls.push(ball)
  }
  return balls
}

function checkStreak(currentBoard: (Ball | null)[][], ball: Ball) {

  const { position: { row, col } } = ball
  const directions = [[1, 0], [1, 1], [0, 1], [-1, 1]]
  // const streaks: Position[] = [{ row, col }]
  const streaks: Ball[] = [ball]
  for (const direction of directions) {
    const balls: Ball[] = []
    // const positions: Position[] = []
    let i = 1
    while (true) {
      const r = row + direction[0] * i
      const c = col + direction[1] * i
      const pos = { row: r, col: c }
      if (!isValidPosotion(pos)) {
        break
      }
      const currentBall = getBall(currentBoard, pos, false)
      if (currentBall === null || currentBall.color !== ball.color) {
        break
      }
      // positions.push(pos)
      balls.push(currentBall)
      i++
    }
    i = 1
    while (true) {
      const r = row - direction[0] * i
      const c = col - direction[1] * i
      const pos = { row: r, col: c }
      if (!isValidPosotion(pos)) {
        break
      }
      const currentBall = getBall(currentBoard, pos, false)
      if (currentBall === null || currentBall.color !== ball.color) {
        break
      }
      // positions.push(pos)
      balls.push(currentBall)
      i++
    }

    if (balls.length >= STREAK - 1) {
      streaks.push(...balls)
    }
  }
  if (streaks.length >= STREAK) {
    return streaks
  }
  return []
}

function restart() {
  let defaultNumberOfBalls = 5
  const newBoard = Array.from({ length: ROW }, () => Array.from({ length: COL }, () => null))
  const initBalls = randomFutureBall(newBoard, defaultNumberOfBalls, false)
  const randomBalls = randomFutureBall(newBoard, 3, true)
  // board.value = newBoard
  history.value = []
  history.value.push({ board: newBoard, score: 0 })
  round.value = 0
}

function cloneBoard(currentBoard: (Ball | null)[][]) {
  return currentBoard.map(row => row.map(ball => ball !== null ? cloneBall(ball) : null))
}


function isFull(currentBoard: (Ball | null)[][]) {
  return currentBoard.every(row => row.every(ball => {
    return ball !== null
  }))
}

function removeBall(currentBoard: (Ball | null)[][], balls: Ball[]) {
  for (const ball of balls) {
    setBoardValue(currentBoard, ball.position, null)
  }
}
// function removeBall(currentBoard: (Ball | null)[][], positions: Position[]) {
//   for (const position of positions) {
//     setBoardValue(currentBoard, position, null)
//   }
// }

function isTheSamePosition(pos1: Position, pos2: Position) {
  return pos1.row === pos2.row && pos1.col === pos2.col
}

function undo() {
  if (round.value <= 0) {
    return
  }
  round.value -= 1
  selectedBall.value = null
  removingBall.value = []
}

function redo() {
  if (round.value >= history.value.length - 1) {
    return
  }
  round.value += 1
  selectedBall.value = null
  removingBall.value = []
}


const gameboard = computed<(Ball | null)[][]>(() => {
  if (history.value[round.value]) {
    return history.value[round.value].board
  }
  return [[]]
})

const score = computed<number>(() => {
  if (history.value[round.value]) {
    return history.value[round.value].score
  }
  return 0
})

function hasPath(from: Position, dest: Position) {
  const positions: Position[] = [from]
  const directions = [[-1, 0,], [0, 1], [1, 0], [0, -1]]
  const movedPosition = new Set()
  while (positions.length > 0) {
    const pos = positions.shift()
    if (!pos) {
      continue
    }
    if (isTheSamePosition(dest, pos)) {
      return true
    }
    movedPosition.add(generateKeyForSet(pos))
    for (const direction of directions) {
      const r = pos.row + direction[0]
      const c = pos.col + direction[1]
      const newPosition = { row: r, col: c }

      if (isValidPosotion(newPosition) && getBall(gameboard.value, newPosition, false) === null && !movedPosition.has(generateKeyForSet(newPosition))) {
        positions.push(newPosition)
      }
    }
  }
  return false
}
function generateKeyForSet(position: Position) {
  return `${position.row}, ${position.col}`
}

watch(removingBall, () => {
  if (removingBall.value.length === 0) {
    return
  }
  setTimeout(() => {
    for (const ball of removingBall.value) {
      setBoardValue(gameboard.value, ball.position, null)
    }
    removingBall.value = []
  }, 500);
})

//restart()
</script>
<template>
  <div class="game-container">
    <div class="game-header">
      <div class="score">Score: <span id="score">{{ score }}</span></div>
    </div>
    <div class="game-board">
      <template v-for="(row, i) in gameboard">
        <div v-for="(ball, j) in row" class="cell" @click="playRound(i, j)">
          <div v-if="ball !== null" :class="[
            ball.isNextRound ? 'next-round' : 'ball',
            ball.color,
            { 'selectedBall': selectedBall ? isTheSamePosition(ball.position, selectedBall.position) : false },
            { 'removing': removingBall.find(rb => isTheSamePosition(rb.position, ball.position)) !== undefined }
          ]">
          </div>
        </div>
      </template>
    </div>
    <div class="controls">
      <div>
        <button class="btn" :disabled="round <= 0" @click="undo">↻</button>
        <button class="btn" :disabled="round >= history.length - 1" @click="redo">↺</button>
      </div>
      <div>
        <button class="btn new-game-btn" @click="restart">New Game</button>
      </div>
    </div>
  </div>

  <!-- <div> {{ score }}</div>
  <button :disabled="round <= 0" @click="undo">Undo</button>
  <button :disabled="round >= history.length - 1" @click="redo">Redo</button>
  <div class="gameboard-container">
    <template v-for="(row, i) in gameboard">
      <div v-for="(ball, j) in row" class="cell" @click="playRound(i, j)">
        <div v-if="ball !== null" class="ball" :class="[
          { 'next-round': ball.isNextRound },
          ball.color,
          { selectedBall: i === selectedBall?.position?.row && j === selectedBall.position.col }]">
        </div>
      </div>
    </template>
</div>
<button @click="restart">restart</button> -->
</template>

<style scoped>
.game-container {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.2);
  /* width: 100%; */
  /* height: 100%; */
  width: min(95vw, 95vh, 700px);
  height: min(95vw, 95vh, 700px);
  /* height: 100vw; */
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 0 10px;
}

.score {
  font-size: 24px;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}

.game-board {
  display: grid;
  gap: 2px;
  /* grid-template-columns: repeat(v-bind(COL), 50px);
  grid-template-rows: repeat(v-bind(ROW), 50px); */
  grid-template-columns: repeat(9, 1fr);
  grid-template-rows: repeat(9, 1fr);
  justify-content: center;

  background: rgba(0, 0, 0, 0.2);
  padding: 10px;
  border-radius: 10px;
  border: 2px solid rgba(255, 255, 255, 0.2);
  /* max-width: 100%; */
  overflow: hidden;
  
  /* width: 90vmin; */
  /* height: 90vmin; */
  aspect-ratio: 1 / 1;
  align-items: stretch;
  justify-items: stretch;
  overflow: hidden;
  flex: 1;
}

.cell {
  border: 1px solid black;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  cursor: pointer;
  /* transition: all 0.3s ease; */
  border: 1px solid rgba(255, 255, 255, 0.1);
  /* touch-action: manipulation; */
}

.ball {
  width: 70%;
  height: 70%;
  border-radius: 50%;
  box-shadow:
    inset 5px 5px 10px rgba(255, 255, 255, 0.3),
    inset -5px -5px 10px rgba(0, 0, 0, 0.3),
    0 4px 8px rgba(0, 0, 0, 0.3);
  animation: bounce 0.5s ease-out;
}

@keyframes bounce {
  0% {
    transform: scale(0);
  }

  50% {
    transform: scale(1.2);
  }

  100% {
    transform: scale(1);
  }
}

.ball.removing {
  animation: explode 0.5s ease-out forwards;
}

@keyframes explode {
  0% {
    transform: scale(1);
    opacity: 1;
  }

  50% {
    transform: scale(1.5);
  }

  100% {
    transform: scale(0);
    opacity: 0;
  }
}

.ball.selectedBall {
  border: 3px solid white;
  box-shadow: 0 0 20px rgba(255, 255, 255);
  transform: scale(1.5);
}

.next-round {
  width: 30%;
  height: 30%;
  border-radius: 50%;
}

.controls {
  margin-top: 20px;
  font-size: 14px;
  display: flex;
  justify-content: space-between;
}

.btn {
  /* background: linear-gradient(135deg, #667eea, #764ba2); */
  /* color: white; */
  border: none;
  padding: 15px 30px;
  font-size: 18px;
  border-radius: 25px;
  cursor: pointer;
  /* margin-top: 20px; */
  /* transition: all 0.3s ease; */
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
}

.red {
  background: linear-gradient(135deg, crimson, red)
}

.green {
  background: linear-gradient(135deg, lawngreen, greenyellow);
}

.blue {
  background: linear-gradient(135deg, royalblue, dodgerblue);
}

.yellow {
  background: linear-gradient(135deg, khaki, yellow);
}

.purple {
  /* color: hotpink; */
  background: linear-gradient(135deg, hotpink, fuchsia);
}

.brown {
  background: linear-gradient(135deg, aquamarine, aqua);
}

.gray {
  background: linear-gradient(135deg, lightslategray, gainsboro);
}

@media (max-width: 768px) {
  .game-container {
    padding: 15px;
    margin: 5px;
    border-radius: 15px;
  }

  /* .game-board {
    max-width: 600px;
    max-height: 600px;
  } */

  .game-header {
    margin-bottom: 15px;
  }

  .score {
    font-size: 20px;
  }

  .controls {
    font-size: 12px;
    margin-top: 15px;
  }

  .btn {
    padding: 12px 24px;
    font-size: 16px;
    margin-top: 15px;
  }
}

@media (max-width: 480px) {
  .game-container {
    padding: 10px;
    border-radius: 10px;
  }

  .score {
    font-size: 18px;
  }

  .game-board {
    padding: 6px;
  }


  .ball {
    box-shadow:
      inset 3px 3px 6px rgba(255, 255, 255, 0.3),
      inset -3px -3px 6px rgba(0, 0, 0, 0.3),
      0 2px 4px rgba(0, 0, 0, 0.3);
  }

  .next-round {
    width: 40%;
    height: 40%;
  }

  .controls {
    font-size: 11px;
  }

  .btn {
    padding: 10px 20px;
    font-size: 14px;
  }
}

@media (max-width: 320px) {
  .game-board {
    padding: 4px;
  }
}
</style>