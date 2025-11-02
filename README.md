# GAME.github.io
四則運算大挑戰
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>數學藏寶圖尋寶遊戲</title>
    <link rel="stylesheet" href="style.css">
    </head>
<body>

    <div id="win-animation-overlay" style="display: none;">
        <div id="win-message">🎉 恭喜找到大寶藏！</div>
    </div>

    <div id="overlay-layer" style="display: none;">
        <div id="display-content"></div>
    </div>

    <header>
        <h1>數學藏寶圖尋寶遊戲</h1>
        <p>收集最多點數，找到寶藏！</p>
    </header>

    <div class="controls">
        <input type="text" id="playerNameInput" placeholder="輸入玩家名稱">
        <button onclick="addPlayer()">加入玩家</button>
        <button id="startGameButton" onclick="startGame()" disabled>開始遊戲</button>
    </div>

    <div class="main-layout-container">
        
        <div class="side-panel" id="players-list-container">
            <h3>玩家行動區</h3>
            <div class="players-action-bar" id="players-list">
                <p class="hint">請先加入玩家</p>
            </div>
        </div>

        <div class="center-map-area">
            <div class="map-container" id="game-map">
                </div>
        </div>

        <div class="side-panel" id="score-info-container">
            <h3>回合紀錄與總分</h3>
            <div id="individual-score-container">
                <p class="hint">回合紀錄將顯示在這裡</p>
                </div>
        </div>
    </div>

    <script src="script.js"></script>
</body>
</html>

// ====== 1. 資料初始化 ======

// 點數分布：總共 20 格 (Path ID 0 到 19)。
const mapPoints = [
    0, 1, 2, 1, 2, 
    1, 2, 3, 1, 2, 
    1, 2, 1, 3, 2, 
    1, 2, 1, 3, 0  // Path ID 19 (索引 19) 是終點/大寶藏 (0 點)
];
// 終點 ID 為 19
const mapLength = mapPoints.length - 1; 

const DICE_WAIT_MS = 600; 
const MOVE_DELAY_MS = 650; 
const DICE_ANIMATION_MS = 1500;

let players = []; 
let nextPlayerId = 1; 
let gameStarted = false; 
let isAnimationActive = false; 

let mapContainer, playersList, individualScoreContainer, playerNameInput, startGameButton, winOverlay;
let overlayLayer, displayContent; 

const playerEmojis = ['😀', '😎', '🤩', '🥳', '🤓', '😇', '🤠', '🤖', '👻', '👽', '🐶', '🐱'];
const playerColors = ['#e74c3c', '#3498db', '#2ecc71', '#f39c12', '#9b59b6', '#1abc9c', '#e67e22', '#2c3e50', '#7f8c8d', '#c0392b', '#16a085', '#d35400']; 

/**
 * 🎯 修正: 確保路徑順序為 P17 -> Grid 13, P18 -> Grid 12, P19 -> Grid 11。
 */
const GRID_ORDER = [
    0, 1, 2, 3, 4,      // P0-4
    9, 14, 19, 18, 17,  // P5-9
    16, 15, 10, 5,      // P10-13
    6, 7, 8,            // P14-16
    13,                 // P17: Grid ID 13
    12,                 // 🎯 P18: Grid ID 12
    11                  // 🎯 P19: Grid ID 11 (大寶藏)
];


// ====== 2. 遊戲初始化函數 (保持不變) ======

function initializeDOMReferences() {
    mapContainer = document.getElementById('game-map');
    playersList = document.getElementById('players-list');
    individualScoreContainer = document.getElementById('individual-score-container');
    playerNameInput = document.getElementById('playerNameInput');
    startGameButton = document.getElementById('startGameButton');
    winOverlay = document.getElementById('win-animation-overlay'); 
    
    overlayLayer = document.getElementById('overlay-layer');
    displayContent = document.getElementById('display-content');

    window.addPlayer = addPlayer;
    window.startGame = startGame;
    window.playerRoll = playerRoll;
    window.removePlayer = removePlayer; 

    return mapContainer && playersList && individualScoreContainer && playerNameInput && startGameButton && winOverlay && overlayLayer && displayContent;
}

