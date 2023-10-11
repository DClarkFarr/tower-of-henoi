<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
type Tower = number[];
type TowerNum = 1 | 2 | 3;
type Towers = Record<TowerNum, Tower>;

type DiscCoords = {
  tower: TowerNum;
  disc: number;
};

type ActionType = "move" | "shift";

type QueueAction = {
  coords: DiscCoords;
  to: TowerNum;
  action: ActionType;
};

type Queue = QueueAction[];

const DESTINATION: TowerNum = 3;

const TOWERS: TowerNum[] = [1, 2, 3];
/**
 * Native methods
 */

function createStartingDisks(numDiscs: number) {
  return Array(numDiscs)
    .fill(0)
    .map((v, k) => k);
}

function moveDisc(
  towers: Towers,
  discCoords: DiscCoords,
  toTowerNum: TowerNum
) {
  if (towers[discCoords.tower][0] !== discCoords.disc) {
    throw new Error("Cannot move a disc unless it is on top");
  }

  const disc = towers[discCoords.tower].shift()!;
  towers[toTowerNum].unshift(disc);
}

function isTopDisc(towers: Towers, towerNum: TowerNum, discNum: number) {
  return towers[towerNum][0] === discNum;
}

function executeQueueAction(
  towers: Towers,
  action: QueueAction,
  queue: Queue,
  actionsTaken: Queue
) {
  moveDisc(towers, action.coords, action.to);

  queue.forEach((a) => {
    if (a.coords.disc === action.coords.disc) {
      a.coords.tower = action.to;
    }
  });

  actionsTaken.push(action);
}

function canMoveDisc(
  towers: Towers,
  discCoords: DiscCoords,
  targetTower: TowerNum
) {
  if (!isTopDisc(towers, discCoords.tower, discCoords.disc)) {
    return false;
  }
  return (
    !towers[targetTower].length || discCoords.disc < towers[targetTower][0]
  );
}

function getDiscsToShift(towers: Towers, coords: DiscCoords) {
  return towers[coords.tower].filter((disc) => disc < coords.disc);
}
function applyNewShiftToQueue(
  towers: Towers,
  steps: number,
  queue: Queue,
  actionsTaken: Queue
) {
  const lastAction = queue[queue.length - 1];
  const discToShift = getDiscsToShift(towers, lastAction.coords);

  if (!discToShift.length) {
    throw new Error("No discs to shift");
  }

  const shiftAlt = lastAction.to;
  const shiftDestination = getMoveAlt(lastAction.coords.tower, shiftAlt);

  return handleQueueIteration(
    towers,
    steps,
    [
      ...queue,
      ...discToShift.reverse().map<QueueAction>((disc, i) => ({
        action: "shift",
        coords: {
          tower: lastAction.coords.tower,
          disc: disc,
        },
        to: i % 2 === 0 ? shiftDestination : shiftAlt,
      })),
    ],
    actionsTaken
  );
}

