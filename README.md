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