function renderMap() {
    if (!mapContainer) return;
    mapContainer.innerHTML = ''; 
    
    mapPoints.forEach((points, pathId) => {
        const gridId = GRID_ORDER[pathId]; 

        const cell = document.createElement('div');
        cell.className = 'cell';
        cell.id = `cell-${pathId}`; 
        
        // 網格定位：(R: 1-4, C: 1-5)
        const row = Math.floor(gridId / 5);
        const col = gridId % 5;
        
        cell.style.gridRow = row + 1;
        cell.style.gridColumn = col + 1;
        
        let content = '';
        if (pathId === 0) {
            content = '⭐'; // 起點
        } else if (pathId === mapLength) {
            content = ''; // 終點 (Path ID 19)
            cell.classList.add('treasure-box');
        } else {
            content = points.toString(); 
            cell.classList.add(`cell-points-${points}`); 
        }
        
        cell.innerHTML = content;
        mapContainer.appendChild(cell);
    });
}

/**
 * 繪製直角路徑線條 (保持不變)
 */
function drawPathLines() {
    if (!mapContainer) return;

    // 1. 清除舊的 SVG
    document.querySelectorAll('.path-svg').forEach(svg => svg.remove());

    const mapRect = mapContainer.getBoundingClientRect();
    
    // 2. 創建新的 SVG 容器
    const svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
    svg.setAttribute('class', 'path-svg');
    svg.setAttribute('width', mapRect.width);
    svg.setAttribute('height', mapRect.height);
    svg.setAttribute('viewBox', `0 0 ${mapRect.width} ${mapRect.height}`);
    mapContainer.appendChild(svg);

    const path = document.createElementNS("http://www.w3.org/2000/svg", "path");
    let dAttribute = '';
    
    // 3. 遍歷所有路徑點 (0 到 19) 計算中心點座標
    for (let i = 0; i <= mapLength; i++) {
        const currentCell = document.getElementById(`cell-${i}`);

        if (currentCell) {
            const currentRect = currentCell.getBoundingClientRect();

            // 相對於 mapContainer 的座標 (中心點)
            const currentX = (currentRect.left + currentRect.right) / 2 - mapRect.left;
            const currentY = (currentRect.top + currentRect.bottom) / 2 - mapRect.top;

            if (i === 0) {
                // 起點：M (MoveTo)
                dAttribute += `M ${currentX} ${currentY} `;
            } else {
                // 之後的點：L (LineTo) 畫直線
                dAttribute += `L ${currentX} ${currentY} `;
            }
        }
    }
    
    path.setAttribute('d', dAttribute.trim());
    svg.appendChild(path);
}

function init() {
    if (initializeDOMReferences()) {
        renderMap(); 
        
        window.addEventListener('load', () => {
            setTimeout(() => {
                drawPathLines();
                players.forEach(player => placeToken(player)); 
            }, 500); 
        });

        overlayLayer.addEventListener('click', hideOverlay);
    } else {
        console.error("初始化失敗：部分 HTML 元素未載入。");
    }
}

document.addEventListener('DOMContentLoaded', init);

window.addEventListener('resize', () => {
    setTimeout(() => {
        if (mapContainer && mapContainer.children.length > 0) {
            drawPathLines();
            players.forEach(player => placeToken(player));
        }
    }, 100);
});
// ====== 3. 玩家管理與流程控制 (無變動) ======

function startGame() {
    if (players.length === 0) {
        alert("請至少加入一位玩家才能開始遊戲！");
        return;
    }
    gameStarted = true;
    startGameButton.disabled = true; 
    disableAllRollButtons(false);
}

function addPlayer() {
    if (!playerNameInput) return;

    const name = playerNameInput.value.trim();
    if (name === "" || players.length >= playerEmojis.length) { 
        alert("請輸入有效名字或玩家數量已達上限！");
        return;
    }

    const usedEmojis = players.map(p => p.emoji);
    const availableEmojis = playerEmojis.filter(emoji => !usedEmojis.includes(emoji));
    
    const usedColors = players.map(p => p.color);
    const availableColors = playerColors.filter(color => !usedColors.includes(color));

    if (availableEmojis.length === 0 || availableColors.length === 0) {
        alert("已無可用的表情符號或顏色。");
        return;
    }

    const randomEmoji = availableEmojis[Math.floor(Math.random() * availableEmojis.length)];
    const randomColor = availableColors[Math.floor(Math.random() * availableColors.length)];

    const newPlayer = {
        id: nextPlayerId++,
        name: name,
        position: 0, 
        currentTotalScore: 0, 
        currentTotalSteps: 0, 
        emoji: randomEmoji, 
        color: randomColor
    };

    players.push(newPlayer);
    playerNameInput.value = '';

    renderPlayersList(); 
    createPlayerScoreTable(newPlayer); 
    placeToken(newPlayer); 
    
    startGameButton.disabled = false;
}

