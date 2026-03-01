# - 游戏
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>星辰轨迹 · 太空避障</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
        }
        body {
            margin: 0;
            min-height: 100vh;
            background: radial-gradient(circle at 20% 30%, #0a0f2e, #030514);
            display: flex;
            align-items: center;
            justify-content: center;
            font-family: 'Segoe UI', system-ui, monospace;
        }
        .game-container {
            background: #0b0e1a;
            border-radius: 2.5rem;
            padding: 1.5rem;
            box-shadow: 0 20px 40px rgba(0, 10, 30, 0.8), 0 0 0 2px #2e3b4e inset;
        }
        canvas {
            display: block;
            width: 800px;
            height: 500px;
            border-radius: 1.8rem;
            background: #030614;
            box-shadow: 0 0 30px #3f5f9f66;
            cursor: none;
        }
        .status-bar {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 1rem;
            padding: 0.5rem 1.8rem;
            background: #131a2c;
            border-radius: 3rem;
            color: #b7e1fa;
            font-weight: 600;
            text-shadow: 0 0 5px #3ea0ff;
            border: 1px solid #2c405c;
            letter-spacing: 1px;
        }
        .score-box {
            background: #0b101f;
            padding: 0.3rem 1.4rem;
            border-radius: 3rem;
            font-size: 1.7rem;
            font-family: 'Courier New', monospace;
            color: #f5e56b;
            box-shadow: 0 0 10px #fabc2c;
        }
        button {
            background: #1e2b41;
            border: none;
            color: white;
            font-size: 1.2rem;
            font-weight: bold;
            padding: 0.5rem 2rem;
            border-radius: 3rem;
            cursor: pointer;
            box-shadow: 0 5px 0 #0b121f, 0 0 10px #5899dd;
            transition: 0.1s ease;
            font-family: inherit;
            letter-spacing: 1px;
        }
        button:hover {
            background: #264663;
            transform: translateY(-2px);
            box-shadow: 0 7px 0 #0b121f, 0 0 18px #7bb9fe;
        }
        button:active {
            transform: translateY(5px);
            box-shadow: 0 2px 0 #0b121f, 0 0 15px #7bb9fe;
        }
        .hint {
            display: flex;
            gap: 2rem;
            align-items: center;
            color: #7b96c5;
            font-size: 1rem;
        }
        .hint i {
            font-style: normal;
            display: inline-block;
            width: 12px;
            height: 12px;
            background: #a8d5ff;
            border-radius: 50%;
            box-shadow: 0 0 12px cyan;
            margin-right: 6px;
        }
    </style>
</head>
<body>
<div class="game-container">
    <canvas id="gameCanvas" width="800" height="500"></canvas>
    <div class="status-bar">
        <div class="hint">
            <span><i></i> 鼠标移动 · 躲开流星</span>
        </div>
        <div class="score-box" id="scoreDisplay">0000</div>
        <button id="restartBtn">🔄 重启航行</button>
    </div>
</div>

<script>
    (function() {
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreSpan = document.getElementById('scoreDisplay');

        // 固定尺寸参数
        const CW = 800, CH = 500;

        // ---------- 飞船参数 ----------
        const SHIP_RADIUS = 16;          // 飞船圆形碰撞半径
        let shipX = CW * 0.5, shipY = CH * 0.8;   // 初始位置靠下
        let targetX = shipX, targetY = shipY;     // 鼠标目标平滑移动

        // ---------- 障碍物(小行星) ----------
        let obstacles = [];
        const MAX_OBSTACLES = 22;
        const OBSTACLE_RADIUS = 14;       // 小行星半径
        const BASE_SPEED = 2.2;            // 下落基础速度

        // ---------- 游戏状态 ----------
        let score = 0;
        let gameActive = true;

        // 帧率控制 & 动态难度 (速度随分数上升)
        let currentSpeed = BASE_SPEED;
        const SPEED_CAP = 7.0;

        // 鼠标追踪 (canvas内才更新)
        let mouseInside = true;    // 默认内部 (刚加载还未移出)
        canvas.addEventListener('mouseenter', () => mouseInside = true);
        canvas.addEventListener('mouseleave', () => mouseInside = false);

        // 鼠标移动 -> 更新目标位置 (限制边界，防止飞船越界)
        canvas.addEventListener('mousemove', (e) => {
            if (!gameActive) return;  // 游戏结束不响应鼠标移动，但依旧可以记录用于重启后？干脆直接限制
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;

            // 计算canvas坐标系内的坐标
            let rawX = (e.clientX - rect.left) * scaleX;
            let rawY = (e.clientY - rect.top) * scaleY;

            // 边界限制 (不让飞船中心过于靠近边缘，保留一些视野)
            const minX = SHIP_RADIUS + 4;
            const maxX = CW - SHIP_RADIUS - 4;
            const minY = SHIP_RADIUS + 4;
            const maxY = CH - SHIP_RADIUS - 4;
            targetX = Math.min(maxX, Math.max(minX, rawX));
            targetY = Math.min(maxY, Math.max(minY, rawY));
        });

        // 重启游戏
        function restartGame() {
            gameActive = true;
            score = 0;
            currentSpeed = BASE_SPEED;
            obstacles = [];
            // 重置飞船位置为中间偏下，目标同步防止突变
            shipX = CW * 0.5;
            shipY = CH * 0.8;
            targetX = shipX;
            targetY = shipY;
            updateScoreDisplay();
        }

        document.getElementById('restartBtn').addEventListener('click', () => {
            restartGame();
        });

        // 辅助：更新分数显示 (四位数补零)
        function updateScoreDisplay() {
            let str = String(Math.floor(score));
            while (str.length < 4) str = '0' + str;
            scoreSpan.innerText = str;
        }

        // 生成一个新的小行星 (确保不会和飞船重叠出生，但概率很低，加一点判断)
        function spawnObstacle() {
            if (!gameActive && obstacles.length >= MAX_OBSTACLES) return; // 游戏结束不再增加，但为了整洁可限制

            let tries = 0;
            const maxTries = 30;
            let newX, newY, safe = false;

            while (!safe && tries < maxTries) {
                newX = 30 + Math.random() * (CW - 60);   // 不贴边
                newY = -25 - Math.random() * 40;         // 从顶外出现
                // 避免出生时直接和飞船重叠 (几乎不会，因为飞船在下半区，但新游戏瞬间可能)
                const dx = newX - shipX;
                const dy = newY - shipY;
                const dist = Math.sqrt(dx*dx + dy*dy);
                if (dist > SHIP_RADIUS + OBSTACLE_RADIUS + 20) safe = true;
                tries++;
            }
            // 即使重叠也生成，概率极低，碰撞检测会在下一帧处理
            obstacles.push({
                x: newX,
                y: newY,
                r: OBSTACLE_RADIUS * (0.8 + Math.random() * 0.5) // 略微变化大小 0.8~1.3倍
            });
        }

        // 更新障碍物位置，碰撞检测，分数与速度
        function updateGame() {
            if (!gameActive) return;

            // 1. 飞船平滑跟随鼠标 (缓动效果，手感更柔和)
            shipX += (targetX - shipX) * 0.25;
            shipY += (targetY - shipY) * 0.2;
            // 二次边界保护 (由于缓动可能短暂溢出, 再钳位一下)
            shipX = Math.min(CW - SHIP_RADIUS - 2, Math.max(SHIP_RADIUS + 2, shipX));
            shipY = Math.min(CH - SHIP_RADIUS - 2, Math.max(SHIP_RADIUS + 2, shipY));

            // 2. 动态速度：分数每增加30，速度提升0.15，有上限
            currentSpeed = Math.min(SPEED_CAP, BASE_SPEED + Math.floor(score / 30) * 0.4);

            // 3. 移动所有小行星 + 碰撞检测/越界移除
            for (let i = obstacles.length - 1; i >= 0; i--) {
                const obs = obstacles[i];
                // 下落
                obs.y += currentSpeed;

                // 碰撞检测 (圆心距离)
                const dx = obs.x - shipX;
                const dy = obs.y - shipY;
                const dist = Math.sqrt(dx*dx + dy*dy);
                if (dist < SHIP_RADIUS + obs.r) {
                    gameActive = false;   // 游戏结束
                    // 可以立即返回，停止更新 (但为了让画面冻结显示碰撞瞬间，直接break循环)
                    // 这里直接设置 gameActive = false，后面的绘制会显示结束画面
                    return;  // 停止更新，不再生成新障碍或加分
                }

                // 超出屏幕底部 或 完全消失在视野外 (加分并移除)
                if (obs.y - obs.r > CH + 40) {
                    obstacles.splice(i, 1);
                    if (gameActive) {   // 只有在活跃时加分
                        score += 5;
                        updateScoreDisplay();
                    }
                }
            }

            // 4. 根据当前障碍数量生成新障碍 (控制数量)
            if (obstacles.length < MAX_OBSTACLES && gameActive) {
                // 随机概率：每帧大约 6% 几率生成一个，避免暴增
                if (Math.random() < 0.07) {
                    spawnObstacle();
                }
                // 偶尔一次生成两个? 不用，保持稀疏度
            }

            // 分数显示刷新 (可能因加分变化)
            updateScoreDisplay();
        }

        // ---------- 绘制图形 ----------
        function draw() {
            ctx.clearRect(0, 0, CW, CH);

            // ---- 深邃星空背景 ----
            ctx.fillStyle = '#030614';
            ctx.fillRect(0, 0, CW, CH);
            // 星星 (随机闪烁)
            for (let i = 0; i < 150; i++) {
                if (i % 2 === 0) continue; // 伪随机减少绘制量
                const x = (i * 23) % CW, y = (i * 17) % CH;
                const brightness = Math.sin(Date.now() * 0.002 + i) * 0.3 + 0.7;
                ctx.fillStyle = `rgba(255, 240, 200, ${brightness*0.6})`;
                ctx.beginPath();
                ctx.arc(x, y, 1.2, 0, Math.PI*2);
                ctx.fill();
            }

            // ---- 小行星带 (障碍物) ----
            for (let obs of obstacles) {
                // 石块立体感
                const gradient = ctx.createRadialGradient(obs.x-4, obs.y-4, 3, obs.x, obs.y, obs.r+2);
                gradient.addColorStop(0, '#7c6e5e');
                gradient.addColorStop(0.7, '#3a2e24');
                gradient.addColorStop(1, '#1f1a14');
                ctx.beginPath();
                ctx.arc(obs.x, obs.y, obs.r, 0, Math.PI*2);
                ctx.fillStyle = gradient;
                ctx.fill();
                // 高光
                ctx.shadowColor = '#b57c48';
                ctx.shadowBlur = 12;
                ctx.beginPath();
                ctx.arc(obs.x-2, obs.y-2, obs.r*0.25, 0, Math.PI*2);
                ctx.fillStyle = '#ad9e8b';
                ctx.fill();
                ctx.shadowBlur = 0;
                ctx.shadowColor = 'transparent';
                // 坑洼小点
                ctx.fillStyle = '#261e16';
                ctx.beginPath();
                ctx.arc(obs.x+3, obs.y-1, 3, 0, 2*Math.PI);
                ctx.fill();
            }

            // ---- 飞船 (主角) - 星辰轨迹 ----
            // 引擎火焰 (动态摇曳)
            if (gameActive) {
            const flame = Math.sin(Date.now() * 0.02) * 2 + 6;
            ctx.shadowColor = '#ffb347';
            ctx.shadowBlur = 20;
            ctx.beginPath();
            ctx.moveTo(shipX - 10, shipY + 12);
            ctx.lineTo(shipX, shipY + 26 + flame);
            ctx.lineTo(shipX + 10, shipY + 12);
            ctx.fillStyle = '#f39c12';
            ctx.fill();
            }
            // 主体: 菱形飞船 (类X翼)
            ctx.shadowColor = '#3ec1ff';
            ctx.shadowBlur = 24;

            // 中心驾驶舱
            ctx.beginPath();
            ctx.ellipse(shipX, shipY-2, 8, 12, 0, 0, Math.PI*2);
            ctx.fillStyle = '#8ecae6';
            ctx.fill();
            // 机翼 / 太阳能板
            ctx.fillStyle = '#5296c5';
            ctx.shadowBlur = 18;
            ctx.beginPath();
            ctx.moveTo(shipX - 19, shipY - 5);
            ctx.lineTo(shipX - 10, shipY - 18);
            ctx.lineTo(shipX - 10, shipY + 2);
            ctx.closePath();
            ctx.fill();
            ctx.beginPath();
            ctx.moveTo(shipX + 19, shipY - 5);
            ctx.lineTo(shipX + 10, shipY - 18);
            ctx.lineTo(shipX + 10, shipY + 2);
            ctx.closePath();
            ctx.fill();

            // 中心光泽
            ctx.shadowBlur = 12;
            ctx.beginPath();
            ctx.arc(shipX-2, shipY-6, 4, 0, 2*Math.PI);
            ctx.fillStyle = '#caf0f8';
            ctx.fill();

            // 驾驶舱光点
            ctx.shadowBlur = 16;
            ctx.beginPath();
            ctx.arc(shipX, shipY-8, 2.5, 0, 2*Math.PI);
            ctx.fillStyle = '#f8f3e0';
            ctx.fill();

            // 重置阴影
            ctx.shadowBlur = 0;
            ctx.shadowColor = 'transparent';

            // 碰撞提示 (如果游戏结束)
            if (!gameActive) {
                ctx.fillStyle = 'rgba(30, 10, 30, 0.7)';
                ctx.fillRect(0, 0, CW, CH);
                ctx.font = 'bold 40px "Segoe UI", "Courier New", monospace';
                ctx.fillStyle = '#fbbf7c';
                ctx.shadowColor = '#ff4444';
                ctx.shadowBlur = 20;
                ctx.fillText('💥 坠毁', 260, 220);
                ctx.font = '24px monospace';
                ctx.fillStyle = '#b0daf0';
                ctx.fillText('点击"重启航行"', 300, 320);
                ctx.shadowBlur = 0;

                // 同时显示最终分数
                ctx.font = '28px "Courier New"';
                ctx.fillStyle = '#f5e56b';
                ctx.fillText('得分: ' + Math.floor(score), 330, 400);
            }

            // 边缘提示 (鼠标离开画布时)
            if (!mouseInside && gameActive) {
                ctx.fillStyle = 'rgba(0,0,0,0.5)';
                ctx.fillRect(0, 0, CW, 60);
                ctx.font = '18px monospace';
                ctx.fillStyle = '#ffe68f';
                ctx.fillText('⬆️ 鼠标移回画布控制飞船 ⬆️', 240, 40);
            }

            // 显示当前速度 (调试感，但增加氛围)
            ctx.font = '14px monospace';
            ctx.fillStyle = '#7aa5c2';
            ctx.fillText('⚡' + currentSpeed.toFixed(1) + '节', 700, 40);
        }

        // 游戏循环
        function gameLoop() {
            if (gameActive) {
                updateGame();
            }
            draw();
            requestAnimationFrame(gameLoop);
        }

        // 启动循环
        gameLoop();

        // 初始生成几个障碍物，让游戏不空旷 (但避免太多)
        for (let i = 0; i < 6; i++) {
            spawnObstacle();
        }

        // 窗口失去焦点时鼠标事件可能不正常，但无所谓。额外保证画布内鼠标可用
        // 触摸设备简单忽略——仅鼠标控制，但保留提示
        // 补充: 如果没有鼠标（触摸板）也可能工作，但移动端不好操作，不过代码是可行的。
    })();
</script>
</body>
</html>
