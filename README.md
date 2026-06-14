<!DOCTYPE html>
<html lang="fa">
<head>
    <meta charset="UTF-8">
    <title>Plato Grand Battle Arena</title>
    <style>
        body {
            margin: 0; background: #070a12; color: #c9d1d9;
            font-family: 'Segoe UI', Tahoma, sans-serif;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            min-height: 100vh; overflow-y: auto; padding: 20px 0; direction: rtl;
        }
        h1 { margin: 0; font-size: 24px; text-shadow: 0 0 15px #00f2fe; color: #00f2fe; }
        #status { font-size: 15px; color: #f0883e; margin: 5px 0; font-weight: bold; min-height: 24px; }
        
        #top-bar { display: flex; gap: 25px; font-size: 14px; margin-bottom: 10px; background: #161b22; padding: 8px 15px; border-radius: 8px; border: 1px solid #30363d;}

        @keyframes shake {
            0% { transform: translate(1px, 1px) rotate(0deg); }
            20% { transform: translate(-2px, -1px) rotate(1deg); }
            100% { transform: translate(1px, -2px) rotate(0deg); }
        }
        .shake-effect { animation: shake 0.2s; }

        #board {
            display: grid; 
            grid-template-columns: repeat(10, 48px); 
            grid-template-rows: repeat(16, 48px);
            gap: 3px; background: #161b22; padding: 8px; border-radius: 12px;
            box-shadow: 0 0 35px rgba(0, 242, 254, 0.15); border: 2px solid #30363d; position: relative;
        }
        .cell {
            background: #21262d; border-radius: 4px; display: flex; align-items: center;
            justify-content: center; position: relative; cursor: pointer; transition: background 0.2s;
            width: 48px; height: 48px;
        }
        .cell:hover { background: #30363d; }
        .destroyed-row { background: #2a080c !important; border: 1px solid #da3633 !important; cursor: not-allowed; }
        
        .valid-move { background: rgba(0, 242, 254, 0.15) !important; border: 1px dashed #00f2fe; }
        .valid-attack { background: rgba(242, 107, 107, 0.25) !important; border: 1px dashed #f85149; }

        .obstacle { background: #2d333b !important; border: 1px solid #444; cursor: not-allowed; }
        .obstacle::after { content: '🛑'; font-size: 12px; opacity: 0.5; }

        .portal { background: #1f1937 !important; border: 1px solid #8a2be2; }
        .portal::after { content: '🌀'; font-size: 14px; }

        .piece {
            width: 38px; height: 38px; border-radius: 50%;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            font-weight: bold; font-size: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.6);
            user-select: none; transition: all 0.3s ease; z-index: 2; position: relative;
        }
        .hp-bar { font-size: 7px; background: rgba(0,0,0,0.8); color: #ff3b30; padding: 1px 2px; border-radius: 3px; margin-top: -2px; font-weight: bold;}
        .selected { transform: scale(1.15); box-shadow: 0 0 15px #00f2fe !important; border: 2px solid #fff !important; }

        .white-p { background: #ffffff; color: #000; border: 2px solid #ccc; }
        .black-p { background: #1f1f1f; color: #fff; border: 2px solid #444; }
        .blue-p { background: #1f6feb; color: #fff; border: 2px solid #58a6ff; }
        .red-p { background: #da3633; color: #fff; border: 2px solid #f85149; }
        .king-p { background: linear-gradient(135deg, #ffd700, #ffa500) !important; color: #000 !important; border: 2px solid #fff !important; box-shadow: 0 0 12px #ffd700 !important; }

        #particleCanvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 5; }

        #shop-container { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 12px; background: #161b22; padding: 10px; border-radius: 10px; border: 1px solid #30363d; max-width: 500px; justify-content: center; }
        .card {
            background: #1f242c; border: 1px solid #30363d; border-radius: 6px; padding: 6px 12px;
            cursor: pointer; font-size: 11px; font-weight: bold; transition: 0.2s; color: #fff;
        }
        .card:hover { border-color: #00f2fe; background: #262c36; }
    </style>
</head>
<body>

    <h1>آرنای استراتژیک بزرگ پلاتو</h1>
    
    <div id="top-bar">
        <div>💰 سکه شما: <span id="coin-count" style="color:#ffd700;">40</span></div>
        <div>🤖 سکه بات: <span id="bot-coin-count" style="color:#ffd700;">40</span></div>
        <div>⏱️ نوبت: <span id="turn-count">1</span>/20</div>
    </div>

    <div id="status">بازی با ۴۰ سکه آغاز شد. برای اژدها یا تسخیر مهره، ۱۰۰ سکه جمع کنید!</div>
    
    <div style="position: relative;">
        <div id="board"></div>
        <canvas id="particleCanvas"></canvas>
    </div>

    <div id="shop-container">
        <b style="color: #00f2fe; font-size:12px; width:100%; text-align:center;">🛒 فروشگاه کارت‌های حماسی:</b>
        <button class="card" onclick="buyEpicSpell('dragon')">🐉 احضار اژدها (۱۰۰ سکه)</button>
        <button class="card" onclick="buyEpicSpell('mind_control')">🔮 طلسم تسخیر مهره (۱۰۰ سکه)</button>
    </div>

    <script>
        const boardEl = document.getElementById('board');
        const statusEl = document.getElementById('status');
        const coinEl = document.getElementById('coin-count');
        const botCoinEl = document.getElementById('bot-coin-count');
        const turnEl = document.getElementById('turn-count');
        const pCanvas = document.getElementById('particleCanvas');
        const pCtx = pCanvas.getContext('2d');
        
        const COLS = 10; const ROWS = 16;
        let currentTurn = 'PLAYER'; let turnCount = 1;
        let selectedPieceId = null; let validMoves = []; let particles = [];
        let playerCoins = 40; let botCoins = 40;
        let destroyedRows = [];
        let activeSpellMode = null; // وضعیت انتخاب مهره حریف برای جادو

        let obstacles = []; let portals = [];
        // تولید رندوم موانع و پورتال‌ها در بخش میانی زمین بزرگ
        while(obstacles.length < 5) {
            let obs = { x: Math.floor(Math.random() * COLS), y: 7 + Math.floor(Math.random() * 2) };
            if (!obstacles.some(o => o.x === obs.x && o.y === obs.y)) obstacles.push(obs);
        }
        while(portals.length < 5) {
            let por = { x: Math.floor(Math.random() * COLS), y: 7 + Math.floor(Math.random() * 2) };
            if (!obstacles.some(o => o.x === por.x && o.y === por.y) && !portals.some(p => p.x === por.x && p.y === por.y)) portals.push(por);
        }

        // ارتش ۹ نفره جدید: ۱ شاه (۲ جان)، ۴ خطی، ۴ ضربدری برای هر طرف
        let gameBoard = {
            'p_king': { x: 5, y: 15, team: 'PLAYER', color: 'king', type: 'linear', isKing: true, hp: 2 },
            'p_l1': { x: 1, y: 15, team: 'PLAYER', color: 'white', type: 'linear', isKing: false },
            'p_l2': { x: 3, y: 15, team: 'PLAYER', color: 'white', type: 'linear', isKing: false },
            'p_l3': { x: 7, y: 15, team: 'PLAYER', color: 'white', type: 'linear', isKing: false },
            'p_l4': { x: 9, y: 15, team: 'PLAYER', color: 'white', type: 'linear', isKing: false },
            'p_d1': { x: 0, y: 14, team: 'PLAYER', color: 'black', type: 'diagonal', isKing: false },
            'p_d2': { x: 2, y: 14, team: 'PLAYER', color: 'black', type: 'diagonal', isKing: false },
            'p_d3': { x: 6, y: 14, team: 'PLAYER', color: 'black', type: 'diagonal', isKing: false },
            'p_d4': { x: 8, y: 14, team: 'PLAYER', color: 'black', type: 'diagonal', isKing: false },

            'b_king': { x: 4, y: 0, team: 'BOT', color: 'king', type: 'linear', isKing: true, hp: 2 },
            'b_l1': { x: 0, y: 0, team: 'BOT', color: 'red', type: 'linear', isKing: false },
            'b_l2': { x: 2, y: 0, team: 'BOT', color: 'red', type: 'linear', isKing: false },
            'b_l3': { x: 6, y: 0, team: 'BOT', color: 'red', type: 'linear', isKing: false },
            'b_l4': { x: 8, y: 0, team: 'BOT', color: 'red', type: 'linear', isKing: false },
            'b_d1': { x: 1, y: 1, team: 'BOT', color: 'blue', type: 'diagonal', isKing: false },
            'b_d2': { x: 3, y: 1, team: 'BOT', color: 'blue', type: 'diagonal', isKing: false },
            'b_d3': { x: 7, y: 1, team: 'BOT', color: 'blue', type: 'diagonal', isKing: false },
            'b_d4': { x: 9, y: 1, team: 'BOT', color: 'blue', type: 'diagonal', isKing: false }
        };

        setTimeout(() => { pCanvas.width = boardEl.clientWidth; pCanvas.height = boardEl.clientHeight; }, 100);

        function renderGrid() {
            boardEl.innerHTML = '';
            for (let y = 0; y < ROWS; y++) {
                for (let x = 0; x < COLS; x++) {
                    const cell = document.createElement('div');
                    cell.className = 'cell'; cell.id = `cell_${x}_${y}`;
                    if (destroyedRows.includes(y)) cell.classList.add('destroyed-row');
                    else {
                        if (obstacles.some(o => o.x === x && o.y === y)) cell.classList.add('obstacle');
                        if (portals.some(p => p.x === x && p.y === y)) cell.classList.add('portal');
                        cell.addEventListener('click', () => onCellClick(x, y));
                    }
                    boardEl.appendChild(cell);
                }
            }
        }
        renderGrid();

        function triggerScreenShake() { boardEl.classList.add('shake-effect'); setTimeout(() => boardEl.classList.remove('shake-effect'), 200); }
        
        function createExplosion(x, y, color, count=15) {
            let startX = x * 51 + 24; let startY = y * 51 + 24;
            for(let i=0; i<count; i++) {
                particles.push({ x: startX, y: startY, vx: (Math.random() - 0.5) * 6, vy: (Math.random() - 0.5) * 6, size: Math.random() * 3 + 1, color: color, alpha: 1 });
            }
        }

        function animateParticles() {
            pCtx.clearRect(0, 0, pCanvas.width, pCanvas.height);
            for(let i = particles.length - 1; i >= 0; i--) {
                let p = particles[i]; p.x += p.vx; p.y += p.vy; p.alpha -= 0.02;
                pCtx.save(); pCtx.globalAlpha = p.alpha; pCtx.beginPath(); pCtx.arc(p.x, p.y, p.size, 0, Math.PI*2); pCtx.fillStyle = p.color; pCtx.fill(); pCtx.restore();
                if(p.alpha <= 0) particles.splice(i, 1);
            }
            requestAnimationFrame(animateParticles);
        }
        animateParticles();

        function updateUI() {
            document.querySelectorAll('.cell').forEach(cell => {
                if(!cell.classList.contains('obstacle') && !cell.classList.contains('portal') && !cell.classList.contains('destroyed-row')) cell.innerHTML = '';
                cell.classList.remove('valid-move', 'valid-attack');
            });

            validMoves.forEach(m => {
                const cell = document.getElementById(`cell_${m.x}_${m.y}`);
                if (cell) cell.classList.add(m.isAttack ? 'valid-attack' : 'valid-move');
            });

            for (let id in gameBoard) {
                let p = gameBoard[id]; const cell = document.getElementById(`cell_${p.x}_${p.y}`);
                if (cell && !destroyedRows.includes(p.y)) {
                    const pieceEl = document.createElement('div');
                    let colorClass = p.color + '-p';
                    let typeName = p.isKing ? '👑 شاه' : (p.type === 'linear' ? '↕' : '❌');
                    
                    pieceEl.className = `piece ${colorClass}`;
                    if (id === selectedPieceId) pieceEl.className += ' selected';
                    pieceEl.innerText = typeName;
                    
                    if(p.isKing) {
                        const hpEl = document.createElement('div'); hpEl.className = 'hp-bar'; hpEl.innerText = 'HP: ' + p.hp; pieceEl.appendChild(hpEl);
                    }
                    cell.appendChild(pieceEl);
                }
            }
            coinEl.innerText = playerCoins; botCoinEl.innerText = botCoins; turnEl.innerText = turnCount;
            checkGameOver();
        }

        function getValidMoves(piece, boardState = gameBoard) {
            let moves = []; let directions = [];
            // شاه ضعیف است و فقط ۱ خانه در تمام جهات حرکت می‌کند
            if (piece.type === 'linear' || piece.isKing) directions = [[1,0], [-1,0], [0,1], [0,-1]];
            if (piece.type === 'diagonal' || piece.isKing) directions = directions.concat([[1,1], [-1,1], [1,-1], [-1,-1]]);

            directions.forEach(d => {
                let tx = piece.x + d[0]; let ty = piece.y + d[1];
                if (tx >= 0 && tx < COLS && ty >= 0 && ty < ROWS) {
                    if (obstacles.some(o => o.x === tx && o.y === ty) || destroyedRows.includes(ty)) return;
                    let targetPieceId = Object.keys(boardState).find(id => boardState[id].x === tx && boardState[id].y === ty);
                    if (!targetPieceId) { moves.push({ x: tx, y: ty, isAttack: false }); }
                    else if (boardState[targetPieceId].team !== piece.team) { moves.push({ x: tx, y: ty, isAttack: true, targetId: targetPieceId }); }
                }
            });
            return moves;
        }

        function buyEpicSpell(type) {
            if (currentTurn !== 'PLAYER') return;
            if (playerCoins < 100) { alert("برای فعال‌سازی جادوهای حماسی به ۱۰۰ سکه نیاز دارید!"); return; }
            activeSpellMode = type;
            statusEl.innerText = type === 'dragon' ? "🐉 اژدها آماده پرواز! یک مهره معمولی بات را انتخاب کنید تا ذوب شود." : "🔮 یک مهره معمولی بات را انتخاب کنید تا به ارتش شما ملحق شود!";
        }

        function handlePortal(piece, targetY) {
            if (portals.some(p => p.x === piece.x && p.y === targetY)) {
                let jump = piece.team === 'PLAYER' ? -4 : 4; let newY = targetY + jump;
                if (newY >= 0 && newY < ROWS && !obstacles.some(o => o.x === piece.x && o.y === newY) && !destroyedRows.includes(newY)) {
                    createExplosion(piece.x, targetY, '#8a2be2', 25); return newY;
                }
            }
            return targetY;
        }

        function onCellClick(x, y) {
            let clickedPieceId = Object.keys(gameBoard).find(id => gameBoard[id].x === x && gameBoard[id].y === y);

            // مدیریت جادوهای ۱۰۰ سکه‌ای بازیکن
            if (activeSpellMode && clickedPieceId) {
                let target = gameBoard[clickedPieceId];
                if (target.team !== 'BOT' || target.isKing) { alert("فقط می‌توانید مهره‌های معمولی بات را هدف قرار دهید!"); return; }

                playerCoins -= 100;
                triggerScreenShake();

                if (activeSpellMode === 'dragon') {
                    createExplosion(x, y, '#ff4500', 40);
                    delete gameBoard[clickedPieceId];
                    statusEl.innerText = "🐉 اژدها مهره حریف را سوزاند و به خاکستر تبدیل کرد!";
                } else if (activeSpellMode === 'mind_control') {
                    createExplosion(x, y, '#00f2fe', 40);
                    target.team = 'PLAYER';
                    target.color = target.type === 'linear' ? 'white' : 'black';
                    statusEl.innerText = "🔮 مهره حریف تسخیر شد و به ارتش شما پیوست!";
                }
                activeSpellMode = null; currentTurn = 'BOT'; updateUI();
                setTimeout(advancedBotAI, 600); return;
            }

            if (currentTurn !== 'PLAYER') return;

            if (clickedPieceId && gameBoard[clickedPieceId].team === 'PLAYER') {
                selectedPieceId = clickedPieceId; validMoves = getValidMoves(gameBoard[clickedPieceId]); updateUI();
            } else if (selectedPieceId) {
                let move = validMoves.find(m => m.x === x && m.y === y);
                if (move) {
                    if (move.isAttack) {
                        let deadPiece = gameBoard[move.targetId]; triggerScreenShake();
                        if (deadPiece.isKing) {
                            deadPiece.hp--; playerCoins += 35; createExplosion(deadPiece.x, deadPiece.y, '#ffd700', 30);
                            if(deadPiece.hp <= 0) delete gameBoard[move.targetId];
                        } else {
                            playerCoins += 15; createExplosion(deadPiece.x, deadPiece.y, '#da3633');
                            delete gameBoard[move.targetId];
                        }
                    } else {
                        playerCoins += 5; // هر حرکت موفق ۵ سکه جایزه دارد
                    }
                    
                    let piece = gameBoard[selectedPieceId]; piece.x = x; piece.y = handlePortal(piece, y);
                    selectedPieceId = null; validMoves = []; currentTurn = 'BOT';
                    statusEl.innerText = "بات در حال تحلیل تهدیدهای زمین بزرگ..."; updateUI();
                    setTimeout(advancedBotAI, 600);
                }
            }
        }

        function advancedBotAI() {
            // هوش مالی بات: اگر بات ۱۰۰ سکه داشته باشد، فوراً قوی‌ترین جادو را روی بازیکن می‌زند!
            if (botCoins >= 100) {
                let playerNormalPieces = Object.keys(gameBoard).filter(id => gameBoard[id].team === 'PLAYER' && !gameBoard[id].isKing);
                if (playerNormalPieces.length > 0) {
                    botCoins -= 100; triggerScreenShake();
                    let randomTargetId = playerNormalPieces[Math.floor(Math.random() * playerNormalPieces.length)];
                    let target = gameBoard[randomTargetId];
                    
                    if (Math.random() > 0.5) { // ۵۰٪ اژدها، ۵۰٪ تسخیر
                        createExplosion(target.x, target.y, '#ff4500', 40); delete gameBoard[randomTargetId];
                        statusEl.innerText = "🔥 غافلگیری! بات اژدها فرستاد و مهره شما را خاکستر کرد!";
                    } else {
                        createExplosion(target.x, target.y, '#da3633', 40);
                        target.team = 'BOT'; target.color = 'red';
                        statusEl.innerText = "🔮 شوم! بات یکی از مهره‌های شما را تسخیر کرد!";
                    }
                    turnCount++; currentTurn = 'PLAYER'; updateUI(); return;
                }
            }

            let botPieces = Object.keys(gameBoard).filter(id => gameBoard[id].team === 'BOT');
            if (botPieces.length === 0) return;

            let bestScore = -Infinity; let bestMove = null;

            botPieces.forEach(pieceId => {
                let p = gameBoard[pieceId]; let moves = getValidMoves(p, gameBoard);
                moves.forEach(move => {
                    let tempBoard = JSON.parse(JSON.stringify(gameBoard)); let finalY = move.y;
                    if (portals.some(pr => pr.x === move.x && pr.y === move.y)) { if (move.y + 4 < ROWS) finalY += 4; }
                    
                    if (move.isAttack) {
                        let target = tempBoard[move.targetId];
                        if(target.isKing) { target.hp--; if(target.hp<=0) delete tempBoard[move.targetId]; }
                        else delete tempBoard[move.targetId];
                    }
                    tempBoard[pieceId].x = move.x; tempBoard[pieceId].y = finalY;

                    let currentScore = evaluateBoard(tempBoard);
                    if (currentScore > bestScore) { bestScore = currentScore; bestMove = { pieceId: pieceId, move: move, finalY: finalY }; }
                });
            });

            if (bestMove) {
                let piece = gameBoard[bestMove.pieceId];
                if (bestMove.move.isAttack) {
                    let target = gameBoard[bestMove.move.targetId]; triggerScreenShake();
                    if(target.isKing) { target.hp--; botCoins += 35; if(target.hp <= 0) delete gameBoard[bestMove.move.targetId]; }
                    else { botCoins += 15; delete gameBoard[bestMove.move.targetId]; }
                    statusEl.innerText = "💥 بات به یکی از مهره‌های شما پاتک زد!";
                } else {
                    botCoins += 5; statusEl.innerText = "نوبت شماست؛ حرکت بعدی را برنامه‌ریزی کنید.";
                }
                piece.x = bestMove.move.x; piece.y = bestMove.finalY;
            }

            // قانون آخرالزمانی ریزش زمین (در نوبت ۲۰)
            if (turnCount === 20) { destroyedRows.push(0); destroyedRows.push(15); for(let id in gameBoard) { if(destroyedRows.includes(gameBoard[id].y)) delete gameBoard[id]; } renderGrid(); statusEl.innerText = "🚨 نوبت ۲۰! مرزهای بالا و پایین زمین سقوط کردند!"; }

            turnCount++; currentTurn = 'PLAYER'; updateUI();
        }

        function evaluateBoard(boardState) {
            let score = 0;
            for (let id in boardState) {
                let p = boardState[id]; if (destroyedRows.includes(p.y)) continue;
                let pieceValue = p.isKing ? (p.hp * 400) : 100;
                if (p.team === 'BOT') score += pieceValue + p.y * 10;
                else if (p.team === 'PLAYER') score -= (pieceValue + (15 - p.y) * 10);
            }
            return score + botCoins;
        }

        function checkGameOver() {
            if (!gameBoard['p_king'] || gameBoard['p_king'].hp <= 0) { statusEl.innerHTML = "<span style='color:#da3633; font-size: 20px;'>💀 شاه شما سقوط کرد! شکست خوردید.</span>"; currentTurn = 'GAMEOVER'; }
            else if (!gameBoard['b_king'] || gameBoard['b_king'].hp <= 0) { statusEl.innerHTML = "<span style='color:#00f2fe; font-size: 20px;'>🏆 شاهکار حماسی! شاه هوش مصنوعی را متلاشی کردید!</span>"; currentTurn = 'GAMEOVER'; }
        }

        updateUI();
    </script>
</body>
</html>
