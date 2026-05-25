[섀도우대거.html](https://github.com/user-attachments/files/28217478/default.html)
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Shadow Dagger</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #111;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    overflow: hidden;
    font-family: 'Courier New', monospace;
  }
  canvas {
    border: 2px solid #333;
    image-rendering: pixelated;
    image-rendering: crisp-edges;
    background: #1a1a2e;
  }
  #ui-overlay {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    pointer-events: none;
    z-index: 10;
  }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<script>
// ============================================================
// CONSTANTS
// ============================================================
const CANVAS_W = 1280;
const CANVAS_H = 720;
const GROUND_Y = 600;
const ARENA_LEFT = 160;
const ARENA_RIGHT = 1120;
const ARENA_TOP = 40;

const GRAVITY = 0.55;
const PLAYER_SPEED = 4.5;
const PLAYER_JUMP = -10.5;
const PLAYER_DOUBLE_JUMP = -9.2;

const COLORS = {
  bg: '#0a0a1a',
  ground: '#2a2230',
  groundLine: '#4a3040',
  wall: '#3d2b1f',
  player: '#4fc3f7',
  playerDark: '#0288d1',
  playerLight: '#b3e5fc',
  dagger: '#e0e0e0',
  daggerGlow: '#ffffff',
  boss: '#7b1fa2',
  bossDark: '#4a148c',
  bossEye: '#ff1744',
  bossAccent: '#ce93d8',
  minion: '#546e7a',
  minionDark: '#37474f',
  minionEye: '#ff9100',
  projectile: '#4fc3f7',
  enemyProjectile: '#ff5252',
  hpBar: '#4caf50',
  hpBarBg: '#333',
  mpBar: '#2196f3',
  mpBarBg: '#333',
  bossHpBar: '#f44336',
  bossHpBarBg: '#333',
  text: '#e0e0e0',
  hitFlash: '#ffffff',
  particlePlayer: '#4fc3f7',
  particleBoss: '#ce93d8',
  particleHit: '#ffeb3b',
};

// ============================================================
// SPRITE SYSTEM - Pixel Art Data
// ============================================================
// '.' = transparent, '0'-'9' = palette index
// Sprite: 2D array of palette indices, rendered pixel-by-pixel

const SPRITES = {};

// ----- Player Sprite (14x18) -----
// Palette: 0=outline, 1=dark, 2=main, 3=detail, 4=light, 5=skin, 6=eye glow, 7=accent
const PLAYER_PAL = ['#0a0a1a','#121240','#1e3060','#2979b8','#4fc3f7','#e8ddd0','#ffeb3b','#ff1744'];

function pixelSprite(rows, pal) {
  const h = rows.length, w = rows[0].length;
  const pixels = rows.map(row => {
    const out = new Int8Array(w);
    for (let c = 0; c < w; c++) out[c] = row[c] === '.' ? -1 : +row[c];
    return out;
  });
  return { pixels, pal, w, h };
}

// Player idle - hooded assassin
SPRITES.playerIdle = pixelSprite([
  '..............',
  '......11......',
  '....012210....',
  '...01222210...',
  '...01222210...',
  '...01666210...',
  '...01222210...',
  '....012210....',
  '....05550.....',
  '...01777710...',
  '..0111211110..',
  '..0113331110..',
  '..0111111110..',
  '..0111111110..',
  '...01...10....',
  '..001...100...',
  '..000...000...',
  '..............',
], PLAYER_PAL);

// Player attack lean (forward with dagger extended)
SPRITES.playerAttack = pixelSprite([
  '..............',
  '......11......',
  '....012210....',
  '...01222210...',
  '...01666210...',
  '...01222210...',
  '....012210....',
  '....05550.....',
  '...01777710...',
  '..0111211110..',
  '..0113331110..',
  '..0111111110..',
  '...01111110...',
  '....011110....',
  '....01.10.....',
  '...001.100....',
  '..000...000...',
  '..............',
], PLAYER_PAL);

// Player jump (legs tucked)
SPRITES.playerJump = pixelSprite([
  '..............',
  '......11......',
  '....012210....',
  '...01222210...',
  '...01666210...',
  '...01222210...',
  '....012210....',
  '....05550.....',
  '...01777710...',
  '..0111211110..',
  '..0113331110..',
  '..0111111110..',
  '...01111110...',
  '....011110....',
  '.....0000.....',
  '...000.000....',
  '..............',
  '..............',
], PLAYER_PAL);

// ----- Boss Sprite (16x20) -----
// Palette: 0=outline, 1=dark, 2=main, 3=light, 4=highlight, 5=eye, 6=sword
const BOSS_PAL = ['#1a0520','#2d0a3e','#4a148c','#7b1fa2','#ce93d8','#ff1744','#ffffff'];

SPRITES.bossIdle = pixelSprite([
  '................',
  '......1111......',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '...122055221...',
  '...122222221....',
  '....1222221.....',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '..12222222221...',
  '..12333332221...',
  '..12222222221...',
  '...122222221....',
  '....1222221.....',
  '....11...11.....',
  '...111...111....',
  '..111.....111...',
  '................',
], BOSS_PAL);

// Boss charge pose (leaning forward)
SPRITES.bossCharge = pixelSprite([
  '................',
  '.......1111.....',
  '......12221.....',
  '.....1222221....',
  '....122222221...',
  '....122055221...',
  '....122222221...',
  '.....1222221....',
  '......12221.....',
  '.....1222221....',
  '....122222221...',
  '...12222222221..',
  '...12333332221..',
  '...12222222221..',
  '....122222221...',
  '.....1222221....',
  '.....11...11....',
  '....111...111...',
  '...111.....111..',
  '................',
], BOSS_PAL);

// Boss slash (arm extended with sword)
SPRITES.bossSlash = pixelSprite([
  '................',
  '......1111......',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '...122055221....',
  '...122222221....',
  '....1222221.....',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '..12222222221...',
  '..12333332221...',
  '..12222222221...',
  '...122222221....',
  '....1222221.....',
  '....11...11.....',
  '...111...111....',
  '..111.....111...',
  '................',
], BOSS_PAL);

// Boss windup (sword raised)
SPRITES.bossWindup = pixelSprite([
  '................',
  '......1111......',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '...122055221....',
  '...122222221....',
  '....1222221.....',
  '.....12221......',
  '....1222221.....',
  '...122222221....',
  '..12222222221...',
  '..12333332221...',
  '..12222222221...',
  '...122222221....',
  '....1222221.....',
  '....11...11.....',
  '...111...111....',
  '..111.....111...',
  '................',
], BOSS_PAL);

// ----- Stage 2 Boss Sprite (32x36) - 거대 공포 몬스터 -----
const BOSS2_PAL = ['#080010','#150828','#2d1048','#4a2070','#ff1744','#ff5252','#ff8a80'];

SPRITES.boss2Idle = pixelSprite([
  '................................',  //  0
  '........1111111111111111........',  //  1 horns
  '.......111111111111111111.......',  //  2
  '......11111111111111111111......',  //  3
  '.....1111111122222111111111.....',  //  4 head
  '.....11111222222222221111111.....',  //  5
  '......11122266622666222111......',  //  6 eyes
  '......11122266622666222111......',  //  7 eyes
  '.....11111222222222221111111.....',  //  8
  '.....1111111122222111111111.....',  //  9
  '......11111111111111111111......',  // 10 neck
  '.....1111111111111111111111.....',  // 11 shoulders
  '....111111111111111111111111....',  // 12
  '...11111111333333333311111111...',  // 13 upper body
  '...11111133333333333333111111...',  // 14
  '..1111111133334444333311111111..',  // 15 chest
  '..1111113334445555444333111111..',  // 16 core
  '..1111113334445555444333111111..',  // 17 core
  '..1111111133334444333311111111..',  // 18
  '...11111133333333333333111111...',  // 19
  '...11111111111111111111111111...',  // 20 waist
  '....111111111111111111111111....',  // 21
  '.....1111111111111111111111.....',  // 22
  '......11111111111111111111......',  // 23 hips
  '......11111111111111111111......',  // 24
  '.....1111111111111111111111.....',  // 25 legs
  '....111111111111111111111111....',  // 26
  '...11111111111111111111111111...',  // 27 thighs
  '...11111111111111111111111111...',  // 28
  '..1111111111111111111111111111..',  // 29
  '..1111111111111111111111111111..',  // 30
  '...11111111111111111111111111...',  // 31
  '....111111111111111111111111....',  // 32
  '.....1111111111111111111111.....',  // 33
  '......11111111111111111111......',  // 34
  '.......111111111111111111.......',  // 35
], BOSS2_PAL);
// ----- Grim Reaper (사신) Sprite (16x20) -----
const REAPER_PAL = ['#08080f','#10101e','#1e1e32','#2e2e4a','#ff1744','#ff5252'];
SPRITES.reaperIdle = pixelSprite([
  '................',
  '...11111111....',  // hood top
  '..122222221...',  // hood
  '..122222221...',
  '..122555221...',  // red eyes
  '..122222221...',
  '.12222222221..',  // face
  '.12222222221..',
  '.12233342221..',  // robe upper
  '.12222222221..',
  '1222222222221.',  // robe body
  '1223333332221.',
  '1223444432221.',
  '1222222222221.',
  '.12222222221..',
  '.1..1111..1..',  // feet
  '11..1111..11.',
  '...............',
], REAPER_PAL);
// All reaper pose sprites share the same pixel data
SPRITES.reaperWindup = SPRITES.reaperSlash = SPRITES.reaperCharge = SPRITES.reaperJump = SPRITES.reaperIdle;
// Scythe weapon (20x20)
const SCYTHE_PAL = ['#08080f','#1a1a2e','#2a2a3e','#e0e0e0','#ffffff','#ff1744'];
SPRITES.scythe = pixelSprite([
  '....................',
  '....................',
  '..11111111111111....',  // blade top
  '.122222222222221...',  // blade outer
  '.123333333333321...',  // blade
  '.123333433333321...',  // blade w/ red
  '.122222222222221...',  // blade inner
  '..11111111111111....',  // blade base
  '......115551.......',  // ferrule
  '......11111.......',  // grip top
  '.....1166661......',  // grip
  '.....1166661......',
  '.....1166661......',
  '.....1166661......',
  '.....1111111......',
  '......11441.......',  // pommel
  '......11441.......',
  '....................',
], SCYTHE_PAL);

// ----- Weapon Sprite (20x10) - Katana/Sword -----
// 0=outline, 1=dark blue, 2=blade light, 3=blade bright, 4=red accent, 5=guard dark, 6=guard light
const WEAPON_PAL = ['#0a0a1a','#0e1e40','#e0e0e0','#ffffff','#ff1744','#4a148c','#7b1fa2'];

SPRITES.weapon = pixelSprite([
  '....................',  //  0
  '......11111111......',  //  1 - outline
  '.....122222221.....',  //  2 - blade outer
  '....12223553221....',  //  3 - blade w/ red accent
  '....12223333221....',  //  4 - blade bright core
  '....12223553221....',  //  5 - blade w/ red accent
  '.....122222221.....',  //  6 - blade outer
  '......11111111.....',  //  7 - outline
  '......1155551......',  //  8 - guard
  '.......1661.......',  //  9 - handle
], WEAPON_PAL);

// Palette: 0=outline, 1=dark, 2=main, 3=light, 4=highlight, 5=eye
const MINION_PAL = ['#0a0a18','#1a1a30','#2a2a4e','#4a4a7e','#78909c','#ff9100'];

SPRITES.minionIdle = pixelSprite([
  '............',
  '....1111....',
  '...12221....',
  '..1222221...',
  '..1222221...',
  '..1255521...',
  '..1222221...',
  '...12221....',
  '..1222221...',
  '..1233321...',
  '..1222221...',
  '...1...1....',
  '..11...11...',
  '............',
], MINION_PAL);

// Sprite draw function
function drawSprite(ctx, sprite, x, y, scale, flip) {
  const { pixels, pal, w, h } = sprite;
  const s = scale || 2;
  let lastColor = null;
  for (let r = 0; r < h; r++) {
    const ry = Math.round(y + r * s);
    for (let c = 0; c < w; c++) {
      const idx = pixels[r][c];
      if (idx < 0) continue;
      const color = pal[idx];
      if (color !== lastColor) { ctx.fillStyle = color; lastColor = color; }
      const sx = flip ? (x + (w - 1 - c) * s) : (x + c * s);
      ctx.fillRect(Math.round(sx), ry, s, s);
    }
  }
}

// ============================================================
// INPUT HANDLER
// ============================================================
class Input {
  constructor() {
    this.keys = {};
    this.justPressed = {};
    window.addEventListener('keydown', e => {
      if (['ArrowLeft','ArrowRight','ArrowUp','ArrowDown',' '].includes(e.key)) e.preventDefault();
      if (!this.keys[e.key]) this.justPressed[e.key] = true;
      this.keys[e.key] = true;
    });
    window.addEventListener('keyup', e => {
      this.keys[e.key] = false;
    });
  }
  isDown(key) { return !!this.keys[key]; }
  isPressed(key) {
    if (this.justPressed[key]) {
      this.justPressed[key] = false;
      return true;
    }
    return false;
  }
  get left() { return this.isDown('ArrowLeft'); }
  get right() { return this.isDown('ArrowRight'); }
  get up() { return this.isDown('ArrowUp'); }
  get down() { return this.isDown('ArrowDown'); }
  get jump() { return this.isPressed(' '); }
  get attack() { return this.isPressed('z') || this.isPressed('Z'); }
  get skillX() { return this.isPressed('x') || this.isPressed('X'); }
  get skillC() { return this.isPressed('c') || this.isPressed('C'); }
  get skillQ() { return this.isPressed('q') || this.isPressed('Q'); }
  get skillE() { return this.isPressed('e') || this.isPressed('E'); }
  get dodge() { return this.isPressed('Shift'); }
  get escape() { return this.isPressed('Escape'); }
  get start() { return this.isPressed(' ') || this.isPressed('Enter'); }
}

// ============================================================
// CAMERA
// ============================================================
class Camera {
  constructor() {
    this.x = 0;
    this.y = 0;
    this.targetX = 0;
    this.targetY = 0;
  }
  follow(target) {
    this.targetX = -(target.x - CANVAS_W / 2);
    this.targetY = -(target.y - CANVAS_H / 2);
    this.targetX = Math.max(-500, Math.min(this.targetX, 500));
    this.targetY = Math.max(-ARENA_TOP, Math.min(this.targetY, CANVAS_H - GROUND_Y + 40));
  }
  update() {
    this.x += (this.targetX - this.x) * 0.08;
    this.y += (this.targetY - this.y) * 0.08;
  }
  get transform() {
    return { x: -Math.round(this.x), y: -Math.round(this.y) };
  }
}

// ============================================================
// SCREEN SHAKE
// ============================================================
let shakeX = 0, shakeY = 0, shakeIntensity = 0;

function triggerShake(intensity, duration) {
  shakeIntensity = intensity;
  setTimeout(() => shakeIntensity = 0, duration);
}

// ============================================================
// PARTICLE SYSTEM
// ============================================================
class Particle {
  constructor(x, y, vx, vy, color, size, life) {
    this.x = x; this.y = y;
    this.vx = vx; this.vy = vy;
    this.color = color;
    this.size = size || 3;
    this.life = life || 30;
    this.maxLife = this.life;
    this.alive = true;
  }
  update() {
    this.x += this.vx;
    this.y += this.vy;
    this.vy += 0.05;
    this.life--;
    if (this.life <= 0) this.alive = false;
  }
  draw(ctx, cam) {
    ctx.globalAlpha = this.life / this.maxLife;
    ctx.fillStyle = this.color;
    const hs = this.size * 0.5;
    ctx.fillRect(Math.round(this.x + cam.x) - hs, Math.round(this.y + cam.y) - hs, this.size, this.size);
  }
}

let particles = [];

function spawnParticles(x, y, count, color, speed, size, life) {
  const sz = size || 3, lf = life || 30;
  for (let i = 0; i < count; i++) {
    const a = Math.random() * Math.PI * 2;
    const spd = (Math.random() * 0.5 + 0.5) * speed;
    particles.push(new Particle(x, y, Math.cos(a) * spd, Math.sin(a) * spd - 1, color, sz, lf));
  }
}

// ============================================================
// COLLISION HELPERS
// ============================================================
function rectCollide(a, b) {
  const ahw = a.w * 0.5, bhw = b.w * 0.5;
  return a.x - ahw < b.x + bhw &&
         a.x + ahw > b.x - bhw &&
         a.y - a.h < b.y &&
         a.y > b.y - b.h;
}

function pointInRect(px, py, r) {
  return px > r.x - r.w/2 && px < r.x + r.w/2 &&
         py > r.y - r.h && py < r.y;
}

// ============================================================
// PLAYER CLASS
// ============================================================
class Player {
  constructor(x, y) {
    this.x = x; this.y = y;
    this.w = 26; this.h = 36;
    this.vx = 0; this.vy = 0;
    this.facingRight = true;
    const vitMult = getVitMult();
    this.hp = Math.round(100 * vitMult); this.maxHp = Math.round(100 * vitMult);
    this.mp = 100; this.maxMp = 100;
    this.mpRegenTimer = 0;
    this.isGrounded = false;
    this.jumpsLeft = 2;
    this.isDead = false;
    this.invincible = 0;
    this.hitFlashTimer = 0;

    // Attack
    this.attackCombo = 0;
    this.attackTimer = 0;
    this.comboWindow = 12;
    this.attackActive = 0;
    this.attackHit = false;

    // Skills
    this.cooldownX = 0;
    this.cooldownC = 0;
    this.cooldownShift = 0;
    this.cooldownQ = 0;
    this.cooldownE = 0;
    this.hasSummon = false;

    // Ultimate
    this.isUltimate = false;
    this.ultimateTimer = 0;
    this.ultimateFrames = 28;
    this.ultimateVx = 0;
    this.ultimateTargetX = 0;
    this.ultimateTargetY = 0;
    this.ultimateHit = false;
    this.ultStartX = 0;
    this.ultStartY = 0;
    this.ultAngle = 0;

    // Dodge
    this.isDodging = false;
    this.dodgeTimer = 0;
    this.dodgeFrames = 12;
    this.dodgeVx = 0;

    // Shadow dash state
    this.isDashing = false;
    this.dashTimer = 0;
    this.dashFrames = 8;
    this.dashVx = 0;
    this.dashHit = false;

    // Dash dagger (throw → teleport)
    this.dashDagger = null; // { x, y, vx, vy, stuck, stuckX, stuckY, stuckTimer }

    // Visual
    this.comboStep = 0;
    this.invFlash = false;
  }

  get box() { return { x: this.x, y: this.y, w: this.w, h: this.h }; }

  reset(x, y) {
    this.x = x; this.y = y;
    this.vx = 0; this.vy = 0;
    this.hp = this.maxHp;
    this.mp = this.maxMp;
    this.isDead = false;
    this.isGrounded = false;
    this.jumpsLeft = 2;
    this.invincible = 0;
    this.attackCombo = 0;
    this.attackTimer = 0;
    this.attackActive = 0;
    this.attackHit = false;
    this.isDodging = false;
    this.isDashing = false;
    this.cooldownX = 0;
    this.cooldownC = 0;
    this.cooldownShift = 0;
    this.cooldownQ = 0;
    this.cooldownE = 0;
    this.isUltimate = false;
    this.ultimateTimer = 0;
    this.ultStartX = 0;
    this.ultStartY = 0;
    this.ultimateTargetY = 0;
    this.dashDagger = null;
  }

