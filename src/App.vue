<script setup>
import {computed, onMounted, onUnmounted, reactive, ref} from "vue";

const wsUrl = "/ws"
let wsClient = undefined
let wsHeartbeatTimer = undefined
const discardZoneSize = 80

const gameState = ref({
  board: {},
  hole: {},
  decks: [],
  players: []
})
const availableDecks = ref({})
const cardHTML = reactive({})
const movingId = ref('')
const moving = ref({startClientX: 0, startClientY: 0, startX: 0, startY: 0, x: 0, y: 0, op_id: 0})
const selectedHole = ref('')

const playerName = ref("")
const roomName = ref("")
const enterDisabled = ref(false)
const showWelcome = ref(true)
const showDeckMenu = ref(false)
const showDrawDialog = ref(false)
const drawingIdx = ref(0)
const drawingTarget = ref('')
const drawingCount = ref(1)
const showAddDeckMenu = ref(false)
const showAddDeckParam = ref(false)
const addingDeck = ref("")
const addingDeckParam = ref({})

const enter = _ => {
  enterDisabled.value = true
  wsClient = new WebSocket(wsUrl)
  wsClient.onmessage = handleWebSocketMessage
  wsClient.onclose = handleWebSocketClose
  wsClient.onerror = handleWebSocketError
  wsHeartbeatTimer = setInterval(wsHeartbeat, 20000)
  wsClient.onopen = () => {
    wsRequest("join")
  }
}

onMounted(() => {
  document.addEventListener("pointermove", e => {
    if (movingId.value === '') return
    movingBoard(e)
    e.preventDefault()
  })
  document.addEventListener("pointerup", e => {
    if (movingId.value === '') return
    moveBoardEnd(e)
    e.preventDefault()
  })
})

onUnmounted(() => {
  if (wsClient !== undefined) wsClient.close(1000);
  if (wsHeartbeatTimer !== undefined) clearInterval(wsHeartbeatTimer);
})

const wsRequest = (cmd, data = null) => {
  wsClient.send(JSON.stringify({
    cmd: cmd,
    player: playerName.value,
    room: roomName.value,
    data: data
  }));
}

const handleWebSocketMessage = message => {
  const msg = JSON.parse(message.data)
  switch (msg.type) {
    case "welcome":
      availableDecks.value = msg.data.available_decks
      handleBroadcast(msg.data.broadcast)
      showWelcome.value = false
      enterDisabled.value = false
      break
    case "broadcast":
      handleBroadcast(msg.data)
      break
    case "download":
      const key = `${msg.data.deck}@${msg.data.card}`
      cardHTML[key] = msg.data.html
      break
    case "error":
    case "fatal":
      showNotification(`错误：${msg.data}`)
      enterDisabled.value = false
      break
  }
}

const handleBroadcast = broadcast => {
  if (!gameState.value.hole.hasOwnProperty(selectedHole.value)) {
    selectedHole.value = ""
  }
  if (!gameState.value.board.hasOwnProperty(movingId.value)) {
    movingId.value = ""
  }
  gameState.value = broadcast
}

const handleWebSocketClose = event => {
  console.log('Websocket closed:', event.code, event.reason)
  wsClient = undefined
}

const handleWebSocketError = event => {
  console.error('Websocket error:', event)
}

const moveBoardStart = (e, id) => {
  const rect = e.target.getBoundingClientRect()
  const x = (rect.left + rect.right) * .5
  const y = (rect.top + rect.bottom) * .5
  movingId.value = id
  moving.value = {
    startClientX: e.clientX,
    startClientY: e.clientY,
    startX: x,
    startY: y,
    x: x,
    y: y,
    op_id: gameState.value.board[id].op_id
  }
}

const movingBoard = e => {
  const m = moving.value
  const offsetX = e.clientX - m.startClientX
  const offsetY = e.clientY - m.startClientY
  m.x = m.startX + offsetX
  m.y = m.startY + offsetY
  moving.value = m
}

const moveBoardEnd = e => {
  movingBoard(e)
  const m = moving.value
  const isDiscardZone = m.x > window.innerWidth - discardZoneSize && m.y < discardZoneSize;
  if (isDiscardZone) {
    discardBoard(movingId.value, moving.value.op_id)
    movingId.value = ''
    return
  }
  const boardRect = document.getElementById("public-cards-container").getBoundingClientRect()
  if (m.y > boardRect.height) {
    wsRequest("collect", {
      id: movingId.value,
      op_id: moving.value.op_id
    })
    movingId.value = ''
    return
  }
  const x = m.x / boardRect.width
  const y = m.y / boardRect.height
  wsRequest("move", {
    id: movingId.value,
    op_id: moving.value.op_id,
    x: x,
    y: y
  })
  movingId.value = ''
}