function removePlayer(playerId) {
    const playerIndex = players.findIndex(p => p.id === playerId);
    if (playerIndex === -1) return;

    const removedPlayer = players[playerIndex];
    players.splice(playerIndex, 1);

    const scoreBox = document.getElementById(`score-box-player-${playerId}`);
    if (scoreBox) scoreBox.remove();
    
    const token = document.querySelector(`.token[data-player-id="${playerId}"]`);
    if (token) token.remove();
    
    renderPlayersList();
    if (players.length === 0) {
        startGameButton.disabled = true;
    }

    alert(`玩家 ${removedPlayer.name} (${removedPlayer.emoji}) 已被移除。`);
}

function renderPlayersList() {
    if (players.length === 0) {
        playersList.innerHTML = '<p class="hint">請先加入玩家</p>';
        return;
    }
    
    let html = '';
    players.forEach(player => {
        const isDisabled = !gameStarted || player.position === mapLength || isAnimationActive; 
        
        html += `
            <div class="player-control" data-player-id="${player.id}" style="border-left: 5px solid ${player.color};">
                <strong>${player.name} (${player.emoji})</strong> 
                <button id="roll-btn-${player.id}" onclick="playerRoll(${player.id})" ${isDisabled ? 'disabled' : ''}>🎲 骰骰子</button>
                <button class="remove-player-btn" onclick="removePlayer(${player.id})">移除</button>
            </div>
        `;
    });
    playersList.innerHTML = html;
}

function disableAllRollButtons(disable) {
    players.forEach(player => {
        const button = document.getElementById(`roll-btn-${player.id}`);
        if (button) button.disabled = disable;
    });
}

// ====== 4. 遊戲核心邏輯：骰子與移動 (無變動) ======

function showOverlay(contentHtml, isPermanent = false, callback) {
    isAnimationActive = true;
    disableAllRollButtons(true);
    
    displayContent.innerHTML = contentHtml;
    overlayLayer.style.display = 'flex';
    
    if (!isPermanent && callback) {
         setTimeout(callback, 500); 
    }
}

function hideOverlay() {
    const contentDiv = displayContent.querySelector('#points-display');
    if (overlayLayer.style.display === 'flex' && isAnimationActive && contentDiv) {
        overlayLayer.style.display = 'none';
        displayContent.innerHTML = '';
        isAnimationActive = false;
        disableAllRollButtons(false);
        renderPlayersList();
    }
}

function showDiceAnimation(steps, callback) {
    
    const diceHtml = `<div id="dice-display">${Math.floor(Math.random() * 6) + 1}</div>`;
    
    showOverlay(diceHtml, false);

    const interval = setInterval(() => {
        document.getElementById('dice-display').textContent = Math.floor(Math.random() * 6) + 1; 
    }, 100);

    setTimeout(() => {
        clearInterval(interval);
        document.getElementById('dice-display').textContent = steps;
        
        setTimeout(() => {
            overlayLayer.style.display = 'none';
            displayContent.innerHTML = '';
            if (callback) callback();
        }, DICE_WAIT_MS); 
    }, DICE_ANIMATION_MS); 
}

function showPointsAnimation(points) {
    
    let message = '';
    if (points === 0) {
         message = `<strong>終點！</strong>`;
    } else {
         message = `最終點數：<br><strong>${points}</strong>`;
    }
    
    const pointsHtml = `<div id="points-display">${message}</div>`;
    
    showOverlay(pointsHtml, true); 
}

function moveTokenSequentially(player, totalSteps, finalPosition, callback) {
    
    if (totalSteps <= 0 || player.position === mapLength) {
        if (callback) callback();
        return;
    }
    
    let nextPosition = Math.min(player.position + 1, mapLength);
    
    player.position = nextPosition;
    placeToken(player); 
    
    setTimeout(() => {
        moveTokenSequentially(player, totalSteps - 1, finalPosition, callback);
    }, MOVE_DELAY_MS); 
}