  update(input) {
    if (this.isDead) return;

    // Cooldowns
    if (this.cooldownX > 0) this.cooldownX--;
    if (this.cooldownC > 0) this.cooldownC--;
    if (this.cooldownShift > 0) this.cooldownShift--;
    if (this.cooldownQ > 0) this.cooldownQ--;
    if (this.cooldownE > 0) this.cooldownE--;
    if (this.invincible > 0) this.invincible--;
    if (this.hitFlashTimer > 0) this.hitFlashTimer--;

    // MP regen
    this.mpRegenTimer++;
    if (this.mpRegenTimer >= 6) {
      this.mpRegenTimer = 0;
      this.mp = Math.min(this.maxMp, this.mp + 1);
    }

    // Dodge state — in place, dark aura
    if (this.isDodging) {
      this.dodgeTimer--;
      if (this.dodgeTimer <= 0) {
        this.isDodging = false;
      }
      // Still apply gravity during dodge
      this.vy += GRAVITY;
      this.y += this.vy;
      this.clampToArena();
      this.landCheck();
      return;
    }

    // Shadow dash state
    if (this.isDashing) {
      this.dashTimer--;
      this.x += this.dashVx;
      if (this.dashTimer <= 0) {
        this.isDashing = false;
      }
      this.vy += GRAVITY;
      this.y += this.vy;
      this.clampToArena();
      this.landCheck();
      return;
    }

    // Ultimate state
    if (this.isUltimate) {
      this.ultimateTimer--;
      const dx = this.ultimateTargetX - this.x;
      if (Math.abs(dx) > 8) {
        this.x += Math.sign(dx) * Math.min(Math.abs(this.ultimateVx), Math.abs(dx));
      } else {
        this.ultimateHit = true;
        if (this.ultimateTimer > 5) this.ultimateTimer = 5;
      }
      if (Math.random() < 0.4) {
        spawnParticles(this.x, this.y - Math.random() * 40, 5, '#29b6f6', 4, 4, 15);
        spawnParticles(this.x + (Math.random() - 0.5) * 30, this.y - 10 - Math.random() * 60, 3, '#ffffff', 3, 3, 10);
      }
      if (this.ultimateTimer <= 0) {
        this.ultimateTargetY = this.y;
        this.isUltimate = false;
      }
      this.vy += GRAVITY;
      this.y += this.vy;
      this.clampToArena();
      this.landCheck();
      return;
    }

    // Attack timer
    if (this.attackActive > 0) {
      this.attackActive--;
      if (this.attackActive <= 0) {
        this.attackHit = false;
      }
      this.attackTimer++;
      if (this.attackTimer > this.comboWindow && this.attackActive <= 0) {
        this.attackCombo = 0;
      }
    } else {
      this.attackTimer++;
      if (this.attackTimer > this.comboWindow) {
        this.attackCombo = 0;
      }
    }

    // Horizontal movement
    if (input.left) {
      this.vx = Math.max(this.vx - 0.8, -this.getSpeed());
      this.facingRight = false;
    } else if (input.right) {
      this.vx = Math.min(this.vx + 0.8, this.getSpeed());
      this.facingRight = true;
    } else {
      this.vx *= 0.7;
      if (Math.abs(this.vx) < 0.1) this.vx = 0;
    }

    // Jump
    if (input.jump && this.jumpsLeft > 0) {
      this.vy = (this.jumpsLeft === 2 && !this.isGrounded) ? PLAYER_DOUBLE_JUMP : PLAYER_JUMP;
      this.jumpsLeft--;
      this.isGrounded = false;
      spawnParticles(this.x, this.y, 5, COLORS.particlePlayer, 2, 3, 15);
    }

    // Gravity
    this.vy += GRAVITY;
    if (this.vy > 15) this.vy = 15;

    // Apply velocity
    this.x += this.vx;
    this.y += this.vy;

    // Ground collision
    this.landCheck();

    // Arena bounds
    this.clampToArena();

    // Attack input (no cooldown, instant chaining)
    if (input.attack) {
      this.performAttack();
    }

    // Dodge input
    if (input.dodge && this.cooldownShift <= 0 && !this.isDodging) {
      this.performDodge();
    }

    // Skill X - Throwing Star
    if (input.skillX && this.cooldownX <= 0 && this.mp >= 30) {
      this.performSkillX();
    }

    // Skill C - Shadow Dash
    if (input.skillC && this.cooldownC <= 0 && this.mp >= 40) {
      this.performSkillC();
    }

    // Skill Q - Ultimate: Lightning Charge
    if (input.skillQ && this.cooldownQ <= 0 && this.mp >= 100) {
      this.performUltimate();
    }

    // Skill E - Summon Minion (learned from boss)
    if (input.skillE && this.cooldownE <= 0 && this.mp >= 60 && this.hasSummon) {
      this.performSummon();
    }

    // Dash dagger update (throw → stick → teleport)
    if (this.dashDagger) {
      if (!this.dashDagger.stuck) {
        // Move dagger fast (straight line, no gravity)
        this.dashDagger.x += this.dashDagger.vx;
        this.dashDagger.y += this.dashDagger.vy;
        // Trail particles
        spawnParticles(this.dashDagger.x, this.dashDagger.y, 1, '#4fc3f7', 2, 3, 8);
        // Hit boss
        if (boss && boss.alive && rectCollide({x:this.dashDagger.x,y:this.dashDagger.y,w:8,h:8}, boss.box)) {
          this.dashDagger.stuck = true;
          this.dashDagger.stuckX = boss.x;
          this.dashDagger.stuckY = boss.y - 8;
          boss.takeDamage(10, this.facingRight ? 1 : -1);
          spawnParticles(this.dashDagger.x, this.dashDagger.y, 15, '#ffffff', 4, 4, 15);
        }
        // Hit minions
        for (const m of minions) {
          if (m.alive && rectCollide({x:this.dashDagger.x,y:this.dashDagger.y,w:8,h:8}, m.box)) {
            this.dashDagger.stuck = true;
            this.dashDagger.stuckX = m.x;
            this.dashDagger.stuckY = m.y - 4;
            m.takeDamage(10, this.facingRight ? 1 : -1);
            spawnParticles(this.dashDagger.x, this.dashDagger.y, 12, '#ffffff', 4, 4, 15);
            break;
          }
        }
        // Miss (hit ground or out of bounds)
        if (this.dashDagger.y >= GROUND_Y || this.dashDagger.x < ARENA_LEFT || this.dashDagger.x > ARENA_RIGHT) {
          this.dashDagger.stuck = true;
          this.dashDagger.stuckX = this.dashDagger.x;
          this.dashDagger.stuckY = Math.min(this.dashDagger.y, GROUND_Y);
          this.dashDagger.targetRef = null;
          spawnParticles(this.dashDagger.x, this.dashDagger.y, 8, '#4fc3f7', 3, 3, 12);
        }
      } else {
        // Stuck — follow target if still alive
        if (this.dashDagger.targetRef && this.dashDagger.targetRef.alive) {
          this.dashDagger.stuckX = this.dashDagger.targetRef.x;
          this.dashDagger.stuckY = this.dashDagger.targetRef.y - 8;
        }
        this.dashDagger.stuckTimer--;
        // Stuck visual pulse
        if (this.dashDagger.stuckTimer % 6 === 0) {
          spawnParticles(this.dashDagger.stuckX, this.dashDagger.stuckY, 5, '#4fc3f7', 3, 4, 12);
        }
        if (this.dashDagger.stuckTimer <= 0) {
          // Teleport to dagger!
          const sx = this.x;
          const sy = this.y - 8;
          const tx = this.dashDagger.stuckX;
          const ty = Math.min(this.dashDagger.stuckY, GROUND_Y);
          // Dramatic dash beam trail
          const dist = Math.sqrt((tx - sx) ** 2 + (ty - sy) ** 2);
          const steps = Math.min(Math.ceil(dist / 3), 50);
          // Thick glowing beam
          for (let i = 0; i < steps; i++) {
            const t = i / steps;
            for (let w = -2; w <= 2; w++) {
              const nx = -(ty - sy) / dist * w * 4;
              const ny = (tx - sx) / dist * w * 4;
              const lx = sx + (tx - sx) * t + nx + (Math.random() - 0.5) * 6;
              const ly = sy + (ty - sy) * t + ny + (Math.random() - 0.5) * 6;
              const col = w === 0 ? '#ffffff' : (Math.abs(w) === 1 ? '#81d4fa' : '#0288d1');
              particles.push(new Particle(lx, ly, (Math.random() - 0.5) * 0.8, (Math.random() - 0.5) * 0.8, col, 3 + Math.abs(w) * 2, 18 + Math.random() * 8));
            }
          }
          // Electric sparks along the path
          for (let i = 0; i < steps * 2; i++) {
            const t = i / steps / 2;
            const lx = sx + (tx - sx) * t + (Math.random() - 0.5) * 20;
            const ly = sy + (ty - sy) * t + (Math.random() - 0.5) * 20;
            particles.push(new Particle(lx, ly, (Math.random() - 0.5) * 4, (Math.random() - 0.5) * 4, '#e1bee7', 2 + Math.random() * 3, 10 + Math.random() * 8));
          }
          this.x = tx;
          this.y = ty;
          this.vx = 0; this.vy = 0;
          this.invincible = 30;
          this.dashDagger = null;
          // Big teleport burst
          spawnParticles(tx, ty, 30, '#4fc3f7', 8, 5, 28);
          spawnParticles(tx, ty, 20, '#ffffff', 6, 4, 22);
          spawnParticles(tx, ty, 15, '#0288d1', 7, 4, 24);
          for (let i = 0; i < 20; i++) {
            const a = (Math.PI * 2 / 20) * i;
            particles.push(new Particle(tx, ty, Math.cos(a) * 5, Math.sin(a) * 5 - 1, '#4fc3f7', 4, 18));
          }
          triggerShake(6, 250);
          screenFlash = 8;
          screenFlashColor = '#4fc3f7';
        }
      }
    }

    // Invincibility flash
    this.invFlash = (this.invincible > 0) && (Math.floor(this.invincible / 3) % 2 === 0);
  }

  landCheck() {
    if (this.y >= GROUND_Y) {
      this.y = GROUND_Y;
      this.vy = 0;
      if (!this.isGrounded) {
        spawnParticles(this.x, this.y, 3, COLORS.particlePlayer, 1.5, 2, 10);
      }
      this.isGrounded = true;
      this.jumpsLeft = 2;
      // Dodge ends on landing if falling
      if (this.isDodging && this.vy >= 0) {
        // Keep dodging
      }
    } else {
      this.isGrounded = false;
    }
  }

  clampToArena() {
    this.x = Math.max(ARENA_LEFT + this.w/2, Math.min(this.x, ARENA_RIGHT - this.w/2));
    this.y = Math.max(ARENA_TOP + this.h, Math.min(this.y, GROUND_Y));
  }

  performAttack() {
    if (this.attackTimer > this.comboWindow) {
      this.attackCombo = 0;
    }
    this.attackCombo = (this.attackCombo % 3) + 1;
    this.attackTimer = 0;
    this.attackActive = 5;
    this.attackHit = false;
    this.comboStep = this.attackCombo;

    const dir = this.facingRight ? 1 : -1;
    const hx = this.x + dir * 15;
    const hy = this.y - 10;
    const combo = this.attackCombo;

    // Combo-specific colors — all RED and flashy
    const colors = [
      { main: '#ff1744', bright: '#ff5252', dark: '#b71c1c', glow: '#ff8a80' },
      { main: '#ff5252', bright: '#ff8a80', dark: '#d50000', glow: '#ff1744' },
      { main: '#ff1744', bright: '#ffffff', dark: '#b71c1c', glow: '#ff9100' },
    ];
    const c = colors[combo - 1];

    // Screen flash for each hit (red flash)
    screenFlash = combo === 3 ? 10 : 5;
    screenFlashColor = combo === 3 ? '#ffffff' : c.bright;
    triggerShake(combo * 2, combo === 3 ? 200 : 100);

    // Burst particles (main) — red explosion
    spawnParticles(hx, hy, combo === 3 ? 30 : 12, c.main, 6 + combo, 4 + combo, 15 + combo * 5);
    spawnParticles(hx, hy, combo === 3 ? 18 : 8, c.bright, 4, 3, 12);
    spawnParticles(hx, hy, combo === 3 ? 10 : 4, c.dark, 5, 4, 18);

    // Spark trail (red wave)
    for (let i = 0; i < (combo === 3 ? 8 : 4); i++) {
      const sx = hx + dir * (6 + i * 5);
      const sy = hy - 4 + (Math.random() - 0.5) * 10;
      particles.push(new Particle(sx, sy, dir * (2 + Math.random() * 2), (Math.random() - 0.5) * 3, c.main, 3 + Math.random() * 4, 10 + combo * 3));
    }

    // Multiple crossing straight slashes — fill the hitbox rectangle (wide × short)
    const range = this.getAttackRange();
    const halfH = 15;
    // Hitbox rectangle bounds (world space)
    const rectL = hx;
    const rectR = hx + dir * range - 10;
    const rectT = hy - halfH;
    const rectB = hy + halfH;
    const rectCx = (rectL + rectR) / 2;
    const rectCy = (rectT + rectB) / 2;
    const slashCount = combo === 3 ? 8 : 6;
    // Generate crossing slashes — random diagonals from left side to right side
    for (let si = 0; si < slashCount; si++) {
      // Start point on left edge (random height)
      const sy1 = rectT + Math.random() * (rectB - rectT);
      // End point on right edge (random height)
      const sy2 = rectT + Math.random() * (rectB - rectT);
      const sx1 = dir > 0 ? rectL : rectR;
      const sx2 = dir > 0 ? rectR : rectL;
      const steps = 8 + Math.floor(Math.random() * 4);
      for (let j = 0; j < steps; j++) {
        const t = j / steps;
        const lx = sx1 + (sx2 - sx1) * t + (Math.random() - 0.5) * 4;
        const ly = sy1 + (sy2 - sy1) * t + (Math.random() - 0.5) * 4;
        const col = j % 2 === 0 ? c.main : c.bright;
        particles.push(new Particle(lx, ly, dir * (3 + Math.random() * 3), (Math.random() - 0.5) * 2, col, 4 + Math.random() * 5, 10 + j));
      }
      // Edge spark at right end
      const edgeX = dir > 0 ? rectR : rectL;
      spawnParticles(edgeX + (Math.random() - 0.5) * 6, sy2 + (Math.random() - 0.5) * 6, 3, '#ffffff', 3, 2, 8);
    }
    // Extra crossing lines from center-out for more drama
    for (let si = 0; si < 3; si++) {
      const ang = (Math.random() - 0.5) * 1.0;
      const len2 = range * 0.45;
      for (let j = 0; j < 10; j++) {
        const t = j / 10;
        const lx = rectCx + dir * Math.cos(ang) * t * len2 + (Math.random() - 0.5) * 3;
        const ly = rectCy + Math.sin(ang) * t * len2 + (Math.random() - 0.5) * 3;
        particles.push(new Particle(lx, ly, dir * Math.cos(ang) * (1 + t * 2), Math.sin(ang) * (1 + t * 2), c.glow, 3 + Math.random() * 3, 8 + j));
      }
    }
    // Explosion at center
    spawnParticles(rectCx, rectCy, combo * 4, c.glow, 4, 3, 12);

    // Small forward lunge
    const lungeDist = combo === 3 ? 18 : 10;
    this.x += dir * lungeDist;
    this.x = Math.max(ARENA_LEFT + this.w/2, Math.min(this.x, ARENA_RIGHT - this.w/2));
  }

  getSpeed() { return PLAYER_SPEED * getDexMult(); }

  getAttackDamage() {
    const base = this.attackCombo === 1 ? 2 : this.attackCombo === 2 ? 3 : 5;
    return Math.round(base * getStrMult());
  }

  getAttackRange() {
    return this.attackCombo === 1 ? 140 : this.attackCombo === 2 ? 160 : 200;
  }

  getAttackKnockback() {
    return this.attackCombo === 3 ? 8 : 4;
  }

  performDodge() {
    this.isDodging = true;
    this.dodgeTimer = this.dodgeFrames;
    this.dodgeVx = 0;
    this.invincible = this.dodgeFrames + 30;
    this.cooldownShift = 90;
    this.mp = Math.max(0, this.mp - 10);
    // Dark aura swirl — in place
    for (let i = 0; i < 24; i++) {
      const a = (Math.PI * 2 / 24) * i + Math.random() * 0.3;
      const r = 12 + Math.random() * 10;
      const px = this.x + Math.cos(a) * r + (Math.random() - 0.5) * 4;
      const py = this.y - 8 + Math.sin(a) * r * 0.3 + (Math.random() - 0.5) * 4;
      const col = ['#1a1a2e','#0d0d1a','#2d2d44','#3d3d5c'][Math.floor(Math.random() * 4)];
      particles.push(new Particle(px, py, Math.cos(a) * 0.3, -0.2 + Math.random() * 0.4, col, 5 + Math.random() * 5, 18 + Math.random() * 10));
    }
  }

  performSkillX() {
    this.mp -= 30;
    this.cooldownX = 420; // 7s cooldown
    const dir = this.facingRight ? 1 : -1;
    const angles = [-0.3, -0.1, 0.1, 0.3];
    const starColors = ['#ce93d8','#e1bee7','#ab47bc','#8e24aa'];
    for (let i = 0; i < angles.length; i++) {
      const rad = angles[i] * dir;
      const p = new Projectile(
        this.x + dir * 18, this.y - 8,
        Math.cos(rad) * 30 * dir, Math.sin(rad) * 30 * 0.3,
        15, 'player', 45
      );
      p.starColor = starColors[i % starColors.length];
      projectiles.push(p);
    }
    // Launch effects — massive purple burst
    spawnParticles(this.x + dir * 12, this.y - 8, 35, '#ce93d8', 7, 6, 26);
    spawnParticles(this.x + dir * 12, this.y - 8, 25, '#ffffff', 5, 5, 20);
    spawnParticles(this.x + dir * 12, this.y - 8, 20, '#ab47bc', 7, 5, 22);
    spawnParticles(this.x + dir * 12, this.y - 8, 15, '#7b1fa2', 6, 4, 18);
    // Ring burst — purple
    for (let i = 0; i < 30; i++) {
      const a = (Math.PI * 2 / 30) * i;
      particles.push(new Particle(this.x + dir * 8, this.y - 8, Math.cos(a) * 6 + dir * 2, Math.sin(a) * 6, ['#ce93d8','#e1bee7','#ab47bc','#ffffff'][i % 4], 5, 22));
    }
    triggerShake(5, 120);
  }

  performSkillC() {
    this.mp -= 40;
    this.cooldownC = 600; // 10s

    // Find nearest enemy
    let target = null;
    let bestDist = Infinity;
    if (boss && boss.alive) {
      const d = Math.abs(boss.x - this.x);
      if (d < bestDist) { bestDist = d; target = boss; }
    }
    for (const m of minions) {
      if (!m.alive) continue;
      const d = Math.abs(m.x - this.x);
      if (d < bestDist) { bestDist = d; target = m; }
    }

    if (target) {
      const dir = target.x > this.x ? 1 : -1;
      const dx = target.x - this.x;
      const dy = target.y - this.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      // Throw dagger very fast toward target
      this.dashDagger = {
        x: this.x + dir * 16, y: this.y - 8,
        vx: (dx / dist) * 22, vy: (dy / dist) * 22,
        stuck: false, stuckX: 0, stuckY: 0, stuckTimer: 30,
        targetRef: target
      };
      // Launch burst
      spawnParticles(this.x + dir * 16, this.y - 8, 20, '#4fc3f7', 5, 4, 18);
      spawnParticles(this.x + dir * 16, this.y - 8, 12, '#ffffff', 3, 3, 14);
      spawnParticles(this.x + dir * 16, this.y - 8, 8, '#0288d1', 4, 3, 16);
    } else {
      // No target — short forward dash as fallback
      const dir = this.facingRight ? 1 : -1;
      this.isDashing = true;
      this.dashTimer = this.dashFrames;
      this.dashVx = dir * 10;
      this.vy = -0.5;
      this.invincible = this.dashFrames + 5;
      this.dashHit = false;
      spawnParticles(this.x, this.y - 8, 15, '#4fc3f7', 4, 3, 14);
    }
    triggerShake(2, 80);
  }

  performUltimate() {
    this.mp -= 100;
    this.cooldownQ = 900; // 15s
    this.isUltimate = true;
    this.ultimateTimer = this.ultimateFrames;
    this.ultimateHit = false;
    const dir = this.facingRight ? 1 : -1;
    let targetX = this.x + dir * 250; // default dash distance
    let bestDist = 99999;
    if (boss && boss.alive) {
      const d = (boss.x - this.x) * dir;
      if (d > 0 && d < bestDist) { bestDist = d; targetX = boss.x - dir * 40; }
    }
    for (const m of minions) {
      if (!m.alive) continue;
      const d = (m.x - this.x) * dir;
      if (d > 0 && d < bestDist) { bestDist = d; targetX = m.x - dir * 40; }
    }
    this.ultStartX = this.x;
    this.ultStartY = this.y;
    this.ultimateTargetX = targetX;
    this.ultimateVx = dir * 22;
    this.vy = -1;
    // Angle along predicted dash trajectory (parabolic arc)
    const n = this.ultimateFrames;
    const yDisp = this.vy * n + 0.5 * GRAVITY * n * (n - 1);
    this.ultAngle = Math.atan2(yDisp, this.ultimateVx * n);
    this.invincible = this.ultimateFrames + 40;

    // Apply slow to boss
    if (boss && boss.alive) boss.slowTimer = 120;
    for (const m of minions) if (m.alive) m.slowTimer = 120;

    // Screen effects
    screenFlash = 30;
    screenFlashColor = '#ffffff';
    triggerShake(15, 500);
    worldSplitTimer = 55;

    // Massive electrical burst
    spawnParticles(this.x, this.y - 8, 50, '#29b6f6', 10, 5, 40);
    spawnParticles(this.x, this.y - 8, 35, '#ffffff', 7, 4, 30);
    spawnParticles(this.x, this.y - 8, 25, '#0d47a1', 8, 5, 35);
    spawnParticles(this.x, this.y - 8, 20, '#4fc3f7', 6, 3, 25);

    // Lightning ring burst
    for (let i = 0; i < 24; i++) {
      const a = (Math.PI * 2 / 24) * i;
      const spd = 4 + Math.random() * 4;
      particles.push(new Particle(this.x, this.y - 8, Math.cos(a) * spd, Math.sin(a) * spd - 0.5, '#29b6f6', 3 + Math.random() * 3, 20 + Math.random() * 15));
    }

    // Chain lightning particles
    for (let i = 0; i < 12; i++) {
      const a = Math.random() * Math.PI * 2;
      const dist = 30 + Math.random() * 60;
      particles.push(new Particle(
        this.x + Math.cos(a) * dist, this.y - 8 + Math.sin(a) * dist * 0.5,
        -Math.cos(a) * 2, -Math.sin(a) * 2, '#e3f2fd', 2 + Math.random() * 3, 18
      ));
    }
  }

  performSummon() {
    this.mp -= 60;
    this.cooldownE = 600; // 10s

    // Summon circle on ground
    for (let i = 0; i < 30; i++) {
      const a = Math.random() * Math.PI * 2;
      const r = 20 + Math.random() * 30;
      const sx = this.x + Math.cos(a) * r;
      const sy = this.y - 4 + Math.random() * 4;
      particles.push(new Particle(sx, sy, Math.cos(a) * 0.5, -0.5 - Math.random() * 1, '#ce93d8', 3 + Math.random() * 3, 20 + Math.random() * 15));
    }
    // Rising energy pillars
    for (let i = 0; i < 16; i++) {
      const a = (Math.PI * 2 / 16) * i;
      const r = 25;
      const sx = this.x + Math.cos(a) * r;
      particles.push(new Particle(sx, this.y - 4, Math.cos(a) * 0.3, -2 - Math.random() * 3, '#e1bee7', 3, 15 + Math.random() * 10));
    }

    // Screen flash
    screenFlash = 6;
    screenFlashColor = '#ce93d8';
    triggerShake(5, 200);

    // Spawn minions with dramatic effect
    for (let i = 0; i < 2; i++) {
      const spawnX = this.x + (i === 0 ? -50 : 50);
      const m = new Minion(spawnX, this.y - 10);
      m.owner = 'player';
      m.aggro = true;
      m.agroRange = 600;
      minions.push(m);
      // Each minion spawn burst
      spawnParticles(spawnX, this.y - 10, 15, '#ce93d8', 4, 4, 20);
      spawnParticles(spawnX, this.y - 10, 8, '#ffffff', 3, 3, 14);
      spawnParticles(spawnX, this.y - 10, 6, '#7b1fa2', 5, 3, 18);
    }

    // Big center burst
    spawnParticles(this.x, this.y - 8, 25, '#ce93d8', 6, 4, 28);
    spawnParticles(this.x, this.y - 8, 15, '#ffffff', 4, 3, 20);
    spawnParticles(this.x, this.y - 8, 10, '#7b1fa2', 5, 4, 24);
  }

  takeDamage(dmg) {
    if (this.invincible > 0 || this.isDead) return;
    dmg = Math.round(dmg * getDefMult());
    this.hp -= dmg;
    this.invincible = 30;
    this.hitFlashTimer = 6;
    spawnParticles(this.x, this.y, 12, COLORS.particleHit, 4, 3, 20);
    triggerShake(4, 120);
    if (this.hp <= 0) {
      this.hp = 0;
      this.isDead = true;
    }
  }