const discardBoard = (id, op_id) => {
  wsRequest('discard_board', {
    id: id,
    op_id: op_id
  })
}

const discardBoardCM = (id, op_id, e) => {
  e.preventDefault()
  discardBoard(id, op_id)
}

const wsHeartbeat = () => {
  wsRequest("nop")
}

const clickSideButton = _ => {
  showDeckMenu.value = true
}

const clickCloseDeckMenu = _ => {
  showDeckMenu.value = false
}

const clickDeck = idx => {
  showDeckMenu.value = false
  showDrawDialog.value = true
  drawingIdx.value = idx
}

const clickAddDeck = _ => {
  showDeckMenu.value = false
  showAddDeckMenu.value = true
}

const clickCloseAddDeckMenu = _ => {
  showAddDeckMenu.value = false
}

const clickAddSpecificDeck = name => {
  addingDeck.value = name
  showAddDeckMenu.value = false
  showAddDeckParam.value = true
  let param = {}
  availableDecks.value[addingDeck.value].forEach(p => {
    if (p.default) {
      param[p.key] = p.default
    } else {
      switch (p.type) {
        case "int":
          param[p.key] = 0
          break
        case "bool":
          param[p.key] = false
          break
        case "string":
          param[p.key] = ""
          break
      }
    }
  })
  console.log(param)
  addingDeckParam.value = param
}

const clickDrawConfirm = _ => {
  wsRequest("draw", {deck: drawingIdx.value, target: drawingTarget.value, num: drawingCount.value});
  showNotification("已发牌")
}

const clickDrawCancel = _ => {
  showDrawDialog.value = false
}

const clickAddDeckConfirm = _ => {
  showAddDeckParam.value = false
  wsRequest("add_deck", {name: addingDeck.value, params: addingDeckParam.value});
}

const clickAddDeckCancel = _ => {
  showAddDeckParam.value = false
}

const selectHoleCard = c => {
  selectedHole.value = c
}

const clearSelectedHole = () => {
  selectedHole.value = ''
}

const pushHoleCard = () => {
  wsRequest("announce", {
    card: selectedHole.value,
    x: Math.random() * 0.5 + 0.25,
    y: Math.random() * 0.25 + 0.5,
  })
}

const discardHole = c => {
  wsRequest("discard_hole", c)
}

const discardHoleBtn = () => {
  const selected = selectedHole.value
  if (gameState.value.hole[selectedHole.value] <= 1) {
    const cards = Object.keys(gameState.value.hole)
    if (cards.length > 1) {
      for (let i in cards) {
        if (cards[i] === selectedHole.value) continue
        selectedHole.value = cards[i]
        break
      }
    } else {
      selectedHole.value = ''
    }
  }
  discardHole(selected)
}

const discardHoleCM = (c, e) => {
  e.preventDefault()
  discardHole(c)
}

const drawFirstBtn = () => {
  wsRequest("draw", {deck: 0, target: playerName.value, num: 1});
}

const clearGameBtn = () => {
  wsRequest('reset')
}

const allCollect = () => {
  wsRequest('all_collect')
}

const getCardHtml = dc => {
  const [id, c] = dc.split('@')
  const type = gameState.value.decks[Number.parseInt(id)].type
  const key = `${type}@${c}`
  if (!cardHTML.hasOwnProperty(key)) {
    wsRequest("download", {deck: type, card: c})
    const ret = `<div style="width: 50px; height: 50px; background-color: white"/>`
    cardHTML[key] = ret
    return ret
  }
  return cardHTML[key]
}

const showNotification = message => {
  const notification = document.createElement('div');
  notification.className = 'notification';
  notification.textContent = message;
  notification.style.cssText = `
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            background-color: #3498db;
            color: white;
            padding: 12px 24px;
            border-radius: 8px;
            font-size: 14px;
            font-weight: 600;
            z-index: 10000;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
            animation: fadeInOut 2s ease-in-out forwards;
        `;

  const style = document.createElement('style');
  style.textContent = `
            @keyframes fadeInOut {
                0% { opacity: 0; transform: translateX(-50%) translateY(-20px); }
                10% { opacity: 1; transform: translateX(-50%) translateY(0); }
                80% { opacity: 1; transform: translateX(-50%) translateY(0); }
                100% { opacity: 0; transform: translateX(-50%) translateY(-20px); }
            }
        `;
  document.head.appendChild(style);
  document.body.appendChild(notification);
  setTimeout(() => {
    notification.remove();
    style.remove();
  }, 2000);
}

