<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
type Tower = number[];
type TowerNum = 1 | 2 | 3;
type Towers = Record<TowerNum, Tower> & { discs: Record<number, TowerNum> };

type QueueAction = {
  from: TowerNum;
  disc: number;
  to: TowerNum;
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

function moveDisc(towers: Towers, disc: number, toTowerNum: TowerNum) {
  const discTower = getTowerByDisc(towers, disc);

  if (towers[discTower][0] !== disc) {
    throw new Error("Cannot move a disc unless it is on top");
  }

  // console.log(
  //   "moving",
  //   disc,
  //   "(" + towers[discTower][0] + ")",
  //   "from tower",
  //   discTower,
  //   "to",
  //   toTowerNum
  // );
  towers[discTower].shift()!;
  towers[toTowerNum].unshift(disc);
  towers.discs[disc] = toTowerNum;
}

function isTopDisc(towers: Towers, towerNum: TowerNum, discNum: number) {
  return towers[towerNum][0] === discNum;
}

function executeQueueAction(
  towers: Towers,
  action: QueueAction,
  actionsTaken: Queue
) {
  moveDisc(towers, action.disc, action.to);
  actionsTaken.push(action);
}

function canMoveDisc(towers: Towers, disc: number, targetTower: TowerNum) {
  if (!isTopDisc(towers, getTowerByDisc(towers, disc), disc)) {
    return false;
  }
  return !towers[targetTower].length || disc < towers[targetTower][0];
}

function getTowerByDisc(towers: Towers, disc: number) {
  return towers.discs[disc];
}

function getDiscsToShift(towers: Towers, towerNum: TowerNum, disc: number) {
  return towers[towerNum].filter((d) => d < disc);
}

function applySubShiftToQueue(
  towers: Towers,
  steps: number,
  queue: Queue,
  actionsTaken: Queue
): [number, Queue] {
  const lastAction = queue[queue.length - 1];

  const towersByBase = TOWERS.map((towerNum) => {
    const filtered = towers[towerNum].filter((n) => n < lastAction.disc);
    return {
      towerNum,
      biggestBase: filtered.length ? Math.max(...filtered) : -1,
    };
  })
    .filter((base) => base.biggestBase > -1)
    .sort((a, b) => a.biggestBase - b.biggestBase);

  let discsToShift: number[] = [];
  let fromTower = getTowerByDisc(towers, lastAction.disc);
  let toDestinationTower = lastAction.to;
  let toAltTower = getMoveAlt(fromTower, toDestinationTower);

  if (
    towersByBase.length === 1 &&
    towersByBase[0].towerNum === lastAction.from
  ) {
    // target has discs on top of it
    discsToShift = getDiscsToShift(towers, fromTower, lastAction.disc);

    // swap base destination/alt because we want our sub stack to not block main destination
    toDestinationTower = toAltTower;
    toAltTower = getMoveAlt(toDestinationTower, fromTower);
  } else if (towersByBase.length === 1) {
    // there's an empty column
    toDestinationTower = getMoveAlt(towersByBase[0].towerNum, fromTower);
    fromTower = towersByBase[0].towerNum;
    toAltTower = getMoveAlt(toDestinationTower, fromTower);
    discsToShift = getDiscsToShift(towers, fromTower, lastAction.disc);
  } else if (towersByBase.length === 2) {
    // need to shift two smaller towers into 1
    const [smallerBase, biggerBase] = towersByBase;
    toDestinationTower = biggerBase.towerNum;
    fromTower = smallerBase.towerNum;
    toAltTower = getMoveAlt(fromTower, toDestinationTower);
    discsToShift = getDiscsToShift(towers, fromTower, lastAction.disc - 1);
  }

  if (!discsToShift.length) {
    throw new Error("No discs to shift");
  }

  return handleQueueIteration(
    towers,
    steps,
    [
      ...queue,
      ...discsToShift.reverse().map<QueueAction>((disc, i) => ({
        from: fromTower,
        disc,
        to: i % 2 === 0 ? toDestinationTower : toAltTower,
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

  if (canMoveDisc(towersCopy, lastAction.disc, lastAction.to)) {
    queue.pop();
    executeQueueAction(towersCopy, lastAction, actionsTaken);

    if (queue.length) {
      return handleQueueIteration(towersCopy, steps + 1, queue, actionsTaken);
    } else {
      return [steps + 1, actionsTaken];
    }
  } else {
    return applySubShiftToQueue(towersCopy, steps, queue, actionsTaken);
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
          from: 1,
          disc,
          to: DESTINATION,
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
const minMoves = ref(0);
const queue = ref<Queue>([]);
const queueIndex = ref(-1);
const towers = ref<Towers>({
  1: [],
  2: [],
  3: [],
  discs: {},
});

const towerHanoi = (numDiscs: number) => {
  const discs = createStartingDisks(numDiscs);

  queueIndex.value = -1;
  queue.value = [];
  towers.value = {
    1: discs,
    2: [],
    3: [],
    discs: discs.reduce((acc, disc) => {
      acc[disc] = 1;
      return acc;
    }, {} as Record<number, TowerNum>),
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
      }, 500);
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

  executeQueueAction(towers.value, action, []);
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
  const base: Omit<Towers, "discs"> = {
    1: [],
    2: [],
    3: [],
  };
  (TOWERS.map(Number) as TowerNum[]).forEach((towerNum) => {
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
              {{ towersToRender[towerNum][index] + 1 }}
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
          <span>{{ queueIndex + 1 }}</span>
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
            :key="`${index}-${action.disc}-${action.from}-${action.to}`"
          >
            <td>{{ index + 1 }}</td>
            <td>{{ action.disc + 1 }}</td>
            <td>{{ action.from }}</td>
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