  draw(ctx, cam) {
    if (this.isDead) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    const f = this.facingRight ? 1 : -1;
    const scale = 3;

    // Invincibility flash
    if (this.invFlash) ctx.globalAlpha = 0.5;

    // Shadow
    if (!this.isGrounded) {
      ctx.fillStyle = 'rgba(0,0,0,0.2)';
      ctx.fillRect(px - 14, Math.round(GROUND_Y + cam.y) - 2, 28, 4);
    }

    // Sprite position (bottom-center of sprite aligns with py)
    const sp = { x: px - (14 * scale / 2), y: py - (18 * scale) };

    // Dodge — dark aura + semi-transparency
    if (this.isDodging) {
      // Dark swirling aura (outer)
      ctx.globalAlpha = 0.15 + Math.sin(Date.now() * 0.015) * 0.08;
      const auraR = 20 + Math.sin(Date.now() * 0.02) * 5;
      const darkGrad = ctx.createRadialGradient(sp.x + 7 * scale, sp.y + 9 * scale, 0, sp.x + 7 * scale, sp.y + 9 * scale, auraR);
      darkGrad.addColorStop(0, 'rgba(20,20,40,0.3)');
      darkGrad.addColorStop(0.5, 'rgba(10,10,25,0.15)');
      darkGrad.addColorStop(1, 'rgba(0,0,0,0)');
      ctx.fillStyle = darkGrad;
      ctx.beginPath(); ctx.arc(sp.x + 7 * scale, sp.y + 9 * scale, auraR, 0, Math.PI * 2); ctx.fill();
      // Dark ring pulse
      ctx.globalAlpha = 0.1 + Math.sin(Date.now() * 0.02) * 0.05;
      ctx.strokeStyle = '#1a1a2e';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.arc(sp.x + 7 * scale, sp.y + 9 * scale, 14 + Math.sin(Date.now() * 0.03) * 4, 0, Math.PI * 2);
      ctx.stroke();
      ctx.globalAlpha = 1;
    }

    // Dash trail
    if (this.isDashing) {
      // Multi-colored ghost trail
      const dashColors = ['#4fc3f7','#29b6f6','#03a9f4','#0288d1','#01579b','#0d47a1'];
      for (let i = 7; i >= 0; i--) {
        ctx.globalAlpha = 0.18 - i * 0.02;
        const ci = Math.min(i, dashColors.length - 1);
        ctx.fillStyle = dashColors[ci];
        const gx = sp.x - this.dashVx * (i + 1) * 3;
        ctx.fillRect(gx - 2, sp.y - 2, 14 * scale + 4, 18 * scale + 4);
        drawSprite(ctx, SPRITES.playerIdle, gx, sp.y, scale, !this.facingRight);
      }
      // Speed lines (more)
      ctx.globalAlpha = 1;
      for (let i = 0; i < 8; i++) {
        ctx.globalAlpha = 0.06 + Math.random() * 0.14;
        ctx.fillStyle = ['#4fc3f7','#29b6f6','#ffffff'][Math.floor(Math.random() * 3)];
        const sl = 8 + Math.random() * 24;
        const sy = py - 12 - Math.random() * 24;
        ctx.fillRect(sp.x - this.dashVx * (4 + Math.random() * 3) - 3, sy, 5, sl);
      }
      // Blue energy trail
      ctx.globalAlpha = 0.08 + Math.sin(Date.now() * 0.02) * 0.04;
      ctx.fillStyle = '#4fc3f7';
      for (let i = 0; i < 5; i++) {
        const ex = sp.x - this.dashVx * (2 + Math.random() * 6);
        const ey = sp.y + Math.random() * (18 * scale);
        ctx.fillRect(ex - 1, ey, 3, 2 + Math.random() * 4);
      }
      ctx.globalAlpha = this.invFlash ? 0.5 : 1;
    }

    // Ultimate trail (blue energy)
    if (this.isUltimate) {
      // Ghost trail with electric colors
      const ultColors = ['#e3f2fd','#4fc3f7','#29b6f6','#0288d1','#01579b','#0d47a1'];
      for (let i = 6; i >= 0; i--) {
        ctx.globalAlpha = 0.3 - i * 0.04;
        const ci = Math.min(i, ultColors.length - 1);
        ctx.fillStyle = ultColors[ci];
        const gx = sp.x - this.ultimateVx * (i + 1) * 3;
        ctx.fillRect(gx - 3, sp.y - 3, 14 * scale + 6, 18 * scale + 6);
        drawSprite(ctx, SPRITES.playerIdle, gx, sp.y, scale, !this.facingRight);
      }
      // Blue glow aura (pulsing)
      ctx.globalAlpha = 0.25 + Math.sin(Date.now() * 0.015) * 0.1;
      ctx.fillStyle = '#0d47a1';
      ctx.fillRect(sp.x - 12, sp.y - 12, 14 * scale + 24, 18 * scale + 24);
      // Inner bright aura
      ctx.globalAlpha = 0.15 + Math.sin(Date.now() * 0.02) * 0.07;
      ctx.fillStyle = '#29b6f6';
      ctx.fillRect(sp.x - 6, sp.y - 6, 14 * scale + 12, 18 * scale + 12);
      // Electric sparks around body
      ctx.globalAlpha = 0.4 + Math.random() * 0.3;
      ctx.fillStyle = '#ffffff';
      for (let i = 0; i < 4; i++) {
        const ex = sp.x + Math.random() * (14 * scale);
        const ey = sp.y + Math.random() * (18 * scale);
        ctx.fillRect(ex, ey, 2 + Math.random() * 3, 1);
      }
      ctx.globalAlpha = this.invFlash ? 0.5 : 1;
    }

    // Dash dagger (thrown dagger → stuck → teleport)
    if (this.dashDagger) {
      const dx2 = Math.round(this.dashDagger.x + cam.x);
      const dy2 = Math.round(this.dashDagger.y + cam.y);
      if (this.dashDagger.stuck) {
        // Stuck in enemy — pulsing glow
        const pulse = 0.5 + Math.sin(Date.now() * 0.01) * 0.3;
        ctx.fillStyle = `rgba(79,195,247,${pulse * 0.3})`;
        ctx.fillRect(dx2 - 8, dy2 - 8, 16, 16);
        ctx.fillStyle = `rgba(255,255,255,${pulse * 0.5})`;
        ctx.fillRect(dx2 - 2, dy2 - 2, 4, 4);
        // Draw weapon sprite
        drawSprite(ctx, SPRITES.weapon, dx2 - 10, dy2 - 5, 1, false);
      } else {
        // Flying dagger — trail
        ctx.fillStyle = 'rgba(79,195,247,0.4)';
        ctx.fillRect(dx2 - 4, dy2 - 4, 8, 8);
        drawSprite(ctx, SPRITES.weapon, dx2 - 10, dy2 - 5, 1, this.dashDagger.vx < 0);
      }
    }

    // Choose sprite based on state
    let sprite = SPRITES.playerIdle;
    if (this.isDodging) {
      sprite = SPRITES.playerIdle;
    } else if (!this.isGrounded) {
      sprite = SPRITES.playerJump;
    } else if (this.attackActive > 0) {
      sprite = SPRITES.playerAttack;
    }

    // Draw sprite (semi-transparent during dodge)
    if (this.isDodging) ctx.globalAlpha = 0.35;
    drawSprite(ctx, sprite, sp.x, sp.y, scale, !this.facingRight);
    if (this.isDodging) ctx.globalAlpha = this.invFlash ? 0.5 : 1;

    // Dagger overlay (direction-aware)
    if (!this.isDodging && !this.isDashing) {
      if (this.attackActive > 0) {
        const combo = this.attackCombo;
        const prog = 1 - (this.attackActive / 5);
        const dir2 = this.facingRight ? 1 : -1;
        const colors = [
          { main: '#ff1744', dark: '#b71c1c', glow: '#ff8a80' },
          { main: '#ff5252', dark: '#d50000', glow: '#ff8a80' },
          { main: '#ff1744', dark: '#b71c1c', glow: '#ffffff' },
        ];
        const c = colors[combo - 1];
        const range = this.getAttackRange();
        const halfH = 15;
        const alpha = 0.9 - prog * 0.6;

        // Hitbox-aligned crossing slashes — wide × short
        ctx.save();
        ctx.translate(px, py - 14);
        ctx.scale(dir2, 1);

        // Multiple crossing diagonal lines
        const numLines = combo === 3 ? 7 : 5;
        ctx.lineCap = 'round';
        for (let li = 0; li < numLines; li++) {
          // Random start/end heights within hitbox
          const sy1 = - halfH + Math.random() * halfH * 2;
          const sy2 = - halfH + Math.random() * halfH * 2;
          const sx1 = 8;
          const sx2 = 8 + range;
          // Glow layer (wider, darker)
          ctx.globalAlpha = alpha * 0.25;
          ctx.strokeStyle = c.dark;
          ctx.lineWidth = 12 + combo * 4;
          ctx.beginPath();
          ctx.moveTo(sx1, sy1);
          ctx.lineTo(sx2, sy2);
          ctx.stroke();
          // Main line
          ctx.globalAlpha = alpha * 0.6;
          ctx.strokeStyle = c.main;
          ctx.lineWidth = 5 + combo * 2;
          ctx.beginPath();
          ctx.moveTo(sx1, sy1);
          ctx.lineTo(sx2, sy2);
          ctx.stroke();
          // Bright core
          ctx.globalAlpha = alpha * 0.8;
          ctx.strokeStyle = c.glow;
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.moveTo(dir2 > 0 ? sx1 : sx2, sy1);
          ctx.lineTo(dir2 > 0 ? sx2 : sx1, sy2);
          ctx.stroke();
        }
        // End-point burst sparks
        if (this.attackActive > 2) {
          ctx.globalAlpha = alpha * 0.7;
          for (let i = 0; i < 8 + combo * 4; i++) {
            const ex = 8 + range + (Math.random() - 0.5) * 20;
            const ey = (Math.random() - 0.5) * halfH * 2;
            const sz = 1 + Math.random() * 4;
            ctx.fillStyle = Math.random() > 0.5 ? '#ffffff' : c.glow;
            ctx.fillRect(ex, ey, sz, sz);
          }
        }
        ctx.globalAlpha = 1;
        ctx.restore();

        // Weapon sprite with glow
        const dx = f > 0 ? px + 2 : px - 2 - 20;
        const dy = py - 28;
        // Outer glow
        ctx.fillStyle = `rgba(255,255,255,${0.15 + Math.sin(Date.now() * 0.03) * 0.08})`;
        ctx.fillRect(dx - 3, dy - 2, 26, 14);
        // Blade trail
        ctx.fillStyle = `${c.main},${0.2})`;
        const trail = 24 + combo * 6;
        const tx = f > 0 ? px + 8 : px - 8 - trail;
        ctx.fillRect(tx, py - 22, trail, 3);
        // Draw weapon sprite
        drawSprite(ctx, SPRITES.weapon, dx, dy, 1, f < 0);
        // Tip sparkle (only during attack)
        ctx.fillStyle = `rgba(255,255,255,${0.3 + Math.random() * 0.4})`;
        const tipX = f > 0 ? dx + 18 : dx + 2;
        ctx.fillRect(tipX, dy + 3, 2, 2);
      } else {
        // Idle weapon (sheathed)
        const dx = f > 0 ? px + 8 : px - 8 - 16;
        const dy = py - 24;
        ctx.globalAlpha = 0.35;
        drawSprite(ctx, SPRITES.weapon, dx, dy, 1, f < 0);
        ctx.globalAlpha = this.invFlash ? 0.5 : 1;
      }
    }

    // Hit flash (white overlay on sprite)
    if (this.hitFlashTimer > 0) {
      ctx.fillStyle = 'rgba(255,255,255,0.45)';
      ctx.fillRect(sp.x, sp.y, 14 * scale, 18 * scale);
    }

    // Show summon ready glow around player
    if (this.hasSummon) {
      ctx.globalAlpha = 0.06 + Math.sin(Date.now() * 0.006) * 0.04;
      ctx.fillStyle = '#ce93d8';
      ctx.fillRect(sp.x - 6, sp.y - 6, 14 * scale + 12, 18 * scale + 12);
      ctx.globalAlpha = 1;
    }

    ctx.globalAlpha = 1;
  }
}

// ============================================================
// PROJECTILE CLASS
// ============================================================
class Projectile {
  constructor(x, y, vx, vy, damage, owner, life) {
    this.x = x; this.y = y;
    this.vx = vx; this.vy = vy;
    this.w = 8; this.h = 8;
    this.damage = damage;
    this.owner = owner;
    this.life = life || 60;
    this.alive = true;
    this.hit = false;
    this.starColor = '#4fc3f7';
  }

  get box() { return { x: this.x, y: this.y, w: this.w, h: this.h }; }

  update() {
    this.x += this.vx;
    this.y += this.vy;
    this.life--;
    if (this.life <= 0) this.alive = false;
    // Bounds check
    if (this.x < ARENA_LEFT - 50 || this.x > ARENA_RIGHT + 50 || this.y < ARENA_TOP - 50 || this.y > GROUND_Y + 10) {
      this.alive = false;
    }
  }

  draw(ctx, cam) {
    if (!this.alive) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    if (this.owner !== 'player') {
      ctx.fillStyle = COLORS.enemyProjectile;
      ctx.fillRect(px - this.w/2, py - this.h/2, this.w, this.h);
      ctx.fillStyle = 'rgba(255,82,82,0.3)';
      ctx.fillRect(px - this.w, py - this.h, this.w * 2, this.h * 2);
      return;
    }
    // Player throwing star - spectular rotating effect
    const rot = Date.now() * 0.012;
    const time = Date.now() * 0.003;
    const color = this.starColor || '#ce93d8';
    const s = 5;

    // Epic purple particle trail — long and massive
    const trailCount = 18;
    const speed = Math.sqrt(this.vx * this.vx + this.vy * this.vy);
    const trailSpread = 3 + speed * 0.15;
    for (let i = 0; i < trailCount; i++) {
      const t = i / trailCount;
      const back = (i + 1) * (2.5 + speed * 0.06);
      const tx = px - this.vx * back + (Math.random() - 0.5) * trailSpread;
      const ty = py - this.vy * back + (Math.random() - 0.5) * trailSpread;
      ctx.globalAlpha = (0.6 - t * 0.45) * (0.7 + Math.random() * 0.3);
      const trailColors = [color, '#ffd4fa', '#ffffff', '#e1bee7', '#7b1fa2', '#ce93d8', '#4a148c', '#ea80fc'];
      ctx.fillStyle = trailColors[i % trailColors.length];
      const ts = (8 - t * 6) * (0.6 + Math.random() * 0.8);
      ctx.fillRect(tx - ts/2, ty - ts/2, ts, ts);
      // Sparkles interleaved
      if (i % 2 === 1) {
        ctx.fillStyle = '#ffffff';
        ctx.fillRect(tx - 2, ty - 2, 4, 4);
      }
    }
    // Ghost after-image rings — bigger, more
    ctx.globalAlpha = 0.15;
    for (let i = 0; i < 5; i++) {
      const back = (i + 1) * (4 + speed * 0.08);
      const rx = px - this.vx * back + (Math.random() - 0.5) * 12;
      const ry = py - this.vy * back + (Math.random() - 0.5) * 12;
      ctx.strokeStyle = ['#ce93d8','#ab47bc','#e1bee7','#7b1fa2','#ffffff'][i];
      ctx.lineWidth = 2 + i * 0.3;
      ctx.globalAlpha = 0.15 - i * 0.02;
      ctx.beginPath(); ctx.arc(rx, ry, 6 + i * 4, 0, Math.PI * 2); ctx.stroke();
    }
    ctx.globalAlpha = 1;

    // Outer glow pulsing (purple, extra big)
    const glowSize = 32 + Math.sin(time * 5) * 8;
    const glow = ctx.createRadialGradient(px, py, 0, px, py, glowSize);
    glow.addColorStop(0, 'rgba(206,147,216,0.5)');
    glow.addColorStop(0.2, 'rgba(171,71,188,0.35)');
    glow.addColorStop(0.5, 'rgba(142,36,170,0.2)');
    glow.addColorStop(1, 'rgba(206,147,216,0)');
    ctx.fillStyle = glow;
    ctx.beginPath(); ctx.arc(px, py, glowSize, 0, Math.PI * 2); ctx.fill();

    // Purple inner glow — bigger
    ctx.globalAlpha = 0.4;
    ctx.fillStyle = '#ce93d8';
    ctx.beginPath(); ctx.arc(px, py, 18 + Math.sin(time * 4) * 4, 0, Math.PI * 2); ctx.fill();
    // White hot core
    ctx.globalAlpha = 0.6;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath(); ctx.arc(px, py, 6 + Math.sin(time * 6) * 2, 0, Math.PI * 2); ctx.fill();
    ctx.globalAlpha = 1;

    // Rotating star (4-pointed + diagonal)
    ctx.save();
    ctx.translate(px, py);
    ctx.rotate(rot);
    // Main blades
    ctx.fillStyle = color;
    ctx.fillRect(-2, -s - 2, 4, s * 2 + 4);
    ctx.fillRect(-s - 2, -2, s * 2 + 4, 4);
    // Diagonal blades
    ctx.fillStyle = '#ffffff';
    const d = s * 0.8;
    ctx.fillRect(-1.5, -d - 1, 3, d * 2 + 2);
    ctx.fillRect(-d - 1, -1.5, d * 2 + 2, 3);
    // Secondary points
    ctx.fillStyle = 'rgba(255,255,255,0.4)';
    for (let i = 0; i < 4; i++) {
      const a = (Math.PI / 4) + (Math.PI / 2) * i;
      ctx.fillRect(Math.cos(a) * (s + 3) - 1, Math.sin(a) * (s + 3) - 1, 3, 3);
    }
    ctx.restore();

    // Bright core
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.arc(px, py, 3 + Math.sin(time * 5) * 1, 0, Math.PI * 2);
    ctx.fill();
  }
}

let projectiles = [];

// ============================================================
// MINION CLASS
// ============================================================
class Minion {
  constructor(x, y) {
    this.x = x; this.y = y;
    this.w = 18; this.h = 22;
    this.vx = 0; this.vy = 0;
    this.hp = 20; this.maxHp = 20;
    this.speed = 1.8;
    this.damage = 5;
    this.originX = x;
    this.patrolDir = Math.random() < 0.5 ? 1 : -1;
    this.patrolRange = 60;
    this.agroRange = 250;
    this.alive = true;
    this.isGrounded = false;
    this.facingRight = true;
    this.hitFlashTimer = 0;
    this.aggro = false;
    this.attackCooldown = 0;
    this.slowTimer = 0;
    this.owner = 'enemy';
  }

  get box() { return { x: this.x, y: this.y, w: this.w, h: this.h }; }

  update(player, boss) {
    if (!this.alive) return;
    if (this.hitFlashTimer > 0) this.hitFlashTimer--;
    if (this.slowTimer > 0) this.slowTimer--;

    this.vy += GRAVITY;
    if (this.vy > 10) this.vy = 10;

    this.attackCooldown--;

    // Target selection: player minions chase boss, enemy minions chase player
    let target;
    if (this.owner === 'player' && boss && boss.alive) {
      target = boss;
    } else {
      target = player;
    }
    const dx = target.x - this.x;
    const dy = target.y - this.y;
    this.aggro = (dx * dx + dy * dy) < this.agroRange * this.agroRange;

    if (this.aggro && dy < 100) {
      // Chase player
      const dir = dx > 0 ? 1 : -1;
      const speedMod = this.slowTimer > 0 ? 0.35 : 1;
      this.vx = dir * this.speed * 1.3 * speedMod;
      this.facingRight = dir > 0;
    } else {
      // Patrol
      const speedMod = this.slowTimer > 0 ? 0.35 : 1;
      this.vx = this.patrolDir * this.speed * 0.6 * speedMod;
      if (Math.abs(this.x - this.originX) > this.patrolRange) {
        this.patrolDir *= -1;
      }
      this.facingRight = this.patrolDir > 0;
    }

    this.x += this.vx;
    this.y += this.vy;

    // Ground
    if (this.y >= GROUND_Y) {
      this.y = GROUND_Y;
      this.vy = 0;
      this.isGrounded = true;
    } else {
      this.isGrounded = false;
    }

    // Arena bound
    this.x = Math.max(ARENA_LEFT + this.w/2, Math.min(this.x, ARENA_RIGHT - this.w/2));
  }

  takeDamage(dmg, knockbackX) {
    this.hp -= dmg;
    this.hitFlashTimer = 5;
    this.vx = knockbackX * 3;
    spawnParticles(this.x, this.y, 6, COLORS.particleHit, 3, 2, 15);
    if (this.hp <= 0) {
      this.alive = false;
      spawnParticles(this.x, this.y, 12, COLORS.minion, 4, 3, 25);
    }
  }

  draw(ctx, cam) {
    if (!this.alive) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    const scale = 2;
    const sp = { x: px - (12 * scale / 2), y: py - (14 * scale) };

    // Aggro glow
    if (this.aggro) {
      ctx.fillStyle = 'rgba(255, 145, 0, 0.12)';
      ctx.fillRect(sp.x - 4, sp.y - 4, 12 * scale + 8, 14 * scale + 8);
    }

    // Slow aura
    if (this.slowTimer > 0) {
      ctx.fillStyle = `rgba(41, 182, 246, ${0.08 + Math.sin(Date.now() * 0.01) * 0.04})`;
      ctx.fillRect(sp.x - 4, sp.y - 4, 12 * scale + 8, 14 * scale + 8);
    }

    // Draw sprite
    drawSprite(ctx, SPRITES.minionIdle, sp.x, sp.y, scale, !this.facingRight);

    // Hit flash
    if (this.hitFlashTimer > 0) {
      ctx.fillStyle = 'rgba(255,255,255,0.4)';
      ctx.fillRect(sp.x, sp.y, 12 * scale, 14 * scale);
    }
  }
}

let minions = [];

// ============================================================
// BOSS CLASS
// ============================================================
class Boss {
  constructor(x, y) {
    this.x = x; this.y = y;
    this.w = 56; this.h = 64;
    this.vx = 0; this.vy = 0;
    this.hp = 200; this.maxHp = 200;
    this.speed = 2.2;
    this.facingRight = false;
    this.alive = true;
    this.isGrounded = false;
    this.hitFlashTimer = 0;

    // AI
    this.state = 'idle'; // idle, slash_windup, slash_active, slash_recovery, charge_windup, charge_active, charge_stop, summon
    this.stateTimer = 0;
    this.patternCooldown = 0;
    this.lastPattern = null;

    // Slash
    this.slashWindup = 28;
    this.slashActive = 10;
    this.slashRecovery = 15;

    // Charge
    this.chargeWindup = 22;
    this.chargeDuration = 35;
    this.chargeSpeed = 10;
    this.chargeDir = 0;

    // Summon
    this.summonCooldown = 0;
    this.canSummon = false;

    // Enrage
    this.isEnraged = false;

    // Slow
    this.slowTimer = 0;

    // Intro
    this.introTimer = 60;

    // hurtbox flash
    this.hitFlash = false;
  }

  get box() { return { x: this.x, y: this.y, w: this.w, h: this.h }; }

  getSpeedMult() {
    let mult = this.isEnraged ? 1.5 : 1;
    if (this.slowTimer > 0) mult *= 0.35;
    return mult;
  }