function applyMicroShiftToQueue(
  towers: Towers,
  steps: number,
  queue: Queue,
  actionsTaken: Queue
) {
  const lastAction = queue[queue.length - 1];

  const destination = lastAction.to;

  const towersByMicroBase = TOWERS.map((towerNum) => {
    const filtered = towers[towerNum].filter((n) => n < lastAction.coords.disc);
    return {
      towerNum,
      biggestBase: filtered.length ? Math.max(...filtered) : -1,
    };
  })
    .filter((base) => base.biggestBase > -1)
    .sort((a, b) => a.biggestBase - b.biggestBase);

  let microDestination = towersByMicroBase[1].towerNum;
  let microAlt = destination;
  let microDisc = towersByMicroBase[1].biggestBase;
  let miniDiscsToShift = getDiscsToShift(towers, {
    tower: destination,
    disc: microDisc,
  });

  if (towersByMicroBase[0].towerNum !== microAlt) {
    microDisc = towersByMicroBase[0].biggestBase;
    microAlt = towersByMicroBase[0].towerNum;
    miniDiscsToShift = getDiscsToShift(towers, {
      tower: microAlt,
      disc: microDisc + 1,
    });
  }

  return handleQueueIteration(
    towers,
    steps,
    [
      ...queue,
      ...miniDiscsToShift.reverse().map<QueueAction>((disc, i) => ({
        action: "shift",
        coords: {
          tower: microAlt,
          disc: disc,
        },
        to:
          i % 2 === 0
            ? microDestination
            : getMoveAlt(microAlt, microDestination),
      })),
    ],
    actionsTaken
  );
}
function handleQueueIteration(
  towers: Towers,
  steps: number,
  queue: Queue,
  actionsTaken: Queue
): [number, Queue] {
  const lastAction = queue[queue.length - 1];
  const towersCopy = JSON.parse(JSON.stringify(towers)) as Towers;

  if (canMoveDisc(towersCopy, lastAction.coords, lastAction.to)) {
    queue.pop();
    executeQueueAction(towersCopy, lastAction, queue, actionsTaken);

    if (queue.length) {
      return handleQueueIteration(towersCopy, steps + 1, queue, actionsTaken);
    } else {
      return [steps + 1, actionsTaken];
    }
  } else {
    if (lastAction.action === "move") {
      if (
        isTopDisc(towersCopy, lastAction.coords.tower, lastAction.coords.disc)
      ) {
        // target tower is obstructed. Must do a micro shift.
        return applyMicroShiftToQueue(towersCopy, steps, queue, actionsTaken);
      } else {
        return applyNewShiftToQueue(towersCopy, steps, queue, actionsTaken);
      }
    } else {
      return applyMicroShiftToQueue(towersCopy, steps, queue, actionsTaken);
    }
  }
}

function getMoveAlt(currentTower: TowerNum, targetTower: TowerNum) {
  return TOWERS.filter(
    (tower) => tower !== currentTower && tower !== targetTower
  )[0];
}

function queueMoves(towers: Towers) {
  return handleQueueIteration(
    towers,
    0,
    [
      ...towers[1].map((disc) => {
        return {
          coords: {
            tower: 1,
            disc,
          },
          to: DESTINATION,
          action: "move",
        } as QueueAction;
      }),
    ],
    []
  );
}

/**
 * Vue Methods
 */
const numDiscs = ref(4);
const totalMoves = ref(0);
const minMoves = ref(0);
const queue = ref<Queue>([]);
const queueIndex = ref(-1);
const towers = ref<Towers>({
  1: [],
  2: [],
  3: [],
});

const towerHanoi = (numDiscs: number) => {
  queueIndex.value = -1;
  queue.value = [];
  towers.value = {
    1: createStartingDisks(numDiscs),
    2: [],
    3: [],
  };

  const [steps, actionsTaken] = queueMoves(towers.value);
  minMoves.value = steps;
  queue.value = actionsTaken;
};

const onClickPlay = async () => {
  const pluck = () => {
    return new Promise((resolve) => {
      setTimeout(() => {
        onClickNext();
        resolve(null);
      }, 750);
    });
  };
  while (queueIndex.value < queue.value.length - 1) {
    await pluck();
  }
};

const onClickNextAll = () => {
  while (queueIndex.value < queue.value.length - 1) {
    onClickNext();
  }
};

const onClickNext = () => {
  if (!queue.value.length) {
    return;
  }
  if (queueIndex.value > queue.value.length - 1) {
    queueIndex.value = -1;
  }

  queueIndex.value += 1;

  const action = queue.value[queueIndex.value];

  executeQueueAction(towers.value, action, queue.value, []);
};

const onChangeNumDiscs = (e: Event) => {
  const target = e.target as HTMLInputElement;
  const value = parseInt(target.value);
  if (value < 3 || value > 10) {
    return;
  }
  numDiscs.value = value;
  towerHanoi(value);
};