function playerRoll(playerId) {
    if (!gameStarted || isAnimationActive) return; 

    const player = players.find(p => p.id === playerId);
    if (!player || player.position === mapLength) return;

    // 骰子機率平均分配 (1-6)
    const steps = Math.floor(Math.random() * 6) + 1; 
    
    const finalPosition = Math.min(player.position + steps, mapLength);

    isAnimationActive = true;
    disableAllRollButtons(true); 

    // 1. 顯示骰子動畫
    showDiceAnimation(steps, () => {
        
        // 2. 開始逐步移動
        moveTokenSequentially(player, steps, finalPosition, () => {
            
            // 3. 移動結束後，計算分數並更新 UI
            const newPosition = player.position;
            const isFinished = newPosition === mapLength;
            const finalPoints = mapPoints[newPosition];
            
            player.currentTotalScore += finalPoints;
            player.currentTotalSteps += steps;

            recordPlayerTurn(player, steps, finalPoints);
            updateUI(player, steps, finalPoints, isFinished);
            
            // 4. 顯示點數提示 (手動關閉)
            showPointsAnimation(finalPoints);
            
            // 5. 處理終點到達事件
            if (isFinished) {
                showWinAnimation(player);
            }
        }); 
    });
}

// ====== 5. 畫面更新與統計 (無變動) ======

function showWinAnimation(winner) {
    if (!winOverlay) return;

    const winMessage = document.getElementById('win-message');
    winMessage.innerHTML = `🎉 恭喜 ${winner.name} (${winner.emoji}) 找到大寶藏！`;

    winOverlay.style.display = 'flex';
    
    setTimeout(() => {
        winOverlay.style.display = 'none';
    }, 3000);
}

function updateUI(player, steps, points, isFinished) {
    if (isFinished) {
        const button = document.getElementById(`roll-btn-${player.id}`);
        if (button) button.disabled = true; 
    }
    if (player.position === mapLength) {
        // 終點時移除角色
        const token = document.querySelector(`.token[data-player-id="${player.id}"]`);
        if (token) token.style.display = 'none';
        
        const treasureCell = document.getElementById(`cell-${mapLength}`);
        if (treasureCell) {
             const container = treasureCell.querySelector('.token-container');
             if (container) container.remove();
        }
    }
    renderPlayersList();
}

function placeToken(player) {
    const targetCell = document.getElementById(`cell-${player.position}`);
    if (!targetCell) return;
    
    // 1. 移除舊的 token (確保只移除當前玩家的舊 token)
    const oldToken = document.querySelector(`.token[data-player-id="${player.id}"]`);
    if (oldToken) {
        const container = oldToken.closest('.token-container');
        oldToken.remove();
        if (container && container.children.length === 0) {
            container.remove();
        }
    }

    // 2. 尋找或創建 token-container
    let container = targetCell.querySelector('.token-container');
    if (!container) {
        container = document.createElement('div');
        container.className = 'token-container';
        targetCell.appendChild(container);
    }
    
    // 3. 創建並放置新的 token
    const token = document.createElement('div');
    token.className = 'token';
    token.setAttribute('data-player-id', player.id);
    token.textContent = player.emoji; 
    token.style.backgroundColor = player.color;
    
    container.appendChild(token);
}

function createPlayerScoreTable(player) {
    const hint = individualScoreContainer.querySelector('.hint');
    if (hint) hint.remove();

    const tableContainer = document.createElement('div');
    tableContainer.style.border = `2px solid ${player.color}`;
    tableContainer.style.borderRadius = '5px';
    tableContainer.style.padding = '10px';
    tableContainer.id = `score-box-player-${player.id}`;

    const table = document.createElement('table');
    table.className = 'player-score-table';
    table.id = `score-table-player-${player.id}`;
    
    table.innerHTML = `
        <thead>
            <tr><th colspan="2" style="background-color: ${player.color}; color: white; text-align: center;">${player.name} (${player.emoji}) 的回合記錄 (總步數: 0, 總點數: 0)</th></tr>
            <tr>
                <th>步數</th>
                <th>點數</th>
            </tr>
        </thead>
        <tbody></tbody>
    `;

    // 🏆 修正處：將 table 加入 tableContainer
    tableContainer.appendChild(table); // ⭐️ 將表格 (table) 加入表格容器 (tableContainer)
    
    // 將表格容器加入側邊欄
    individualScoreContainer.appendChild(tableContainer);
}

