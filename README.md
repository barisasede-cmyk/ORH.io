<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Blob Eater - Battle Royale</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #f0f0f0; font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; }
        #menu-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.85); display: flex; justify-content: center; align-items: center; z-index: 100; }
        #menu-box { background: white; padding: 40px; border-radius: 20px; text-align: center; width: 350px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        input[type="text"] { width: 80%; padding: 12px; margin: 20px 0; border: 2px solid #ddd; border-radius: 8px; font-size: 18px; }
        .color-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 10px; margin-bottom: 25px; }
        .color-option { width: 35px; height: 35px; border-radius: 50%; cursor: pointer; border: 3px solid transparent; }
        .color-option.selected { border-color: #333; transform: scale(1.1); }
        #play-btn { background: #2ecc71; color: white; border: none; padding: 15px 50px; font-size: 20px; font-weight: bold; border-radius: 30px; cursor: pointer; }
        #ui-layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; display: none; }
        #score-board { position: absolute; top: 20px; left: 20px; background: rgba(0, 0, 0, 0.7); color: white; padding: 10px 20px; border-radius: 8px; font-size: 18px; }
        #leaderboard { position: absolute; top: 20px; right: 20px; background: rgba(0, 0, 0, 0.6); color: white; padding: 15px; border-radius: 10px; width: 180px; font-size: 14px; }
        #leaderboard h3 { margin: 0 0 10px 0; text-align: center; border-bottom: 1px solid #aaa; }
        .lb-item { display: flex; justify-content: space-between; margin-bottom: 4px; }
        .lb-me { color: #f1c40f; font-weight: bold; }
        #minimap-container { position: absolute; bottom: 20px; right: 20px; width: 180px; height: 180px; background: rgba(0, 0, 0, 0.5); border: 2px solid white; border-radius: 5px; }
        #minimap { width: 100%; height: 100%; }
    </style>
</head>
<body>

    <div id="menu-overlay">
        <div id="menu-box">
            <h1>Blob Eater</h1>
            <input type="text" id="playerName" placeholder="Your Nickname" maxlength="12">
            <div class="color-grid" id="colorGrid"></div>
            <button id="play-btn">START BATTLE</button>
        </div>
    </div>

    <div id="ui-layer">
        <div id="score-board">Size: <span id="scoreText">25</span></div>
        <div id="leaderboard">
            <h3>Leaderboard</h3>
            <div id="lb-list"></div>
        </div>
        <div id="minimap-container"><canvas id="minimap"></canvas></div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const minimapCanvas = document.getElementById('minimap');
        const minimapCtx = minimapCanvas.getContext('2d');
        const scoreText = document.getElementById('scoreText');
        const lbList = document.getElementById('lb-list');
        const menuOverlay = document.getElementById('menu-overlay');
        const uiLayer = document.getElementById('ui-layer');

        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        minimapCanvas.width = 180;
        minimapCanvas.height = 180;

        const WORLD_WIDTH = 4000, WORLD_HEIGHT = 4000, FOOD_COUNT = 800, ENEMY_COUNT = 20;
        const COLORS = ['#e74c3c', '#f1c40f', '#9b59b6', '#2ecc71', '#e67e22', '#1abc9c', '#3498db', '#ff7675', '#74b9ff', '#55efc4', '#fd79a8', '#6c5ce7', '#fab1a0', '#00b894', '#d63031'];
        const BOT_NAMES = ["Alpha", "Shadow", "Hunter", "Titan", "Ghost", "Rex", "Storm", "Void", "Blaze", "Ninja", "Warrior", "King", "Bouncer", "Zinger", "Killer", "NoobMaster", "ProBlob", "SkyWalker", "Turbo", "Vortex"];

        let gameActive = false;
        let player = { x: 2000, y: 2000, radius: 25, color: COLORS[0], name: 'YOU', isBoosting: false };
        let foods = [], enemies = [], mouseX = 0, mouseY = 0;

        COLORS.forEach((c, i) => {
            const div = document.createElement('div');
            div.className = 'color-option' + (i === 0 ? ' selected' : '');
            div.style.backgroundColor = c;
            div.onclick = () => {
                document.querySelectorAll('.color-option').forEach(el => el.classList.remove('selected'));
                div.classList.add('selected');
                player.color = c;
            };
            document.getElementById('colorGrid').appendChild(div);
        });

        function random(min, max) { return Math.random() * (max - min) + min; }

        function initGame() {
            foods = Array.from({length: FOOD_COUNT}, () => ({
                x: random(0, WORLD_WIDTH), y: random(0, WORLD_HEIGHT),
                color: COLORS[Math.floor(random(0, COLORS.length))], radius: 6
            }));
            let shuffledNames = [...BOT_NAMES].sort(() => 0.5 - Math.random());
            enemies = Array.from({length: ENEMY_COUNT}, (_, i) => ({
                id: i, x: random(0, WORLD_WIDTH), y: random(0, WORLD_HEIGHT),
                radius: random(20, 70), color: COLORS[Math.floor(random(0, COLORS.length))],
                name: shuffledNames[i] || "Bot_" + i, targetX: random(0, WORLD_WIDTH), targetY: random(0, WORLD_HEIGHT)
            }));
        }

        window.addEventListener('mousemove', (e) => { mouseX = e.clientX; mouseY = e.clientY; });
        window.addEventListener('keydown', (e) => { if (e.key === 'Shift') player.isBoosting = true; });
        window.addEventListener('keyup', (e) => { if (e.key === 'Shift') player.isBoosting = false; });

        document.getElementById('play-btn').onclick = () => {
            const n = document.getElementById('playerName').value;
            if(n) player.name = n.toUpperCase();
            menuOverlay.style.display = 'none'; uiLayer.style.display = 'block';
            initGame(); gameActive = true;
        };

        function updateLeaderboard() {
            let all = [...enemies, player].sort((a, b) => b.radius - a.radius).slice(0, 10);
            lbList.innerHTML = all.map((entity, i) => `
                <div class="lb-item ${entity === player ? 'lb-me' : ''}">
                    <span>${i+1}. ${entity.name}</span>
                    <span>${Math.floor(entity.radius)}</span>
                </div>
            `).join('');
        }

        function updateEnemies() {
            enemies.forEach(e => {
                let speed = Math.max(1.1, 55 / e.radius);
                let perception = 600; 
                let targetEntity = null;

                // Kendinden küçük olan EN YAKIN varlığı bul (Oyuncu veya diğer botlar)
                let potentialTargets = [...enemies.filter(other => other.id !== e.id), player];
                
                potentialTargets.forEach(other => {
                    let d = Math.hypot(other.x - e.x, other.y - e.y);
                    if (d < perception) {
                        if (e.radius > other.radius * 1.1) {
                            // Avla!
                            e.targetX = other.x; e.targetY = other.y;
                            targetEntity = other;
                        } else if (other.radius > e.radius * 1.1) {
                            // Kaç!
                            e.targetX = e.x - (other.x - e.x);
                            e.targetY = e.y - (other.y - e.y);
                            targetEntity = other;
                        }
                    }
                });

                const dx = e.targetX - e.x, dy = e.targetY - e.y;
                const dist = Math.sqrt(dx*dx + dy*dy);
                if (dist < 20) {
                    e.targetX = random(0, WORLD_WIDTH); e.targetY = random(0, WORLD_HEIGHT);
                } else {
                    e.x += (dx/dist)*speed; e.y += (dy/dist)*speed;
                }

                e.x = Math.max(e.radius, Math.min(WORLD_WIDTH - e.radius, e.x));
                e.y = Math.max(e.radius, Math.min(WORLD_HEIGHT - e.radius, e.y));

                // Yemek yeme
                foods.forEach(f => {
                    if (Math.hypot(e.x-f.x, e.y-f.y) < e.radius) {
                        e.radius = Math.sqrt(e.radius**2 + 5**2);
                        f.x = random(0, WORLD_WIDTH); f.y = random(0, WORLD_HEIGHT);
                    }
                });
            });
        }

        function updatePlayer() {
            scoreText.innerText = Math.floor(player.radius);
            let speed = Math.max(1.4, 95 / player.radius);
            if (player.isBoosting && player.radius > 20) { speed *= 1.8; player.radius -= 0.05; }
            const dx = mouseX - (canvas.width/2), dy = mouseY - (canvas.height/2);
            const dist = Math.sqrt(dx*dx + dy*dy);
            if (dist > 5) { player.x += (dx/dist)*speed; player.y += (dy/dist)*speed; }
            player.x = Math.max(player.radius, Math.min(WORLD_WIDTH - player.radius, player.x));
            player.y = Math.max(player.radius, Math.min(WORLD_HEIGHT - player.radius, player.y));

            foods.forEach(f => {
                if (Math.hypot(player.x-f.x, player.y-f.y) < player.radius) {
                    player.radius = Math.sqrt(player.radius**2 + 10**2);
                    f.x = random(0, WORLD_WIDTH); f.y = random(0, WORLD_HEIGHT);
                }
            });
        }

        function checkEat() {
            // Oyuncu botu yer mi veya bot oyuncuyu yer mi?
            for (let i = enemies.length - 1; i >= 0; i--) {
                let e = enemies[i];
                let d = Math.hypot(player.x - e.x, player.y - e.y);
                if (d < player.radius && player.radius > e.radius * 1.1) {
                    player.radius = Math.sqrt(player.radius**2 + e.radius**2);
                    respawnEnemy(i);
                } else if (d < e.radius && e.radius > player.radius * 1.1) {
                    alert("GAME OVER! Eaten by " + e.name); location.reload();
                }
            }

            // Botlar birbirini yer mi? (En önemli kısım)
            for (let i = 0; i < enemies.length; i++) {
                for (let j = 0; j < enemies.length; j++) {
                    if (i === j) continue;
                    let e1 = enemies[i];
                    let e2 = enemies[j];
                    let d = Math.hypot(e1.x - e2.x, e1.y - e2.y);
                    
                    if (d < e1.radius && e1.radius > e2.radius * 1.1) {
                        e1.radius = Math.sqrt(e1.radius**2 + e2.radius**2);
                        respawnEnemy(j);
                    }
                }
            }
        }

        function respawnEnemy(index) {
            let names = [...BOT_NAMES];
            enemies[index].radius = random(20, 50);
            enemies[index].x = random(0, WORLD_WIDTH);
            enemies[index].y = random(0, WORLD_HEIGHT);
            enemies[index].name = names[Math.floor(random(0, names.length))];
        }

        function draw() {
            if (!gameActive) return;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.save();
            ctx.translate(canvas.width/2 - player.x, canvas.height/2 - player.y);
            ctx.strokeStyle = '#ddd';
            for(let i=0; i<=WORLD_WIDTH; i+=100) {
                ctx.beginPath(); ctx.moveTo(i, 0); ctx.lineTo(i, WORLD_HEIGHT); ctx.stroke();
                ctx.beginPath(); ctx.moveTo(0, i); ctx.lineTo(WORLD_WIDTH, i); ctx.stroke();
            }
            foods.forEach(f => {
                ctx.beginPath(); ctx.arc(f.x, f.y, f.radius, 0, Math.PI*2);
                ctx.fillStyle = f.color; ctx.fill();
            });
            enemies.forEach(e => {
                ctx.beginPath(); ctx.arc(e.x, e.y, e.radius, 0, Math.PI*2);
                ctx.fillStyle = e.color; ctx.fill();
                ctx.strokeStyle = '#000'; ctx.lineWidth = 2; ctx.stroke();
                ctx.fillStyle = "white"; ctx.font = "bold 12px Arial"; ctx.textAlign = "center";
                ctx.fillText(e.name, e.x, e.y + 5);
            });
            ctx.beginPath(); ctx.arc(player.x, player.y, player.radius, 0, Math.PI*2);
            ctx.fillStyle = player.color; ctx.fill();
            ctx.strokeStyle = '#000'; ctx.lineWidth = 3; ctx.stroke();
            ctx.fillStyle = "white"; ctx.font = "bold 14px Arial"; ctx.textAlign = "center";
            ctx.fillText(player.name, player.x, player.y + 5);
            ctx.restore();

            minimapCtx.clearRect(0, 0, 180, 180);
            enemies.forEach(e => {
                minimapCtx.fillStyle = "red";
                minimapCtx.fillRect(e.x/22.2, e.y/22.2, 3, 3);
            });
            minimapCtx.fillStyle = player.color;
            minimapCtx.fillRect(player.x/22.2, player.y/22.2, 5, 5);
        }

        function gameLoop() {
            if(gameActive) {
                updatePlayer(); updateEnemies(); checkEat(); updateLeaderboard(); draw();
            }
            requestAnimationFrame(gameLoop);
        }
        gameLoop();
    </script>
</body>
</html>
