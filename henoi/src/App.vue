<script setup lang="ts">
import { onMounted, ref } from "vue";
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
/**
 * Native methods
 */

function createStartingDisks(numDiscs: number) {
  return Array(numDiscs)
    .fill(0)
    .map((v, k) => k);
}
function getTowerMax(towers: Towers, towerNum: TowerNum) {
  const num = Math.max(...towers[towerNum]);
  return isFinite(num) ? num : -1;
}
function getNextDiscCoords(towers: Towers): DiscCoords {
  const tower1Max = getTowerMax(towers, 1);
  const tower2Max = getTowerMax(towers, 2);
  return {
    tower: tower1Max > tower2Max ? 1 : 2,
    disc: tower1Max > tower2Max ? tower1Max : tower2Max,
  };
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
  console.log(
    "Execute: move disc ",
    disc,
    "from tower ",
    discCoords.tower,
    "to ",
    toTowerNum
  );
  towers[toTowerNum].unshift(disc);
}
function getTopDiscCoords(towers: Towers, towerNum: TowerNum) {
  return {
    tower: towerNum,
    disc: towers[towerNum][0],
  };
}
function isTopDisc(towers: Towers, towerNum: TowerNum, discNum: number) {
  return towers[towerNum][0] === discNum;
}

function executeQueueAction(towers: Towers, action: QueueAction, queue: Queue) {
  moveDisc(towers, action.coords, action.to);

  queue.forEach((a) => {
    if (a.coords.disc === action.coords.disc) {
      a.coords.tower = action.to;
    }
  });
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
function applyNewShiftToQueue(towers: Towers, steps: number, queue: Queue) {
  const lastAction = queue[queue.length - 1];
  const discToShift = getDiscsToShift(towers, lastAction.coords);

  if (!discToShift.length) {
    throw new Error("No discs to shift");
  }

  const shiftAlt = lastAction.to;
  const shiftDestination = getMoveAlt(lastAction.coords.tower, shiftAlt);

  return handleQueueIteration(towers, steps, [
    ...queue,
    ...discToShift.reverse().map<QueueAction>((disc, i) => ({
      action: "shift",
      coords: {
        tower: lastAction.coords.tower,
        disc: disc,
      },
      to: i % 2 === 0 ? shiftDestination : shiftAlt,
    })),
  ]);
}

function applyMicroShiftToQueue(towers: Towers, steps: number, queue: Queue) {
  const lastAction = queue[queue.length - 1];

  const destination = lastAction.to;

  const towersByMicroBase = ([1, 2, 3] as TowerNum[])
    .map((towerNum) => {
      const filtered = towers[towerNum].filter(
        (n) => n < lastAction.coords.disc
      );
      return {
        towerNum,
        biggestBase: filtered.length ? Math.max(...filtered) : -1,
      };
    })
    .filter((base) => base.biggestBase > -1)
    .sort((a, b) => a.biggestBase - b.biggestBase);

  if (towersByMicroBase[0].towerNum !== destination) {
    throw new Error("towersByMicroBase was not destination. Rethink!");
  }

  const microDestination = towersByMicroBase[1].towerNum;
  const microDisc = towersByMicroBase[1].biggestBase;
  const miniDiscsToShift = getDiscsToShift(towers, {
    tower: destination,
    disc: microDisc,
  });

  return handleQueueIteration(towers, steps, [
    ...queue,
    ...miniDiscsToShift.reverse().map<QueueAction>((disc, i) => ({
      action: "shift",
      coords: {
        tower: destination,
        disc: disc,
      },
      to:
        i % 2 === 0
          ? microDestination
          : getMoveAlt(destination, microDestination),
    })),
  ]);
}
function handleQueueIteration(
  towersCopy: Towers,
  steps: number,
  queue: Queue
): number {
  if (steps > 100) {
    throw new Error("way to many steps, buddy");
  }
  const lastAction = queue[queue.length - 1];

  // console.log("start iteration", JSON.parse(JSON.stringify(queue)));

  if (canMoveDisc(towersCopy, lastAction.coords, lastAction.to)) {
    queue.pop();
    executeQueueAction(towersCopy, lastAction, queue);

    if (queue.length) {
      return handleQueueIteration(towersCopy, steps + 1, queue);
    } else {
      return steps + 1;
    }
  } else {
    if (lastAction.action === "move") {
      if (
        isTopDisc(towersCopy, lastAction.coords.tower, lastAction.coords.disc)
      ) {
        // target tower is obstructed. Must do a micro shift.
        return applyMicroShiftToQueue(towersCopy, steps, queue);
      } else {
        return applyNewShiftToQueue(towersCopy, steps, queue);
      }
    } else {
      return applyMicroShiftToQueue(towersCopy, steps, queue);
    }
  }

  return steps;
}

function getMoveAlt(currentTower: TowerNum, targetTower: TowerNum) {
  return ([1, 2, 3] as TowerNum[]).filter(
    (tower) => tower !== currentTower && tower !== targetTower
  )[0];
}

function makeMove(towers: Towers) {
  // const nextDiscCoords = getNextDiscCoords(towers);

  return handleQueueIteration(towers, 0, [
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
  ]);
}

/**
 * Vue Methods
 */
const numDiscs = ref(4);
const totalMoves = ref(0);
const towers = ref<Towers>({
  1: [],
  2: [],
  3: [],
});

const towerHanoi = (numDiscs: number) => {
  towers.value[1] = createStartingDisks(numDiscs);
};

const onClickNext = () => {
  totalMoves.value += makeMove(towers.value);
};

onMounted(() => {
  towerHanoi(numDiscs.value);
});
</script>

<template>
  <div id="app">
    <div class="left">
      <div class="towers">
        <div class="tower tower-1">
          <div
            class="disc"
            :class="[
              'disc-index-' + index,
              'disc-' + disc,
              towers[1][index] !== undefined ? 'full' : 'empty',
            ]"
            v-for="(disc, index) in numDiscs"
            :key="`1-${index}`"
          >
            <span class="disc-title" v-if="towers[1][index] !== undefined">
              {{ towers[1][index] }}
            </span>
          </div>
          <div class="tower-label">Tower<br />1</div>
        </div>
        <div class="tower tower-2">
          <div
            class="disc"
            :class="[
              'disc-index-' + index,
              'disc-' + disc,
              towers[2][index] !== undefined ? 'full' : 'empty',
            ]"
            v-for="(disc, index) in numDiscs"
            :key="`2-${index}`"
          >
            <span class="disc-title" v-if="towers[2][index] !== undefined">
              {{ towers[2][index] }}
            </span>
          </div>
          <div class="tower-label">Tower<br />2</div>
        </div>
        <div class="tower tower-3">
          <div
            class="disc"
            :class="[
              'disc-index-' + index,
              'disc-' + disc,
              towers[3][index] !== undefined ? 'full' : 'empty',
            ]"
            v-for="(disc, index) in numDiscs"
            :key="`3-${index}`"
          >
            <span class="disc-title" v-if="towers[3][index] !== undefined">
              {{ towers[3][index] }}
            </span>
          </div>
          <div class="tower-label">Tower<br />3</div>
        </div>
      </div>
      <div class="actions">
        <button @click="onClickNext">Next Move</button>
      </div>
    </div>
    <div class="right">
      <div class="state">
        <div>
          <b>Total Moves: </b>
          <span>{{ totalMoves }}</span>
        </div>
        <b>State </b>
        <pre>{{ towers }}</pre>
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
  height: 100%;
  font-family: sans-serif;
}
#app {
  display: flex;
  width: 100%;
  height: 100%;
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
}
.tower-label {
  text-align: center;
}
.actions {
  border-top: solid 4px;
  margin-top: 25px;
  padding-top: 15px;
}
</style>