function recordPlayerTurn(player, steps, points) {
    const table = document.getElementById(`score-table-player-${player.id}`);
    if (!table) return;

    const tBody = table.querySelector('tbody');
    const tHeadRow = table.querySelector('thead tr:first-child th');
    
    tHeadRow.innerHTML = `${player.name} (${player.emoji}) 的回合記錄 (總步數: ${player.currentTotalSteps}, 總點數: ${player.currentTotalScore})`;

    const newRow = tBody.insertRow(0); 
    
    newRow.insertCell().textContent = steps; 
    newRow.insertCell().textContent = points; 
}

/* 基本重置和字體設定 */
body {
    font-family: 'Comic Sans MS', cursive, sans-serif; 
    background-color: #f5f0e8; 
    color: #4a4a4a; 
    margin: 0;
    padding: 10px; 
    text-align: center;
    min-height: 100vh; 
    display: flex;
    flex-direction: column;
}

header {
    background-color: #8b4513; 
    color: white;
    padding: 15px; 
    margin-bottom: 15px;
    border-radius: 8px; 
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.controls {
    text-align: center;
    padding: 10px;
    margin-bottom: 15px;
    background-color: #fff;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.main-layout-container {
    display: flex;
    flex: 1; 
    gap: 15px;
    max-width: 1400px; 
    margin: 0 auto;
    width: 100%;
}

.side-panel {
    width: 250px; 
    background-color: #fff;
    border: 3px solid #a07a4a;
    border-radius: 8px;
    padding: 10px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    display: flex;
    flex-direction: column;
    overflow-y: auto; 
}
#players-list-container { order: 1; }
#score-info-container { order: 3; }

.center-map-area {
    flex: 1; 
    display: flex;
    justify-content: center;
    align-items: center;
    order: 2; 
    min-width: 500px; 
}

/* 地圖容器 - 5x4 網格佈局 */
.map-container {
    display: grid;
    grid-template-columns: repeat(5, 1fr); 
    grid-template-rows: repeat(4, minmax(60px, 1fr)); 
    gap: 5px; 
    padding: 15px;
    background-color: #c9b79c; 
    width: 100%; 
    max-width: 750px; 
    aspect-ratio: 5 / 4.5; 
    margin: 0; 
    position: relative; 
    border-radius: 10px;
    box-shadow: 0 0 20px rgba(0, 0, 0, 0.3); 
}

/* 地圖背景圖案 */
.map-container::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
    pointer-events: none; 
    
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='100' height='100'%3E%3Ctext x='50%25' y='50%25' font-size='30' text-anchor='middle' alignment-baseline='middle'%3E%F0%9F%8C%B4%3C/text%3E%3C/svg%3E"), 
                      url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='100' height='100'%3E%3Ctext x='50%25' y='50%25' font-size='30' text-anchor='middle' alignment-baseline='middle'%3E%F0%9F%92%B0%3C/text%3E%3C/svg%3E"),
                      url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='100' height='100'%3E%3Ctext x='50%25' y='50%25' font-size='30' text-anchor='middle' alignment-baseline='middle'%3E%E2%98%A0%EF%B8%8F%3C/text%3E%3C/svg%3E");

    background-repeat: repeat;
    background-size: 100px 100px, 120px 120px, 80px 80px;
    background-position: 0 0, 50px 50px, 25px 25px;

    opacity: 0.15;
    filter: sepia(0.8) grayscale(0.5); 
}

/* 地圖格子樣式 */
.cell {
    border: 2px solid #6b4c3b; 
    background-color: #f7e0b5; 
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    
    font-size: 2.2em; 
    font-weight: bold;
    position: relative; 
    padding: 5px;
    overflow: hidden; 
    border-radius: 50%; 
    aspect-ratio: 1 / 1; 
    z-index: 2; 
    box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    color: #4a4a4a; 
    transition: all 0.2s; 
    min-height: 40px; 
    max-width: 80px; 
    margin: auto; 
}