  update(player) {
    if (!this.alive) return;

    if (this.introTimer > 0) {
      this.introTimer--;
      return;
    }

    if (this.hitFlashTimer > 0) this.hitFlashTimer--;
    if (this.slowTimer > 0) this.slowTimer--;

    // Phase check
    const hpRatio = this.hp / this.maxHp;
    this.canSummon = hpRatio <= 0.5;
    if (hpRatio <= 0.2 && !this.isEnraged) {
      this.isEnraged = true;
      spawnParticles(this.x, this.y, 20, COLORS.bossAccent, 5, 4, 30);
    }

    if (this.summonCooldown > 0) this.summonCooldown--;

    // Gravity
    if (!this.isGrounded) {
      this.vy += GRAVITY * 0.8;
      this.y += this.vy;
    }
    if (this.y >= GROUND_Y) {
      this.y = GROUND_Y;
      this.vy = 0;
      this.isGrounded = true;
    }

    // Arena clamp
    this.x = Math.max(ARENA_LEFT + this.w/2, Math.min(this.x, ARENA_RIGHT - this.w/2));

    // State machine
    this.stateTimer--;

    switch (this.state) {
      case 'idle':
        this.vx *= 0.9;
        this.face(player);
        if (this.patternCooldown > 0) { this.patternCooldown--; break; }

        // Decide next pattern
        const patterns = ['slash'];
        if (this.canSummon && this.summonCooldown <= 0) patterns.push('summon');
        patterns.push('charge');

        let chosen = patterns[Math.floor(Math.random() * patterns.length)];
        if (chosen === this.lastPattern) {
          // Try different pattern
          const alt = patterns.filter(p => p !== chosen);
          if (alt.length > 0) chosen = alt[Math.floor(Math.random() * alt.length)];
        }
        this.lastPattern = chosen;
        this.state = chosen + '_windup';
        this.stateTimer = chosen === 'slash' ? this.slashWindup : (chosen === 'charge' ? this.chargeWindup : 10);
        if (chosen === 'summon') {
          this.state = 'summon';
          this.stateTimer = 20;
        }
        break;

      case 'slash_windup':
        this.vx *= 0.9;
        this.face(player);
        if (this.stateTimer <= 0) {
          this.state = 'slash_active';
          this.stateTimer = this.slashActive;
        }
        break;

      case 'slash_active':
        if (this.stateTimer <= 0) {
          this.state = 'slash_recovery';
          this.stateTimer = this.slashRecovery;
        }
        break;

      case 'slash_recovery':
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 40 + Math.floor(Math.random() * 40);
        }
        break;

      case 'charge_windup':
        this.vx *= 0.9;
        this.face(player);
        this.chargeDir = player.x > this.x ? 1 : -1;
        if (this.stateTimer <= 0) {
          this.state = 'charge_active';
          this.stateTimer = this.chargeDuration;
          spawnParticles(this.x, this.y, 8, COLORS.bossAccent, 3, 3, 15);
        }
        break;

      case 'charge_active':
        this.vx = this.chargeDir * this.chargeSpeed * this.getSpeedMult();
        this.x += this.vx;
        // Check wall collision
        if (this.x <= ARENA_LEFT + this.w/2 || this.x >= ARENA_RIGHT - this.w/2) {
          this.state = 'charge_stop';
          this.stateTimer = 15;
          spawnParticles(this.x, this.y, 10, COLORS.bossAccent, 4, 3, 20);
          triggerShake(3, 80);
        }
        if (this.stateTimer <= 0) {
          this.state = 'charge_stop';
          this.stateTimer = 15;
        }
        break;

      case 'charge_stop':
        this.vx *= 0.7;
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 30 + Math.floor(Math.random() * 30);
        }
        break;

      case 'summon':
        if (this.stateTimer <= 0) {
          this.spawnMinions();
          this.summonCooldown = 480; // 8s
          this.state = 'idle';
          this.patternCooldown = 30;
        }
        break;
    }

    // Natural movement in idle
    if (this.state === 'idle') {
      const dist = player.x - this.x;
      if (Math.abs(dist) > 150) {
        this.vx += Math.sign(dist) * 0.15;
        this.vx = Math.max(-this.speed * 0.6, Math.min(this.vx, this.speed * 0.6));
      }
    }

    this.x += this.vx;
    this.face(player);

    // Check player attack hit
    this.checkPlayerHit(player);
  }

  face(player) {
    this.facingRight = player.x > this.x;
  }

  checkPlayerHit(player) {
    if (player.attackActive > 0 && !player.attackHit) {
      const range = player.getAttackRange();
      const dir = player.facingRight ? 1 : -1;
      const attackBox = {
        x: player.x + dir * range / 2,
        y: player.y - 10,
        w: range,
        h: 30
      };
      if (rectCollide({ x: attackBox.x, y: attackBox.y, w: attackBox.w, h: attackBox.h }, this.box)) {
        this.takeDamage(player.getAttackDamage(), player.facingRight ? 1 : -1);
        player.attackHit = true;
      }
    }

    // Shadow dash hit
    if (player.isDashing && !player.dashHit) {
      if (rectCollide(this.box, player.box)) {
        this.takeDamage(18, player.facingRight ? 1 : -1);
        player.dashHit = true;
      }
    }

    // Projectile hits
    for (const p of projectiles) {
      if (p.alive && p.owner === 'player' && !p.hit) {
        if (rectCollide(this.box, p.box)) {
          this.takeDamage(p.damage, p.vx > 0 ? 1 : -1);
          p.hit = true;
          p.alive = false;
          spawnParticles(p.x, p.y, 6, COLORS.particleHit, 3, 2, 15);
        }
      }
    }
  }

  takeDamage(dmg, dir) {
    this.hp -= dmg;
    this.hitFlashTimer = 5;
    this.hitFlash = true;
    this.vx += dir * 2 * (this.isEnraged ? 0.5 : 1);
    spawnParticles(this.x, this.y, 8, COLORS.particleBoss, 3, 3, 18);
    triggerShake(2, 60);
    if (this.hp <= 0) {
      this.hp = 0;
      this.alive = false;
      spawnParticles(this.x, this.y, 40, COLORS.bossAccent, 6, 5, 40);
      triggerShake(8, 300);
    }
  }

  spawnMinions() {
    for (let i = 0; i < 2; i++) {
      const spawnX = this.x + (i === 0 ? -60 : 60);
      minions.push(new Minion(spawnX, this.y - 20));
    }
    spawnParticles(this.x, this.y, 15, COLORS.bossAccent, 4, 3, 20);
  }

  draw(ctx, cam) {
    if (!this.alive) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    const f = this.facingRight ? 1 : -1;
    const scale = 3;
    const sp = { x: px - (16 * scale / 2), y: py - (20 * scale) };

    // Intro glow
    if (this.introTimer > 0) {
      ctx.fillStyle = `rgba(206, 147, 216, ${0.2 + Math.sin(this.introTimer * 0.2) * 0.15})`;
      ctx.fillRect(sp.x - 12, sp.y - 12, 16 * scale + 24, 20 * scale + 24);
    }

    // Enrage glow
    if (this.isEnraged) {
      ctx.fillStyle = 'rgba(255, 23, 68, 0.15)';
      ctx.fillRect(sp.x - 8, sp.y - 8, 16 * scale + 16, 20 * scale + 16);
    }

    // Slow aura
    if (this.slowTimer > 0) {
      ctx.fillStyle = `rgba(41, 182, 246, ${0.08 + Math.sin(Date.now() * 0.01) * 0.04})`;
      ctx.fillRect(sp.x - 8, sp.y - 8, 16 * scale + 16, 20 * scale + 16);
    }

    // Shadow
    ctx.fillStyle = 'rgba(0,0,0,0.3)';
    ctx.fillRect(px - 24, Math.round(GROUND_Y + cam.y) - 2, 48, 4);

    // Choose sprite based on state
    let sprite = SPRITES.bossIdle;
    if (this.state === 'slash_windup') sprite = SPRITES.bossWindup;
    else if (this.state === 'slash_active') sprite = SPRITES.bossSlash;
    else if (this.state === 'charge_windup' || this.state === 'charge_active') sprite = SPRITES.bossCharge;

    // Draw sprite
    drawSprite(ctx, sprite, sp.x, sp.y, scale, !this.facingRight);

    // Sword overlay (direction-aware)
    ctx.fillStyle = '#ce93d8';
    if (this.state === 'slash_windup') {
      const bx = f > 0 ? px + 16 : px - 16 - 8;
      ctx.fillRect(bx, py - 56, 8, 28);
      const gx = f > 0 ? px + 14 : px - 14 - 12;
      ctx.fillRect(gx, py - 58, 12, 6);
    } else if (this.state === 'slash_active') {
      const sx = f > 0 ? px + 24 : px - 24 - 18;
      ctx.fillRect(sx, py - 22, 18, 5);
      ctx.fillStyle = 'rgba(206,147,216,0.3)';
      const ex = f > 0 ? px + 22 : px - 22 - 26;
      ctx.fillRect(ex, py - 26, 26, 14);
    } else {
      const bx = f > 0 ? px + 22 : px - 22 - 6;
      ctx.fillRect(bx, py - 40, 6, 24);
    }

    // Charge windup glow
    if (this.state === 'charge_windup') {
      ctx.fillStyle = `rgba(255, 23, 68, ${0.3 + Math.sin(this.stateTimer * 0.4) * 0.2})`;
      ctx.fillRect(sp.x - 6, sp.y - 6, 16 * scale + 12, 20 * scale + 12);
    }

    // Charge trail
    if (this.state === 'charge_active') {
      for (let i = 2; i >= 0; i--) {
        ctx.globalAlpha = 0.1 - i * 0.03;
        drawSprite(ctx, sprite, sp.x - this.vx * (i + 1) * 3, sp.y, scale, !this.facingRight);
      }
      ctx.globalAlpha = 1;
    }

    // Hit flash
    if (this.hitFlashTimer > 0) {
      ctx.fillStyle = 'rgba(255,255,255,0.3)';
      ctx.fillRect(sp.x, sp.y, 16 * scale, 20 * scale);
    }
  }
}

// ============================================================
// STAGE 2 BOSS - Big Guardian (stationary)
// ============================================================
class Stage2Boss {
  constructor(x, y) {
    this.x = x; this.y = y;
    this.w = 80; this.h = 90;
    this.hp = 500; this.maxHp = 500;
    this.alive = true;
    this.facingRight = true;
    this.hitFlashTimer = 0;
    this.slowTimer = 0;

    // State
    this.state = 'idle'; // idle, slam_telegraph, slam_active, slam_recovery, laser_telegraph, laser_active, laser_recovery
    this.stateTimer = 0;
    this.patternCooldown = 0;

    // Slam attack
    this.slamTelegraphFrames = 30;
    this.slamActiveFrames = 12;
    this.slamRecoveryFrames = 18;
    this.slamZone = { x: 0, w: 140 };

    // Laser attack
    this.laserTelegraphFrames = 35;
    this.laserActiveFrames = 22;
    this.laserRecoveryFrames = 15;
    this.laserY = 0;
  }

  get box() { return { x: this.x, y: this.y, w: this.w, h: this.h }; }

  update(player) {
    if (!this.alive) return;
    if (this.hitFlashTimer > 0) this.hitFlashTimer--;
    if (this.slowTimer > 0) this.slowTimer--;
    if (this.patternCooldown > 0) this.patternCooldown--;

    // Fixed position at center
    this.x = (ARENA_LEFT + ARENA_RIGHT) / 2;
    this.y = GROUND_Y;

    this.stateTimer--;

    switch (this.state) {
      case 'idle': {
        if (this.patternCooldown > 0) break;
        const patterns = ['slam', 'laser'];
        const chosen = patterns[Math.floor(Math.random() * patterns.length)];
        if (chosen === 'slam') {
          this.state = 'slam_telegraph';
          this.stateTimer = this.slamTelegraphFrames;
          const zoneW = this.slamZone.w;
          this.slamZone.x = ARENA_LEFT + 30 + Math.random() * (ARENA_RIGHT - ARENA_LEFT - zoneW - 60);
        } else {
          this.state = 'laser_telegraph';
          this.stateTimer = this.laserTelegraphFrames;
          this.laserY = GROUND_Y - 15;
        }
        break;
      }

      case 'slam_telegraph': {
        if (this.stateTimer <= 0) {
          this.state = 'slam_active';
          this.stateTimer = this.slamActiveFrames;
          triggerShake(8, 200);
        }
        break;
      }

      case 'slam_active': {
        if (!player.isDead && !player.invincible) {
          const zoneBox = { x: this.slamZone.x + this.slamZone.w/2, y: GROUND_Y - 10, w: this.slamZone.w, h: 20 };
          if (rectCollide(player.box, zoneBox)) {
            player.takeDamage(20);
          }
        }
        if (this.stateTimer <= 0) {
          this.state = 'slam_recovery';
          this.stateTimer = this.slamRecoveryFrames;
        }
        break;
      }

      case 'slam_recovery': {
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 40 + Math.floor(Math.random() * 30);
        }
        break;
      }

      case 'laser_telegraph': {
        if (this.stateTimer <= 0) {
          this.state = 'laser_active';
          this.stateTimer = this.laserActiveFrames;
          triggerShake(4, 150);
        }
        break;
      }

      case 'laser_active': {
        if (!player.isDead && !player.invincible && player.isGrounded) {
          const laserBox = { x: (ARENA_LEFT + ARENA_RIGHT) / 2, y: GROUND_Y - 5, w: ARENA_RIGHT - ARENA_LEFT - 40, h: 12 };
          if (rectCollide(player.box, laserBox)) {
            player.takeDamage(25);
          }
        }
        if (this.stateTimer <= 0) {
          this.state = 'laser_recovery';
          this.stateTimer = this.laserRecoveryFrames;
        }
        break;
      }

      case 'laser_recovery': {
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 50 + Math.floor(Math.random() * 30);
        }
        break;
      }
    }

    // Check player attack hits (same logic as Boss)
    this.checkPlayerHit(player);
  }

  checkPlayerHit(player) {
    if (player.attackActive > 0 && !player.attackHit) {
      const range = player.getAttackRange();
      const dir = player.facingRight ? 1 : -1;
      const attackBox = {
        x: player.x + dir * range / 2,
        y: player.y - 10,
        w: range,
        h: 30
      };
      if (rectCollide(attackBox, this.box)) {
        this.takeDamage(player.getAttackDamage());
        player.attackHit = true;
      }
    }
    if (player.isDashing && !player.dashHit) {
      if (rectCollide(this.box, player.box)) {
        this.takeDamage(18);
        player.dashHit = true;
      }
    }
    for (const p of projectiles) {
      if (p.alive && p.owner === 'player' && !p.hit) {
        if (rectCollide(this.box, p.box)) {
          this.takeDamage(p.damage);
          p.hit = true;
          p.alive = false;
          spawnParticles(p.x, p.y, 6, COLORS.particleHit, 3, 2, 15);
        }
      }
    }
    // Ultimate
    if (player.isUltimate && !player.ultimateHit) {
      const dir = player.facingRight ? 1 : -1;
      const ultBox = {
        x: player.x + dir * 30,
        y: player.y - 10,
        w: 60,
        h: 40
      };
      if (rectCollide(ultBox, this.box)) {
        this.takeDamage(50);
        player.ultimateHit = true;
        spawnParticles(player.x, player.y, 30, '#ffffff', 8, 5, 30);
        screenFlash = 15;
        screenFlashColor = '#ffffff';
      }
    }
  }

  takeDamage(dmg) {
    this.hp -= dmg;
    this.hitFlashTimer = 5;
    spawnParticles(this.x, this.y, 8, COLORS.particleBoss, 3, 3, 18);
    triggerShake(2, 60);
    if (this.hp <= 0) {
      this.hp = 0;
      this.alive = false;
      spawnParticles(this.x, this.y, 50, '#ce93d8', 7, 6, 45);
      triggerShake(10, 400);
    }
  }

  draw(ctx, cam) {
    if (!this.alive) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    const scale = 3;
    const sp = { x: px - (32 * scale / 2), y: py - (36 * scale) };

    // Slow aura
    if (this.slowTimer > 0) {
      ctx.fillStyle = `rgba(41,182,246,${0.08 + Math.sin(Date.now() * 0.01) * 0.04})`;
      ctx.fillRect(sp.x - 10, sp.y - 10, 32 * scale + 20, 36 * scale + 20);
    }

    // Big shadow
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(px - 48, Math.round(GROUND_Y + cam.y) - 2, 96, 8);

    // Draw sprite
    drawSprite(ctx, SPRITES.boss2Idle, sp.x, sp.y, scale, false);

    // Hit flash
    if (this.hitFlashTimer > 0) {
      ctx.fillStyle = 'rgba(255,255,255,0.3)';
      ctx.fillRect(sp.x, sp.y, 32 * scale, 36 * scale);
    }

    // ---- TELEGRAPH & ATTACK VISUALS ----
    const prog = this.stateTimer / (this.state === 'slam_telegraph' ? this.slamTelegraphFrames : 1);

    // Slam telegraph — red zone on ground
    if (this.state === 'slam_telegraph') {
      const pulse = 0.3 + Math.sin(this.stateTimer * 0.3) * 0.2;
      ctx.fillStyle = `rgba(255,0,0,${pulse})`;
      const zx = Math.round(this.slamZone.x + cam.x);
      const zy = Math.round(GROUND_Y + cam.y - 4);
      ctx.fillRect(zx, zy, this.slamZone.w, 8);
      ctx.strokeStyle = `rgba(255,50,50,${pulse + 0.2})`;
      ctx.lineWidth = 2;
      ctx.strokeRect(zx - 1, zy - 1, this.slamZone.w + 2, 10);
      // Danger icon
      ctx.fillStyle = `rgba(255,200,0,${0.5 + Math.sin(this.stateTimer * 0.4) * 0.3})`;
      ctx.font = '16px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('⚠', zx + this.slamZone.w / 2, zy - 8);
      ctx.textAlign = 'left';
    }

    // Slam active — impact effect
    if (this.state === 'slam_active') {
      ctx.fillStyle = `rgba(255,200,100,${0.8 - (1 - prog) * 0.6})`;
      const zx = Math.round(this.slamZone.x + cam.x);
      const zy = Math.round(GROUND_Y + cam.y);
      ctx.fillRect(zx - 10, zy - 4, this.slamZone.w + 20, 14);
      // Dust particles
      if (Math.random() < 0.4) {
        spawnParticles(this.slamZone.x + this.slamZone.w / 2, GROUND_Y, 5, '#8d6e63', 4, 3, 12);
      }
    }

    // Laser telegraph — beam warning line
    if (this.state === 'laser_telegraph') {
      const pulse = 0.2 + Math.sin(this.stateTimer * 0.25) * 0.15;
      ctx.fillStyle = `rgba(255,0,0,${pulse})`;
      const ly = Math.round(this.laserY + cam.y);
      ctx.fillRect(ARENA_LEFT + Math.round(cam.x) - 10, ly, ARENA_RIGHT - ARENA_LEFT + 20, 10);
      ctx.fillStyle = `rgba(255,50,50,${pulse + 0.1})`;
      ctx.fillRect(ARENA_LEFT + Math.round(cam.x) - 10, ly + 1, ARENA_RIGHT - ARENA_LEFT + 20, 2);
      // Warning text
      ctx.fillStyle = `rgba(255,100,100,${0.4 + Math.sin(this.stateTimer * 0.3) * 0.2})`;
      ctx.font = 'bold 11px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('JUMP!', px, ly - 14);
      ctx.textAlign = 'left';
    }

    // Laser active — red beam
    if (this.state === 'laser_active') {
      const beamAlpha = 0.6 + Math.sin(Date.now() * 0.02) * 0.3;
      ctx.fillStyle = `rgba(255,0,0,${beamAlpha})`;
      const ly = Math.round(this.laserY + cam.y);
      ctx.fillRect(ARENA_LEFT + Math.round(cam.x) - 10, ly, ARENA_RIGHT - ARENA_LEFT + 20, 12);
      // Bright core
      ctx.fillStyle = `rgba(255,200,200,${beamAlpha * 0.6})`;
      ctx.fillRect(ARENA_LEFT + Math.round(cam.x) - 10, ly + 3, ARENA_RIGHT - ARENA_LEFT + 20, 4);
      // Overlay glow
      ctx.fillStyle = `rgba(255,0,0,${0.08 + Math.sin(Date.now() * 0.01) * 0.04})`;
      ctx.fillRect(0, ly - 20, CANVAS_W, 50);
    }
  }
}

// ============================================================
// GRIM REAPER - Stage 3 Final Boss
// ============================================================
class GrimReaper {
  constructor(x, y) {
    this.x = x; this.y = y;
    this.w = 48; this.h = 60;
    this.hp = 450; this.maxHp = 450;
    this.vx = 0; this.vy = 0;
    this.facingRight = false;
    this.alive = true;
    this.isGrounded = true;
    this.hitFlashTimer = 0;
    this.slowTimer = 0;
    this.invulnerable = 0;

    // AI
    this.state = 'idle';
    this.stateTimer = 0;
    this.patternCooldown = 0;
    this.lastPattern = null;

    // Slash
    this.slashWindup = 30;
    this.slashActive = 12;
    this.slashRecovery = 20;

    // Scythe Charge
    this.chargeWindup = 25;
    this.chargeDuration = 35;
    this.chargeSpeed = 11;
    this.chargeDir = 0;

    // Aerial Slam
    this.slamWindup = 18;
    this.slamRise = 25;
    this.slamFall = 20;
    this.slamRecovery = 25;
    this.slamTargetX = 0;
    this.slamPeakY = 0;

    // Enrage
    this.isEnraged = false;

    // Intro
    this.introTimer = 50;
  }

  get box() { return { x: this.x, y: this.y - this.h / 2, w: this.w, h: this.h }; }

  getSpeedMult() {
    let mult = this.isEnraged ? 1.4 : 1;
    if (this.slowTimer > 0) mult *= 0.35;
    return mult;
  }

