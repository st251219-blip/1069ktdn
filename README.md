<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Smash Clone</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #111;
            color: #fff;
            font-family: 'Arial Black', Impact, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            user-select: none;
        }

        #game-wrapper {
            position: relative;
            box-shadow: 0 0 30px rgba(0, 0, 0, 1);
            border: 4px solid #444;
            border-radius: 8px;
            background: linear-gradient(to bottom, #1e3c72 0%, #2a5298 100%);
        }

        canvas {
            display: block;
            width: 800px;
            height: 500px;
            object-fit: contain;
        }

        /* UIオーバーレイ共通設定 */
        .overlay {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.8);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 10;
        }

        h1 {
            font-size: 64px;
            color: #ffcc00;
            text-shadow: 4px 4px 0px #d32f2f;
            letter-spacing: 4px;
            margin-bottom: 40px;
        }

        .controls-info {
            display: flex;
            gap: 50px;
            margin-bottom: 40px;
            background: rgba(255,255,255,0.1);
            padding: 20px;
            border-radius: 10px;
            font-family: sans-serif;
            font-weight: bold;
        }

        .p1-color { color: #ff5252; }
        .p2-color { color: #448aff; }

        kbd {
            background-color: #eee;
            border-radius: 4px;
            border: 1px solid #b4b4b4;
            box-shadow: 0 2px 0px rgba(0,0,0,.2), 0 2px 0 0 rgba(255,255,255,.7) inset;
            color: #333;
            display: inline-block;
            font-size: 14px;
            padding: 4px 8px;
            margin: 0 2px;
        }

        .button-group {
            display: flex;
            gap: 20px;
        }

        button {
            padding: 15px 40px;
            font-size: 24px;
            font-weight: bold;
            font-family: inherit;
            background-color: #ffcc00;
            color: #000;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            box-shadow: 0 6px 0 #cc9900;
            transition: transform 0.1s, box-shadow 0.1s;
        }

        button:active {
            transform: translateY(6px);
            box-shadow: 0 0 0 #cc9900;
        }

        #winnerText {
            font-size: 72px;
            margin-bottom: 30px;
            text-shadow: 4px 4px 0 #000;
        }
    </style>
</head>
<body>

<div id="game-wrapper">
    <canvas id="gameCanvas" width="800" height="500"></canvas>
    
    <!-- スタート画面 -->
    <div id="start-screen" class="overlay">
        <h1>SMASH CLONE</h1>
        <div class="controls-info">
            <div class="p1-color">
                <h3>1P (RED) 操作</h3>
                移動: <kbd>A</kbd> <kbd>D</kbd><br><br>
                ジャンプ: <kbd>W</kbd> (2段可)<br><br>
                攻撃(弱): 止まって <kbd>F</kbd><br><br>
                攻撃(強): 歩きながら <kbd>F</kbd>
            </div>
            <div class="p2-color">
                <h3>2P (BLUE) 操作</h3>
                移動: <kbd>←</kbd> <kbd>→</kbd><br><br>
                ジャンプ: <kbd>↑</kbd> (2段可)<br><br>
                攻撃(弱): 止まって <kbd>/</kbd><br><br>
                攻撃(強): 歩きながら <kbd>/</kbd>
            </div>
        </div>
        <div class="button-group">
            <button onclick="startGame('cpu')">1P vs CPU</button>
            <button onclick="startGame('p2')">1P vs 2P</button>
        </div>
    </div>

    <!-- ゲームオーバー画面 -->
    <div id="game-over" class="overlay" style="display: none;">
        <div id="winnerText">PLAYER 1 WINS!</div>
        <button onclick="showStartScreen()">タイトルへ戻る</button>
    </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// --- キー入力管理 ---
const keys = {};
const prevKeys = {};

window.addEventListener('keydown', e => keys[e.code] = true);
window.addEventListener('keyup', e => keys[e.code] = false);

// 押した瞬間だけ判定する（ジャンプや攻撃用）
function isKeyJustPressed(code) {
    return keys[code] && !prevKeys[code];
}

// --- グローバル変数 ---
let particles = [];
let screenShake = 0;
let hitStop = 0; // 攻撃が当たった瞬間の硬直（打撃感の演出）
let gameMode = 'cpu'; // 'cpu' or 'p2'
let isPlaying = false;

// 物理演算定数
const GRAVITY = 0.6;
const FRICTION_GROUND = 0.8;
const FRICTION_AIR = 0.95;
const MAX_FALL_SPEED = 16;
// 撃墜（バースト）になる画面外の境界線
const BOUNDS = { top: -200, bottom: 600, left: -150, right: 950 };

// ステージの足場定義
const platforms = [
    { x: 150, y: 350, w: 500, h: 40, type: 'solid', color: '#555' }, // メイン土台
    { x: 230, y: 240, w: 100, h: 10, type: 'jumpthrough', color: '#888' }, // 左の浮遊足場
    { x: 470, y: 240, w: 100, h: 10, type: 'jumpthrough', color: '#888' }, // 右の浮遊足場
    { x: 350, y: 140, w: 100, h: 10, type: 'jumpthrough', color: '#888' }  // 上の浮遊足場
];

// --- エフェクト関連 ---
class Particle {
    constructor(x, y, vx, vy, life, color, size) {
        this.x = x;
        this.y = y;
        this.vx = vx;
        this.vy = vy;
        this.life = life;
        this.maxLife = life;
        this.color = color;
        this.size = size;
    }

    update() {
        this.x += this.vx;
        this.y += this.vy;
        this.life--;
        this.size *= 0.95; // だんだん小さくなる
    }

    draw(ctx) {
        ctx.globalAlpha = Math.max(0, this.life / this.maxLife);
        ctx.fillStyle = this.color;
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
        ctx.fill();
        ctx.globalAlpha = 1.0;
    }
}

// 攻撃ヒット時の火花エフェクト
function createHitEffect(x, y, isStrong) {
    let count = isStrong ? 25 : 10;
    let color = isStrong ? '#f44336' : '#ffeb3b';
    for (let i = 0; i < count; i++) {
        let angle = Math.random() * Math.PI * 2;
        let speed = Math.random() * (isStrong ? 12 : 6) + 2;
        particles.push(new Particle(x, y, Math.cos(angle)*speed, Math.sin(angle)*speed, 20, color, isStrong ? 8 : 4));
    }
}

// 撃墜時の大爆発エフェクト
function createBurstEffect(x, y, color) {
    x = Math.max(0, Math.min(x, canvas.width));
    y = Math.max(0, Math.min(y, canvas.height));
    for (let i = 0; i < 60; i++) {
        let angle = Math.random() * Math.PI * 2;
        let speed = Math.random() * 20 + 5;
        particles.push(new Particle(x, y, Math.cos(angle)*speed, Math.sin(angle)*speed, 45, color, 10));
        particles.push(new Particle(x, y, Math.cos(angle)*speed*0.6, Math.sin(angle)*speed*0.6, 35, '#fff', 5));
    }
}

// 矩形の当たり判定
function rectIntersect(r1, r2) {
    return !(r2.x > r1.x + r1.w || 
             r2.x + r2.w < r1.x || 
             r2.y > r1.y + r1.h || 
             r2.y + r2.h < r1.y);
}

// --- プレイヤークラス ---
class Player {
    constructor(id, x, y, color, controls) {
        this.id = id;
        this.w = 40;
        this.h = 40;
        this.spawnX = x;
        this.spawnY = y;
        this.x = x;
        this.y = y;
        this.vx = 0;
        this.vy = 0;
        this.prevY = y;
        this.color = color;
        this.controls = controls;
        
        this.damage = 0; // %ダメージ
        this.lives = 3;  // 残機
        this.isGrounded = false;
        this.facingRight = (id === 1);
        
        this.jumps = 0;
        this.maxJumps = 2; // 2段ジャンプ
        this.jumpForce = 14;
        this.speed = 1.5;
        this.maxSpeed = 8;
        
        this.attackFrame = 0;
        this.isAttacking = false;
        this.isStrongAttack = false;
        this.attackCooldown = 0;
        this.hitbox = null;
        this.hasHit = false;
        
        this.hitStun = 0; // ふっとび・硬直時間
        this.invincible = 0; // 復活時の無敵時間
    }

    respawn() {
        this.damage = 0;
        this.x = this.spawnX;
        this.y = 50; // 上空から復帰
        this.vx = 0;
        this.vy = 0;
        this.hitStun = 0;
        this.isAttacking = false;
        this.invincible = 120; // 約2秒間無敵
    }

    update(opponents) {
        if (this.lives <= 0) return;
        
        if (this.invincible > 0) this.invincible--;
        if (this.attackCooldown > 0) this.attackCooldown--;
        
        this.prevY = this.y;

        // --- 入力と移動 ---
        if (this.hitStun > 0) {
            // ふっとび中は操作不能、空気抵抗で減速
            this.hitStun--;
            this.vx *= 0.98;
        } else {
            // 左右移動
            if (this.controls.left && keys[this.controls.left]) {
                this.vx -= this.speed;
                if (!this.isAttacking) this.facingRight = false;
            }
            if (this.controls.right && keys[this.controls.right]) {
                this.vx += this.speed;
                if (!this.isAttacking) this.facingRight = true;
            }

            // 速度制限
            if (Math.abs(this.vx) > this.maxSpeed) {
                this.vx = Math.sign(this.vx) * this.maxSpeed;
            }

            // ジャンプ (押した瞬間)
            if (this.controls.jump && isKeyJustPressed(this.controls.jump)) {
                this.doJump();
            }

            // 攻撃 (押した瞬間)
            if (this.controls.attack && isKeyJustPressed(this.controls.attack) && this.attackCooldown === 0 && !this.isAttacking) {
                // 左右キーの入力状態を見て強攻撃か判定する
                let isMoving = (this.controls.left && keys[this.controls.left]) || (this.controls.right && keys[this.controls.right]);
                this.doAttack(isMoving);
            }
        }

        // --- 摩擦と重力 ---
        let noHorizontalInput = (!this.controls.left || !keys[this.controls.left]) && 
                                (!this.controls.right || !keys[this.controls.right]);
                                
        if (noHorizontalInput && this.hitStun === 0) {
            this.vx *= this.isGrounded ? FRICTION_GROUND : FRICTION_AIR;
            if (Math.abs(this.vx) < 0.1) this.vx = 0;
        }
        
        this.vy += GRAVITY;
        if (this.vy > MAX_FALL_SPEED) this.vy = MAX_FALL_SPEED;

        // --- 攻撃判定の処理 ---
        if (this.isAttacking) {
            this.attackFrame--;
            
            // 攻撃発生フレーム（強・弱でタイミングと持続を変える）
            let isHitboxActive = false;
            if (this.isStrongAttack) {
                isHitboxActive = (this.attackFrame > 15 && this.attackFrame < 22); // 発生少し遅め、判定やや長め
            } else {
                isHitboxActive = (this.attackFrame > 8 && this.attackFrame < 13);  // 発生早い、判定短め
            }
            
            if (isHitboxActive) {
                let hw = this.isStrongAttack ? 60 : 40; // 強攻撃は判定が広い
                let hx = this.facingRight ? this.x + this.w : this.x - hw;
                let hy = this.y + 10;
                this.hitbox = { x: hx, y: hy, w: hw, h: 25 };

                // 相手に当たるかチェック
                if (!this.hasHit) {
                    opponents.forEach(opp => {
                        if (opp.lives > 0 && opp.invincible === 0 && rectIntersect(this.hitbox, opp)) {
                            this.hasHit = true;
                            
                            // 基本ダメージとふっとばし力の計算
                            let baseDmg, baseKbX, baseKbY;
                            if (this.isStrongAttack) {
                                baseDmg = 10 + Math.floor(Math.random() * 5); // 強: 10〜14%
                                baseKbX = 6;  // ふっ飛びの初速を控えめに
                                baseKbY = 6;
                            } else {
                                baseDmg = 2 + Math.floor(Math.random() * 3);  // 弱: 2〜4%
                                baseKbX = 2.5;
                                baseKbY = 3;
                            }
                            
                            opp.damage += baseDmg;
                            
                            // 50%程度で撃墜できるよう、ダメージによる乗数カーブを調整
                            let kbMultiplier = 1 + (opp.damage * 0.06); 
                            
                            opp.vx = (this.facingRight ? 1 : -1) * (baseKbX * kbMultiplier);
                            opp.vy = -(baseKbY + (opp.damage * 0.04));
                            
                            // 相手を硬直させる
                            opp.hitStun = (this.isStrongAttack ? 20 : 10) + Math.floor(opp.damage * 0.25);
                            
                            // ゲームフィールのためのヒットストップと画面揺れ
                            hitStop = Math.min(12, Math.floor(kbMultiplier * (this.isStrongAttack ? 3 : 1)));
                            screenShake = Math.min(30, Math.floor(kbMultiplier * (this.isStrongAttack ? 6 : 2)));
                            
                            createHitEffect(this.hitbox.x + hw/2, this.hitbox.y + 10, this.isStrongAttack);
                        }
                    });
                }
            } else {
                this.hitbox = null; // 判定消失
            }

            // 攻撃終了
            if (this.attackFrame <= 0) {
                this.isAttacking = false;
                this.attackCooldown = this.isStrongAttack ? 20 : 8; // 強攻撃は隙が大きい
            }
        }

        // --- X軸の移動と壁判定 ---
        this.x += this.vx;
        for (let plat of platforms) {
            if (plat.type === 'solid' && rectIntersect(this, plat)) {
                // 上から乗っている、または下から頭を擦っている状態は壁(X軸)として押し戻さない
                if (this.prevY + this.h <= plat.y + 12 || this.prevY >= plat.y + plat.h - 12) {
                    continue;
                }

                if (this.vx > 0) this.x = plat.x - this.w;
                else if (this.vx < 0) this.x = plat.x + plat.w;
                this.vx = 0;
            }
        }

        // --- Y軸の移動と床判定 ---
        this.y += this.vy;
        this.isGrounded = false;
        
        for (let plat of platforms) {
            if (plat.type === 'solid') {
                if (rectIntersect(this, plat)) {
                    if (this.vy > 0 && this.prevY + this.h <= plat.y + 10) { // 上から着地
                        this.y = plat.y - this.h;
                        this.isGrounded = true;
                        this.vy = 0;
                        this.jumps = 0;
                    } else if (this.vy < 0) { // 下から頭をぶつける
                        this.y = plat.y + plat.h;
                        this.vy = 0;
                    }
                }
            } else if (plat.type === 'jumpthrough') {
                // すり抜け床 (下から上へは突き抜ける、上から乗る)
                if (this.vy >= 0 && this.prevY + this.h <= plat.y && this.y + this.h >= plat.y) {
                    if (this.x < plat.x + plat.w && this.x + this.w > plat.x) {
                        // 下キー入力ですり抜けて降りる処理 (キー設定がある場合)
                        if (this.controls.down && keys[this.controls.down]) {
                            // すり抜ける
                        } else {
                            this.y = plat.y - this.h;
                            this.isGrounded = true;
                            this.vy = 0;
                            this.jumps = 0;
                        }
                    }
                }
            }
        }

        // --- 画面外での撃墜（バースト）判定 ---
        if (this.y > BOUNDS.bottom || this.x < BOUNDS.left || this.x > BOUNDS.right || this.y < BOUNDS.top) {
            createBurstEffect(this.x, this.y, this.color);
            screenShake = 40;
            this.lives--;
            if (this.lives > 0) {
                this.respawn();
            }
        }
    }

    doJump() {
        if (this.isGrounded) {
            this.vy = -this.jumpForce;
            this.jumps = 1;
            this.isGrounded = false;
        } else if (this.jumps < this.maxJumps) {
            this.vy = -this.jumpForce * 0.85; // 空中ジャンプ
            this.jumps++;
            // 既存のcreateHitEffectを引数3つで呼ぶ場合は影響ないが、安全のためfalseを渡しておく
            createHitEffect(this.x + this.w/2, this.y + this.h, false); 
        }
    }

    doAttack(isStrong = false) {
        this.isAttacking = true;
        this.isStrongAttack = isStrong;
        // 強攻撃はモーションが長い
        this.attackFrame = isStrong ? 35 : 18; 
        this.hasHit = false;
    }

    draw(ctx) {
        if (this.lives <= 0) return;

        // 無敵時の点滅
        if (this.invincible > 0 && Math.floor(Date.now() / 100) % 2 === 0) {
            ctx.globalAlpha = 0.4;
        }

        ctx.save();
        ctx.translate(this.x + this.w/2, this.y + this.h/2);

        // ふっとび中はキャラが回転する
        if (this.hitStun > 0) {
            ctx.rotate((this.hitStun * 0.4) * (this.vx > 0 ? 1 : -1));
        }

        // キャラクター本体
        ctx.fillStyle = this.color;
        ctx.fillRect(-this.w/2, -this.h/2, this.w, this.h);

        // 目（向きを示す）
        ctx.fillStyle = '#fff';
        let eyeOffsetX = this.facingRight ? 5 : -15;
        ctx.fillRect(eyeOffsetX, -10, 10, 10);
        
        ctx.fillStyle = '#000';
        let pupilOffsetX = this.facingRight ? 10 : -15;
        ctx.fillRect(pupilOffsetX, -5, 5, 5);

        // 攻撃中のエフェクト（パンチや剣の軌跡のようなもの）
        if (this.isAttacking) {
            let isHitboxActive = false;
            if (this.isStrongAttack) {
                isHitboxActive = (this.attackFrame > 15 && this.attackFrame < 22);
            } else {
                isHitboxActive = (this.attackFrame > 8 && this.attackFrame < 13);
            }

            if (isHitboxActive) {
                ctx.fillStyle = this.isStrongAttack ? '#f44336' : '#fff';
                let effectW = this.isStrongAttack ? 45 : 25;
                let effectH = this.isStrongAttack ? 15 : 10;
                
                if (this.facingRight) {
                    ctx.fillRect(this.w/2, -effectH/2, effectW, effectH);
                } else {
                    ctx.fillRect(-this.w/2 - effectW, -effectH/2, effectW, effectH);
                }
            }
        }

        ctx.restore();
        ctx.globalAlpha = 1.0;

        // P1/P2マーカー
        ctx.fillStyle = this.color;
        ctx.font = '14px Arial Black';
        ctx.textAlign = 'center';
        let label = (this instanceof CPUPlayer) ? 'CPU' : `P${this.id}`;
        ctx.fillText(label, this.x + this.w/2, this.y - 10);
    }
}

// --- CPUプレイヤークラス (Playerを継承) ---
class CPUPlayer extends Player {
    constructor(id, x, y, color) {
        // CPUはコントロールキーを持たない
        super(id, x, y, color, {}); 
        this.decisionTimer = 0;
    }

    update(opponents) {
        if (this.lives <= 0) return;
        
        // ターゲット(1P)を取得
        let target = opponents[0];

        // --- CPU AI ロジック ---
        if (target && target.lives > 0 && this.hitStun === 0) {
            let dx = target.x - this.x;
            let dy = target.y - this.y;
            this.decisionTimer++;

            // 1. 復帰ロジック（画面外に落ちそうな時はジャンプして戻る）
            if (this.y > 350 || this.x < 100 || this.x > 700) {
                // 中央に向かって移動
                if (this.x < 400) this.simulateKey('right');
                else this.simulateKey('left');
                
                // 落ちていたらジャンプ
                if (this.vy > 0 && this.jumps < this.maxJumps && this.decisionTimer % 10 === 0) {
                    this.doJump();
                }
            } 
            // 2. 攻撃ロジック（相手が近い場合）
            else if (Math.abs(dx) < 80 && Math.abs(dy) < 40) {
                // さらに近ければ立ち止まって弱攻撃、少し離れていれば走りながら強攻撃を狙う
                if (Math.abs(dx) < 45) {
                    this.simulateKey(null); // 移動停止
                }
                
                // 向きを合わせる
                this.facingRight = dx > 0;
                
                // ランダムに攻撃
                if (this.attackCooldown === 0 && !this.isAttacking && Math.random() < 0.2) {
                    let isMoving = (this.vx !== 0);
                    this.doAttack(isMoving);
                }
            } 
            // 3. 追跡ロジック
            else {
                if (dx > 30) this.simulateKey('right');
                else if (dx < -30) this.simulateKey('left');
                
                // 相手が上にいるか、足場を乗り継ぐためにジャンプ
                if (dy < -60 && Math.random() < 0.05 && this.jumps < this.maxJumps) {
                    this.doJump();
                }
            }
        }

        // 親クラスのupdateを呼び出す（物理演算などを実行）
        super.update(opponents);
    }

    simulateKey(direction) {
        // 仮想的なキー入力を処理
        if (direction === 'left') {
            this.vx -= this.speed;
            if (!this.isAttacking) this.facingRight = false;
        } else if (direction === 'right') {
            this.vx += this.speed;
            if (!this.isAttacking) this.facingRight = true;
        }
    }
}

// --- ゲーム管理 ---
let p1, p2;

function startGame(mode) {
    gameMode = mode;
    document.getElementById('start-screen').style.display = 'none';
    document.getElementById('game-over').style.display = 'none';

    // プレイヤー1の生成
    p1 = new Player(1, 250, 150, '#ff5252', {
        left: 'KeyA', right: 'KeyD', jump: 'KeyW', down: 'KeyS', attack: 'KeyF'
    });

    // プレイヤー2 または CPU の生成
    if (mode === 'cpu') {
        p2 = new CPUPlayer(2, 550, 150, '#888888'); // CPUはグレー
    } else {
        p2 = new Player(2, 550, 150, '#448aff', {
            left: 'ArrowLeft', right: 'ArrowRight', jump: 'ArrowUp', down: 'ArrowDown', attack: 'Slash'
        });
    }
    p2.facingRight = false;

    particles = [];
    isPlaying = true;
    
    // アニメーションループ開始
    requestAnimationFrame(updateGame);
}

function showStartScreen() {
    isPlaying = false;
    document.getElementById('start-screen').style.display = 'flex';
    document.getElementById('game-over').style.display = 'none';
    ctx.clearRect(0, 0, canvas.width, canvas.height);
}

// --- UI描画 ---
function drawUI() {
    // 画面下部の黒背景
    ctx.fillStyle = 'rgba(0, 0, 0, 0.85)';
    ctx.fillRect(0, 420, canvas.width, 80);

    // P1 UI
    drawPlayerUI(p1, 200, 460);
    
    // P2 UI
    drawPlayerUI(p2, 600, 460);
}

function drawPlayerUI(player, x, y) {
    // ダメージ量に応じて色を赤くし、震えさせる
    let dmgColor = '#fff';
    if (player.damage > 50) dmgColor = '#ffeb3b';
    if (player.damage > 100) dmgColor = '#ff9800';
    if (player.damage > 150) dmgColor = '#f44336';

    let shakeX = 0, shakeY = 0;
    if (player.damage > 100) {
        shakeX = (Math.random() - 0.5) * (player.damage / 40);
        shakeY = (Math.random() - 0.5) * (player.damage / 40);
    }

    ctx.save();
    ctx.translate(x + shakeX, y + shakeY);
    
    // パーセンテージ表示
    ctx.fillStyle = dmgColor;
    ctx.font = 'bold 48px "Arial Black", Impact';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    
    ctx.lineWidth = 4;
    ctx.strokeStyle = '#000';
    ctx.strokeText(`${player.damage}%`, 0, 0);
    ctx.fillText(`${player.damage}%`, 0, 0);
    
    ctx.restore();

    // 残機（ストック）のアイコン表示
    for (let i = 0; i < player.lives; i++) {
        ctx.fillStyle = player.color;
        ctx.beginPath();
        // アイコンを横に並べる
        ctx.arc(x - 25 + (i * 25), y + 32, 8, 0, Math.PI * 2);
        ctx.fill();
        ctx.lineWidth = 2;
        ctx.strokeStyle = '#fff';
        ctx.stroke();
    }
}

// --- メインループ ---
function updateGame() {
    if (!isPlaying) return;

    // ヒットストップ（硬直）中は更新をスキップして停止させる
    if (hitStop > 0) {
        hitStop--;
        draw(); 
        // キーの状態だけは更新しておく
        Object.assign(prevKeys, keys);
        requestAnimationFrame(updateGame);
        return;
    }

    // プレイヤーの更新
    p1.update([p2]);
    p2.update([p1]);

    // 勝敗判定
    if (p1.lives <= 0 || p2.lives <= 0) {
        isPlaying = false;
        let winnerText = p1.lives > 0 ? "PLAYER 1 WINS!" : (gameMode === 'cpu' ? "CPU WINS!" : "PLAYER 2 WINS!");
        let winnerColor = p1.lives > 0 ? p1.color : p2.color;
        
        document.getElementById('winnerText').innerText = winnerText;
        document.getElementById('winnerText').style.color = winnerColor;
        document.getElementById('game-over').style.display = 'flex';
    }

    // パーティクルの更新
    particles = particles.filter(p => p.life > 0);
    particles.forEach(p => p.update());

    // 画面揺れの減衰
    if (screenShake > 0) screenShake *= 0.85;
    if (screenShake < 0.5) screenShake = 0;

    // キーの状態を保存（次フレームのエッジ判定用）
    Object.assign(prevKeys, keys);

    draw();

    if (isPlaying) {
        requestAnimationFrame(updateGame);
    }
}

// --- 描画処理 ---
function draw() {
    // 画面クリア
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    ctx.save();
    
    // 画面シェイクの適用
    if (screenShake > 0) {
        let dx = (Math.random() - 0.5) * screenShake;
        let dy = (Math.random() - 0.5) * screenShake;
        ctx.translate(dx, dy);
    }

    // 背景のグリッド線
    ctx.strokeStyle = 'rgba(255,255,255,0.05)';
    ctx.lineWidth = 1;
    for(let i = 0; i < canvas.width; i += 50) {
        ctx.beginPath();
        ctx.moveTo(i, 0);
        ctx.lineTo(i, canvas.height);
        ctx.stroke();
    }
    for(let i = 0; i < canvas.height; i += 50) {
        ctx.beginPath();
        ctx.moveTo(0, i);
        ctx.lineTo(canvas.width, i);
        ctx.stroke();
    }

    // 足場（プラットフォーム）の描画
    platforms.forEach(plat => {
        ctx.fillStyle = plat.color;
        ctx.fillRect(plat.x, plat.y, plat.w, plat.h);
        
        // 立体感を出すためのハイライト
        ctx.fillStyle = 'rgba(255,255,255,0.15)';
        ctx.fillRect(plat.x, plat.y, plat.w, 4);
        
        if (plat.type === 'solid') {
            // 下部の影
            ctx.fillStyle = 'rgba(0,0,0,0.5)';
            ctx.fillRect(plat.x, plat.y + plat.h - 6, plat.w, 6);
        }
    });

    // パーティクルの描画
    particles.forEach(p => p.draw(ctx));

    // プレイヤーの描画
    p1.draw(ctx);
    p2.draw(ctx);

    ctx.restore();

    // UIは画面揺れの影響を受けないよう一番最後に描画
    drawUI();
}

// 初期画面の描画
showStartScreen();
</script>
</body>
</html>