const towersToRender = computed(() => {
  const base: Towers = {
    1: [],
    2: [],
    3: [],
  };
  (Object.keys(towers.value).map(Number) as TowerNum[]).forEach((towerNum) => {
    base[towerNum] = Array(numDiscs.value - towers.value[towerNum].length).fill(
      -1
    );
    base[towerNum].push(...towers.value[towerNum]);
  });

  return base;
});

onMounted(() => {
  towerHanoi(numDiscs.value);
});
</script>

<template>
  <div id="app">
    <div class="left">
      <div class="towers">
        <div
          v-for="towerNum of TOWERS"
          class="tower"
          :class="[`tower-${towerNum}`]"
          :key="`tower-${towerNum}`"
        >
          <div
            class="disc"
            :class="[
              'disc-index-' + index,
              'disc-' + disc,
              towersToRender[towerNum][index] > -1 ? 'full' : 'empty',
            ]"
            v-for="(disc, index) in numDiscs"
            :key="`1-${index}`"
          >
            <span
              class="disc-title"
              v-if="towersToRender[towerNum][index] > -1"
            >
              {{ towersToRender[towerNum][index] }}
            </span>
          </div>
          <div class="tower-label">Tower<br />{{ towerNum }}</div>
        </div>
      </div>
      <div class="actions">
        <button @click="onClickNext" :disabled="queueIndex >= queue.length - 1">
          Next Move
        </button>
        <button
          @click="onClickNextAll"
          :disabled="queueIndex >= queue.length - 1"
        >
          Next All
        </button>
        <button @click="onClickPlay" :disabled="queueIndex >= queue.length - 1">
          Play Through
        </button>
      </div>
      <div class="inputs">
        <div class="form-group">
          <label>Number of Discs</label>
          <input
            class="input"
            :value="numDiscs"
            type="number"
            @input="onChangeNumDiscs"
          />
        </div>
      </div>
    </div>
    <div class="right">
      <div class="state">
        <div>
          <b>Total Moves: </b>
          <span>{{ totalMoves }}</span>
        </div>
        <div>
          <b>Min Moves: </b>
          <span>{{ minMoves }}</span>
        </div>
        <div style="display: none">
          <b>Current State </b>
          <pre>{{ towers }}</pre>
        </div>
      </div>
      <div class="queue">
        <h4>Queue</h4>
        <table class="table">
          <tr>
            <th>#</th>
            <th>Disc</th>
            <th>From Tower</th>
            <th>To Tower</th>
            <th>Taken</th>
          </tr>
          <tr
            class="queue-disc"
            v-for="(action, index) in queue"
            :key="`${index}-${action.coords.disc}-${action.coords.tower}-${action.to}`"
          >
            <td>{{ index + 1 }}</td>
            <td>{{ action.coords.disc }}</td>
            <td>{{ action.coords.tower }}</td>
            <td>{{ action.to }}</td>
            <td>{{ index > queueIndex ? "No" : "Yes" }}</td>
          </tr>
        </table>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
* {
  box-sizing: border-box;
}

body,
html {
  padding: 0;
  margin: 0;
  min-height: 100%;
  font-family: sans-serif;
}
#app {
  display: flex;
  width: 100%;
  min-height: 100%;
  background: black;
  padding: 10px;
  color: black;
}

.left {
  width: 500px;
  background: #efefef;
}

.towers {
  display: flex;
  justify-content: space-around;
}

.tower {
  border: solid 1px #ccc;
}
.disc {
  height: 30px;
  width: 30px;
  background: #aaa;
  margin-bottom: 3px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: auto;
  margin-right: auto;

  &.full {
    background: blue;
    color: white;
  }
}

.right {
  color: #000;
  background: #ababab;
  width: 100%;
  padding: 10px;
}
.tower-label {
  text-align: center;
}
.actions {
  border-top: solid 4px;
  margin-top: 25px;
  padding-top: 15px;
}

.queue {
  .table {
    th,
    td {
      padding-left: 5px;
      padding-right: 5px;
    }
    th {
      font-size: 16px;
      font-weight: 600;
      min-width: 50px;
      background: #eaeaea;
    }
    td {
      text-align: center;
      background: #eee;
    }
  }
}
</style>