  update(player) {
    if (!this.alive) return;
    if (this.introTimer > 0) { this.introTimer--; return; }
    if (this.hitFlashTimer > 0) this.hitFlashTimer--;
    if (this.slowTimer > 0) this.slowTimer--;
    if (this.invulnerable > 0) this.invulnerable--;

    // Enrage at 30% HP
    const hpRatio = this.hp / this.maxHp;
    if (hpRatio <= 0.3 && !this.isEnraged) {
      this.isEnraged = true;
      spawnParticles(this.x, this.y - 20, 30, '#ff1744', 6, 5, 35);
      triggerShake(8, 300);
    }

    // Gravity
    if (!this.isGrounded) {
      this.vy += GRAVITY * 0.7;
      this.y += this.vy;
    }
    if (this.y >= GROUND_Y) {
      this.y = GROUND_Y;
      this.vy = 0;
      this.isGrounded = true;
    }
    this.x = Math.max(ARENA_LEFT + this.w / 2, Math.min(this.x, ARENA_RIGHT - this.w / 2));
    this.stateTimer--;

    switch (this.state) {
      case 'idle':
        this.vx *= 0.9;
        this.face(player);
        if (this.patternCooldown > 0) { this.patternCooldown--; break; }

        const patterns = ['slash', 'charge', 'slam'];
        let chosen = patterns[Math.floor(Math.random() * patterns.length)];
        if (chosen === this.lastPattern) {
          const alt = patterns.filter(p => p !== chosen);
          if (alt.length > 0) chosen = alt[Math.floor(Math.random() * alt.length)];
        }
        this.lastPattern = chosen;
        if (chosen === 'slash') {
          this.state = 'slash_windup';
          this.stateTimer = this.slashWindup;
        } else if (chosen === 'charge') {
          this.state = 'charge_windup';
          this.stateTimer = this.chargeWindup;
        } else {
          this.state = 'slam_windup';
          this.stateTimer = this.slamWindup;
          this.slamTargetX = player.x;
        }
        break;

      // ---- SLASH ----
      case 'slash_windup':
        this.vx *= 0.9;
        this.face(player);
        if (this.stateTimer <= 0) {
          this.state = 'slash_active';
          this.stateTimer = this.slashActive;
          triggerShake(4, 120);
        }
        break;
      case 'slash_active':
        if (this.stateTimer <= 0) {
          this.state = 'slash_recovery';
          this.stateTimer = this.slashRecovery;
        }
        break;
      case 'slash_recovery':
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 50 + Math.floor(Math.random() * 30);
        }
        break;

      // ---- CHARGE ----
      case 'charge_windup':
        this.vx *= 0.9;
        this.face(player);
        this.chargeDir = player.x > this.x ? 1 : -1;
        if (this.stateTimer <= 0) {
          this.state = 'charge_active';
          this.stateTimer = this.chargeDuration;
          spawnParticles(this.x, this.y - 10, 10, '#ff1744', 4, 3, 18);
          triggerShake(2, 60);
        }
        break;
      case 'charge_active':
        this.vx = this.chargeDir * this.chargeSpeed * this.getSpeedMult();
        this.x += this.vx;
        // Scythe sweep particles
        if (Math.random() < 0.3) {
          spawnParticles(this.x + this.chargeDir * 20, this.y - 15, 3, '#e0e0e0', 3, 2, 8);
        }
        if (this.x <= ARENA_LEFT + this.w / 2 || this.x >= ARENA_RIGHT - this.w / 2) {
          this.state = 'charge_stop';
          this.stateTimer = 15;
          spawnParticles(this.x, this.y - 10, 15, '#ff1744', 5, 4, 22);
          triggerShake(5, 150);
        }
        if (this.stateTimer <= 0) {
          this.state = 'charge_stop';
          this.stateTimer = 15;
        }
        break;
      case 'charge_stop':
        this.vx *= 0.7;
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 40 + Math.floor(Math.random() * 30);
        }
        break;

      // ---- AERIAL SLAM ----
      case 'slam_windup':
        this.vx *= 0.9;
        this.face(player);
        if (this.stateTimer <= 0) {
          this.state = 'slam_rise';
          this.stateTimer = this.slamRise;
          this.vy = -9;
          this.isGrounded = false;
          this.slamPeakY = this.y - 160;
          spawnParticles(this.x, this.y, 10, '#2a2a3e', 4, 3, 15);
        }
        break;
      case 'slam_rise':
        // Slowed upward arc
        if (this.stateTimer <= 0) {
          this.state = 'slam_fall';
          this.stateTimer = this.slamFall;
          this.vy = 0;
        }
        break;
      case 'slam_fall':
        this.vy = Math.min(this.vy + GRAVITY * 0.8, 16);
        if (this.isGrounded) {
          // Impact!
          this.state = 'slam_recovery';
          this.stateTimer = this.slamRecovery;
          triggerShake(12, 350);
          screenFlash = 10;
          screenFlashColor = '#ff1744';
          spawnParticles(this.x, this.y, 40, '#e0e0e0', 8, 6, 30);
          spawnParticles(this.x, this.y, 25, '#ff1744', 7, 5, 25);
          // Shockwave hitbox
          for (const m of minions) {
            if (m.alive && Math.abs(m.x - this.x) < 120) {
              m.takeDamage(40, this.x > m.x ? 1 : -1);
            }
          }
          if (!player.invincible && Math.abs(player.x - this.x) < 120) {
            player.takeDamage(30);
            player.vx += (player.x > this.x ? 1 : -1) * 8;
          }
        }
        break;
      case 'slam_recovery':
        if (this.stateTimer <= 0) {
          this.state = 'idle';
          this.patternCooldown = 60 + Math.floor(Math.random() * 30);
        }
        break;
    }

    // Natural movement in idle
    if (this.state === 'idle') {
      const dist = player.x - this.x;
      if (Math.abs(dist) > 180) {
        this.vx += Math.sign(dist) * 0.12;
        this.vx = Math.max(-4, Math.min(this.vx, 4));
      }
    }

    this.x += this.vx;
    this.face(player);
    this.checkPlayerHit(player);
  }

  face(player) { this.facingRight = player.x > this.x; }

  checkPlayerHit(player) {
    // Slash hit
    if (this.state === 'slash_active') {
      const dir = this.facingRight ? 1 : -1;
      const slashBox = {
        x: this.x + dir * 40,
        y: this.y - 25,
        w: 70,
        h: 50
      };
      if (!player.invincible && rectCollide(player.box, slashBox)) {
        player.takeDamage(this.isEnraged ? 22 : 16);
      }
    }
    // Charge hit
    if (this.state === 'charge_active') {
      if (!player.invincible && rectCollide(this.box, player.box)) {
        player.takeDamage(this.isEnraged ? 20 : 14);
        player.vx += this.chargeDir * 6;
      }
    }
    // Player attacks hit reaper
    if (player.attackActive > 0 && !player.attackHit) {
      const range = player.getAttackRange();
      const dir = player.facingRight ? 1 : -1;
      const attackBox = { x: player.x + dir * range / 2, y: player.y - 10, w: range, h: 30 };
      if (rectCollide(attackBox, this.box)) {
        this.takeDamage(player.getAttackDamage(), dir);
        player.attackHit = true;
      }
    }
    if (player.isDashing && !player.dashHit) {
      if (rectCollide(this.box, player.box)) {
        this.takeDamage(18, player.facingRight ? 1 : -1);
        player.dashHit = true;
      }
    }
    for (const p of projectiles) {
      if (p.alive && p.owner === 'player' && !p.hit) {
        if (rectCollide(this.box, p.box)) {
          this.takeDamage(p.damage, p.vx > 0 ? 1 : -1);
          p.hit = true;
          p.alive = false;
          spawnParticles(p.x, p.y, 6, COLORS.particleHit, 3, 2, 15);
        }
      }
    }
    if (player.isUltimate && !player.ultimateHit) {
      const dir = player.facingRight ? 1 : -1;
      const ultBox = { x: player.x + dir * 30, y: player.y - 10, w: 60, h: 40 };
      if (rectCollide(ultBox, this.box)) {
        this.takeDamage(50, dir);
        player.ultimateHit = true;
        spawnParticles(player.x, player.y, 30, '#ffffff', 8, 5, 30);
        screenFlash = 15;
        screenFlashColor = '#ffffff';
      }
    }
  }

  takeDamage(dmg, dir) {
    if (this.invulnerable > 0) return;
    this.hp -= dmg;
    this.hitFlashTimer = 5;
    this.vx += dir * 2 * (this.isEnraged ? 0.4 : 1);
    spawnParticles(this.x, this.y - (this.h / 2), 8, '#ff1744', 3, 3, 18);
    triggerShake(2, 60);
    if (this.hp <= 0) {
      this.hp = 0;
      this.alive = false;
      spawnParticles(this.x, this.y - (this.h / 2), 60, '#e0e0e0', 8, 6, 50);
      spawnParticles(this.x, this.y - (this.h / 2), 40, '#ff1744', 7, 5, 40);
      triggerShake(15, 500);
      screenFlash = 20;
      screenFlashColor = '#ffffff';
    }
  }

  draw(ctx, cam) {
    if (!this.alive) return;
    const px = Math.round(this.x + cam.x);
    const py = Math.round(this.y + cam.y);
    const scale = 3;
    const sp = { x: px - (16 * scale / 2), y: py - (20 * scale) };

    // Intro glow
    if (this.introTimer > 0) {
      ctx.fillStyle = `rgba(255,23,68,${0.15 + Math.sin(this.introTimer * 0.25) * 0.12})`;
      ctx.fillRect(sp.x - 16, sp.y - 16, 16 * scale + 32, 20 * scale + 32);
    }

    // Enrage red aura
    if (this.isEnraged) {
      ctx.fillStyle = `rgba(255,23,68,${0.12 + Math.sin(Date.now() * 0.01) * 0.06})`;
      ctx.fillRect(sp.x - 10, sp.y - 10, 16 * scale + 20, 20 * scale + 20);
    }

    // Slow aura
    if (this.slowTimer > 0) {
      ctx.fillStyle = `rgba(41,182,246,${0.08 + Math.sin(Date.now() * 0.01) * 0.04})`;
      ctx.fillRect(sp.x - 10, sp.y - 10, 16 * scale + 20, 20 * scale + 20);
    }

    // Shadow
    ctx.fillStyle = 'rgba(0,0,0,0.35)';
    ctx.fillRect(px - 24, Math.round(GROUND_Y + cam.y) - 2, 48, 4);

    const f = this.facingRight ? 1 : -1;

    // Choose sprite
    let sprite = SPRITES.reaperIdle;
    let scytheAngle = 0;
    let scytheOfsX = 0, scytheOfsY = 0;
    if (this.state === 'slash_windup') {
      sprite = SPRITES.reaperWindup;
      scytheAngle = -0.3 + Math.sin(this.stateTimer * 0.15) * 0.2;
      scytheOfsX = f > 0 ? 22 : -22;
      scytheOfsY = -20;
    } else if (this.state === 'slash_active') {
      sprite = SPRITES.reaperSlash;
      scytheAngle = 1.2;
      scytheOfsX = f > 0 ? 28 : -28;
      scytheOfsY = -12;
    } else if (this.state === 'charge_windup') {
      sprite = SPRITES.reaperCharge;
      scytheAngle = -0.5;
      scytheOfsX = f > 0 ? 20 : -20;
      scytheOfsY = -18;
    } else if (this.state === 'charge_active') {
      sprite = SPRITES.reaperCharge;
      scytheAngle = 0.8;
      scytheOfsX = f > 0 ? 24 : -24;
      scytheOfsY = -10;
    } else if (this.state === 'slam_rise') {
      sprite = SPRITES.reaperJump;
      scytheAngle = -0.8;
      scytheOfsX = f > 0 ? 18 : -18;
      scytheOfsY = -30;
    } else if (this.state === 'slam_fall') {
      sprite = SPRITES.reaperJump;
      scytheAngle = 1.5;
      scytheOfsX = f > 0 ? 20 : -20;
      scytheOfsY = -8;
    }

    // Draw body sprite
    drawSprite(ctx, sprite, sp.x, sp.y, scale, !this.facingRight);

    // Draw scythe overlay
    ctx.save();
    ctx.translate(px + scytheOfsX, py + scytheOfsY);
    ctx.rotate(scytheAngle * f);
    ctx.globalAlpha = 0.15;
    ctx.fillStyle = '#ff1744';
    ctx.fillRect(-14, -24, 28, 28);
    ctx.globalAlpha = 1;
    drawSprite(ctx, SPRITES.scythe, -10, -20, 1, f < 0);
    ctx.restore();

    // Charge trail
    if (this.state === 'charge_active') {
      const trailSprite = sprite;
      for (let i = 3; i >= 0; i--) {
        ctx.globalAlpha = 0.08 - i * 0.018;
        drawSprite(ctx, trailSprite, sp.x - this.vx * (i + 1) * 3, sp.y, scale, !this.facingRight);
      }
      // Red streak lines
      ctx.globalAlpha = 0.12;
      ctx.fillStyle = '#ff1744';
      for (let i = 0; i < 6; i++) {
        const lx = sp.x - this.vx * (2 + Math.random() * 8);
        const ly = sp.y + Math.random() * (20 * scale);
        ctx.fillRect(lx - 1, ly, 4 + Math.random() * 14, 2);
      }
      ctx.globalAlpha = 1;
    }

    // Slam telegraph
    if (this.state === 'slam_windup') {
      ctx.fillStyle = `rgba(255,23,68,${0.2 + Math.sin(this.stateTimer * 0.4) * 0.15})`;
      ctx.fillRect(sp.x - 10, sp.y - 10, 16 * scale + 20, 20 * scale + 20);
    }

    // Slam landing zone indicator
    if (this.state === 'slam_fall') {
      const pulse = 0.3 + Math.sin(Date.now() * 0.05) * 0.15;
      ctx.fillStyle = `rgba(255,0,0,${pulse * 0.4})`;
      ctx.fillRect(this.slamTargetX + cam.x - 60, GROUND_Y + cam.y - 4, 120, 8);
      ctx.fillStyle = `rgba(255,23,68,${pulse * 0.3})`;
      ctx.fillRect(this.slamTargetX + cam.x - 60, GROUND_Y + cam.y - 6, 120, 4);
    }

    // Shockwave visual on slam recovery
    if (this.state === 'slam_recovery') {
      const prog2 = 1 - this.stateTimer / this.slamRecovery;
      const ringR = 40 + prog2 * 100;
      ctx.globalAlpha = (1 - prog2) * 0.25;
      ctx.strokeStyle = '#e0e0e0';
      ctx.lineWidth = 4;
      ctx.beginPath(); ctx.arc(px, Math.round(GROUND_Y + cam.y) + 2, ringR, 0, Math.PI * 2); ctx.stroke();
      ctx.strokeStyle = '#ff1744';
      ctx.lineWidth = 2;
      ctx.beginPath(); ctx.arc(px, Math.round(GROUND_Y + cam.y) + 2, ringR + 8, 0, Math.PI * 2); ctx.stroke();
      ctx.globalAlpha = 1;
    }

    // Hit flash
    if (this.hitFlashTimer > 0) {
      ctx.fillStyle = 'rgba(255,255,255,0.3)';
      ctx.fillRect(sp.x, sp.y, 16 * scale, 20 * scale);
    }
  }
}

// ============================================================
// UI
// ============================================================
function drawUI(ctx, player, boss) {
  const px = 20, py = 30, hw = CANVAS_W / 2;

  // Rank display — three stats
  const rankColors = ['#9e9e9e','#4fc3f7','#ffeb3b','#ff9800','#f44336','#e040fb'];
  ctx.fillStyle = rankColors[strRank];
  ctx.font = 'bold 11px monospace';
  ctx.fillText(`STR ${RANKS[strRank]}`, px, py - 14);
  ctx.fillStyle = rankColors[dexRank];
  ctx.fillText(`DEX ${RANKS[dexRank]}`, px + 70, py - 14);
  ctx.fillStyle = rankColors[vitRank];
  ctx.fillText(`VIT ${RANKS[vitRank]}`, px + 140, py - 14);
  // Stats effect text
  ctx.fillStyle = 'rgba(255,255,255,0.35)';
  ctx.font = '9px monospace';
  ctx.fillText(`DMGx${getStrMult().toFixed(1)}  SPDx${getDexMult().toFixed(1)}  HPx${getVitMult().toFixed(1)}  DEF${Math.round((1-getDefMult())*100)}%`, px, py - 2);

  // Player HP bar
  ctx.fillStyle = COLORS.hpBarBg;
  ctx.fillRect(px, py, 160, 16);
  const hpRatio = player.hp / player.maxHp;
  ctx.fillStyle = hpRatio > 0.5 ? COLORS.hpBar : (hpRatio > 0.25 ? '#ff9800' : '#f44336');
  ctx.fillRect(px + 1, py + 1, (160 - 2) * hpRatio, 14);
  ctx.fillStyle = COLORS.text;
  ctx.font = '10px monospace';
  ctx.fillText(`HP ${player.hp}/${player.maxHp}`, px + 4, py + 12);

  // MP bar
  ctx.fillStyle = COLORS.mpBarBg;
  ctx.fillRect(px, py + 20, 140, 12);
  const mpRatio = player.mp / player.maxMp;
  ctx.fillStyle = COLORS.mpBar;
  ctx.fillRect(px + 1, py + 21, (140 - 2) * mpRatio, 10);
  ctx.fillStyle = COLORS.text;
  ctx.fillText(`MP ${Math.floor(player.mp)}/${player.maxMp}`, px + 4, py + 31);

  // Skill cooldowns (bottom center)
  const skillY = CANVAS_H - 50;
  const skillStartX = hw - 245;
  const skills = [
    { key: 'Z', name: '베기', cd: player.attackActive > 0 ? '-' : '0', ready: player.attackActive <= 0 },
    { key: 'X', name: '표창', cd: Math.ceil(player.cooldownX / 60), maxCd: 7, ready: player.cooldownX <= 0, mp: 30 },
    { key: 'C', name: '대시', cd: Math.ceil(player.cooldownC / 60), maxCd: 10, ready: player.cooldownC <= 0, mp: 40 },
    { key: 'Q', name: '궁극기', cd: Math.ceil(player.cooldownQ / 60), maxCd: 15, ready: player.cooldownQ <= 0, mp: 100 },
    { key: 'E', name: '소환', cd: Math.ceil(player.cooldownE / 60), maxCd: 10, ready: player.cooldownE <= 0 && player.hasSummon, mp: 60 },
    { key: 'Shift', name: '회피', cd: Math.ceil(player.cooldownShift / 60), maxCd: 1.5, ready: player.cooldownShift <= 0, mp: 10 },
  ];
  for (let i = 0; i < skills.length; i++) {
    const s = skills[i];
    const sx = skillStartX + i * 70;
    const mpOk = s.mp === undefined || player.mp >= s.mp;
    const ready = s.ready && mpOk;

    ctx.fillStyle = ready ? 'rgba(255,255,255,0.15)' : 'rgba(255,255,255,0.05)';
    ctx.fillRect(sx, skillY, 60, 36);

    // Ultimate highlight (Q)
    if (s.key === 'Q') {
      const glow = ready ? '#0d47a1' : '#1a0a2e';
      ctx.strokeStyle = ready ? '#29b6f6' : '#1a237e';
      ctx.lineWidth = 2;
      ctx.strokeRect(sx - 1, skillY - 1, 62, 38);
      if (ready) {
        ctx.fillStyle = `rgba(41, 182, 246, ${0.05 + Math.sin(Date.now() * 0.005) * 0.03})`;
        ctx.fillRect(sx + 1, skillY + 1, 58, 34);
      }
    }

    // Summon highlight (E) - purple glow when acquired
    if (s.key === 'E') {
      const acquired = player.hasSummon;
      ctx.strokeStyle = acquired ? (ready ? '#ce93d8' : '#4a148c') : 'rgba(255,255,255,0.05)';
      ctx.lineWidth = acquired ? 2 : 1;
      ctx.strokeRect(sx - 1, skillY - 1, 62, 38);
      if (acquired && ready) {
        ctx.fillStyle = `rgba(206, 147, 216, ${0.05 + Math.sin(Date.now() * 0.005) * 0.03})`;
        ctx.fillRect(sx + 1, skillY + 1, 58, 34);
      }
      if (!acquired) {
        ctx.fillStyle = 'rgba(0,0,0,0.6)';
        ctx.fillRect(sx + 1, skillY + 1, 58, 34);
        ctx.fillStyle = 'rgba(255,255,255,0.25)';
        ctx.font = '8px monospace';
        ctx.fillText('LOCKED', sx + 12, skillY + 24);
      }
    }

    ctx.strokeStyle = ready ? 'rgba(255,255,255,0.3)' : 'rgba(255,255,255,0.1)';
    ctx.lineWidth = 1;
    ctx.strokeRect(sx, skillY, 60, 36);

    ctx.fillStyle = ready ? COLORS.text : 'rgba(255,255,255,0.3)';
    ctx.font = '11px monospace';
    ctx.fillText(s.key === 'Shift' ? 'Sh' : (s.key.length > 3 ? s.key[0] : s.key), sx + 4, skillY + 14);
    ctx.fillText(s.name, sx + 20, skillY + 14);

    if (!s.ready) {
      ctx.fillStyle = 'rgba(0,0,0,0.5)';
      ctx.fillRect(sx + 1, skillY + 1, 58 * (s.cd / s.maxCd || 0), 34);
      if (s.key !== 'E' || player.hasSummon) {
        ctx.fillStyle = COLORS.text;
        ctx.font = '10px monospace';
        const cdNum = typeof s.cd === 'number' ? s.cd : 0;
        ctx.fillText(`${cdNum.toFixed(1)}s`, sx + 24, skillY + 28);
      }
    } else if (!mpOk) {
      ctx.fillStyle = 'rgba(33,150,243,0.3)';
      ctx.fillRect(sx + 1, skillY + 1, 58, 34);
    }
  }

  // Boss HP bar (top center)
  if (boss && boss.alive) {
    const bhpW = 300;
    const bhpX = CANVAS_W / 2 - bhpW / 2;
    const bhpY = 12;

    ctx.fillStyle = COLORS.bossHpBarBg;
    ctx.fillRect(bhpX, bhpY, bhpW, 18);

    const bossHpRatio = boss.hp / boss.maxHp;
    const bossHpColor = (boss.isEnraged ? '#f44336' : (bossHpRatio > 0.5 ? '#e91e63' : '#ff5722'));
    ctx.fillStyle = bossHpColor;
    ctx.fillRect(bhpX + 1, bhpY + 1, (bhpW - 2) * bossHpRatio, 16);

    ctx.fillStyle = COLORS.text;
    ctx.font = 'bold 11px monospace';
    ctx.textAlign = 'center';
    const bossName = boss.constructor.name === 'Stage2Boss' ? 'BIG GUARDIAN' : 'SHADOW KNIGHT';
    ctx.fillText(bossName, hw, bhpY + 14);
    ctx.textAlign = 'left';

    if (boss.isEnraged) {
      ctx.fillStyle = '#f44336';
      ctx.font = 'bold 10px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('⚡ 광폭화 ⚡', hw, bhpY + 32);
      ctx.textAlign = 'left';
    } else if (boss.canSummon) {
      ctx.fillStyle = '#ff9800';
      ctx.font = '10px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('● HP 50% 이하', CANVAS_W / 2, bhpY + 32);
      ctx.textAlign = 'left';
    }
  }

  // Controls hint
  ctx.fillStyle = 'rgba(255,255,255,0.3)';
  ctx.font = '10px monospace';
  ctx.fillText('← → 이동  SPACE 점프 | Z 베기  X 표창  C 대시  Q 궁극기  E 소환  Shift 회피', 20, CANVAS_H - 10);
}

// ============================================================
// GAME STATE
// ============================================================
// Screen flash effect (for ultimate)
let screenFlash = 0;
let screenFlashColor = '#29b6f6';

// Rank / Stats System — three separate stats
const RANKS = ['C','B','A','S','SS','SSS'];
let strRank = 0; // 힘 C
let dexRank = 1; // 민첩 B
let vitRank = 0; // 맷집 C

function getStrMult() { return 1.0 + strRank * 0.5; }
function getDexMult() { return 1.0 + dexRank * 0.4; }
function getVitMult() { return 1.0 + vitRank * 0.3; }
function getDefMult() { return 1.0 / (1.0 + vitRank * 0.15); }

