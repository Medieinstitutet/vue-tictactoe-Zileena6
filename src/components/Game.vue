<script setup lang="ts">
import { ref, reactive, computed, onMounted, watch } from 'vue';
import Playername from './AddPlayer.vue';
import { Player } from '../models/PlayerModel.js';
import ScoreHistoryModal from './ScoresModal.vue';

const players = ref<Player[]>([]);
const winner = ref<string | null>(null);
const isTie = ref(false);
const gameover = ref(false);
const currentPlayerIndex = ref(0);
const previousGames = ref<string[]>([]);
const isModalVisible = ref(false);
const winningLine = ref<number[][] | null>(null);

let board = reactive([
  ['', '', ''],
  ['', '', ''],
  ['', '', ''],
]);

const currentPlayer = computed(() => players.value[currentPlayerIndex.value]);

const checkWin = (): number[][] | null => {
  const threeInARow = currentPlayer.value.symbol;

  for (let i = 0; i < 3; i++) {
    if (board[i].every((cell) => cell === threeInARow))
      return [
        [i, 0],
        [i, 1],
        [i, 2],
      ];
    if (board.every((row) => row[i] === threeInARow))
      return [
        [0, i],
        [1, i],
        [2, i],
      ];
  }

  if (
    board[0][0] === threeInARow &&
    board[1][1] === threeInARow &&
    board[2][2] === threeInARow
  )
    return [
      [0, 0],
      [1, 1],
      [2, 2],
    ];
  if (
    board[0][2] === threeInARow &&
    board[1][1] === threeInARow &&
    board[2][0] === threeInARow
  )
    return [
      [0, 2],
      [1, 1],
      [2, 0],
    ];

  return null;
};

// const checkWin = () => {
//   const threeInARow = currentPlayer.value.symbol;

//   for (let i = 0; i < 3; i++) {
//     if (board[i].every((cell) => cell === threeInARow)) return true;
//     if (board.every((row) => row[i] === threeInARow)) return true;
//   }

//   if (
//     board[0][0] === threeInARow &&
//     board[1][1] === threeInARow &&
//     board[2][2] === threeInARow
//   )
//     return true;
//   if (
//     board[0][2] === threeInARow &&
//     board[1][1] === threeInARow &&
//     board[2][0] === threeInARow
//   )
//     return true;

//   return false;
// };

const checkTie = () => {
  return board.flat().every((cell) => cell);
};

const isWinningCell = (row: number, col: number) => {
  return winningLine.value?.some(([r, c]) => r === row && c === col);
};

const playMove = (row: number, col: number) => {
  if (!board[row][col] && !winner.value) {
    board[row][col] = currentPlayer.value.symbol;
    const winLine = checkWin();
    if (winLine) {
      winner.value = currentPlayer.value.name;
      winningLine.value = winLine;
      saveGameResult(`${currentPlayer.value.name} wins!`);
    } else if (checkTie()) {
      isTie.value = true;
      saveGameResult("It's a tie!");
    } else {
      currentPlayerIndex.value = currentPlayerIndex.value === 0 ? 1 : 0;
    }
    saveToLocalStorage();
  }
};

// const playMove = (row: number, col: number) => {
//   if (!board[row][col] && !winner.value) {
//     board[row][col] = currentPlayer.value.symbol;
//     if (checkWin()) {
//       winner.value = currentPlayer.value.name;

//       winningLine.value = winLine;
//       saveGameResult(`${currentPlayer.value.name} wins!`);
//     } else if (checkTie()) {
//       isTie.value = true;
//       saveGameResult("It's a tie!");
//     } else {
//       currentPlayerIndex.value = currentPlayerIndex.value === 0 ? 1 : 0;
//     }
//     saveToLocalStorage();
//   }
// };

const reset = () => {
  for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3; j++) {
      board[i][j] = '';
    }
  }
  currentPlayerIndex.value = 0;
  gameover.value = false;
  winner.value = null;
  isTie.value = false;
  winningLine.value = null;

  localStorage.removeItem('gameState');
  localStorage.removeItem('previousGames');
};