const minPlaceId = computed(() => {
  return Object.values(gameState.value.board).reduce((a, c) => Math.min(a, c.pl_id), Infinity)
})
</script>

<template>
  <div id="public-area" class="area public-area">
    <div id="public-cards-container" class="cards-container" @click="clearSelectedHole">
      <template v-for="(c, id) in gameState.board">
        <div class="board-child" v-if="id !== movingId"
             :style="`left: ${c.x * 100}%; top: ${c.y * 100}%; z-index: ${c.pl_id - minPlaceId}`"
             v-html="getCardHtml(c.card)"
             @pointerdown="moveBoardStart($event, id)"
             @contextmenu="discardBoardCM(id, c.op_id, $event)"
        />
      </template>
      <div class="board-child" v-if="movingId"
           :style="`left: ${moving.x}px; top: ${moving.y}px; z-index: 10000`"
           v-html="getCardHtml(gameState.board[movingId].card)"
      />
    </div>
    <button class="hole-push-btn" v-show="selectedHole !== ''" @click="pushHoleCard"> 发牌 </button>
    <button class="hole-discard-btn" v-show="selectedHole !== ''" @click="discardHoleBtn"> 弃牌 </button>
  </div>

  <div id="hand-area" class="area hand-area">
    <div id="hand-cards-container" class="cards-container horizontal-scroll">
      <div v-for="(count, c) in gameState.hole"
           @click="selectHoleCard(c)"
           @contextmenu="discardHoleCM(c, $event)"
           :class="`hole-child ${(selectedHole === c ? 'selected-hole' : '')}`">
        <div v-html="getCardHtml(c)" class="hole-card"/>
        <div class="hole-card-count" v-show="count > 1">
          x{{count}}
        </div>
      </div>
    </div>
  </div>

  <div id="discard-zone" :class="`discard-zone ${movingId ? 'active' : ''}`"/>

  <div v-if="showWelcome" id="welcome-modal" class="modal show">
    <div class="modal-content welcome-content">
      <h2>欢迎来到通用桌游模拟器</h2>
      <div class="form-group">
        <label for="player-name">玩家名称：</label>
        <input type="text" id="player-name" class="form-input" maxlength="20" v-model="playerName">
      </div>
      <div class="form-group">
        <label for="room-name">房间名称：</label>
        <input type="text" id="room-name" class="form-input" maxlength="20" v-model="roomName">
      </div>
      <button id="enter-game" class="btn btn-primary btn-full" :disabled="enterDisabled" @click="enter">进入游戏</button>
    </div>
  </div>

  <button class="side-button" style="left: 0; top: 0" @click="clickSideButton">
    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
      <line x1="3" y1="9" x2="21" y2="9"></line>
      <line x1="9" y1="21" x2="9" y2="9"></line>
    </svg>
  </button>

  <button class="side-button" style="left: 0; top: 75px" v-show="gameState.decks.length > 0" @click="drawFirstBtn">
    <svg viewBox="0 0 24 24" width="24" height="24" fill="currentColor">
      <path d="M11 5h2v6h6v2h-6v6h-2v-6H5v-2h6V5z"/>
    </svg>
  </button>

  <button class="side-button" style="right: 0; top: 0; background-color: red" v-show="movingId === ''" @click="clearGameBtn">
    <svg viewBox="0 0 24 24" width="24" height="24" fill="currentColor">
      <path d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"/>
    </svg>
  </button>

  <button class="side-button" style="right: 0; top: 75px" v-show="Object.keys(gameState.board).length > 0" @click="allCollect">
    <svg viewBox="0 0 24 24" width="24" height="24" fill="currentColor" stroke="currentColor" stroke-width="1.5">
      <line x1="12" y1="4" x2="12" y2="13" stroke="currentColor" stroke-width="1.5"/>
      <line x1="12" y1="13" x2="8" y2="10" stroke="currentColor" stroke-width="1.5"/>
      <line x1="12" y1="13" x2="16" y2="10" stroke="currentColor" stroke-width="1.5"/>
      <line x1="6" y1="18" x2="18" y2="18" stroke="currentColor" stroke-width="1.5"/>
      <line x1="6" y1="12" x2="6" y2="18" stroke="currentColor" stroke-width="1.5"/>
      <line x1="18" y1="12" x2="18" y2="18" stroke="currentColor" stroke-width="1.5"/>
    </svg>
  </button>

  <div v-if="showDeckMenu" id="deck-menu" class="modal show">
    <div class="modal-content">
      <div class="modal-header">
        <h3>牌堆管理</h3>
        <button id="close-deck-menu" class="close-btn" @click="clickCloseDeckMenu">&times;</button>
      </div>
      <div class="modal-body">
        <div class="deck-list" style="color: firebrick">
          <div class="deck-item" v-for="(deck, idx) in gameState.decks" :key="idx" @click="clickDeck(idx)">
            <span class="deck-name">{{deck.name}}</span>
            <span class="deck-count" v-if="deck.rest_len >= 0">{{deck.rest_len}}</span>
            <span class="deck-total" v-if="deck.max_len >= 0">/ {{deck.max_len}}</span>
          </div>
          <div class="deck-item" @click="clickAddDeck">
            <span class="deck-name">+</span>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div v-if="showAddDeckMenu" id="add-deck-menu" class="modal show">
    <div class="modal-content">
      <div class="modal-header">
        <h3>添加牌堆</h3>
        <button id="close-deck-menu" class="close-btn" @click="clickCloseAddDeckMenu">&times;</button>
      </div>
      <div class="modal-body">
        <div class="deck-list">
          <div class="deck-item" v-for="name in Object.keys(availableDecks)" :key="name" @click="clickAddSpecificDeck(name)">
            <span class="deck-name">{{name}}</span>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div v-if="showDrawDialog" id="deal-dialog" class="modal show">
    <div class="modal-content dialog-content">
      <h3>发牌</h3>
      <div class="deal-options">
        <div class="deal-option">
          <label>
            <input type="radio" v-model="drawingTarget" name="drawTargets" :value="playerName" checked>
            发入手牌
          </label>
        </div>
        <div class="deal-option">
          <label>
            <input type="radio" v-model="drawingTarget" name="drawTargets" value="">
            发入公共区
          </label>
        </div>
      </div>
      <div class="deal-input">
        <label for="card-count">牌数：</label>
        <input type="number" id="card-count" min="1" v-model.number="drawingCount">
      </div>
      <div class="deal-buttons">
        <button id="confirm-deal" class="btn btn-primary" @click="clickDrawConfirm">确定</button>
        <button id="cancel-deal" class="btn btn-secondary" @click="clickDrawCancel">取消</button>
      </div>
    </div>
  </div>

  <div v-if="showAddDeckParam" id="deal-dialog" class="modal show">
    <div class="modal-content dialog-content">
      <h3>添加牌堆：{{addingDeck}}</h3>
      <div style="margin-top: 5px" class="form-group" v-for="param in availableDecks[addingDeck]" :key="param.key">
        <label>
          {{param.label}}
          <input
              v-if="param.type === 'bool'"
              type="checkbox"
              :id="param.key"
              v-model="addingDeckParam[param.key]"
              style="margin-left: 10px"
          />
        </label>
        <input
            v-if="param.type === 'int'"
            type="number"
            :id="param.key"
            v-model.number="addingDeckParam[param.key]"
            :min="param.min"
            :max="param.max"
            class="form-input"
        />
        <input
            v-else-if="param.type === 'string'"
            type="text"
            :id="param.key"
            v-model="addingDeckParam[param.key]"
            class="form-input"
        />
      </div>
      <div class="deal-buttons">
        <button id="confirm-deal" class="btn btn-primary" @click="clickAddDeckConfirm">确定</button>
        <button id="cancel-deal" class="btn btn-secondary" @click="clickAddDeckCancel">取消</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.hole-child {
  position: relative;
  width: 65px;
  min-width: 65px;
  height: 90px;
  user-select: none;
  overflow: hidden;
}

.selected-hole {
  transform: translate(0, -5%);
}

.hole-card {
  position: absolute;
  transform: translate(-50%, -50%);
  top: 50%;
  left: 50%;
  width: 100% !important;
  height: auto !important;
}

.hole-push-btn {
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100px;
  height: 60px;
  background-color: #3498db;
  display: grid;
  place-items: center;
}

.hole-discard-btn {
  position: absolute;
  right: 0;
  bottom: 0;
  width: 100px;
  height: 60px;
  background-color: #ff5555;
  display: grid;
  place-items: center;
}

.hole-card-count {
  position: absolute;
  transform: translate(-100%, -100%);
  top: 95%;
  left: 95%;
  font-weight: bold;
  text-shadow:
      1px 1px 0 white,
      -1px -1px 0 white,
      1px -1px 0 white,
      -1px 1px 0 white,
      1px 0 0 white,
      -1px 0 0 white,
      0 1px 0 white,
      0 -1px 0 white;
}

.board-child {
  position: absolute;
  transform: translate(-50%, -50%);
  user-select: none;
}
</style>