function rankUpAll() {
  if (strRank < RANKS.length - 1) strRank++;
  if (dexRank < RANKS.length - 1) dexRank++;
  if (vitRank < RANKS.length - 1) vitRank++;
}

// Lightning bolt draw helper
function drawLightning(ctx, x1, y1, x2, y2, segments, jitter, color, width) {
  ctx.beginPath();
  ctx.moveTo(x1, y1);
  const dx = (x2 - x1) / segments;
  const dy = (y2 - y1) / segments;
  for (let i = 1; i < segments; i++) {
    ctx.lineTo(x1 + dx * i + (Math.random() - 0.5) * jitter, y1 + dy * i + (Math.random() - 0.5) * jitter);
  }
  ctx.lineTo(x2, y2);
  ctx.strokeStyle = color;
  ctx.lineWidth = width || 2;
  ctx.stroke();
}

let gameState = 'title'; // title, playing, training, gameover, victory, stage2
let gameTimer = 0;
let gameoverTimer = 0;
let hasSummonSkill = false;
let worldSplitTimer = 0;
let currentStage = 1;
let menuSelection = 0;
let showControls = false;
let player, boss;
let camera;

// Cherry blossom petals (fusion 사극 atmosphere)
let petals = [];
for (let i = 0; i < 30; i++) {
  petals.push({
    x: Math.random() * CANVAS_W,
    y: Math.random() * CANVAS_H,
    vx: -0.2 - Math.random() * 0.4,
    vy: 0.3 + Math.random() * 0.5,
    size: 2 + Math.random() * 3,
    phase: Math.random() * Math.PI * 2
  });
}

function initGame() {
  strRank = 0; dexRank = 1; vitRank = 0; hasSummonSkill = false;
  player = new Player(400, GROUND_Y);
  boss = new Boss(900, GROUND_Y);
  camera = new Camera();
  projectiles = [];
  minions = [];
  particles = [];
  gameTimer = 0;
  worldSplitTimer = 0;
}

function startGame() {
  initGame();
  gameState = 'playing';
  gameoverTimer = 0;
  currentStage = 1;
  input.keys = {};
  input.justPressed = {};
  player.hasSummon = hasSummonSkill;
}

function startTraining() {
  player = new Player(400, GROUND_Y);
  boss = new Boss(900, GROUND_Y);
  boss.alive = false;
  camera = new Camera();
  projectiles = [];
  minions = [];
  particles = [];
  gameTimer = 0;
  gameState = 'training';
  gameoverTimer = 0;
  worldSplitTimer = 0;
  input.keys = {};
  input.justPressed = {};
  player.hasSummon = hasSummonSkill;
}

function startStage2() {
  player = new Player(400, GROUND_Y);
  player.hasSummon = hasSummonSkill;
  boss = new Stage2Boss((ARENA_LEFT + ARENA_RIGHT) / 2, GROUND_Y);
  camera = new Camera();
  projectiles = [];
  minions = [];
  particles = [];
  gameTimer = 0;
  gameState = 'stage2';
  gameoverTimer = 0;
  worldSplitTimer = 0;
  input.keys = {};
  input.justPressed = {};
}

function startStage3() {
  player = new Player(400, GROUND_Y);
  player.hasSummon = hasSummonSkill;
  boss = new GrimReaper(900, GROUND_Y);
  camera = new Camera();
  projectiles = [];
  minions = [];
  particles = [];
  gameTimer = 0;
  gameState = 'stage3';
  gameoverTimer = 0;
  worldSplitTimer = 0;
  input.keys = {};
  input.justPressed = {};
}

// ============================================================
// MAIN GAME LOOP
// ============================================================
const canvas = document.getElementById('gameCanvas');
canvas.width = CANVAS_W;
canvas.height = CANVAS_H;
const ctx = canvas.getContext('2d');
const input = new Input();

function update() {
  switch (gameState) {
    case 'title':
      if (showControls) {
        if (input.start) showControls = false;
        break;
      }
      if (input.isPressed('ArrowUp')) { menuSelection = (menuSelection + 2) % 3; }
      if (input.isPressed('ArrowDown')) { menuSelection = (menuSelection + 1) % 3; }
      if (input.start) {
        if (menuSelection === 0) { startGame(); }
        else if (menuSelection === 1) { startTraining(); }
        else if (menuSelection === 2) { showControls = true; }
      }
      break;

    case 'playing':
      gameTimer++;
      if (worldSplitTimer > 0) worldSplitTimer--;
      player.update(input);

      if (boss.alive) {
        boss.update(player);

        // Boss hits player
        if (!player.invincible && !player.isDead) {
          // Contact damage
          if (rectCollide(player.box, boss.box)) {
            player.takeDamage(10);
          }
          // Slash hit
          if (boss.state === 'slash_active') {
            const dir = boss.facingRight ? 1 : -1;
            const slashBox = {
              x: boss.x + dir * 35,
              y: boss.y - 15,
              w: 45,
              h: 35
            };
            if (rectCollide(player.box, slashBox)) {
              player.takeDamage(15);
            }
          }
          // Charge hit
          if (boss.state === 'charge_active') {
            const chargeBox = {
              x: boss.x,
              y: boss.y - 20,
              w: boss.w + 20,
              h: boss.h * 0.7
            };
            if (rectCollide(player.box, chargeBox)) {
              player.takeDamage(12);
            }
          }
        }
      } else {
        // Boss dead -> victory
        if (!hasSummonSkill) {
          hasSummonSkill = true;
        }
        rankUpAll();
        gameState = 'victory';
      }

      // Update minions
      for (const m of minions) {
        if (m.alive) m.update(player, boss);
        // Enemy minion hits player
        if (m.alive && m.owner !== 'player' && !player.invincible && !player.isDead) {
          if (rectCollide(player.box, m.box)) {
            player.takeDamage(m.damage);
          }
        }
        // Player minion hits boss
        if (m.alive && m.owner === 'player' && boss && boss.alive && m.attackCooldown <= 0) {
          if (rectCollide(m.box, boss.box)) {
            boss.takeDamage(m.damage, m.facingRight ? 1 : -1);
            m.attackCooldown = 30;
          }
        }
      }
      minions = minions.filter(m => m.alive);

      // Player attack hits minions
      if (player.attackActive > 0 && !player.attackHit) {
        const dir = player.facingRight ? 1 : -1;
        const range = player.getAttackRange();
        const attackBox = {
          x: player.x + dir * range / 2,
          y: player.y - 10,
          w: range,
          h: 30
        };
        for (const m of minions) {
          if (m.alive && rectCollide(attackBox, m.box)) {
            m.takeDamage(player.getAttackDamage(), dir);
            player.attackHit = true;
          }
        }
      }

      // Shadow dash hits minions
      if (player.isDashing && !player.dashHit) {
        for (const m of minions) {
          if (m.alive && rectCollide(player.box, m.box)) {
            m.takeDamage(18, player.facingRight ? 1 : -1);
            player.dashHit = true;
          }
        }
      }

      // Ultimate charge hits enemies
      if (player.isUltimate && !player.ultimateHit) {
        const dir = player.facingRight ? 1 : -1;
        const ultBox = {
          x: player.x + dir * 30,
          y: player.y - 10,
          w: 60,
          h: 40
        };
        if (boss.alive && rectCollide(ultBox, boss.box)) {
          boss.takeDamage(50, dir);
          player.ultimateHit = true;
          spawnParticles(player.x, player.y, 30, '#ffffff', 8, 5, 30);
          screenFlash = 15;
          screenFlashColor = '#ffffff';
        }
        for (const m of minions) {
          if (m.alive && rectCollide(ultBox, m.box)) {
            m.takeDamage(50, dir);
            player.ultimateHit = true;
          }
        }
      }

      // Projectile hits minions
      for (const p of projectiles) {
        if (p.alive && p.owner === 'player' && !p.hit) {
          for (const m of minions) {
            if (m.alive && rectCollide(m.box, p.box)) {
              m.takeDamage(p.damage, p.vx > 0 ? 1 : -1);
              p.hit = true;
              p.alive = false;
              break;
            }
          }
        }
      }

      // Update projectiles
      for (const p of projectiles) p.update();
      projectiles = projectiles.filter(p => p.alive);

      // Update particles
      for (const p of particles) p.update();
      particles = particles.filter(p => p.alive);

      // Camera
      camera.follow(player);
      camera.update();

      // Game over check
      if (player.isDead) {
        gameState = 'gameover';
      }
      break;

    case 'training':
      gameTimer++;
      if (worldSplitTimer > 0) worldSplitTimer--;
      player.update(input);
      // Minions still get updated
      for (const m of minions) {
        if (m.alive) m.update(player, boss);
      }
      minions = minions.filter(m => m.alive);
      // Player attack hits minions
      if (player.attackActive > 0 && !player.attackHit) {
        const dir = player.facingRight ? 1 : -1;
        const range = player.getAttackRange();
        const attackBox = {
          x: player.x + dir * range / 2,
          y: player.y - 10,
          w: range,
          h: 30
        };
        for (const m of minions) {
          if (m.alive && rectCollide(attackBox, m.box)) {
            m.takeDamage(player.getAttackDamage(), dir);
            player.attackHit = true;
          }
        }
      }
      // Shadow dash hits minions
      if (player.isDashing && !player.dashHit) {
        for (const m of minions) {
          if (m.alive && rectCollide(player.box, m.box)) {
            m.takeDamage(18, player.facingRight ? 1 : -1);
            player.dashHit = true;
          }
        }
      }
      // Ultimate hits minions
      if (player.isUltimate && !player.ultimateHit) {
        const dir = player.facingRight ? 1 : -1;
        const ultBox = {
          x: player.x + dir * 30,
          y: player.y - 10,
          w: 60,
          h: 40
        };
        for (const m of minions) {
          if (m.alive && rectCollide(ultBox, m.box)) {
            m.takeDamage(50, dir);
            player.ultimateHit = true;
          }
        }
      }
      // Projectile hits minions in training
      for (const p of projectiles) {
        if (p.alive && p.owner === 'player' && !p.hit) {
          for (const m of minions) {
            if (m.alive && rectCollide(m.box, p.box)) {
              m.takeDamage(p.damage, p.vx > 0 ? 1 : -1);
              p.hit = true;
              p.alive = false;
              break;
            }
          }
        }
      }
      // Update projectiles
      for (const p of projectiles) p.update();
      projectiles = projectiles.filter(p => p.alive);
      // Update particles
      for (const p of particles) p.update();
      particles = particles.filter(p => p.alive);
      // Camera
      camera.follow(player);
      camera.update();
      // Infinite HP in training
      player.hp = player.maxHp;
      player.mp = player.maxMp;
      player.isDead = false;
      // ESC to return
      if (input.escape) { gameState = 'title'; menuSelection = 0; }
      break;

    case 'stage3':
      gameTimer++;
      if (worldSplitTimer > 0) worldSplitTimer--;
      player.update(input);

      if (boss && boss.alive) {
        boss.update(player);
        for (const m of minions) {
          if (m.alive && m.owner === 'player' && boss && boss.alive && m.attackCooldown <= 0) {
            if (rectCollide(m.box, boss.box)) {
              boss.takeDamage(m.damage);
              m.attackCooldown = 30;
            }
          }
        }
      } else if (boss && !boss.alive) {
        rankUpAll();
        gameState = 'victory';
      }

      for (const m of minions) {
        if (m.alive) m.update(player, boss);
        if (m.alive && m.owner !== 'player' && !player.invincible) {
          if (rectCollide(player.box, m.box)) player.takeDamage(m.damage);
        }
      }
      minions = minions.filter(m => m.alive);

      if (player.attackActive > 0 && !player.attackHit) {
        const dir = player.facingRight ? 1 : -1;
        const range = player.getAttackRange();
        const attackBox = { x: player.x + dir * range / 2, y: player.y - 10, w: range, h: 30 };
        for (const m of minions) {
          if (m.alive && rectCollide(attackBox, m.box)) { m.takeDamage(player.getAttackDamage(), dir); player.attackHit = true; }
        }
      }
      if (player.isDashing && !player.dashHit) {
        for (const m of minions) { if (m.alive && rectCollide(player.box, m.box)) { m.takeDamage(18, player.facingRight ? 1 : -1); player.dashHit = true; } }
      }
      for (const p of projectiles) {
        if (p.alive && p.owner === 'player' && !p.hit) {
          for (const m of minions) {
            if (m.alive && rectCollide(m.box, p.box)) { m.takeDamage(p.damage, p.vx > 0 ? 1 : -1); p.hit = true; p.alive = false; break; }
          }
        }
      }
      for (const p of projectiles) p.update();
      projectiles = projectiles.filter(p => p.alive);
      for (const p of particles) p.update();
      particles = particles.filter(p => p.alive);
      camera.follow(player);
      camera.update();
      if (player.isDead) { gameState = 'gameover'; }
      if (input.escape) { gameState = 'title'; menuSelection = 0; }
      break;

    case 'stage2':
      gameTimer++;
      if (worldSplitTimer > 0) worldSplitTimer--;
      player.update(input);

      // Stage2 Boss update (includes checkPlayerHit for attack/dash/projectile/ultimate)
      if (boss && boss.alive) {
        boss.update(player);
        // Player minion hits boss
        for (const m of minions) {
          if (m.alive && m.owner === 'player' && boss && boss.alive && m.attackCooldown <= 0) {
            if (rectCollide(m.box, boss.box)) {
              boss.takeDamage(m.damage);
              m.attackCooldown = 30;
            }
          }
        }
      } else if (boss && !boss.alive) {
        rankUpAll();
        gameState = 'victory';
      }

      for (const m of minions) {
        if (m.alive) m.update(player, boss);
      }
      minions = minions.filter(m => m.alive);

      // Player attack hits minions
      if (player.attackActive > 0 && !player.attackHit) {
        const dir = player.facingRight ? 1 : -1;
        const range = player.getAttackRange();
        const attackBox = {
          x: player.x + dir * range / 2,
          y: player.y - 10,
          w: range,
          h: 30
        };
        for (const m of minions) {
          if (m.alive && rectCollide(attackBox, m.box)) {
            m.takeDamage(player.getAttackDamage(), dir);
            player.attackHit = true;
          }
        }
      }

      // Shadow dash hits minions
      if (player.isDashing && !player.dashHit) {
        for (const m of minions) {
          if (m.alive && rectCollide(player.box, m.box)) {
            m.takeDamage(18, player.facingRight ? 1 : -1);
            player.dashHit = true;
          }
        }
      }

      // Projectile hits minions
      for (const p of projectiles) {
        if (p.alive && p.owner === 'player' && !p.hit) {
          for (const m of minions) {
            if (m.alive && rectCollide(m.box, p.box)) {
              m.takeDamage(p.damage, p.vx > 0 ? 1 : -1);
              p.hit = true;
              p.alive = false;
              break;
            }
          }
        }
      }

      for (const p of projectiles) p.update();
      projectiles = projectiles.filter(p => p.alive);
      for (const p of particles) p.update();
      particles = particles.filter(p => p.alive);
      camera.follow(player);
      camera.update();

      // Game over check
      if (player.isDead) { gameState = 'gameover'; }

      if (input.escape) { gameState = 'title'; menuSelection = 0; }
      break;

    case 'gameover':
      gameoverTimer++;
      if (input.start && gameoverTimer > 60) {
        if (currentStage === 1) { startGame(); }
        else if (currentStage === 2) { startStage2(); }
        else if (currentStage === 3) { startStage3(); }
      }
      if (input.escape) { gameState = 'title'; menuSelection = 0; }
      break;

    case 'victory':
      gameoverTimer++;
      if (input.start && gameoverTimer > 60) {
        if (currentStage === 1) {
          currentStage = 2;
          startStage2();
        } else if (currentStage === 2) {
          currentStage = 3;
          startStage3();
        } else {
          // 모든 스테이지 클리어 -> 타이틀로
          currentStage = 1;
          gameState = 'title';
          menuSelection = 0;
          input.keys = {};
          input.justPressed = {};
        }
      }
      if (input.escape) { gameState = 'title'; menuSelection = 0; }
      break;
  }
}