/* 點數顏色標記 */
.cell-points-1 { background-color: #c8e6c9; } 
.cell-points-2 { background-color: #fff9c4; } 
.cell-points-3 { background-color: #ffccbc; } 

/* 終點寶藏箱樣式 (ID: 19) */
#cell-19 { /* 🎯 修正: 更改 ID 為 19 */
    background-color: gold; 
    font-size: 2.2em; 
    box-shadow: 0 0 20px rgba(255, 215, 0, 0.8); 
    transform: none; 
}
#cell-19::before { /* 🎯 修正: 更改 ID 為 19 */
    content: "💰"; 
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 2em;
}

/* 路徑 SVG 樣式 - 確保直角 */
.path-svg {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; 
    z-index: 1; 
    overflow: hidden; 
}
.path-svg path {
    fill: none;
    stroke: #6b4c3b; 
    stroke-width: 6px; 
    
    stroke-linecap: butt; 
    stroke-linejoin: miter; 
    
    filter: drop-shadow(1px 1px 2px rgba(0,0,0,0.5)); 
}


/* 角色標記 (Token) 相關 */
.token-container {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%); 
    display: flex; 
    justify-content: center;
    align-items: center;
    gap: 1px; 
    z-index: 10;
    pointer-events: none; 
    flex-wrap: nowrap;
}

.token {
    position: static; 
    width: 25px; 
    height: 25px;
    line-height: 25px;
    border-radius: 50%; 
    font-size: 1em; 
    font-weight: bold;
    color: white;
    border: 2px solid white;
    z-index: 1; 
    transition: all 0.6s ease-in-out; 
    text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
    display: flex; 
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.5); 
    flex-shrink: 0; 
}

/* =================================================== */
/* 🎯 START: 您提供的骰子和點數提示樣式 (已整合) */
/* =================================================== */

/* 骰子/點數 滾動動畫疊層 */
#overlay-layer {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.7); 
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000; /* 提高 Z-index 確保覆蓋一切 */
    backdrop-filter: blur(5px); 
}

/* 骰子顯示 (位於 #display-content 內) */
#dice-display {
    width: 150px;
    height: 150px;
    background-color: white;
    border-radius: 25px;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 6em; 
    color: #8b4513;
    border: 5px solid gold;
    box-shadow: 0 0 30px rgba(255, 215, 0, 0.5);
    animation: bounce 0.5s infinite alternate; 
}

/* 點數提示框樣式 */
#points-display {
    width: 280px; 
    height: 150px;
    background-color: white;
    border-radius: 25px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    font-size: 2.2em; 
    color: #8b4513;
    border: 5px solid #3498db; 
    box-shadow: 0 0 30px rgba(52, 152, 219, 0.5);
    animation: fanfare 0.3s forwards; 
    padding: 10px;
}
#points-display strong {
    font-size: 3em; 
    color: #e74c3c;
    white-space: nowrap; 
}

/* 骰子跳動關鍵影格 */
@keyframes bounce {
    from {
        transform: scale(1);
    }
    to {
        transform: scale(1.1);
    }
}

/* =================================================== */
/* 🎯 END: 您提供的骰子和點數提示樣式 (已整合) */
/* =================================================== */


/* 獲勝動畫樣式 */
#win-animation-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 200; 
}

#win-message {
    background: linear-gradient(45deg, gold, yellow, #ffd700);
    color: #8b4513;
    padding: 40px 60px;
    border-radius: 20px;
    font-size: 3em;
    font-weight: bold;
    box-shadow: 0 0 50px gold, 0 0 20px rgba(255, 255, 0, 0.5);
    transform: scale(0.1);
    animation: fanfare 1s forwards cubic-bezier(0.68, -0.55, 0.27, 1.55);
    text-shadow: 2px 2px 5px rgba(0,0,0,0.3);
}

@keyframes fanfare {
    0% { transform: scale(0.1); opacity: 0; }
    80% { transform: scale(1.2); opacity: 1; }
    100% { transform: scale(1); opacity: 1; }
}

/* 分數表格樣式 */
#individual-score-container {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    padding: 10px;
    justify-content: space-around;
}

.player-score-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 10px;
}

.player-score-table th, .player-score-table td {
    border: 1px solid #ccc;
    padding: 8px;
    text-align: center;
}

.player-score-table thead th {
    font-size: 1.1em;
}