const resetGame = () => {
  reset();
  players.value = [];
  localStorage.removeItem('gameState');
};

const saveGameResult = (result: string) => {
  previousGames.value.push(result);
  localStorage.setItem('previousGames', JSON.stringify(previousGames.value));
};

const addPlayer = (player: Player) => {
  players.value.push(player);
  console.log(player.name);
};

const saveToLocalStorage = () => {
  localStorage.setItem(
    'gameState',
    JSON.stringify({
      players: players.value,
      winner: winner.value,
      isTie: isTie.value,
      currentPlayerIndex: currentPlayerIndex.value,
      board: board,
    })
  );
};

const loadFromLocalStorage = () => {
  const savedGameState = localStorage.getItem('gameState');
  if (savedGameState) {
    const parsedState = JSON.parse(savedGameState);
    players.value = parsedState.players;
    winner.value = parsedState.winner;
    isTie.value = parsedState.isTie;
    currentPlayerIndex.value = parsedState.currentPlayerIndex;
    board = reactive(parsedState.board);
  }

  const savedPreviousGames = localStorage.getItem('previousGames');
  if (savedPreviousGames) {
    previousGames.value = JSON.parse(savedPreviousGames);
  }
};

onMounted(() => {
  loadFromLocalStorage();
});

watch(
  [players, winner, isTie, currentPlayerIndex, board],
  () => {
    saveToLocalStorage();
  },
  { deep: true }
);

const showModal = () => {
  isModalVisible.value = true;
};

const closeModal = () => {
  isModalVisible.value = false;
};
</script>

<template>
  <div>
    <h1 class="title">
      <span class="tic">Tic</span><span class="tac">Tac</span
      ><span class="toe">Toe</span>
    </h1>

    <Playername @add-player="addPlayer" v-if="players.length < 2" />

    <div v-if="players.length === 2">
      <h2 v-for="player of players" :key="player.id">
        {{ player.name }} : {{ player.symbol }}
      </h2>
      <h2 class="current">
        Current Player:
        <span class="">
          {{ currentPlayer.name }} : {{ currentPlayer.symbol }}
        </span>
      </h2>
      <div class="row" v-for="(row, rowIndex) of board" :key="rowIndex">
        <div
          class="cell"
          v-for="(cell, cellIndex) of row"
          :key="cellIndex"
          :class="{
            'cell-x': cell === 'X',
            'cell-o': cell === 'O',
            'winner-cell': isWinningCell(rowIndex, cellIndex),
          }"
          @click="playMove(rowIndex, cellIndex)"
        >
          {{ cell }}
        </div>
      </div>

      <div class="">
        <p v-if="winner || isTie" class="result">
          {{ winner ? `${winner} wins! 🎉` : "It's a tie! 👻" }}
        </p>
        <button class="button" @click="reset">Reset Game</button>
      </div>
    </div>

    <div v-if="players.length === 2">
      <button class="button" @click="resetGame">Restart Game!</button>
    </div>
    <button class="button" @click="showModal">Show Score History</button>
  </div>

  <ScoreHistoryModal
    :isVisible="isModalVisible"
    :previousGames="previousGames"
    @close="closeModal"
  />
</template>

<style scoped>
.tic-tac-toe {
  text-align: center;
}

.board {
  display: inline-block;
  margin-top: 20px;
}

.row {
  clear: both;
}

.cell {
  width: 150px;
  height: 150px;
  float: left;
  margin-right: -1px;
  margin-bottom: -1px;
  line-height: 50px;
  text-align: center;
  border: 2px solid #f7f6f6;
  cursor: pointer;
  font-size: 140px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.winner-cell {
  background-color: #06f56a;
}

.result {
  font-size: 2rem;
}

.cell-x {
  color: #ae05fc;
  font-family: 'Rubik Bubbles', system-ui;
}

.cell-o {
  color: #fc05fc;
  font-family: 'Rubik Bubbles', system-ui;
}

.tic {
  color: #ae05fc;
}
.tac {
  color: #05d7fc;
}
.toe {
  color: #fc05fc;
}
</style>