function render() {
  // Fill with dark background first so canvas is never blank
  ctx.fillStyle = '#050510';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // ============================================================
  // STAGE 2 - Dark Forest
  // ============================================================
  if (gameState === 'stage2') {
    renderStage2(ctx);
    return;
  }

  // ============================================================
  // STAGE 3 - Grim Reaper
  // ============================================================
  if (gameState === 'stage3') {
    renderStage3(ctx);
    return;
  }

  // ============================================================
  // BACKGROUND - Fusion Sageuk (한옥 + 달 + 벛꽃)
  // ============================================================

  // Night sky gradient
  const skyGrad = ctx.createLinearGradient(0, 0, 0, CANVAS_H);
  skyGrad.addColorStop(0, '#050510');
  skyGrad.addColorStop(0.3, '#0a0a20');
  skyGrad.addColorStop(0.55, '#1a0a2e');
  skyGrad.addColorStop(0.75, '#1a0a20');
  skyGrad.addColorStop(1, '#0f0815');
  ctx.fillStyle = skyGrad;
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Stars
  ctx.fillStyle = 'rgba(255,255,255,0.7)';
  const stars = [[120,45,2],[280,78,1],[450,30,2],[610,95,1],[780,42,2],
    [950,65,1],[1100,38,2],[200,130,1],[520,140,2],[820,120,1],
    [1050,150,1],[150,200,1],[380,180,1],[680,170,2],[900,190,1]];
  for (const [sx,sy,sz] of stars) ctx.fillRect(sx, sy, sz, sz);

  // Moon
  const moonX = 1040, moonY = 90;
  const moonGlow = ctx.createRadialGradient(moonX, moonY, 0, moonX, moonY, 140);
  moonGlow.addColorStop(0, 'rgba(255,235,200,0.12)');
  moonGlow.addColorStop(0.5, 'rgba(255,235,200,0.05)');
  moonGlow.addColorStop(1, 'rgba(255,235,200,0)');
  ctx.fillStyle = moonGlow;
  ctx.beginPath(); ctx.arc(moonX, moonY, 140, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#ffe8c8';
  ctx.beginPath(); ctx.arc(moonX, moonY, 38, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = 'rgba(200,180,150,0.3)';
  ctx.beginPath(); ctx.arc(moonX - 12, moonY - 8, 7, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(moonX + 8, moonY + 10, 4, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath(); ctx.arc(moonX + 16, moonY - 5, 3, 0, Math.PI * 2); ctx.fill();

  // Mountain silhouettes (far)
  ctx.fillStyle = '#0f0818';
  ctx.beginPath(); ctx.moveTo(0, 370);
  ctx.lineTo(60,300);ctx.lineTo(140,330);ctx.lineTo(220,260);ctx.lineTo(310,300);
  ctx.lineTo(400,240);ctx.lineTo(490,290);ctx.lineTo(580,250);ctx.lineTo(670,310);
  ctx.lineTo(760,260);ctx.lineTo(850,320);ctx.lineTo(940,280);ctx.lineTo(1030,310);
  ctx.lineTo(1120,260);ctx.lineTo(1200,300);ctx.lineTo(1280,340);
  ctx.lineTo(1280,410);ctx.lineTo(0,410);ctx.closePath(); ctx.fill();

  // Mountain silhouettes (near)
  ctx.fillStyle = '#120a20';
  ctx.beginPath(); ctx.moveTo(0, 400);
  ctx.lineTo(100,340);ctx.lineTo(200,370);ctx.lineTo(320,320);ctx.lineTo(450,360);
  ctx.lineTo(560,330);ctx.lineTo(680,370);ctx.lineTo(800,340);ctx.lineTo(920,380);
  ctx.lineTo(1040,350);ctx.lineTo(1160,390);ctx.lineTo(1280,360);
  ctx.lineTo(1280,440);ctx.lineTo(0,440);ctx.closePath(); ctx.fill();

  const cam = camera;

  // Screen shake
  let sx = 0, sy = 0;
  if (shakeIntensity > 0) {
    sx = (Math.random() - 0.5) * shakeIntensity * 2;
    sy = (Math.random() - 0.5) * shakeIntensity * 2;
  }

  ctx.save();
  ctx.translate(sx, sy);

  const camX = Math.round(cam.x), camY = Math.round(cam.y), hw = CANVAS_W / 2, hh = CANVAS_H / 2;
  const floorY = Math.round(GROUND_Y + camY);

  // ---- Screen-space hanok cityscape (always visible) ----
  // Distant city layer (small dark silhouettes)
  ctx.fillStyle = '#0d0618';
  const buildings = [
    { x:30, w:90, h:120, roofH:45 }, { x:140, w:70, h:90, roofH:35 }, { x:230, w:100, h:140, roofH:50 },
    { x:350, w:80, h:100, roofH:38 }, { x:450, w:110, h:150, roofH:55 }, { x:580, w:75, h:85, roofH:32 },
    { x:670, w:95, h:130, roofH:48 }, { x:780, w:85, h:105, roofH:40 }, { x:880, w:105, h:145, roofH:52 },
    { x:1000, w:70, h:95, roofH:36 }, { x:1090, w:100, h:135, roofH:50 }, { x:1200, w:60, h:80, roofH:30 },
  ];
  for (const b of buildings) {
    const baseY = 520;
    const topY = baseY - b.h;
    // Body
    ctx.fillRect(b.x, topY + b.roofH, b.w, b.h - b.roofH);
    // Roof (curved hanok-style)
    ctx.beginPath();
    ctx.moveTo(b.x - 15, topY + b.roofH);
    ctx.lineTo(b.x + b.w/2, topY);
    ctx.lineTo(b.x + b.w + 15, topY + b.roofH);
    ctx.lineTo(b.x + b.w + 10, topY + b.roofH + 3);
    ctx.lineTo(b.x + b.w/2 + 5, topY + 4);
    ctx.lineTo(b.x + b.w/2 - 5, topY + 4);
    ctx.lineTo(b.x - 10, topY + b.roofH + 3);
    ctx.closePath(); ctx.fill();
    // Ridge
    ctx.fillStyle = '#150a20';
    ctx.fillRect(b.x + b.w/2 - 6, topY - 3, 12, 5);
    ctx.fillStyle = '#0d0618';
    // Warm windows
    if (b.h > 100 && Math.random() < 0.6) {
      ctx.fillStyle = 'rgba(255,180,80,0.12)';
      for (let wy = 0; wy < 2; wy++) {
        for (let wx = 0; wx < 2; wx++) {
          ctx.fillRect(b.x + 8 + wx * (b.w/2 - 4), topY + b.roofH + 8 + wy * 22, b.w/2 - 12, 16);
        }
      }
      ctx.fillStyle = '#0d0618';
    }
  }

  // Mid city layer (slightly larger, overlapping)
  ctx.fillStyle = '#120a1e';
  const midBuildings = [
    { x:-10, w:110, h:160, roofH:55 }, { x:120, w:60, h:110, roofH:40 },
    { x:200, w:130, h:180, roofH:60 }, { x:360, w:70, h:120, roofH:42 },
    { x:460, w:120, h:170, roofH:58 }, { x:610, w:65, h:105, roofH:38 },
    { x:700, w:115, h:175, roofH:58 }, { x:840, w:60, h:115, roofH:40 },
    { x:920, w:130, h:165, roofH:56 }, { x:1080, w:75, h:125, roofH:42 },
    { x:1170, w:100, h:155, roofH:52 },
  ];
  for (const b of midBuildings) {
    const baseY = 535;
    const topY = baseY - b.h;
    ctx.fillRect(b.x, topY + b.roofH, b.w, b.h - b.roofH);
    ctx.beginPath();
    ctx.moveTo(b.x - 18, topY + b.roofH);
    ctx.lineTo(b.x + b.w/2, topY - 4);
    ctx.lineTo(b.x + b.w + 18, topY + b.roofH);
    ctx.lineTo(b.x + b.w + 12, topY + b.roofH + 3);
    ctx.lineTo(b.x + b.w/2 + 6, topY);
    ctx.lineTo(b.x + b.w/2 - 6, topY);
    ctx.lineTo(b.x - 12, topY + b.roofH + 3);
    ctx.closePath(); ctx.fill();
    ctx.fillStyle = '#1a0f22';
    ctx.fillRect(b.x + b.w/2 - 7, topY - 5, 14, 6);
    ctx.fillStyle = '#120a1e';
    // Windows with warm glow
    if (Math.random() < 0.5) {
      ctx.fillStyle = 'rgba(255,200,100,0.1)';
      for (let wy = 0; wy < 3; wy++) {
        ctx.fillRect(b.x + 10, topY + b.roofH + 6 + wy * 18, b.w - 20, 12);
      }
      ctx.fillStyle = '#120a1e';
    }
  }

  // ---- Arena background walls (world-space, tempered) ----
  ctx.fillStyle = '#150820';
  ctx.fillRect(ARENA_LEFT + camX - 40, ARENA_TOP + camY - 20, ARENA_RIGHT - ARENA_LEFT + 80, floorY - ARENA_TOP - camY + 20);

  // ---- Wooden pillars at arena edges (world-space) ----
  ctx.fillStyle = '#3d2218';
  ctx.fillRect(ARENA_LEFT + camX - 8, ARENA_TOP + camY - 20, 10, floorY - ARENA_TOP - camY + 20);
  ctx.fillRect(ARENA_RIGHT + camX - 2, ARENA_TOP + camY - 20, 10, floorY - ARENA_TOP - camY + 20);
  // Cross beam at top
  ctx.fillRect(ARENA_LEFT + camX - 8, ARENA_TOP + camY - 24, ARENA_RIGHT - ARENA_LEFT + 18, 6);
  // Roof eave above arena (hanok-style curved edge at top of pillars)
  ctx.fillStyle = '#2a1810';
  ctx.beginPath();
  ctx.moveTo(ARENA_LEFT + camX - 30, ARENA_TOP + camY - 24);
  ctx.lineTo(ARENA_LEFT + camX + 50, ARENA_TOP + camY - 50);
  ctx.lineTo(ARENA_RIGHT + camX - 50, ARENA_TOP + camY - 50);
  ctx.lineTo(ARENA_RIGHT + camX + 30, ARENA_TOP + camY - 24);
  ctx.lineTo(ARENA_RIGHT + camX - 50, ARENA_TOP + camY - 22);
  ctx.lineTo(ARENA_LEFT + camX + 50, ARENA_TOP + camY - 22);
  ctx.closePath(); ctx.fill();
  // Roof ridge line
  ctx.fillStyle = '#3d2218';
  ctx.fillRect(ARENA_LEFT + camX + 50, ARENA_TOP + camY - 52, ARENA_RIGHT - ARENA_LEFT - 100, 5);

  // ---- Paper lanterns hanging from beam ----
  const lanternColors = ['#ff6b35','#ffb74d','#ff5252','#ffb74d','#ff6b35'];
  ctx.shadowBlur = 18;
  for (let i = 0; i < 5; i++) {
    const lx = (ARENA_LEFT + 80 + i * ((ARENA_RIGHT - ARENA_LEFT - 160) / 4)) + camX;
    const ly = ARENA_TOP + camY + 8;
    ctx.strokeStyle = 'rgba(80,50,30,0.4)';
    ctx.lineWidth = 1;
    ctx.beginPath(); ctx.moveTo(lx, ARENA_TOP + camY - 24);
    ctx.lineTo(lx, ly); ctx.stroke();
    ctx.shadowColor = lanternColors[i];
    ctx.fillStyle = lanternColors[i];
    ctx.fillRect(lx - 4, ly, 8, 12);
    ctx.fillStyle = 'rgba(255,200,100,0.25)';
    ctx.fillRect(lx - 7, ly - 1, 14, 14);
  }
  ctx.shadowBlur = 0;
  ctx.shadowColor = 'transparent';

  // ---- Arena floor (stone platform) ----
  ctx.fillStyle = COLORS.ground;
  ctx.fillRect(ARENA_LEFT + camX, floorY, ARENA_RIGHT - ARENA_LEFT, CANVAS_H - floorY + camY);

  ctx.fillStyle = COLORS.groundLine;
  ctx.fillRect(ARENA_LEFT + camX, floorY - 2, ARENA_RIGHT - ARENA_LEFT, 3);

  ctx.fillStyle = 'rgba(255,255,255,0.02)';
  const tileW = 60;
  for (let tx = ARENA_LEFT + camX; tx < ARENA_RIGHT + camX; tx += tileW) {
    const colIdx = Math.floor((tx - ARENA_LEFT - camX) / tileW);
    ctx.fillRect(tx, floorY + 4, tileW - 2, 22);
    ctx.fillRect(tx + (colIdx % 2) * tileW / 2, floorY + 28, tileW - 2, 22);
  }

  ctx.fillStyle = '#3a2830';
  ctx.fillRect(ARENA_LEFT + camX - 4, floorY - 6, ARENA_RIGHT - ARENA_LEFT + 8, 6);
  ctx.fillStyle = '#2a1820';
  ctx.fillRect(ARENA_LEFT + camX - 4, floorY - 6, ARENA_RIGHT - ARENA_LEFT + 8, 2);

  // Cherry blossom petals (정적 배경 효과)
  ctx.fillStyle = 'rgba(255,180,200,0.5)';
  for (const p of petals) {
    p.x += p.vx;
    p.y += p.vy;
    p.vx += Math.sin(p.phase) * 0.003;
    p.phase += 0.02;
    if (p.y > CANVAS_H + 10) { p.y = -10; p.x = Math.random() * (CANVAS_W + 200) - 100; }
    if (p.x < -20) { p.x = CANVAS_W + 10; }
    ctx.fillRect(p.x, p.y, p.size, p.size * 0.6);
    ctx.fillRect(p.x + 1, p.y - 1, p.size * 0.6, p.size * 0.4);
  }

  // Draw game objects (sorted by y for depth)
  const renderables = [];

  for (const m of minions) {
    if (m.alive) renderables.push({ type: 'minion', obj: m, y: m.y });
  }
  if (boss && boss.alive) renderables.push({ type: 'boss', obj: boss, y: boss.y });
  renderables.push({ type: 'player', obj: player, y: player.y });

  renderables.sort((a, b) => a.y - b.y);

  // Draw projectiles behind everything
  for (const p of projectiles) p.draw(ctx, cam);

  for (const r of renderables) r.obj.draw(ctx, cam);

  // Particles
  for (const p of particles) p.draw(ctx, cam);
  ctx.globalAlpha = 1;

  // Lightning bolts during ultimate (world space)
  if (player.isUltimate && gameState === 'playing') {
    ctx.save();
    // Sky-to-ground lightning bolts
    for (let i = 0; i < 6; i++) {
      const bx = player.x + (Math.random() - 0.5) * 300 + camX;
      const by = GROUND_Y + camY - 80 - Math.random() * 120;
      const alpha = 0.3 + Math.random() * 0.5;
      const colors = [`rgba(41,182,246,${alpha})`, `rgba(255,255,255,${alpha * 0.6})`, `rgba(3,169,244,${alpha})`];
      drawLightning(ctx, bx, by - 100 - Math.random() * 60, bx + (Math.random() - 0.5) * 40, by, 12, 15, colors[Math.floor(Math.random() * 3)], 2 + Math.random() * 2);
      // Branch lightning
      for (let b = 0; b < 2; b++) {
        const brx = bx + (Math.random() - 0.5) * 80;
        const bry = by - 40 - Math.random() * 40;
        drawLightning(ctx, bx, by - Math.random() * 50, brx, bry, 6, 8, `rgba(41,182,246,${alpha * 0.4})`, 1.5);
      }
    }
    // Ball lightning orbs around player
    for (let i = 0; i < 4; i++) {
      const angle = Date.now() * 0.003 + (Math.PI / 2) * i;
      const dist = 50 + Math.sin(Date.now() * 0.005 + i) * 10;
      const ox = player.x + camX + Math.cos(angle) * dist;
      const oy = player.y + camY - 20 + Math.sin(angle) * dist * 0.4;
      const grad = ctx.createRadialGradient(ox, oy, 0, ox, oy, 12);
      grad.addColorStop(0, 'rgba(255,255,255,0.6)');
      grad.addColorStop(0.5, 'rgba(41,182,246,0.3)');
      grad.addColorStop(1, 'rgba(41,182,246,0)');
      ctx.fillStyle = grad;
      ctx.beginPath(); ctx.arc(ox, oy, 12, 0, Math.PI * 2); ctx.fill();
      // Spark trail
      ctx.fillStyle = `rgba(255,255,255,${0.2 + Math.random() * 0.3})`;
      ctx.fillRect(ox - 1, oy - 1 + Math.random() * 6, 3, 2);
    }
    ctx.restore();
  }

  ctx.restore();

  // Screen flash effect
  if (screenFlash > 0) {
    ctx.globalAlpha = screenFlash / 35;
    ctx.fillStyle = screenFlashColor;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.globalAlpha = 1;
    screenFlash--;
  }

  // Ultimate screen overlay (blue aura)
  if (player.isUltimate && gameState === 'playing') {
    const pulse = 0.18 + Math.sin(Date.now() * 0.008) * 0.1;
    ctx.fillStyle = `rgba(13, 71, 161, ${pulse})`;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    const vigGrad = ctx.createRadialGradient(hw, hh, 80, hw, hh, CANVAS_W / 1.5);
    vigGrad.addColorStop(0, 'rgba(41,182,246,0)');
    vigGrad.addColorStop(0.5, `rgba(13,71,161,${0.15 + Math.sin(Date.now() * 0.005) * 0.05})`);
    vigGrad.addColorStop(1, `rgba(13,71,161,${0.4 + Math.sin(Date.now() * 0.003) * 0.1})`);
    ctx.fillStyle = vigGrad;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.fillStyle = `rgba(41,182,246,${0.02 + Math.sin(Date.now() * 0.01) * 0.01})`;
    const lineY = (Date.now() * 0.08) % CANVAS_H;
    ctx.fillRect(0, lineY, CANVAS_W, 3);
  }

  // World-splitting effect (ultimate) — FIXED AT INITIAL POSITION/ANGLE
  drawWorldSplitEffect(ctx, camX, camY);

  switch (gameState) {
    case 'title':
      drawTitleScreen(ctx);
      if (showControls) drawControlsOverlay(ctx);
      break;
    case 'playing':
      drawUI(ctx, player, boss);
      if (player.hasSummon) drawSummonHint(ctx);
      break;
    case 'training':
      drawUI(ctx, player, null);
      drawTrainingUI(ctx);
      if (player.hasSummon) drawSummonHint(ctx);
      break;
    case 'gameover':
      drawUI(ctx, player, boss);
      drawGameOverScreen(ctx);
      break;
    case 'victory':
      drawVictoryScreen(ctx);
      break;
  }
}


// Shared worldSplit line effect (start → target, screen-fixed)
function drawWorldSplitEffect(ctx, camX, camY) {
  if (worldSplitTimer <= 0) return;
  const glowA = Math.min(worldSplitTimer / 35, 1);
  const fadeA = Math.min(worldSplitTimer / 8, 1);
  const alpha = glowA * fadeA;

  // Direction vector from start to target (world-space)
  const sx = player.ultStartX + camX;
  const sy = player.ultStartY + camY;
  const tx = player.ultimateTargetX + camX;
  const ty = (player.ultimateTargetY || player.ultStartY) + camY;

  const dx = tx - sx, dy = ty - sy;
  const len = Math.sqrt(dx * dx + dy * dy) || 1;
  // Unit direction and normal
  const ux = dx / len, uy = dy / len;
  const nx = -uy, ny = ux;

  // Extend the line far beyond screen in both directions
  const diag = Math.sqrt(CANVAS_W * CANVAS_W + CANVAS_H * CANVAS_H);
  // Midpoint of the original dash (so the slash is centered on the action)
  const mx = (sx + tx) * 0.5, my = (sy + ty) * 0.5;
  const ex0x = mx - ux * diag, ex0y = my - uy * diag; // extended far back
  const ex1x = mx + ux * diag, ex1y = my + uy * diag; // extended far forward

  ctx.save();

  // ── BACKGROUND SPLIT GLOW (wide, diffuse, dramatic) ──
  ctx.globalAlpha = 0.22 * alpha;
  ctx.strokeStyle = '#0d47a1';
  ctx.lineWidth = 120 * alpha;
  ctx.lineCap = 'butt';
  ctx.beginPath(); ctx.moveTo(ex0x, ex0y); ctx.lineTo(ex1x, ex1y); ctx.stroke();

  ctx.globalAlpha = 0.3 * alpha;
  ctx.strokeStyle = '#29b6f6';
  ctx.lineWidth = 60 * alpha;
  ctx.beginPath(); ctx.moveTo(ex0x, ex0y); ctx.lineTo(ex1x, ex1y); ctx.stroke();

  // ── RED ENERGY TEAR ──
  ctx.globalAlpha = 0.45 * alpha;
  ctx.strokeStyle = '#ff1744';
  ctx.lineWidth = 22 * alpha;
  ctx.beginPath(); ctx.moveTo(ex0x, ex0y); ctx.lineTo(ex1x, ex1y); ctx.stroke();

  // ── BRIGHT BLUE CORE ──
  ctx.globalAlpha = 0.75 * alpha;
  ctx.strokeStyle = '#4fc3f7';
  ctx.lineWidth = 8 * alpha;
  ctx.beginPath(); ctx.moveTo(ex0x, ex0y); ctx.lineTo(ex1x, ex1y); ctx.stroke();

  // ── WHITE HOT CENTER CRACK ──
  ctx.globalAlpha = 1.0 * alpha;
  ctx.strokeStyle = '#ffffff';
  ctx.lineWidth = 2.5;
  ctx.beginPath(); ctx.moveTo(ex0x, ex0y); ctx.lineTo(ex1x, ex1y); ctx.stroke();

  // ── EDGE FRINGE (jagged crack energy) ──
  ctx.globalAlpha = 0.5 * alpha;
  for (let i = 0; i < 12; i++) {
    const t = (i / 11) * 2 - 1; // -1 to 1 along full screen
    const bx = mx + ux * diag * t;
    const by = my + uy * diag * t;
    const jitter = (Math.random() - 0.5) * 24 * alpha;
    const jLen = 8 + Math.random() * 28 * alpha;
    const side = Math.random() > 0.5 ? 1 : -1;
    ctx.strokeStyle = Math.random() > 0.5 ? '#ffffff' : '#4fc3f7';
    ctx.lineWidth = 1 + Math.random() * 2;
    ctx.beginPath();
    ctx.moveTo(bx + nx * jitter, by + ny * jitter);
    ctx.lineTo(bx + nx * (jitter + side * jLen), by + ny * (jitter + side * jLen));
    ctx.stroke();
  }

  // ── SPARK BURST along the full line ──
  ctx.globalAlpha = 1;
  const sparkCount = Math.floor(30 * alpha);
  for (let i = 0; i < sparkCount; i++) {
    const t = Math.random() * 2 - 1; // spread across full line
    const lx = mx + ux * diag * t + nx * (Math.random() - 0.5) * 18 * alpha;
    const ly = my + uy * diag * t + ny * (Math.random() - 0.5) * 18 * alpha;
    const sa = 0.5 + Math.random() * 0.5;
    ctx.fillStyle = Math.random() > 0.4 ? `rgba(255,255,255,${sa})` : (Math.random() > 0.5 ? `rgba(79,195,247,${sa})` : `rgba(255,23,68,${sa})`);
    const ps = 1.5 + Math.random() * 4;
    ctx.fillRect(lx - ps * 0.5, ly - ps * 0.5, ps, ps);
  }

  // ── EPICENTER BURST at midpoint ──
  const burstA = 0.6 * alpha;
  ctx.globalAlpha = burstA * 0.4;
  ctx.fillStyle = '#29b6f6';
  ctx.beginPath(); ctx.arc(mx, my, 55 * alpha, 0, Math.PI * 2); ctx.fill();
  ctx.globalAlpha = burstA * 0.6;
  ctx.fillStyle = '#ffffff';
  ctx.beginPath(); ctx.arc(mx, my, 22 * alpha, 0, Math.PI * 2); ctx.fill();
  ctx.globalAlpha = burstA;
  ctx.fillStyle = '#ff1744';
  ctx.beginPath(); ctx.arc(mx, my, 10 * alpha, 0, Math.PI * 2); ctx.fill();

  ctx.globalAlpha = 1;
  ctx.restore();
}
// ============================================================
// STAGE 2 - Dark Forest (BOSS NOT IMPLEMENTED)
// ============================================================
function renderStage2(ctx) {
  // Dark background fill
  ctx.fillStyle = '#05080f';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Dark forest sky
  const skyGrad = ctx.createLinearGradient(0, 0, 0, CANVAS_H);
  skyGrad.addColorStop(0, '#05080f');
  skyGrad.addColorStop(0.4, '#0a0f18');
  skyGrad.addColorStop(0.7, '#0f1520');
  skyGrad.addColorStop(1, '#0a0f15');
  ctx.fillStyle = skyGrad;
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Fog
  ctx.fillStyle = 'rgba(20,30,45,0.3)';
  for (let i = 0; i < 8; i++) {
    const fx = (i * 160 + Math.sin(Date.now() * 0.0005 + i) * 20) % 1400 - 100;
    const fy = 300 + Math.sin(Date.now() * 0.001 + i * 2) * 40;
    ctx.fillRect(fx, fy, 200 + Math.sin(i) * 40, 80 + Math.sin(i * 3) * 20);
  }

  // Moon (red-tinted)
  const moonX = 140, moonY = 100;
  const moonGlow = ctx.createRadialGradient(moonX, moonY, 0, moonX, moonY, 120);
  moonGlow.addColorStop(0, 'rgba(255,80,80,0.08)');
  moonGlow.addColorStop(1, 'rgba(255,80,80,0)');
  ctx.fillStyle = moonGlow;
  ctx.beginPath(); ctx.arc(moonX, moonY, 120, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#ff6b6b';
  ctx.beginPath(); ctx.arc(moonX, moonY, 30, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = 'rgba(200,50,50,0.3)';
  ctx.beginPath(); ctx.arc(moonX, moonY, 30, 0, Math.PI * 2); ctx.fill();

  // Tree silhouettes
  ctx.fillStyle = '#0a0f15';
  const trees = [
    { x:0, h:280 }, { x:80, h:340 }, { x:170, h:260 }, { x:280, h:380 },
    { x:400, h:310 }, { x:520, h:350 }, { x:650, h:290 }, { x:780, h:370 },
    { x:900, h:320 }, { x:1020, h:360 }, { x:1140, h:300 }, { x:1240, h:350 },
  ];
  for (const t of trees) {
    const baseY = 480;
    // Trunk
    ctx.fillRect(t.x + 3, baseY - t.h + 40, 8, t.h - 40);
    // Canopy (spiky triangle)
    ctx.beginPath();
    ctx.moveTo(t.x - 20, baseY - t.h + 60);
    ctx.lineTo(t.x + 7, baseY - t.h);
    ctx.lineTo(t.x + 35, baseY - t.h + 60);
    ctx.closePath(); ctx.fill();
    ctx.beginPath();
    ctx.moveTo(t.x - 10, baseY - t.h + 90);
    ctx.lineTo(t.x + 7, baseY - t.h + 30);
    ctx.lineTo(t.x + 25, baseY - t.h + 90);
    ctx.closePath(); ctx.fill();
  }

  // Ground
  ctx.fillStyle = '#0d1219';
  const floorY2 = 480;
  ctx.fillRect(0, floorY2, CANVAS_W, CANVAS_H - floorY2);
  ctx.fillStyle = '#1a2535';
  ctx.fillRect(0, floorY2 - 2, CANVAS_W, 3);

  // World-space arena (forest clearing)
  const camX2 = Math.round(camera.x), camY2 = Math.round(camera.y), hw2 = CANVAS_W / 2, hh2 = CANVAS_H / 2;
  const floorY = Math.round(480 + camY2);

  ctx.save();
  // Screen shake
  let sx = 0, sy = 0;
  if (shakeIntensity > 0) {
    sx = (Math.random() - 0.5) * shakeIntensity * 2;
    sy = (Math.random() - 0.5) * shakeIntensity * 2;
  }
  ctx.translate(sx, sy);

  // Arena ground
  ctx.fillStyle = '#0d1219';
  ctx.fillRect(ARENA_LEFT + camX2, floorY, ARENA_RIGHT - ARENA_LEFT, CANVAS_H - floorY + 100);
  ctx.fillStyle = '#182030';
  ctx.fillRect(ARENA_LEFT + camX2, floorY - 2, ARENA_RIGHT - ARENA_LEFT, 3);
  // Stone edge
  ctx.fillStyle = '#1a2535';
  ctx.fillRect(ARENA_LEFT + camX2 - 4, floorY - 4, ARENA_RIGHT - ARENA_LEFT + 8, 4);

  // Mossy pillars
  ctx.fillStyle = '#1a2028';
  ctx.fillRect(ARENA_LEFT + camX2 - 6, ARENA_TOP + camY2, 8, floorY - ARENA_TOP);
  ctx.fillRect(ARENA_RIGHT + camX2 - 2, ARENA_TOP + camY2, 8, floorY - ARENA_TOP);
  // Old cross beam
  ctx.fillRect(ARENA_LEFT + camX2 - 6, ARENA_TOP + camY2 - 4, ARENA_RIGHT - ARENA_LEFT + 12, 6);

  // Draw game objects
  const renderables = [];
  for (const m of minions) if (m.alive) renderables.push({ type: 'minion', obj: m, y: m.y });
  if (boss && boss.alive) renderables.push({ type: 'boss', obj: boss, y: boss.y });
  renderables.push({ type: 'player', obj: player, y: player.y });
  renderables.sort((a, b) => a.y - b.y);
  for (const p of projectiles) p.draw(ctx, camera);
  for (const r of renderables) {
    if (r.type === 'player') r.obj.draw(ctx, camera);
    if (r.type === 'boss') r.obj.draw(ctx, camera);
    if (r.type === 'minion') r.obj.draw(ctx, camera);
  }
  for (const p of particles) p.draw(ctx, camera);

  // Ultimate lightning effects
  if (player.isUltimate) {
    ctx.save();
    for (let i = 0; i < 6; i++) {
      const bx = player.x + (Math.random() - 0.5) * 300 + camX2;
      const by = floorY - 80 - Math.random() * 120;
      const alpha = 0.3 + Math.random() * 0.5;
      const colors = [`rgba(41,182,246,${alpha})`, `rgba(255,255,255,${alpha * 0.6})`, `rgba(3,169,244,${alpha})`];
      drawLightning(ctx, bx, by - 100 - Math.random() * 60, bx + (Math.random() - 0.5) * 40, by, 12, 15, colors[Math.floor(Math.random() * 3)], 2 + Math.random() * 2);
      for (let b = 0; b < 2; b++) {
        const brx = bx + (Math.random() - 0.5) * 80;
        const bry = by - 40 - Math.random() * 40;
        drawLightning(ctx, bx, by - Math.random() * 50, brx, bry, 6, 8, `rgba(41,182,246,${alpha * 0.4})`, 1.5);
      }
    }
    for (let i = 0; i < 4; i++) {
      const angle = Date.now() * 0.003 + (Math.PI / 2) * i;
      const dist = 50 + Math.sin(Date.now() * 0.005 + i) * 10;
      const ox = player.x + camX2 + Math.cos(angle) * dist;
      const oy = player.y + camY2 - 20 + Math.sin(angle) * dist * 0.4;
      const grad = ctx.createRadialGradient(ox, oy, 0, ox, oy, 12);
      grad.addColorStop(0, 'rgba(255,255,255,0.6)');
      grad.addColorStop(0.5, 'rgba(41,182,246,0.3)');
      grad.addColorStop(1, 'rgba(41,182,246,0)');
      ctx.fillStyle = grad;
      ctx.beginPath(); ctx.arc(ox, oy, 12, 0, Math.PI * 2); ctx.fill();
      ctx.fillStyle = `rgba(255,255,255,${0.2 + Math.random() * 0.3})`;
      ctx.fillRect(ox - 1, oy - 1 + Math.random() * 6, 3, 2);
    }
    ctx.restore();
  }

  ctx.restore();

  // Screen flash
  if (screenFlash > 0) {
    ctx.globalAlpha = screenFlash / 35;
    ctx.fillStyle = screenFlashColor;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.globalAlpha = 1;
    screenFlash--;
  }

  // Ultimate screen overlay (blue aura)
  if (player.isUltimate) {
    const pulse = 0.18 + Math.sin(Date.now() * 0.008) * 0.1;
    ctx.fillStyle = `rgba(13, 71, 161, ${pulse})`;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    const vigGrad = ctx.createRadialGradient(hw2, hh2, 80, hw2, hh2, CANVAS_W / 1.5);
    vigGrad.addColorStop(0, 'rgba(41,182,246,0)');
    vigGrad.addColorStop(0.5, `rgba(13,71,161,${0.15 + Math.sin(Date.now() * 0.005) * 0.05})`);
    vigGrad.addColorStop(1, `rgba(13,71,161,${0.4 + Math.sin(Date.now() * 0.003) * 0.1})`);
    ctx.fillStyle = vigGrad;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.fillStyle = `rgba(41,182,246,${0.02 + Math.sin(Date.now() * 0.01) * 0.01})`;
    const lineY = (Date.now() * 0.08) % CANVAS_H;
    ctx.fillRect(0, lineY, CANVAS_W, 3);
  }

  // World-splitting effect (ultimate) — Stage 2 (ALONG DASH TRAJECTORY)
  drawWorldSplitEffect(ctx, camX2, camY2);

  // UI
  drawUI(ctx, player, boss);
  drawStage2UI(ctx);
  if (player.hasSummon) drawSummonHint(ctx);
}

// ============================================================
// STAGE 3 - Grim Reaper
// ============================================================
function renderStage3(ctx) {
  // Dark abyss background
  ctx.fillStyle = '#050508';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Blood-red sky gradient
  const skyGrad = ctx.createLinearGradient(0, 0, 0, CANVAS_H);
  skyGrad.addColorStop(0, '#0a0008');
  skyGrad.addColorStop(0.3, '#15000a');
  skyGrad.addColorStop(0.5, '#1a0010');
  skyGrad.addColorStop(0.7, '#0f050a');
  skyGrad.addColorStop(1, '#050508');
  ctx.fillStyle = skyGrad;
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Mist/fog
  ctx.fillStyle = 'rgba(40,10,20,0.15)';
  for (let i = 0; i < 10; i++) {
    const fx = (i * 140 + Math.sin(Date.now() * 0.0004 + i * 3) * 30) % 1400 - 100;
    const fy = 250 + Math.sin(Date.now() * 0.0008 + i * 2) * 50;
    ctx.fillRect(fx, fy, 180 + Math.sin(i * 5) * 30, 100 + Math.sin(i * 2) * 20);
  }

  // Floating particles (souls)
  for (let i = 0; i < 20; i++) {
    const sx = (i * 67 + Date.now() * 0.02 * (1 + (i % 3) * 0.5)) % 1400 - 60;
    const sy = 100 + Math.sin(i * 3 + Date.now() * 0.003) * 80 + i * 25;
    const pAlpha = 0.05 + Math.sin(i * 7 + Date.now() * 0.005) * 0.03;
    ctx.fillStyle = `rgba(200,200,255,${pAlpha})`;
    const ps = 1 + (i % 3);
    ctx.fillRect(sx, sy, ps, ps);
  }

  const cam = camera;
  let sx3 = 0, sy3 = 0;
  if (shakeIntensity > 0) {
    sx3 = (Math.random() - 0.5) * shakeIntensity * 2;
    sy3 = (Math.random() - 0.5) * shakeIntensity * 2;
  }
  ctx.save();
  ctx.translate(sx3, sy3);

  const camX = Math.round(cam.x), camY = Math.round(cam.y);
  const hw = CANVAS_W / 2, hh = CANVAS_H / 2;
  const floorY = Math.round(GROUND_Y + camY);

  // Dark ground
  ctx.fillStyle = '#0a0a10';
  ctx.fillRect(0, floorY, CANVAS_W, CANVAS_H - floorY);
  ctx.fillStyle = '#1a1018';
  ctx.fillRect(0, floorY - 2, CANVAS_W, 3);

  // Broken pillars in background
  ctx.fillStyle = '#0d070a';
  for (let i = 0; i < 6; i++) {
    const bx = 150 + i * 180 + Math.sin(i * 2) * 20;
    const bh = 120 + Math.sin(i * 3) * 40;
    ctx.fillRect(bx, floorY - bh, 20, bh);
    ctx.fillRect(bx + 14, floorY - bh + 20, 6, bh - 40);
  }

  // Arena tiles
  ctx.fillStyle = 'rgba(30,15,20,0.15)';
  for (let i = 0; i < 12; i++) {
    const tx = ARENA_LEFT + (i * (ARENA_RIGHT - ARENA_LEFT) / 12) + camX;
    ctx.fillRect(tx, floorY - 10, (ARENA_RIGHT - ARENA_LEFT) / 12 - 2, 10);
  }

  // Arena bounds
  ctx.strokeStyle = 'rgba(255,23,68,0.08)';
  ctx.lineWidth = 2;
  ctx.strokeRect(ARENA_LEFT + camX, ARENA_TOP + camY, ARENA_RIGHT - ARENA_LEFT, GROUND_Y - ARENA_TOP);

  // Draw boss and game objects
  const renderables = [];
  if (boss && boss.alive) renderables.push({ type: 'boss', obj: boss });
  for (const m of minions) if (m.alive) renderables.push({ type: 'minion', obj: m });
  renderables.push({ type: 'player', obj: player });

  for (const p of projectiles) p.draw(ctx, cam);
  for (const r of renderables) {
    switch (r.type) {
      case 'player': r.obj.draw(ctx, cam); break;
      case 'boss': r.obj.draw(ctx, cam); break;
      case 'minion': r.obj.draw(ctx, cam); break;
    }
  }
  for (const p of particles) p.draw(ctx, cam);

  ctx.restore();

  // Screen flash
  if (screenFlash > 0) {
    ctx.globalAlpha = screenFlash / 35;
    ctx.fillStyle = screenFlashColor;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.globalAlpha = 1;
    screenFlash--;
  }

  // Ultimate screen overlay
  if (player.isUltimate) {
    const pulse = 0.18 + Math.sin(Date.now() * 0.008) * 0.1;
    ctx.fillStyle = `rgba(13, 71, 161, ${pulse})`;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    const vigGrad = ctx.createRadialGradient(hw, hh, 80, hw, hh, CANVAS_W / 1.5);
    vigGrad.addColorStop(0, 'rgba(41,182,246,0)');
    vigGrad.addColorStop(0.5, `rgba(13,71,161,${0.15 + Math.sin(Date.now() * 0.005) * 0.05})`);
    vigGrad.addColorStop(1, `rgba(13,71,161,${0.4 + Math.sin(Date.now() * 0.003) * 0.1})`);
    ctx.fillStyle = vigGrad;
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
    ctx.fillStyle = `rgba(41,182,246,${0.02 + Math.sin(Date.now() * 0.01) * 0.01})`;
    const lineY = (Date.now() * 0.08) % CANVAS_H;
    ctx.fillRect(0, lineY, CANVAS_W, 3);
  }

  // World-splitting effect
  drawWorldSplitEffect(ctx, camX, camY);

  // UI
  drawUI(ctx, player, boss);
  drawStage3UI(ctx);
  if (player.hasSummon) drawSummonHint(ctx);
}

function drawTitleScreen(ctx) {
  ctx.fillStyle = 'rgba(0,0,0,0.65)';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  // Title
  ctx.fillStyle = '#ce93d8';
  ctx.font = 'bold 52px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('SHADOW DAGGER', CANVAS_W / 2, 220);

  // Subtitle
  ctx.fillStyle = '#4fc3f7';
  ctx.font = '16px monospace';
  ctx.fillText('단검 암살자 vs 그림자 기사', CANVAS_W / 2, 260);

  // Menu items
  const items = [
    { label: '▶ BOSS FIGHT', y: 360, desc: '보스와의 전투' },
    { label: '▶ TRAINING', y: 410, desc: '자유 수련장' },
    { label: '▶ HOW TO PLAY', y: 460, desc: '조작법' },
  ];

  ctx.textAlign = 'center';
  for (let i = 0; i < items.length; i++) {
    const isSel = i === menuSelection;
    ctx.fillStyle = isSel ? '#ffffff' : 'rgba(255,255,255,0.4)';
    ctx.font = isSel ? 'bold 20px monospace' : '16px monospace';
    ctx.fillText(items[i].label, CANVAS_W / 2, items[i].y);
    if (isSel) {
      ctx.fillStyle = 'rgba(255,255,255,0.3)';
      ctx.font = '12px monospace';
      ctx.fillText(items[i].desc, CANVAS_W / 2, items[i].y + 22);
    }
  }

  // Controls hint
  ctx.fillStyle = 'rgba(255,255,255,0.25)';
  ctx.font = '11px monospace';
  ctx.fillText('↑ ↓ 이동 | SPACE 선택', CANVAS_W / 2, 540);
  // Current rank display on title
  ctx.fillStyle = 'rgba(255,255,255,0.2)';
  ctx.font = '10px monospace';
  const rankColors2 = ['#9e9e9e','#4fc3f7','#ffeb3b','#ff9800','#f44336','#e040fb'];
  ctx.fillStyle = rankColors2[strRank];
  ctx.fillText(`STR ${RANKS[strRank]}`, CANVAS_W / 2 - 70, 570);
  ctx.fillStyle = rankColors2[dexRank];
  ctx.fillText(`DEX ${RANKS[dexRank]}`, CANVAS_W / 2, 570);
  ctx.fillStyle = rankColors2[vitRank];
  ctx.fillText(`VIT ${RANKS[vitRank]}`, CANVAS_W / 2 + 70, 570);
  ctx.textAlign = 'left';
}

function drawGameOverScreen(ctx) {
  ctx.fillStyle = 'rgba(0,0,0,0.6)';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  ctx.fillStyle = '#f44336';
  ctx.font = 'bold 48px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('YOU DIED', CANVAS_W / 2, CANVAS_H / 2 - 20);

  ctx.fillStyle = COLORS.text;
  ctx.font = '16px monospace';
  ctx.fillText(`생존 시간: ${Math.floor(gameTimer / 60)}초`, CANVAS_W / 2, CANVAS_H / 2 + 30);

  const blink = Math.floor(Date.now() / 500) % 2 === 0;
  if (blink) {
    ctx.fillText('PRESS SPACE TO RETRY', CANVAS_W / 2, CANVAS_H / 2 + 80);
  }
  ctx.textAlign = 'left';
}

function drawVictoryScreen(ctx) {
  ctx.fillStyle = 'rgba(0,0,0,0.55)';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

  ctx.textAlign = 'center';

  // Boss defeated
  ctx.fillStyle = '#ffeb3b';
  ctx.font = 'bold 48px monospace';
  ctx.fillText('VICTORY!', CANVAS_W / 2, 200);

  ctx.fillStyle = COLORS.text;
  ctx.font = '16px monospace';
  const bossLabels = { 1: 'Shadow Knight', 2: 'Big Guardian', 3: 'Grim Reaper' };
  ctx.fillText(`${bossLabels[currentStage] || 'Boss'} 처치! (${Math.floor(gameTimer / 60)}초)`, CANVAS_W / 2, 250);

  // Skill acquisition (only on first clear, before rank up)
  if (currentStage === 1) {
    ctx.fillStyle = '#ce93d8';
    ctx.font = 'bold 22px monospace';
    ctx.fillText('✦ SKILL ACQUIRED ✦', CANVAS_W / 2, 320);
    ctx.fillStyle = '#e1bee7';
    ctx.font = '16px monospace';
    ctx.fillText('E - MINION SUMMON', CANVAS_W / 2, 350);
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    ctx.font = '13px monospace';
    ctx.fillText('보스의 힘을 흡수하여 그림자 정령을 소환한다', CANVAS_W / 2, 375);
  } else if (currentStage === 3) {
    ctx.fillStyle = '#ff1744';
    ctx.font = 'bold 22px monospace';
    ctx.fillText('✦ GRIM REAPER DEFEATED ✦', CANVAS_W / 2, 320);
    ctx.fillStyle = '#ff8a80';
    ctx.font = '16px monospace';
    ctx.fillText('모든 스테이지를 클리어했습니다!', CANVAS_W / 2, 350);
    ctx.fillStyle = 'rgba(255,255,255,0.5)';
    ctx.font = '13px monospace';
    ctx.fillText('축하합니다! 당신은 진정한 그림자 암살자입니다.', CANVAS_W / 2, 375);
  }
  // Rank Up display
  const rankColors = ['#9e9e9e','#4fc3f7','#ffeb3b','#ff9800','#f44336','#e040fb'];
  ctx.fillStyle = '#e040fb';
  ctx.font = 'bold 22px monospace';
  ctx.fillText('✦ ALL STATS UP! ✦', CANVAS_W / 2, 430);
  ctx.font = '14px monospace';
  ctx.fillStyle = rankColors[strRank];
  ctx.fillText(`STR ${RANKS[strRank]}`, CANVAS_W / 2 - 80, 455);
  ctx.fillStyle = rankColors[dexRank];
  ctx.fillText(`DEX ${RANKS[dexRank]}`, CANVAS_W / 2, 455);
  ctx.fillStyle = rankColors[vitRank];
  ctx.fillText(`VIT ${RANKS[vitRank]}`, CANVAS_W / 2 + 80, 455);
  ctx.fillStyle = 'rgba(255,255,255,0.4)';
  ctx.font = '11px monospace';
  ctx.fillText(`DMGx${getStrMult().toFixed(1)}  SPDx${getDexMult().toFixed(1)}  HPx${getVitMult().toFixed(1)}`, CANVAS_W / 2, 475);

  // Next stage hint
  if (currentStage === 1) {
    ctx.fillStyle = '#4fc3f7';
    ctx.font = '16px monospace';
    ctx.fillText('→ 다음 스테이지로 이동 준비 완료', CANVAS_W / 2, 490);
  }

  const blink = Math.floor(Date.now() / 500) % 2 === 0;
  if (blink && gameoverTimer > 60) {
    ctx.fillStyle = COLORS.text;
    ctx.font = '16px monospace';
    if (currentStage === 1 || currentStage === 2) {
      ctx.fillText('PRESS SPACE TO NEXT STAGE', CANVAS_W / 2, 540);
    } else {
      ctx.fillText('PRESS SPACE TO TITLE', CANVAS_W / 2, 540);
    }
  }
  ctx.textAlign = 'left';
}

function drawTrainingUI(ctx) {
  ctx.fillStyle = 'rgba(0,255,100,0.5)';
  ctx.font = '11px monospace';
  ctx.fillText('⚔ TRAINING MODE', 20, 70);
  ctx.fillStyle = 'rgba(255,255,255,0.3)';
  ctx.fillText('ESC to exit', 20, 90);
  ctx.fillStyle = 'rgba(255,255,255,0.15)';
  ctx.fillText('F1: Spawn Dummy', 20, 110);
}

function drawStage2UI(ctx) {
  ctx.fillStyle = '#ff5252';
  ctx.font = 'bold 14px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('⚔ STAGE 2 - Dark Forest', CANVAS_W / 2, 30);
  ctx.fillStyle = 'rgba(255,255,255,0.4)';
  ctx.font = '11px monospace';
  ctx.fillText('VS BIG GUARDIAN', CANVAS_W / 2, 50);
  ctx.fillStyle = 'rgba(255,255,255,0.15)';
  ctx.fillText('ESC to return to title', CANVAS_W / 2, 70);

  // Boss info (right side)
  ctx.textAlign = 'right';
  ctx.fillStyle = '#ce93d8';
  ctx.font = 'bold 12px monospace';
  ctx.fillText('— BOSS SKILLS —', CANVAS_W - 20, 120);
  ctx.fillStyle = 'rgba(255,255,255,0.5)';
  ctx.font = '11px monospace';
  const bskills = [
    '💥 지면 강타 (Ground Slam) - 붉은 구역 표시 후 충격파',
    '🔦 레이저 (Laser) - JUMP! 경고 후 피하기',
  ];
  for (let i = 0; i < bskills.length; i++) {
    ctx.fillText(bskills[i], CANVAS_W - 20, 140 + i * 22);
  }

  ctx.textAlign = 'left';
}

function drawStage3UI(ctx) {
  ctx.fillStyle = '#ff1744';
  ctx.font = 'bold 14px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('⚔ STAGE 3 - Grim Reaper', CANVAS_W / 2, 30);
  ctx.fillStyle = 'rgba(255,255,255,0.4)';
  ctx.font = '11px monospace';
  ctx.fillText('VS THE GRIM REAPER', CANVAS_W / 2, 50);
  ctx.fillStyle = 'rgba(255,255,255,0.15)';
  ctx.fillText('ESC to return to title', CANVAS_W / 2, 70);

  // Boss info (right side)
  ctx.textAlign = 'right';
  ctx.fillStyle = '#ff1744';
  ctx.font = 'bold 12px monospace';
  ctx.fillText('— GRIM REAPER SKILLS —', CANVAS_W - 20, 120);
  ctx.fillStyle = 'rgba(255,255,255,0.5)';
  ctx.font = '11px monospace';
  const skills = [
    '⚔ 베기 (Scythe Slash) - 넓은 범위 참격',
    '💨 돌진 (Scythe Charge) - 난 휘두르며 돌진',
    '💀 내려찍기 (Aerial Slam) - 공중에서 하강 후 충격파',
  ];
  for (let i = 0; i < skills.length; i++) {
    ctx.fillText(skills[i], CANVAS_W - 20, 140 + i * 22);
  }
  ctx.textAlign = 'left';
}

function drawControlsOverlay(ctx) {
  ctx.fillStyle = 'rgba(0,0,0,0.85)';
  ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#4fc3f7';
  ctx.font = 'bold 24px monospace';
  ctx.fillText('HOW TO PLAY', CANVAS_W / 2, 180);
  ctx.fillStyle = COLORS.text;
  ctx.font = '14px monospace';
  const controls = [
    '← →  Move',
    'SPACE  Jump / Double Jump',
    'Z  Attack (3-hit combo)',
    'X  Throwing Star',
    'C  Shadow Dash',
    'Q  Ultimate: Lightning Charge',
    'E  Summon Minion (acquired)',
    'Shift  Dodge (invincible)',
  ];
  for (let i = 0; i < controls.length; i++) {
    ctx.fillStyle = i === 6 && !hasSummonSkill ? 'rgba(255,255,255,0.2)' : COLORS.text;
    ctx.fillText(controls[i], CANVAS_W / 2, 230 + i * 30);
  }
  ctx.fillStyle = 'rgba(255,255,255,0.3)';
  ctx.font = '12px monospace';
  const blink = Math.floor(Date.now() / 500) % 2 === 0;
  if (blink) ctx.fillText('PRESS SPACE TO CLOSE', CANVAS_W / 2, 520);
  ctx.textAlign = 'left';
}

function drawSummonHint(ctx) {
  const cdRemain = Math.ceil(player.cooldownE / 60);
  const isReady = cdRemain <= 0 && player.mp >= 30;
  ctx.fillStyle = isReady ? 'rgba(206,147,216,0.7)' : 'rgba(206,147,216,0.25)';
  ctx.font = '10px monospace';
  let txt = 'E - MINION SUMMON';
  if (!isReady && cdRemain > 0) txt += ` (${cdRemain}s)`;
  ctx.fillText(txt, CANVAS_W - 140, CANVAS_H - 30);
  // Ready glow
  if (isReady) {
    ctx.fillStyle = `rgba(206,147,216,${0.04 + Math.sin(Date.now() * 0.006) * 0.03})`;
    ctx.fillRect(CANVAS_W - 150, CANVAS_H - 38, 145, 16);
  }
}

// ============================================================
// GAME LOOP
// ============================================================
let lastError = null;
initGame();

function safeCall(fn) {
  try {
    fn();
  } catch (e) {
    lastError = e;
    console.error('Game error:', e);
  }
}

function gameLoop() {
  safeCall(update);
  safeCall(render);
  requestAnimationFrame(gameLoop);
}

gameLoop();

// Draw error overlay if an error occurred
const origRender = render;
render = function() {
  try {
    origRender();
  } catch (e) {
    if (!lastError) { lastError = e; console.error('Render error:', e); }
  }
  if (lastError) {
    ctx.save();
    ctx.fillStyle = 'rgba(255,0,0,0.85)';
    ctx.font = '14px monospace';
    ctx.textAlign = 'center';
    ctx.fillText('ERROR: ' + lastError.message, CANVAS_W / 2, 30);
    ctx.fillText('Check F12 console for details', CANVAS_W / 2, 50);
    ctx.textAlign = 'left';
    ctx.restore();
  }
};
</script>
</body>
</html>
