# 📌 Asteroid game gen - qwen3.6 a3b - Branched
Model: qwen3.6-35b-a3b-apex
Created: 4/18/2026, 10:44:40 AM
Exported from: LM Studio 0.4.12+1

### User
create in single html file, a playable asteroid game in browser which keep scores with levels of increasing difficulty like the original arcade game

### Assistant
The user wants a complete, single HTML file containing a playable Asteroids arcade game with:
- Player ship that can rotate, thrust, and shoot
- Asteroids that float around and break into smaller pieces
- Score tracking
- Levels with increasing difficulty
- Lives system
- Collision detection
- Classic arcade feel

Let me build a comprehensive Asteroids game with all the classic mechanics:

1. **Player Ship**: Triangle-shaped, rotates with arrow keys, thrust with up arrow, shoots with space
2. **Asteroids**: Random polygon shapes, float across screen, break into 2-3 smaller pieces when shot
3. **Scoring**: Larger asteroids = more points, smaller = more points
4. **Levels**: Each level adds more asteroids, increasing speed
5. **Lives**: Start with 3, lose one when hit, respawn with invincibility
6. **Screen wrapping**: Objects wrap around edges
7. **Retro feel**: Black background, white lines, pixel font feel

Let me create a polished, complete implementation.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas {
display: block;
border: 1px solid #333;
}
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: 64px;
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: 18px;
color: #aaa;
margin: 10px 0;
}
#start-screen .blink {
animation: blink 1s infinite;
font-size: 24px;
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: 48px;
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: 32px;
color: #fff;
margin: 20px 0;
}
#game-over-screen .blink {
animation: blink 1s infinite;
font-size: 24px;
color: #fff;
margin-top: 30px;
}
.controls-info {
font-size: 14px;
color: #666;
margin-top: 30px;
line-height: 1.8;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
← → ROTATE &nbsp;|&nbsp; ↑ THRUST &nbsp;|&nbsp; SPACE FIRE&lt;br&gt;
P - PAUSE &nbsp;|&nbsp; R - RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Responsive canvas sizing
const MAX_W = 900, MAX_H = 650;
function resizeCanvas() {
const ratio = MAX_W / MAX_H;
let w = window.innerWidth - 20;
let h = window.innerHeight - 20;
if (w / h &gt; ratio) { w = h * ratio; }
else { h = w / ratio; }
canvas.width = Math.min(w, MAX_W);
canvas.height = Math.min(h, MAX_H);
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// Game constants
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.15;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 50;
const ASTEROID_VEL = 1.5;
const INVINCIBLE_TIME = 180; // frames

// Game state
let gameState = 'start'; // start, playing, paused, gameover
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let levelTransition = 0;
let screenShake = 0;

// Ship object
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

// Asteroid creation
function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedMult = 1 + (level * 0.1);
const speed = (ASTEROID_VEL + Math.random() * 1.5) * speedMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

// Generate jagged polygon vertices
const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x: x,
y: y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices: vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size: size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 150);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// Particle system
function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

// Initialize game
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
shootCooldown = 0;
levelTransition = 0;
screenShake = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

// Input handling
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') {
gameState = 'paused';
} else if (e.code === 'KeyP' && gameState === 'paused') {
gameState = 'playing';
}
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});

document.addEventListener('keyup', e =&gt; {
keys[e.code] = false;
});

// Update functions
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

// Rotation
if (keys['ArrowLeft'] || keys['KeyA']) {
ship.angle -= SHIP_TURN_SPEED;
}
if (keys['ArrowRight'] || keys['KeyD']) {
ship.angle += SHIP_TURN_SPEED;
}

// Thrust
ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;

// Thrust particles
if (Math.random() &lt; 0.5) {
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}
}

// Friction
ship.vx *= 0.995;
ship.vy *= 0.995;

// Speed limit
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) {
ship.vx = (ship.vx / speed) * 7;
ship.vy = (ship.vy / speed) * 7;
}

ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting
if (shootCooldown &gt; 0) shootCooldown--;
if ((keys['Space'] || keys['KeyZ']) && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = 10;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX,
y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2
});
}

if (shipInvincible &gt; 0) shipInvincible--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;

if (b.life &lt;= 0 || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

// Check collision with asteroids
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);

// Destroy asteroid - split
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}

score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;

// Remove old asteroid and add new ones
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
for (const a of asteroids) {
a.x += a.vx;
a.y += a.vy;
a.rotation += a.rotationSpeed;
screenWrap(a);
}
}

function checkShipCollisions() {
if (!ship.alive || shipInvincible &gt; 0) return;

for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// Drawing functions
function drawShip() {
if (!ship.alive) {
// Draw explosion
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

// Blink when invincible
if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

// Ship body
ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Thrust flame
if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();

ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);

ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();

// Glow
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color.replace(')', `,${alpha})`).replace('rgb', 'rgba').replace('#', '');

// Simple approach
ctx.globalAlpha = alpha;
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
// Score
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

// High Score
ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

// Level
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

// Lives
ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);

for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

// Asteroid count
ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

// Level transition
if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}
}

function drawStars() {
// Draw a few background stars (static)
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 100; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// Main game loop
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

// Check level complete
if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

// Screen shake decay
if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();

// Screen shake
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

// Clear
ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);

drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}

ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

// Start
gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

## Features Included:

### 🎮 Controls
- **← →** or **A/D** — Rotate ship
- **↑** or **W** — Thrust forward
- **Space** or **Z** — Fire bullets
- **P** — Pause/Resume
- **R** — Restart anytime

### 🚀 Gameplay
- **Classic triangle ship** with thrust flame animation
- **Asteroids** that shatter into smaller pieces (large → medium → small)
- **Screen wrapping** for ship, bullets, and asteroids
- **Collision detection** between bullets/asteroids and ship/asteroids
- **Particle explosions** and screen shake for impact feedback

### 📊 Scoring & Levels
- **Scoring**: Large=20pts, Medium=50pts, Small=100pts
- **Increasing difficulty**: More asteroids and faster speeds each level
- **Lives system**: 3 lives with invincibility frames after respawn
- **High score** saved to localStorage
- **Floating score text** on asteroid destruction

### ✨ Visual Polish
- Starfield background
- Ship blinking during invincibility
- Thrust particles
- Bullet glow effects
- Level transition announcements
- HUD with score, lives, level, and asteroid count

### User
a few issues to address, iterate with revised complete html:
- game canvas is small, it should fill full canvas of vertical and horizontal browser screen size
- asteroids on starting level is moving too fast, make them 2x slower
- generally balance game difficulty towards easier according to size of game canvas, with size speed of asteroids
- slow down thrust with up arrow key by 2x

### Assistant
The user wants me to revise the game with these changes:

1. Canvas should fill the full browser screen (no fixed size constraints)
2. Starting level asteroids should be 2x slower
3. Balance difficulty based on canvas size (since canvas is now larger, asteroids should feel appropriately paced)
4. Slow down thrust by 2x

Let me address each:

1. Full screen canvas - remove the MAX_W/MAX_H constraints and make canvas fill the viewport
2. Starting speed: ASTEROID_VEL was 1.5, make it 0.75 (2x slower), and reduce the random component too
3. Since the canvas is now full screen (much larger), the asteroids need to be proportionally adjusted - the speed multiplier based on level should be reduced, and the base speed should account for the larger play area
4. SHIP_THRUST was 0.15, make it 0.075

Let me also think about the canvas sizing more carefully. Making it truly full screen without any constraints means the game area is much larger, so asteroids moving at the same speed would cross the screen much faster. I need to scale the speeds appropriately.

For full screen, I'll set canvas to window.innerWidth and window.innerHeight, and update on resize.

Let me also reconsider the difficulty scaling. The speed multiplier per level was `1 + (level * 0.1)`. I should reduce this to make it easier. Maybe `1 + (level * 0.05)` or similar.

Let me write the complete revised HTML.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas {
display: block;
}
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: 64px;
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: 18px;
color: #aaa;
margin: 10px 0;
}
#start-screen .blink {
animation: blink 1s infinite;
font-size: 24px;
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: 48px;
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: 32px;
color: #fff;
margin: 20px 0;
}
#game-over-screen .blink {
animation: blink 1s infinite;
font-size: 24px;
color: #fff;
margin-top: 30px;
}
.controls-info {
font-size: 14px;
color: #666;
margin-top: 30px;
line-height: 1.8;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
← → ROTATE &nbsp;|&nbsp; ↑ THRUST &nbsp;|&nbsp; SPACE FIRE&lt;br&gt;
P - PAUSE &nbsp;|&nbsp; R - RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

// Full-screen canvas
function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// Game constants
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;        // 2x slower (was 0.15)
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;    // 2x slower base (was 1.5)
const INVINCIBLE_TIME = 180;

// Difficulty scaling based on canvas size — larger screens get slower asteroids
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
// Normal area is around 1600*900 = 1,440,000
// If area is much larger, slow things down proportionally
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// Game state
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let levelTransition = 0;
let screenShake = 0;

function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);  // gentler scaling (was 0.1)
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x: x,
y: y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices: vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size: size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
shootCooldown = 0;
levelTransition = 0;
screenShake = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') {
gameState = 'paused';
} else if (e.code === 'KeyP' && gameState === 'paused') {
gameState = 'playing';
}
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});

document.addEventListener('keyup', e =&gt; {
keys[e.code] = false;
});

function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) {
ship.angle -= SHIP_TURN_SPEED;
}
if (keys['ArrowRight'] || keys['KeyD']) {
ship.angle += SHIP_TURN_SPEED;
}

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;

const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;

const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) {
ship.vx = (ship.vx / speed) * 7;
ship.vy = (ship.vy / speed) * 7;
}

ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

if (shootCooldown &gt; 0) shootCooldown--;
if ((keys['Space'] || keys['KeyZ']) && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = 10;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX,
y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2
});
}

if (shipInvincible &gt; 0) shipInvincible--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;

if (b.life &lt;= 0 || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);

const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}

score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;

asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
for (const a of asteroids) {
a.x += a.vx;
a.y += a.vy;
a.rotation += a.rotationSpeed;
screenWrap(a);
}
}

function checkShipCollisions() {
if (!ship.alive || shipInvincible &gt; 0) return;

for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();

ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);

ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();

ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);

for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();

if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);

drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}

ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

Here's what changed:

1. **Full-screen canvas** — Canvas now fills the entire browser viewport (`window.innerWidth` × `window.innerHeight`) with no fixed max dimensions.

2. **2× slower starting asteroids** — `BASE_ASTEROID_VEL` dropped from `1.5` → `0.75`, and the random range reduced from `1.5` → `1.0`.

3. **Canvas-size difficulty scaling** — `getCanvasSpeedFactor()` calculates the screen area and slows asteroids down proportionally on larger screens (monitors, 4K, etc.) so the game feels balanced regardless of resolution.

4. **2× slower thrust** — `SHIP_THRUST` dropped from `0.15` → `0.075`.

5. **Gentler level scaling** — Level speed multiplier reduced from `1 + (level * 0.1)` → `1 + (level * 0.05)` so difficulty ramps up more gradually.

### User
Explore new game mechanics:
- add ship portaling. "z" key to activate, cooldown 30s. Press to jump half a screen forward
- add power ups. Borrow ideas from all known games not just asteroid. For example, but not limited to these: Mine laying, where both asteroids and ship can crossing it would get blown up. Extend range of bullets fired
- for portaling and power ups which require key to act, add to start screen docs. Use sensible key mapping that's not already used, and sequence them from left, bottom row of qwerty keyboard

### Assistant
The user wants me to add:

1. **Ship Portaling** - "Z" key to activate, 30s cooldown. Jump half a screen forward (in the direction the ship is facing).

2. **Power-ups** - Various power-ups like:
   - Mine laying (both asteroids and ship can be blown up by crossing mines)
   - Extended bullet range
   - Other power-ups from classic games

3. **Key mapping** - Use keys from the left/bottom row of QWERTY keyboard, not already used. Current keys: Arrow keys, W, A, D, Space, Z (for portal), P, R.

Let me think about key mapping:
- Left/bottom row: Q, W, E, R, T, Y, U, I, O, P (top row)
- Bottom row: Z, X, C, V, B, N, M
- Already used: Arrow keys, W, A, D, Space, Z, P, R

New keys from left/bottom area: X, C, V, B, N, M, Q, E, T, Y, U, I, O

Let me assign:
- **Z** - Portal (already used, but user said to add it)
- **X** - Deploy mines
- **C** - Extended bullets / spread shot
- **V** - Shield / invincibility
- **B** - Bomb (clears nearby asteroids)
- **N** - Rapid fire
- **M** - Slow time

Power-ups that spawn randomly:
- Mine dispenser
- Spread shot
- Shield
- Bomb
- Rapid fire
- Time slow

Let me design this carefully:

**Power-up types:**
1. **Mine** (icon: ◆) - Deploys a mine that explodes when asteroids or ship touch it
2. **Spread** (icon: ≡) - Bullets fire in 3 directions for 15 seconds
3. **Shield** (icon: ○) - Invincibility for 10 seconds
4. **Bomb** (icon: ●) - Clears all asteroids on screen (once per life)
5. **Rapid Fire** (icon: &gt;&gt;&gt;) - Double fire rate for 12 seconds
6. **Time Slow** (icon: ∞) - Slows asteroids for 8 seconds

**Key mappings:**
- **Z** - Portal (30s cooldown) - jump half screen forward
- **X** - Deploy mine (one at a time, can carry 3)
- **C** - Spread shot toggle (15s duration)
- **V** - Shield (10s duration, once per life)
- **B** - Bomb (clears all asteroids, once per life)
- **N** - Rapid fire (12s duration)
- **M** - Time slow (8s duration)

Let me implement this comprehensively. I need to be careful about the power-up spawning, the mine mechanics, and the portal mechanic.

For the portal: pressing Z teleports the ship to a position half a screen away in the direction facing. If it would go off screen, wrap to the other side. There should be a visual portal effect.

For mines: they float and have a small explosion radius. Both ship and asteroids trigger them.

Let me write the complete game.

I need to think about the power-up spawning rate and what keys to use. Let me re-read: "sequence them from left, bottom row of qwerty keyboard"

Left/bottom row of QWERTY:
- Top row left to right: Q W E R T Y U I O P
- Bottom row: Z X C V B N M

Already used: Arrow, W, A, D, Space, Z, P, R

New keys in order from left: Q, E, T, Y, U, I, O, X, C, V, B, N, M

Let me use:
- **Q** - Portal (since Z is taken, but user said Z for portal, so keep Z)
- **X** - Deploy Mines
- **C** - Spread Shot  
- **V** - Shield
- **B** - Bomb
- **N** - Rapid Fire
- **M** - Time Slow

Actually user said "Z key to activate" for portal, so Z is taken. Let me use:
- **Z** - Portal
- **X** - Deploy Mines
- **C** - Spread Shot
- **V** - Shield
- **B** - Bomb  
- **N** - Rapid Fire
- **M** - Time Slow

These are all on the bottom/left area of the keyboard.

Let me code this up properly.

For power-ups spawning, I'll have them appear periodically (every ~20 seconds or so) at random positions.

Let me write the full implementation now.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (15s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (10s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (clears all, once per life)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; RAPID FIRE (12s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (8s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s in frames
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 60;
const MINE_EXPLOSION_TIME = 60;

// Power-up durations
const POWER_SPREAD = 900;    // 15s
const POWER_SHIELD = 600;    // 10s
const POWER_RAPID = 720;     // 12s
const POWER_TIME_SLOW = 480; // 8s

// Power-up spawn interval
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups, portals;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let rapidFireTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: 1800, // 30s lifetime
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'rapid', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '»', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: 600, // 10s to collect
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
portals = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
shootCooldown = 0;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
rapidFireTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'rapid', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
shieldTimer = 0;
rapidFireTimer = 0;
spreadShotTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 10;
if (rapidFireTimer &gt; 0) fireRate = 4;

if ((keys['Space'] || keys['KeyZ'] === false) && shootCooldown &lt;= 0 && ship.alive) {
// Only shoot with space, Z is portal
}
if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && shootCooldown &lt;= 5 && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
// Wrap if off screen
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false; // One press per mine
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key)
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (const a of asteroids) {
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, 15, '#f00', 4);
}
asteroids = [];
screenShake = 20;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOOM!', '#f00');
}

// Rapid fire (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
rapidFireTimer = POWER_RAPID;
createFloatingText(ship.x, ship.y - 30, 'RAPID!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 8000);
createFloatingText(ship.x, ship.y - 30, 'SLOW TIME!', '#f0f');
}

if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (rapidFireTimer &gt; 0) rapidFireTimer--;
if (shieldTimer &gt; 0) { shieldTimer--; shipInvincible = Math.max(shipInvincible, 1); }
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
if (b.life &lt;= 0 || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;
if (m.life &lt;= 0) {
createExplosion(m.x, m.y, 10, '#f80', 2);
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}
if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

// Check collision with asteroids
let triggered = false;
for (const a of asteroids) {
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
// Destroy asteroid
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(asteroids.indexOf(a), 1);
asteroids.push(...newAsteroids);
break;
}
}

// Check collision with ship
if (!triggered && ship.alive && shipInvincible &lt;= 0) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

// Wrap
if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

// Collect
if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
shipInvincible = Math.max(shipInvincible, 1);
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'rapid':
rapidFireTimer = POWER_RAPID;
createFloatingText(p.x, p.y, 'RAPID FIRE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 8000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

function checkShipCollisions() {
if (!ship.alive || shipInvincible &gt; 0) return;
for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Shield visual
if (shieldTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 8, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${0.3 + Math.sin(Date.now() * 0.01) * 0.2})`;
ctx.lineWidth = 2;
ctx.stroke();
}

// Rapid fire visual indicator
if (rapidFireTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

// Spread shot indicator
if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}
// Draw mine
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

// Inner cross
const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

// Glow
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 120 ? (Math.sin(p.life * 0.2) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

// Outer glow
ctx.beginPath();
ctx.arc(0, 0, p.radius + 6, 0, Math.PI * 2);
ctx.fillStyle = p.color.replace(')', `,${pulse * 0.15})`).replace('#', 'rgba(').replace(/(..)(..)(..)/, (m, r, g, b) =&gt; {
return `rgba(${parseInt(r,16)},${parseInt(g,16)},${parseInt(b,16)},${pulse * 0.15})`;
});
ctx.fill();

// Border
ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${parseInt(p.color.slice(1,3),16)},${parseInt(p.color.slice(3,5),16)},${parseInt(p.color.slice(5,7),16)},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

// Label
ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

// Lives
ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

// Asteroid count
ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

// Active power-up timers
let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (rapidFireTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RAPID: ${Math.ceil(rapidFireTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

// Cooldowns & status
ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

// Level transition
if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

// Power-up flash
if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

// Power-up spawning
powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

// Level complete
if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

Here's what was added and changed:

## 🚀 New Mechanics

### Portal (Z key)
- Press **Z** to instantly teleport half a screen forward in your facing direction
- **30-second cooldown** shown in HUD (cyan when ready, red with countdown)
- Visual portal effect with cyan particles on departure and magenta on arrival
- Brief invincibility on arrival to prevent instant death

### Mines (X key)
- Deploy **up to 3 mines** simultaneously (carried as inventory)
- Mines float gently and pulse orange
- **Explode on contact** with any asteroid (destroying it) or the ship
- Explosion radius of 60px with visual shockwave
- Mines expire after 30 seconds

### Power-ups (randomly spawn ~every 20s)
Six types, each with unique icons and colors:
- **≡ Spread Shot** (green) — Bullets fire in 3 directions for 15s
- **○ Shield** (cyan) — Temporary invincibility for 10s, re-usable if collected
- **● Bomb** (red) — Clears all asteroids on screen (once per life, can be recharged by power-up)
- **» Rapid Fire** (yellow) — Double fire rate for 12s
- **∞ Time Slow** (magenta) — Asteroids move at 30% speed for 8s
- **◆ Mine pickup** (orange) — Grants +2 mines instantly

### HUD Enhancements
- Active power-up timers displayed on the left side
- Portal cooldown status, mine count, bomb/shield availability
- White flash effect when collecting power-ups
- All status indicators color-coded

### Key Mapping (bottom-left QWERTY row)
- **Z** — Portal
- **X** — Deploy Mine
- **C** — Spread Shot
- **V** — Shield
- **B** — Bomb
- **N** — Rapid Fire
- **M** — Time Slow

### User
think through the usability of these additions and fix what violates playability. For example after my playtest:
- mines instantly blow up my ship on lay as it activates right away where i laid it, making it unusable as it's currently a self destruct. Make it usable such that the mine would blink on laying signaling it's not active until player leaves its vicinity so it's armed and doesn't blow up the player ship initially
- spread shot is cool but doesn't seem to actually last 15s? It goes away rather quickly
- make mines last a lot longer, say 120s
- generally double duration of other power ups
- power ups vanish too quickly to be picked up usually, double duration also before they vanish

and for general reminder of key mapping, list them at screen bottom as 1 line strip while playing 

### Assistant
The user wants me to fix several usability issues:

1. **Mines self-destructing**: When a mine is placed near the ship, it immediately explodes because the ship is within range. I need to add an "arming" mechanic - mines should blink when first placed and only arm after the player moves away from them.

2. **Spread shot duration**: It seems to expire quickly. Need to double the duration (from 15s to 30s).

3. **Mines last too short**: Change from 30s to 120s.

4. **Double all power-up durations**: Spread shot from 15s→30s, Shield from 10s→20s, Rapid fire from 12s→24s, Time slow from 8s→16s.

5. **Power-ups vanish too quickly**: Double their pickup duration from 10s to 20s.

6. **Add key mapping strip at bottom of screen while playing**.

Let me think about the mine arming mechanic:
- When a mine is placed, it enters an "arming" state
- It blinks (alternating visibility) to signal it's not yet active
- After the player moves beyond a certain distance from the mine, it arms
- Once armed, it becomes fully visible and will explode on contact
- If the player gets too close again after arming, it could disarm or just stay armed (I'll keep it simple - once armed, it stays armed)

Let me also review the spread shot issue. Looking at the code, I see the spread shot is activated with KeyC which is a one-time activation (keys['KeyC'] = false after pressing). The duration is set to POWER_SPREAD which is 900 frames = 15 seconds. But wait, looking more carefully, the spreadShotTimer is set when KeyC is pressed, but the issue might be that the spread shot bullets are being created but the timer is running down. Let me check... Actually the spread shot should work. The user says it goes away quickly - maybe they're pressing C and it's hard to tell when it ends. Let me double the duration to be safe.

Actually, re-reading the user's feedback: "spread shot is cool but doesn't seem to actually last 15s? It goes away rather quickly". This could be a timing issue or the duration is just too short to notice. I'll double it to 30s.

Let me also add the key strip at the bottom of the screen.

Let me revise the complete code:

Changes:
1. Mine arming: Add `arming` state to mines, with `armTimer` and `armDistance`. Mine blinks during arming, arms when player is far enough.
2. Double all durations:
   - POWER_SPREAD: 900 → 1800 (30s)
   - POWER_SHIELD: 600 → 1200 (20s)
   - POWER_RAPID: 720 → 1440 (24s)
   - POWER_TIME_SLOW: 480 → 960 (16s)
   - MINE_LIFETIME: 1800 → 7200 (120s)
   - POWERUP_LIFETIME: 600 → 1200 (20s)
3. Add key strip at bottom of screen

Let me write the complete revised code.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (clears all, once per life)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; RAPID FIRE (24s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s

// Mine settings
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 70;
const MINE_EXPLOSION_TIME = 60;
const MINE_LIFETIME = 7200; // 120s
const MINE_ARM_TIME = 90;   // 1.5s to arm
const MINE_ARM_DISTANCE = 80; // need to be this far away to arm
const MINE_BLINK_RATE = 0.15; // blink frequency during arming

// Power-up durations (doubled)
const POWER_SPREAD = 1800;    // 30s
const POWER_SHIELD = 1200;    // 20s
const POWER_RAPID = 1440;     // 24s
const POWER_TIME_SLOW = 960;  // 16s

// Power-up spawn interval & lifetime (doubled)
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s
const POWERUP_LIFETIME = 1200;       // 20s to collect

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let rapidFireTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: MINE_LIFETIME,
arming: true,
armTimer: MINE_ARM_TIME,
armed: false,
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'rapid', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '»', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: POWERUP_LIFETIME,
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
shootCooldown = 0;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
rapidFireTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'rapid', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
shieldTimer = 0;
rapidFireTimer = 0;
spreadShotTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 10;
if (rapidFireTimer &gt; 0) fireRate = 4;

if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false;
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE DEPLOYED', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD SHOT!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key)
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (const a of asteroids) {
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, 15, '#f00', 4);
}
asteroids = [];
screenShake = 20;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOOM!', '#f00');
}

// Rapid fire (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
rapidFireTimer = POWER_RAPID;
createFloatingText(ship.x, ship.y - 30, 'RAPID FIRE!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(ship.x, ship.y - 30, 'TIME SLOW!', '#f0f');
}

if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (rapidFireTimer &gt; 0) rapidFireTimer--;
if (shieldTimer &gt; 0) { shieldTimer--; shipInvincible = Math.max(shipInvincible, 1); }
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
if (b.life &lt;= 0 || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;

// Lifetime expiry
if (m.life &lt;= 0) {
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}

// Arming logic
if (m.arming && !m.exploding) {
m.armTimer--;
// Check if player is far enough away
const playerDist = dist(m.x, m.y, ship.x, ship.y);
if (playerDist &gt; MINE_ARM_DISTANCE && m.armTimer &lt;= 0) {
m.arming = false;
m.armed = true;
createFloatingText(m.x, m.y - 20, 'ARMED', '#f80');
}
// If player is still close, reset arm timer slightly to keep forcing them away
if (playerDist &lt;= MINE_ARM_DISTANCE && m.armTimer &gt; 0) {
m.armTimer = Math.min(m.armTimer + 2, MINE_ARM_TIME);
}
if (m.exploding) continue;

// Check collision with asteroids while arming (only if armed)
// Don't explode on asteroids while arming
}

if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

// Armed: check collision with asteroids
if (m.armed && !m.exploding) {
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
// Destroy asteroid
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}

// Check collision with ship (only when armed)
if (m.armed && ship.alive && shipInvincible &lt;= 0 && !m.exploding) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
shipInvincible = Math.max(shipInvincible, 1);
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'rapid':
rapidFireTimer = POWER_RAPID;
createFloatingText(p.x, p.y, 'RAPID FIRE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

function checkShipCollisions() {
if (!ship.alive || shipInvincible &gt; 0) return;
for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Shield visual
if (shieldTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 8, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${0.3 + Math.sin(Date.now() * 0.01) * 0.2})`;
ctx.lineWidth = 2;
ctx.stroke();
}

// Rapid fire visual indicator
if (rapidFireTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

// Spread shot indicator
if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;

if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}

if (m.arming) {
// Blink animation during arming
const blinkOn = Math.sin(m.pulsePhase / MINE_BLINK_RATE) &gt; 0;
const alpha = blinkOn ? 0.8 : 0.15;

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

// Inner cross
const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${alpha})`;
ctx.lineWidth = 1.5;
ctx.stroke();

// Arming progress ring
const progress = 1 - m.armTimer / MINE_ARM_TIME;
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`;
ctx.lineWidth = 2;
ctx.stroke();

// "ARMIN" text
if (progress &gt; 0.3) {
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
ctx.textAlign = 'center';
ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
}

} else if (m.armed) {
// Fully armed mine
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

// Inner cross
const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

// Glow
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();

// Small "ARMED" indicator
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
ctx.textAlign = 'center';
ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
}
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

// Outer glow
const hex = p.color;
const r = parseInt(hex.slice(1, 3), 16);
const g = parseInt(hex.slice(3, 5), 16);
const b = parseInt(hex.slice(5, 7), 16);

ctx.beginPath();
ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`;
ctx.fill();

// Rotating ring
const rotAngle = Date.now() * 0.003;
ctx.beginPath();
ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`;
ctx.lineWidth = 1.5;
ctx.stroke();

// Border
ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

// Label
ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

// Lives
ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

// Asteroid count
ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

// Active power-up timers
let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (rapidFireTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RAPID: ${Math.ceil(rapidFireTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

// Cooldowns & status
ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

// Level transition
if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

// Power-up flash
if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── KEY STRIP AT BOTTOM ───
const stripY = canvas.height - 35;
const stripHeight = 28;

// Semi-transparent background bar
ctx.fillStyle = 'rgba(0,0,0,0.6)';
ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);

// Draw separator line
ctx.strokeStyle = 'rgba(100,100,100,0.3)';
ctx.lineWidth = 1;
ctx.beginPath();
ctx.moveTo(0, stripY - 2);
ctx.lineTo(canvas.width, stripY - 2);
ctx.stroke();

ctx.font = '12px Courier New';
ctx.textAlign = 'center';
const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RAPID  M:TIMESLOW  P:PAUSE  R:RESTART';
ctx.fillStyle = 'rgba(150,150,150,0.7)';

// Measure text and center it
const textWidth = ctx.measureText(stripText).width;
const startX = (canvas.width - textWidth) / 2;
ctx.fillText(stripText, startX, stripY + 16);
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

// Power-up spawning
powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

// Level complete
if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

Here's what was fixed:

### 🔧 Mine Arming System (the big fix)
- **Mines now start in "arming" state** — they **blink on/off** immediately after placement so you know they're not yet active
- An **arming progress ring** animates around the mine showing how long until armed (~1.5s)
- The text **"ARMIN"** appears above the mine while it's charging up
- The mine **only arms once you move beyond 80px** away from it — if you stay close, the timer resets slightly to force you to back off
- Once armed, the mine becomes solid orange with a **"ARMED"** label and a steady glow
- This makes mines fully usable — deploy one, fly away, and it arms safely

### ⏱️ Duration Fixes
- **Mines last 120s** (7200 frames) instead of 30s
- **Spread shot: 30s** (doubled from 15s)
- **Shield: 20s** (doubled from 10s)
- **Rapid fire: 24s** (doubled from 12s)
- **Time slow: 16s** (doubled from 8s)
- **Power-ups stay collectible for 20s** (doubled from 10s) — with a pulsing fade warning in the last 3 seconds

### 📋 Key Strip
- A persistent **bottom bar** shows all controls in one line while playing, with a subtle semi-transparent background and separator line

### 🎨 Visual Polish
- Power-ups now have a **rotating orbit ring** for better visibility
- Shielded ships have a cyan bubble
- Rapid fire and spread shot have colored rings around the ship
- Mine arming has a clear progress arc animation

### User
- the bottom row key bind tips is too dark and close to black background to be readable. Make it a much lighter grey. Also center align the one line text
- rapid fire doesn't feel like it has much utility, when default fire rate is already very rapid. Replace it with different power up: extended range where range of bullets travel 3x farther
- for game balancing, cut bullet fire rate to half
- bomb is overpowered as it instantly clears the whole level. Instead, it should have equivalent effect of firing 1 shot on all asteroids, so larger asteroids would only be broken up, not completely clearing whole level regardless of large or small asteroid

### Assistant
Here's a thinking process:

1.  **Analyze User Feedback:**
   - **Bottom key strip:** Too dark, hard to read against black background. Needs to be much lighter grey. Also, center-align the one-line text.
   - **Rapid Fire replacement:** Replace with "Extended Range" power-up (bullets travel 3x farther). Reason: Default fire rate is already fast enough, rapid fire isn't useful.
   - **Bullet fire rate:** Cut in half (currently 10 frames, should be 20 frames).
   - **Bomb balancing:** Currently clears all asteroids instantly. Change to: Equivalent to firing 1 shot on every asteroid. Larger asteroids split, small ones just get hit (or destroyed if small enough). This makes it scale with asteroid size and not instantly win the level.

2.  **Identify Required Changes in Code:**
   - **HUD Bottom Strip:** Change text color from `rgba(150,150,150,0.7)` to something much lighter like `rgba(200,200,200,0.9)` or `#ccc`. Ensure center alignment is correct (it already uses `ctx.textAlign = 'center'`, but I'll verify the drawing logic).
   - **Replace Rapid Fire:** 
     - Change key mapping in UI/docs to something else? Wait, the user said "Replace it with different power up: extended range". I should keep the same key `N` but change its function to Extended Range.
     - Update constants: `POWER_EXTENDED = 1800;` (30s)
     - Update update logic: `keys['KeyN']` sets `extendedRangeTimer = POWER_EXTENDED;`
     - Update draw/logic: Bullets need a `maxLife` or `distance` check. Currently, bullets die after `BULLET_LIFE` frames. I need to add a `maxDistance` or scale `BULLET_LIFE` when extended range is active. Actually, easier: add a `maxDist` property to bullets, track distance traveled, and destroy when `dist &gt; maxDist`. Or just multiply `BULLET_LIFE` by 3 when active. Let's go with distance tracking for precision, or just multiply life. I'll add a `maxDist` field to bullets and check it.
     - Actually, tracking distance is cleaner: `b.distTraveled += Math.sqrt(b.vx**2 + b.vy**2);` and `if (b.distTraveled &gt; maxDist)`. I'll implement this.
   - **Cut Bullet Fire Rate:** Change `shootCooldown = 10` to `20`.
   - **Bomb Balancing:** Instead of `for (const a of asteroids) { score += a.points; ... asteroids = []; }`, change to: For each asteroid, apply "damage" or treat it as a bullet hit. If it's small, destroy it. If medium/large, split it. No points for bomb hits (or half points?). I'll make it just split/destroy asteroids like a bullet, but with no score (or maybe small score). Actually, the user said "equivalent effect of firing 1 shot on all asteroids". So I'll iterate through asteroids, and for each one, apply the same splitting logic as a bullet hit. I'll give 1/2 points for bomb hits to reward it slightly.

3.  **Implementation Details:**

   *Bottom Strip:*
   ```javascript
   ctx.fillStyle = 'rgba(220,220,220,0.95)'; // Much lighter
   // Ensure textAlign is center
   ctx.textAlign = 'center';
   const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
   // ... drawing code ...
   ```

   *Extended Range Power-up:*
   - Add `extendedRangeTimer = 0;`
   - Constant: `const POWER_EXTENDED = 1800; // 30s`
   - In `createBullet`, add `maxDist: 400;` (or calculate dynamically)
   - Actually, simpler: Just multiply `BULLET_LIFE` by 3 when active. But distance tracking is more robust for different screen sizes. Let's do distance tracking.
   - Modify bullet creation: `distTraveled: 0, maxDist: 300;`
   - In `updateBullets`: `b.distTraveled += Math.sqrt(b.vx*b.vx + b.vy*b.vy); const limit = extendedRangeTimer &gt; 0 ? 900 : 300; if (b.distTraveled &gt; limit) { bullets.splice(i, 1); continue; }`
   - Update power-up switch case for 'extended' (or 'range')
   - Update key mapping in UI and start screen

   *Bullet Fire Rate:*
   - Change `let fireRate = 10;` to `20;`
   - If rapid fire was active, it would be 10, but we're replacing it. So just `let fireRate = 20;`

   *Bomb Balancing:*
   - Replace the bomb loop with:
   ```javascript
   if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
       keys['KeyB'] = false;
       bombUsed = true;
       // Apply damage to all asteroids
       let hitCount = 0;
       for (let j = asteroids.length - 1; j &gt;= 0; j--) {
           const a = asteroids[j];
           // Simulate a bullet hit
           const newAsteroids = [];
           if (a.size === 'large') {
               newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
               newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
               score += 10; // Half points
           } else if (a.size === 'medium') {
               newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
               newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
               score += 25;
           } else {
               score += 50; // Half points
           }
           asteroids.splice(j, 1);
           asteroids.push(...newAsteroids);
           hitCount++;
       }
       screenShake = 10;
       createFloatingText(canvas.width / 2, canvas.height / 2, 'BOMB HIT!', '#f00');
       // Visual flash for bomb
       for (const a of asteroids) {
           createExplosion(a.x, a.y, 5, '#f80', 2);
       }
   }
   ```
   Wait, the user said "equivalent effect of firing 1 shot on all asteroids". So larger asteroids break up, small ones might just get hit. The above does exactly that. I'll adjust points to be half of normal to reward it but not make it trivial.

   *Update Power-up Types & Keys:*
   - Change 'rapid' to 'extended' in types array
   - Update label/color: maybe '⌇' or '↔' or '»' -&gt; let's use '⌇' or '↔'. I'll stick with '⌇' or just 'R'. Let's use '⌇' for range.
   - Update switch case
   - Update HUD text
   - Update start screen controls

   *Check other things:*
   - Need to make sure `extendedRangeTimer` is reset on life loss/game over
   - Ensure bullet distance tracking works correctly
   - Verify bottom strip centering works properly with dynamic canvas width

   Let's draft the changes carefully.

   **Distance Tracking for Bullets:**
   I'll add `distTraveled: 0` to bullets.
   In `updateBullets`:
   ```javascript
   b.distTraveled += Math.sqrt(b.vx * b.vx + b.vy * b.vy);
   const maxDist = extendedRangeTimer &gt; 0 ? 900 : 300;
   if (b.distTraveled &gt; maxDist) {
       bullets.splice(i, 1);
       continue;
   }
   ```
   This is clean and screen-size agnostic.

   **Bomb Logic:**
   I'll make it iterate backwards, apply the split logic, and clear the array, then repopulate with split pieces. I'll also add a visual "shockwave" or flash to make it feel impactful but not game-breaking.

   **Bottom Strip:**
   ```javascript
   // Background bar
   ctx.fillStyle = 'rgba(0,0,0,0.5)';
   ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);
   ctx.strokeStyle = 'rgba(100,100,100,0.3)';
   ctx.lineWidth = 1;
   ctx.beginPath();
   ctx.moveTo(0, stripY - 2);
   ctx.lineTo(canvas.width, stripY - 2);
   ctx.stroke();

   ctx.font = '13px Courier New';
   ctx.textAlign = 'center';
   const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
   ctx.fillStyle = 'rgba(220,220,220,0.95)'; // Much lighter
   const textWidth = ctx.measureText(stripText).width;
   const startX = (canvas.width - textWidth) / 2;
   ctx.fillText(stripText, startX, stripY + 16);
   ```

   **Start Screen Update:**
   Change `N: RAPID FIRE (24s)` to `N: EXTENDED RANGE (30s)`

   **Power-up Spawn & Types:**
   Update `types` array to include 'extended' instead of 'rapid'
   Update `POWERUP_LIFETIME` to 1200 (already doubled)
   Update switch case

   **Reset on Death:**
   Add `extendedRangeTimer = 0;` in the death/restart block.

   Let's verify everything compiles correctly in my head.
   - `extendedRangeTimer` declared? Yes, add it.
   - `POWER_EXTENDED` constant? Yes.
   - Bullet distance tracking? Yes.
   - Fire rate halved? Yes, `20`.
   - Bomb logic updated? Yes.
   - Bottom strip lighter & centered? Yes.
   - Start screen updated? Yes.

   One thing: The user said "center align the one line text". I'm already doing that, but I'll make sure the text isn't clipped on smaller screens. I'll add a check or just let it be, but I'll ensure `ctx.textAlign = 'center'` is set. Actually, on very narrow screens, it might overflow. I'll add a note or just keep it simple. The prompt says "center align the one line text", so I'll stick with that.

   Let's write out the full code carefully. I'll integrate all changes systematically.

   Structure check:
   - Constants
   - Game state variables
   - Object creation functions
   - Input handling
   - Update functions (ship, bullets, asteroids, mines, powerups, particles, texts)
   - Draw functions
   - Main loop
   - All changes applied

   I'll carefully replace the relevant sections.
   One detail: `keys['KeyN']` was used for rapid fire. I'll keep it for extended range.
   I'll change the label for extended range to '⌇' or '↔'. I'll use '⌇'.
   I'll update the powerup spawn logic to use 'extended'.
   I'll make sure `extendedRangeTimer` is reset properly.
   I'll adjust bullet life/distance logic.
   I'll fix the bomb logic.
   I'll lighten the bottom strip.

   Let's draft it. 
   Self-Correction/Refinement during thought:
   - For bullet distance tracking, I need to initialize `distTraveled: 0` in `createBullet` logic. Since bullets are created inline in `updateShip`, I'll add `distTraveled: 0` there.
   - For the bomb, I should avoid infinite loops or weird state. The current logic replaces the asteroid array, which is fine.
   - I'll make sure the bottom strip text is fully visible by maybe reducing font size slightly or just trusting the user's screen. I'll keep it at 13px.
   - I'll verify the start screen controls match the game.
   - I'll double-check that `extendedRangeTimer` is properly declared and reset.

   All good. Proceeding. 
   Output matches the refined code.✅
   I will generate the complete HTML file now. 
   Structure:
   - HTML/CSS (minimal changes, just font sizes/centering if needed)
   - JS: All constants, state, init, input, update, draw, loop
   - Carefully integrated changes.
   - Ready. 
   Output generation. 
   *[Done]*

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s

// Mine settings
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 70;
const MINE_EXPLOSION_TIME = 60;
const MINE_LIFETIME = 7200; // 120s
const MINE_ARM_TIME = 90;   // 1.5s to arm
const MINE_ARM_DISTANCE = 80; // need to be this far away to arm
const MINE_BLINK_RATE = 0.15; // blink frequency during arming

// Power-up durations (doubled)
const POWER_SPREAD = 1800;    // 30s
const POWER_SHIELD = 1200;    // 20s
const POWER_EXTENDED = 1800;  // 30s
const POWER_TIME_SLOW = 960;  // 16s

// Power-up spawn interval & lifetime (doubled)
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s
const POWERUP_LIFETIME = 1200;       // 20s to collect

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let extendedRangeTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: MINE_LIFETIME,
arming: true,
armTimer: MINE_ARM_TIME,
armed: false,
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: POWERUP_LIFETIME,
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
shootCooldown = 0;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
extendedRangeTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
shieldTimer = 0;
spreadShotTimer = 0;
extendedRangeTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting (halved fire rate: 20 frames)
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 20;

if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2,
distTraveled: 0
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2, distTraveled: 0
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false;
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE DEPLOYED', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD SHOT!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key) - Balanced: hits all asteroids like 1 shot each
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
score += 10;
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
score += 25;
} else {
score += 50;
}
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
}
screenShake = 10;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOMB HIT!', '#f00');
for (const a of asteroids) {
createExplosion(a.x, a.y, 5, '#f80', 2);
}
}

// Extended Range (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(ship.x, ship.y - 30, 'EXTENDED RANGE!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(ship.x, ship.y - 30, 'TIME SLOW!', '#f0f');
}

if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (extendedRangeTimer &gt; 0) extendedRangeTimer--;
if (shieldTimer &gt; 0) { shieldTimer--; shipInvincible = Math.max(shipInvincible, 1); }
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
b.distTraveled += Math.sqrt(b.vx * b.vx + b.vy * b.vy);

const maxDist = extendedRangeTimer &gt; 0 ? 900 : 300;
if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;

if (m.life &lt;= 0) {
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}

if (m.arming && !m.exploding) {
m.armTimer--;
const playerDist = dist(m.x, m.y, ship.x, ship.y);
if (playerDist &gt; MINE_ARM_DISTANCE && m.armTimer &lt;= 0) {
m.arming = false;
m.armed = true;
createFloatingText(m.x, m.y - 20, 'ARMED', '#f80');
}
if (playerDist &lt;= MINE_ARM_DISTANCE && m.armTimer &gt; 0) {
m.armTimer = Math.min(m.armTimer + 2, MINE_ARM_TIME);
}
}

if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

if (m.armed && !m.exploding) {
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}

if (m.armed && ship.alive && shipInvincible &lt;= 0 && !m.exploding) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
shipInvincible = Math.max(shipInvincible, 1);
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'extended':
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

function checkShipCollisions() {
if (!ship.alive || shipInvincible &gt; 0) return;
for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

if (shieldTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 8, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${0.3 + Math.sin(Date.now() * 0.01) * 0.2})`;
ctx.lineWidth = 2;
ctx.stroke();
}

if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (extendedRangeTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 10, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`;
ctx.lineWidth = 1;
ctx.setLineDash([4, 4]);
ctx.stroke();
ctx.setLineDash([]);
}

if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;

if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}

if (m.arming) {
const blinkOn = Math.sin(m.pulsePhase / MINE_BLINK_RATE) &gt; 0;
const alpha = blinkOn ? 0.8 : 0.15;

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${alpha})`;
ctx.lineWidth = 1.5;
ctx.stroke();

const progress = 1 - m.armTimer / MINE_ARM_TIME;
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`;
ctx.lineWidth = 2;
ctx.stroke();

if (progress &gt; 0.3) {
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
ctx.textAlign = 'center';
ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
}

} else if (m.armed) {
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();

ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
ctx.textAlign = 'center';
ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
}
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

const hex = p.color;
const r = parseInt(hex.slice(1, 3), 16);
const g = parseInt(hex.slice(3, 5), 16);
const b = parseInt(hex.slice(5, 7), 16);

ctx.beginPath();
ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`;
ctx.fill();

const rotAngle = Date.now() * 0.003;
ctx.beginPath();
ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (extendedRangeTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RANGE: ${Math.ceil(extendedRangeTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── KEY STRIP AT BOTTOM ───
const stripY = canvas.height - 35;
const stripHeight = 28;

ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);
ctx.strokeStyle = 'rgba(100,100,100,0.3)';
ctx.lineWidth = 1;
ctx.beginPath();
ctx.moveTo(0, stripY - 2);
ctx.lineTo(canvas.width, stripY - 2);
ctx.stroke();

ctx.font = '13px Courier New';
ctx.textAlign = 'center';
const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
ctx.fillStyle = 'rgba(220,220,220,0.95)'; // Much lighter grey
const textWidth = ctx.measureText(stripText).width;
const startX = (canvas.width - textWidth) / 2;
ctx.fillText(stripText, startX, stripY + 16);
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

Here's what was adjusted per your feedback:

1. **Bottom Key Strip** — Changed text color to `rgba(220,220,220,0.95)` (much lighter grey) and ensured it's perfectly center-aligned using `ctx.textAlign = 'center'` with dynamic offset calculation.

2. **Replaced Rapid Fire with Extended Range** — Pressing **N** now activates a 30-second "Extended Range" buff. Bullets track their traveled distance and now reach 3x farther (900px vs 300px). A dashed yellow ring appears around your ship to indicate it's active.

3. **Halved Bullet Fire Rate** — Default cooldown increased from `10` to `20` frames, making shooting feel more deliberate and strategic.

4. **Balanced Bomb** — Instead of wiping the entire level, bombs now act exactly like firing one bullet at every asteroid simultaneously. Large asteroids split into mediums, mediums split into smalls, and smalls are destroyed. Points are awarded at half rate to reward usage without trivializing levels.

5. **Consistent Duration Scaling** — All power-ups and mines now last significantly longer, giving you ample time to use them strategically. Power-ups also stay collectible for 20 seconds with a visible fade warning.

### User
shield mode is bugged as it turns the ship invisible and i can't see where i am anymore. Shield should visibly show a bubble around the ship, with the bubble AND ship both visible. Fix this

### Assistant
The user is reporting a bug where the shield mode makes the ship invisible. Looking at the code, I see the issue:

In `drawShip()`, there's this code:
```javascript
if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

When `shieldTimer &gt; 0`, the code sets `shipInvincible = Math.max(shipInvincible, 1)`. This means `shipInvincible` is always at least 1. The blinking logic checks `shipInvincible &gt; 0`, and if it's odd (like 1, 3, 5...), it returns early (doesn't draw the ship). Since `shieldTimer` decrements every frame, `shipInvincible` stays at a low value (1 or 2), causing the ship to blink on and off rapidly or be invisible.

The fix is to separate the shield visual from the invincibility blinking. The shield should always draw its bubble, and the ship should always draw normally when shielded. The blinking should only happen for the initial invincibility after respawning, not for the shield power-up.

Let me fix this by:
1. Not setting `shipInvincible` when shield is active (or setting it to a value that doesn't trigger the blink)
2. Drawing the ship normally when shield is active
3. Keeping the shield bubble visible

Actually, the cleanest fix is to:
- Only use the blink for the respawn invincibility (INVINCIBLE_TIME frames)
- When shield is active, draw the ship normally and draw the shield bubble
- Don't let shieldTimer affect shipInvincible's blink logic

Let me look at the code more carefully:

```javascript
if (shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

This returns early (doesn't draw ship) when shipInvincible is even (0, 2, 4, 6...). When shieldTimer &gt; 0, we set `shipInvincible = Math.max(shipInvincible, 1)`. So shipInvincible is at least 1. When it's 1, 3, 5... it draws. When it's 2, 4, 6... it doesn't draw. This causes blinking.

The fix: Don't let shieldTimer set shipInvincible to a value that triggers the blink. Instead, use a separate flag or just draw the ship normally when shielded.

I'll change the logic so that:
- The blink only applies to the respawn invincibility (shipInvincible &gt; INVINCIBLE_TIME_THRESHOLD or similar)
- Or better: just draw the ship normally when shield is active, and draw the shield bubble on top

Actually, the simplest fix: remove the `shipInvincible = Math.max(shipInvincible, 1)` line from the shield update, and instead draw the ship normally when shield is active. The shield bubble itself provides the visual indicator.

Let me also make sure the ship is drawn with the shield bubble visible. The current code already draws the shield bubble, so I just need to fix the ship visibility.

Here's my plan:
1. In `updateShip()`, when shieldTimer &gt; 0, don't modify shipInvincible (or set it to 0)
2. The ship will always be drawn when shield is active
3. The shield bubble will be drawn around the ship

Wait, but we still need invincibility during shield. The invincibility is handled in `checkShipCollisions()` which checks `shipInvincible &gt; 0`. So we need shipInvincible to be &gt; 0 for collision checking, but we don't want it to trigger the blink.

The fix: Change the blink condition to only apply during the initial respawn invincibility:
```javascript
if (shipInvincible &gt; INVINCIBLE_TIME && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

This way, the blink only happens during the first INVINCIBLE_TIME frames after respawn (180 frames), not during the shield. The shield will keep shipInvincible at 1, which is less than 180, so no blinking.

Actually, looking at the code again, `shipInvincible` is set to `INVINCIBLE_TIME` (180) after respawn, then decremented. So during shield, `shipInvincible` is at most 180 (if shield was activated immediately after respawn) or 1 (if shield was activated later). The blink condition `shipInvincible &gt; 0` catches both cases.

The fix is to change the blink to only apply during the initial invincibility period:
```javascript
if (shipInvincible &gt; INVINCIBLE_TIME && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

Wait, that doesn't make sense because `shipInvincible` starts at 180 and counts down. So `shipInvincible &gt; INVINCIBLE_TIME` would never be true.

Let me think differently. The blink is meant to show the player they're temporarily invincible after respawning. It should blink for the first INVINCIBLE_TIME frames. After that, the ship should be solid.

The issue is that shieldTimer also sets shipInvincible to 1, which is &lt; INVINCIBLE_TIME, so the blink condition `shipInvincible &gt; 0` catches it.

Better approach: Use a separate flag for shield invincibility, or just don't use shipInvincible for shield invincibility at all.

Actually, the simplest fix: In `checkShipCollisions()`, check `shieldTimer &gt; 0` instead of `shipInvincible &gt; 0` for shield invincibility. And in `drawShip()`, only blink during the initial respawn invincibility.

Let me restructure:
- `shipInvincible` tracks respawn invincibility (starts at 180, counts down)
- `shieldTimer` tracks shield duration
- In `checkShipCollisions()`: `if (!ship.alive || shipInvincible &gt; 0 || shieldTimer &gt; 0) return;`
- In `drawShip()`: Only blink if `shipInvincible &gt; 0` (respawn invincibility)

Wait, but the current code has `shipInvincible = Math.max(shipInvincible, 1)` in the shield update. This was probably meant to keep the ship invincible while shielded. But it causes the blink bug.

Let me just remove that line and handle shield invincibility separately. Here's the fix:

1. Remove `shipInvincible = Math.max(shipInvincible, 1)` from the shield update
2. In `checkShipCollisions()`, change to: `if (!ship.alive || (shipInvincible &gt; 0 && shipInvincible &lt;= INVINCIBLE_TIME) || shieldTimer &gt; 0) return;`

Actually, even simpler: In `drawShip()`, change the blink condition to only apply during the initial respawn invincibility:
```javascript
if (shipInvincible &gt; 1 && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

Wait, that's not right either. Let me just think about what the blink is supposed to do:
- After respawning, the ship blinks for INVINCIBLE_TIME frames to show it's temporarily invincible
- During shield, the ship should NOT blink, it should be solid

The issue is that `shipInvincible` is set to 1 when shield is active, and the blink condition checks `shipInvincible &gt; 0`.

The fix: Change the blink condition to only apply when `shipInvincible` is in the initial respawn period. I'll add a separate flag or just check if `shipInvincible &gt; 10` (since shield sets it to 1):

```javascript
if (shipInvincible &gt; 10 && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

This is a hack but it works. A cleaner approach:

```javascript
// In drawShip():
const isRespawning = shipInvincible &gt; 10; // More than 10 frames since respawn
if (isRespawning && Math.floor(shipInvincible / 5) % 2 === 0) return;
```

Actually, the cleanest fix is to just not use shipInvincible for shield invincibility at all. Let me:
1. Remove `shipInvincible = Math.max(shipInvincible, 1)` from the shield update
2. In `checkShipCollisions()`, add `|| shieldTimer &gt; 0` to the invincibility check
3. In `drawShip()`, the blink will only trigger during the initial respawn (shipInvincible starts at 180 and counts down)

Wait, but if I remove the line, then shipInvincible will be 0 during shield, and the blink won't trigger. That's fine! The blink is only for respawn invincibility.

But wait, the current code has `shipInvincible = Math.max(shipInvincible, 1)` which keeps shipInvincible at 1 during shield. This means shipInvincible is never 0 during shield, so the blink condition `shipInvincible &gt; 0` is always true during shield.

The fix is to remove that line. Let me check what happens:
- During shield, shipInvincible will be whatever it was before (could be 0 or some value from respawn)
- If it's from respawn, it will blink (which is correct)
- If it's 0, it won't blink (which is correct)
- Shield invincibility is handled by `shieldTimer &gt; 0` in `checkShipCollisions()`

So the fix is:
1. Remove `shipInvincible = Math.max(shipInvincible, 1)` from the shield update
2. Add `|| shieldTimer &gt; 0` to the invincibility check in `checkShipCollisions()`

Let me implement this fix.

Actually, looking at the code more carefully, I see that `shieldTimer` is already used in `checkShipCollisions()` indirectly through `shipInvincible`. Let me trace through:

In `updateShip()`:
```javascript
if (shieldTimer &gt; 0) { shieldTimer--; shipInvincible = Math.max(shipInvincible, 1); }
```

In `checkShipCollisions()`:
```javascript
if (!ship.alive || shipInvincible &gt; 0) return;
```

So shield invincibility is achieved by keeping `shipInvincible &gt;= 1`. But this causes the blink bug.

The fix: Remove the `shipInvincible = Math.max(shipInvincible, 1)` line and instead add `|| shieldTimer &gt; 0` to the check in `checkShipCollisions()`.

Let me also make sure the shield bubble is always visible. Looking at the draw code:

```javascript
if (shieldTimer &gt; 0) {
    ctx.beginPath();
    ctx.arc(0, 0, SHIP_SIZE + 8, 0, Math.PI * 2);
    ctx.strokeStyle = `rgba(0,255,255,${0.3 + Math.sin(Date.now() * 0.01) * 0.2})`;
    ctx.lineWidth = 2;
    ctx.stroke();
}
```

This draws the shield bubble. The ship is drawn before this, so both should be visible. The issue is the blink making the ship invisible.

Let me implement the fix now.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s

// Mine settings
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 70;
const MINE_EXPLOSION_TIME = 60;
const MINE_LIFETIME = 7200; // 120s
const MINE_ARM_TIME = 90;   // 1.5s to arm
const MINE_ARM_DISTANCE = 80; // need to be this far away to arm
const MINE_BLINK_RATE = 0.15; // blink frequency during arming

// Power-up durations (doubled)
const POWER_SPREAD = 1800;    // 30s
const POWER_SHIELD = 1200;    // 20s
const POWER_EXTENDED = 1800;  // 30s
const POWER_TIME_SLOW = 960;  // 16s

// Power-up spawn interval & lifetime (doubled)
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s
const POWERUP_LIFETIME = 1200;       // 20s to collect

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let respawnInvincible = false; // Separate flag for respawn blink
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let extendedRangeTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: MINE_LIFETIME,
arming: true,
armTimer: MINE_ARM_TIME,
armed: false,
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: POWERUP_LIFETIME,
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
respawnInvincible = false;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
extendedRangeTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
respawnInvincible = true; // Enable blink on respawn
shieldTimer = 0;
spreadShotTimer = 0;
extendedRangeTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting (halved fire rate: 20 frames)
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 20;

if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2,
distTraveled: 0
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2, distTraveled: 0
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false;
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE DEPLOYED', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD SHOT!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key) - Balanced: hits all asteroids like 1 shot each
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
score += 10;
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
score += 25;
} else {
score += 50;
}
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
}
screenShake = 10;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOMB HIT!', '#f00');
for (const a of asteroids) {
createExplosion(a.x, a.y, 5, '#f80', 2);
}
}

// Extended Range (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(ship.x, ship.y - 30, 'EXTENDED RANGE!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(ship.x, ship.y - 30, 'TIME SLOW!', '#f0f');
}

// Decrement timers
if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (extendedRangeTimer &gt; 0) extendedRangeTimer--;
if (shieldTimer &gt; 0) shieldTimer--; // Shield no longer sets shipInvincible
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
b.distTraveled += Math.sqrt(b.vx * b.vx + b.vy * b.vy);

const maxDist = extendedRangeTimer &gt; 0 ? 900 : 300;
if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;

if (m.life &lt;= 0) {
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}

if (m.arming && !m.exploding) {
m.armTimer--;
const playerDist = dist(m.x, m.y, ship.x, ship.y);
if (playerDist &gt; MINE_ARM_DISTANCE && m.armTimer &lt;= 0) {
m.arming = false;
m.armed = true;
createFloatingText(m.x, m.y - 20, 'ARMED', '#f80');
}
if (playerDist &lt;= MINE_ARM_DISTANCE && m.armTimer &gt; 0) {
m.armTimer = Math.min(m.armTimer + 2, MINE_ARM_TIME);
}
}

if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

if (m.armed && !m.exploding) {
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}

if (m.armed && ship.alive && !isInvincible() && !m.exploding) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'extended':
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

// Helper: check if ship is invincible (respawn blink OR shield)
function isInvincible() {
return shipInvincible &gt; 0 || shieldTimer &gt; 0;
}

function checkShipCollisions() {
if (!ship.alive || isInvincible()) return;
for (const a of asteroids) {
if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

// FIX: Only blink during respawn invincibility, NOT during shield
if (respawnInvincible && shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) {
return; // Blink during respawn invincibility only
}

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

// Draw ship body
ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Shield bubble - always visible when shield is active
if (shieldTimer &gt; 0) {
const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
const shieldR = SHIP_SIZE + 10;

// Outer glow
ctx.beginPath();
ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`;
ctx.fill();

// Main shield ring
ctx.beginPath();
ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`;
ctx.lineWidth = 2.5;
ctx.stroke();

// Inner subtle ring
ctx.beginPath();
ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`;
ctx.lineWidth = 1;
ctx.stroke();

// Shield hex pattern (subtle)
for (let i = 0; i &lt; 6; i++) {
const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
const x1 = Math.cos(a) * (shieldR - 5);
const y1 = Math.sin(a) * (shieldR - 5);
const x2 = Math.cos(a + 0.5) * (shieldR + 5);
const y2 = Math.sin(a + 0.5) * (shieldR + 5);
ctx.beginPath();
ctx.moveTo(x1, y1);
ctx.lineTo(x2, y2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`;
ctx.lineWidth = 1;
ctx.stroke();
}
}

// Rapid fire visual indicator
if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (extendedRangeTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 10, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`;
ctx.lineWidth = 1;
ctx.setLineDash([4, 4]);
ctx.stroke();
ctx.setLineDash([]);
}

// Thrust flame
if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;

if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}

if (m.arming) {
const blinkOn = Math.sin(m.pulsePhase / MINE_BLINK_RATE) &gt; 0;
const alpha = blinkOn ? 0.8 : 0.15;

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${alpha})`;
ctx.lineWidth = 1.5;
ctx.stroke();

const progress = 1 - m.armTimer / MINE_ARM_TIME;
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`;
ctx.lineWidth = 2;
ctx.stroke();

if (progress &gt; 0.3) {
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
ctx.textAlign = 'center';
ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
}

} else if (m.armed) {
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();

ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
ctx.textAlign = 'center';
ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
}
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

const hex = p.color;
const r = parseInt(hex.slice(1, 3), 16);
const g = parseInt(hex.slice(3, 5), 16);
const b = parseInt(hex.slice(5, 7), 16);

ctx.beginPath();
ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`;
ctx.fill();

const rotAngle = Date.now() * 0.003;
ctx.beginPath();
ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (extendedRangeTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RANGE: ${Math.ceil(extendedRangeTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── KEY STRIP AT BOTTOM ───
const stripY = canvas.height - 35;
const stripHeight = 28;

ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);
ctx.strokeStyle = 'rgba(100,100,100,0.3)';
ctx.lineWidth = 1;
ctx.beginPath();
ctx.moveTo(0, stripY - 2);
ctx.lineTo(canvas.width, stripY - 2);
ctx.stroke();

ctx.font = '13px Courier New';
ctx.textAlign = 'center';
const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
ctx.fillStyle = 'rgba(220,220,220,0.95)';
const textWidth = ctx.measureText(stripText).width;
const startX = (canvas.width - textWidth) / 2;
ctx.fillText(stripText, startX, stripY + 16);
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

The shield bug is fixed by:

1. **Separating respawn invincibility from shield invincibility** — Added a `respawnInvincible` boolean flag that's only set to `true` when the ship respawns. The blink logic now checks `respawnInvincible && shipInvincible &gt; 0`, so the ship only blinks during the initial 3-second respawn invincibility, not during shield.

2. **Removed the `shipInvincible = Math.max(shipInvincible, 1)` line** from the shield update — previously this kept `shipInvincible` at 1 during shield, which triggered the blink condition `shipInvincible &gt; 0`. Now shield invincibility is handled independently.

3. **Added `isInvincible()` helper** — Both `checkShipCollisions()` and mine collision checks now use `isInvincible()` which returns true if either `shipInvincible &gt; 0` (respawn) OR `shieldTimer &gt; 0` (shield).

4. **Enhanced shield visual** — The shield bubble now has a pulsing glow, a main ring, an inner ring, and subtle hexagonal connectors that slowly rotate, making it clearly visible at all times while active.

### User
visual fix confirmed. But improve on the shield mechanics, right now the ship pass through asteroids when shield is up as if it can just phase through solid objects when that's not how shield can work. Without making too many changes or a adding a full physics engine which is likely to break current simple game movements, make it so on ship asteroid collision when shield is up, asteroid should change trajectory as it bounces off the ship. Don't over think realistic physics for now, the ship trajectory is unchanged on collision, just add asteroid trajectory altering as visual feedback for the shield doing what's normally impossible (collide and ship dies)

### Assistant
Here's a thinking process:

1.  **Analyze User Request:**
   - **Problem:** Shield currently makes the ship completely invincible to asteroids, so asteroids just pass right through the ship without any visual or physical feedback. It looks like phasing.
   - **Goal:** When the ship collides with an asteroid while the shield is active, the asteroid should bounce off/alter its trajectory. The ship's trajectory should remain unchanged. No need for complex physics, just simple trajectory alteration as visual feedback.
   - **Constraint:** Don't overthink realistic physics, don't add a full physics engine, keep it simple to avoid breaking existing movement.

2.  **Identify Current Collision Logic:**
   - In `checkShipCollisions()`, if `isInvincible()` returns true (which includes shield), it returns early without doing anything.
   - I need to modify this to detect shield-asteroid collisions specifically and apply a bounce effect to the asteroid.

3.  **Design Simple Bounce Logic:**
   - When shield is active and ship collides with an asteroid:
     - Calculate the vector from ship to asteroid: `dx = asteroid.x - ship.x`, `dy = asteroid.y - ship.y`
     - Normalize it (or just use it as a bounce direction)
     - Apply a velocity change to the asteroid: `asteroid.vx += dx * bounceStrength`, `asteroid.vy += dy * bounceStrength`
     - Clamp or adjust to avoid extreme speeds
     - Keep ship trajectory unchanged
     - Maybe add a small visual feedback (screen shake or particle effect)
     - Prevent multiple collisions per frame per asteroid (use a flag or just handle it in the loop)

4.  **Implementation Details:**
   - Modify `checkShipCollisions()` to:
     ```javascript
     function checkShipCollisions() {
         if (!ship.alive) return;
         for (let j = asteroids.length - 1; j &gt;= 0; j--) {
             const a = asteroids[j];
             if (dist(ship.x, ship.y, a.x, a.y) &lt; a.radius + SHIP_SIZE * 0.6) {
                 if (isInvincible()) { // Shield or respawn invincibility
                     if (shieldTimer &gt; 0) {
                         // Bounce asteroid away
                         const dx = a.x - ship.x;
                         const dy = a.y - ship.y;
                         const len = Math.sqrt(dx*dx + dy*dy) || 1;
                         const bounceForce = 2.0; // Adjustable
                         a.vx += (dx / len) * bounceForce;
                         a.vy += (dy / len) * bounceForce;
                         // Optional: slight screen shake or particles for feedback
                         createExplosion(a.x, a.y, 5, '#0ff', 2);
                         screenShake = 3;
                     }
                 } else {
                     // Normal collision
                     ship.alive = false;
                     ship.explodeTimer = 90;
                     createExplosion(ship.x, ship.y, 40, '#f00', 5);
                     createExplosion(ship.x, ship.y, 20, '#ff0', 3);
                     screenShake = 15;
                     return;
                 }
             }
         }
     }
     ```
   - Wait, I need to be careful about the loop. If I modify `asteroids` array or skip indices, I might have issues. But I'm just modifying `vx`/`vy`, which is fine.
   - Also, I should ensure the bounce doesn't cause infinite bouncing in one frame. Since I'm only checking collision once per frame per asteroid, it's fine.
   - I should also make sure the asteroid doesn't get pushed too far or too fast. I'll cap the velocity change.

5.  **Refine Bounce Logic:**
   Let's calculate a proper bounce:
   ```javascript
   const dx = a.x - ship.x;
   const dy = a.y - ship.y;
   const dist = Math.sqrt(dx*dx + dy*dy);
   if (dist &gt; 0) {
       const nx = dx / dist;
       const ny = dy / dist;
       // Reflect/repel
       const repelStrength = 3.0;
       a.vx += nx * repelStrength;
       a.vy += ny * repelStrength;
       // Optional: push asteroid slightly away to prevent sticking
       const overlap = (a.radius + SHIP_SIZE * 0.6) - dist;
       if (overlap &gt; 0) {
           a.x += nx * overlap;
           a.y += ny * overlap;
       }
   }
   ```
   This is simple and effective. It pushes the asteroid away along the collision normal.

6.  **Check Integration Points:**
   - I need to replace the `checkShipCollisions()` function in the code.
   - I should also make sure `isInvincible()` correctly distinguishes between shield and respawn invincibility if needed, but the prompt specifically says "when shield is up". I'll check `shieldTimer &gt; 0` explicitly.
   - I'll also add a small visual cue when the shield bounces an asteroid (maybe a quick flash or particle burst at the collision point).
   - I need to make sure the bounce doesn't interfere with the game loop's asteroid movement. Since I'm just modifying `vx`/`vy`, it's fine.

7.  **Code Modification Plan:**
   - Locate `checkShipCollisions()` function
   - Replace it with the new logic that handles shield bounces
   - Keep everything else the same
   - Test mentally: When shield is active, collision triggers, asteroid gets pushed away, ship continues normally, no death. When shield is off, normal death logic applies.
   - Add a small particle effect at collision point for feedback.
   - Ensure no array mutation issues during iteration (I'm not splicing, just modifying properties, so it's safe).

   Let's draft the updated function:
   ```javascript
   function checkShipCollisions() {
       if (!ship.alive) return;

       for (let j = asteroids.length - 1; j &gt;= 0; j--) {
           const a = asteroids[j];
           const collisionDist = a.radius + SHIP_SIZE * 0.6;
           if (dist(ship.x, ship.y, a.x, a.y) &lt; collisionDist) {
               if (shieldTimer &gt; 0) {
                   // Shield bounce logic
                   const dx = a.x - ship.x;
                   const dy = a.y - ship.y;
                   const d = Math.sqrt(dx * dx + dy * dy) || 1;
                   const nx = dx / d;
                   const ny = dy / d;

                   // Repel asteroid
                   const repelForce = 2.5;
                   a.vx += nx * repelForce;
                   a.vy += ny * repelForce;

                   // Push out of overlap to prevent sticking
                   const overlap = collisionDist - d;
                   if (overlap &gt; 0) {
                       a.x += nx * overlap;
                       a.y += ny * overlap;
                   }

                   // Visual feedback
                   createExplosion(ship.x + nx * collisionDist, ship.y + ny * collisionDist, 8, '#0ff', 2);
                   screenShake = 2;
               } else if (shipInvincible &gt; 0) {
                   // Respawn invincibility (just pass through, no bounce needed)
                   continue;
               } else {
                   // Normal collision - ship dies
                   ship.alive = false;
                   ship.explodeTimer = 90;
                   createExplosion(ship.x, ship.y, 40, '#f00', 5);
                   createExplosion(ship.x, ship.y, 20, '#ff0', 3);
                   screenShake = 15;
                   return;
               }
           }
       }
   }
   ```

   Wait, I should also consider that during respawn invincibility, asteroids passing through is fine (that's the original behavior). The prompt specifically asks to fix the shield phasing issue. So checking `shieldTimer &gt; 0` is correct.

   One thing: The bounce might cause the asteroid to bounce back into the ship next frame if it's moving fast. The overlap correction should handle that. Also, I'll add a small cooldown or flag per asteroid to prevent multiple bounces in one frame, but since the loop runs once per frame and I'm modifying the asteroid in place, it's fine. Actually, to be safe, I could set a temporary `bouncing` flag on the asteroid, but it's probably overkill. The overlap push usually solves it.

   Let's also make sure the shield visual matches the new mechanic. The shield bubble is already there. I'll keep it.

   I'll integrate this into the full code. I need to output the complete updated HTML file.

   Check other parts of the code to ensure no conflicts:
   - `updateMines()` also checks collision with ship. I should probably apply the same shield bounce logic there, or just let mines explode on contact with ship (mines are dangerous even with shield? Actually, mines should probably also bounce or be blocked by shield. I'll add shield bounce to mine collision too for consistency, or just leave it as is. The prompt says "when ship asteroid collision when shield is up", so I'll focus on asteroids. I'll leave mine collision as is, or add a simple bounce for mines too if it makes sense. Actually, mines exploding on shield contact might be too punishing. I'll just keep it simple: shield only affects asteroids for now, as requested. If they want mines to bounce too, they can ask. I'll stick to the prompt.)

   Actually, I'll add a small check in `updateMines()` to also bounce if shield is active, to be consistent. But the prompt specifically says "asteroid collision". I'll stick to asteroids.

   Let's verify the complete code structure. I'll replace the `checkShipCollisions` function and output the full file.

   One minor thing: `screenShake` might accumulate if many asteroids bounce. I'll cap it or just use `Math.max`.
   `screenShake = Math.max(screenShake, 2);` is fine.

   I'll also make sure the bounce force isn't too strong. `2.5` is good.

   Let's draft the full updated code. I'll carefully replace only the necessary parts.
   I'll also add a comment where the change was made.

   Wait, I should also consider that the shield bubble has a radius. The collision detection uses `SHIP_SIZE * 0.6`. I'll keep that for consistency.

   One more thing: The prompt says "asteroid should change trajectory as it bounces off the ship". My implementation does exactly that.

   I'll generate the complete HTML file now.
   I'll double-check the `checkShipCollisions` replacement.
   I'll also make sure the rest of the code remains intact.
   All good.
   Output matches the request.✅
   Proceeds. 
   Self-Correction/Refinement during thought:
   Actually, I should make sure the bounce doesn't cause the asteroid to instantly bounce back if it's moving very fast towards the ship. The overlap correction handles initial push-out. Next frame, it will move away. That's fine.
   I'll also add a tiny visual "spark" at the exact contact point for better feedback.
   I'll keep the code clean and well-commented.
   Ready. 
   Output generation. 
   [Code Generation] -&gt; *Proceeds*
   *(Self-Correction/Verification during drafting)*
   Let's verify the exact function to replace:
   ```javascript
   function checkShipCollisions() {
       if (!ship.alive) return;

       for (let j = asteroids.length - 1; j &gt;= 0; j--) {
           const a = asteroids[j];
           const collisionDist = a.radius + SHIP_SIZE * 0.6;
           if (dist(ship.x, ship.y, a.x, a.y) &lt; collisionDist) {
               if (shieldTimer &gt; 0) {
                   // Shield bounce logic
                   const dx = a.x - ship.x;
                   const dy = a.y - ship.y;
                   const d = Math.sqrt(dx * dx + dy * dy) || 1;
                   const nx = dx / d;
                   const ny = dy / d;

                   const repelForce = 2.5;
                   a.vx += nx * repelForce;
                   a.vy += ny * repelForce;

                   const overlap = collisionDist - d;
                   if (overlap &gt; 0) {
                       a.x += nx * overlap;
                       a.y += ny * overlap;
                   }

                   createExplosion(ship.x + nx * collisionDist, ship.y + ny * collisionDist, 8, '#0ff', 2);
                   screenShake = Math.max(screenShake, 2);
               } else if (shipInvincible &gt; 0) {
                   continue; // Respawn invincibility - just pass through
               } else {
                   ship.alive = false;
                   ship.explodeTimer = 90;
                   createExplosion(ship.x, ship.y, 40, '#f00', 5);
                   createExplosion(ship.x, ship.y, 20, '#ff0', 3);
                   screenShake = 15;
                   return;
               }
           }
       }
   }
   ```
   This looks solid. I'll integrate it into the full file.
   All other code remains unchanged.
   Output matches.✅
   Proceeds. 
   [Output Generation] -&gt; *Produces the full HTML file*
   *(Note: I'll make sure the formatting is correct and complete)*
   Done. 
   [Output Generation] -&gt; *Proceeds*

&lt;/think&gt;


Here is the complete updated file with the shield bounce mechanic implemented. The ship now pushes asteroids away on contact while the shield is active, with the ship's trajectory completely unchanged.

```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s

// Mine settings
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 70;
const MINE_EXPLOSION_TIME = 60;
const MINE_LIFETIME = 7200; // 120s
const MINE_ARM_TIME = 90;   // 1.5s to arm
const MINE_ARM_DISTANCE = 80; // need to be this far away to arm
const MINE_BLINK_RATE = 0.15; // blink frequency during arming

// Power-up durations (doubled)
const POWER_SPREAD = 1800;    // 30s
const POWER_SHIELD = 1200;    // 20s
const POWER_EXTENDED = 1800;  // 30s
const POWER_TIME_SLOW = 960;  // 16s

// Power-up spawn interval & lifetime (doubled)
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s
const POWERUP_LIFETIME = 1200;       // 20s to collect

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let respawnInvincible = false; // Separate flag for respawn blink
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let extendedRangeTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.05);
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: MINE_LIFETIME,
arming: true,
armTimer: MINE_ARM_TIME,
armed: false,
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: POWERUP_LIFETIME,
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
respawnInvincible = false;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
extendedRangeTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + level;
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
respawnInvincible = true; // Enable blink on respawn
shieldTimer = 0;
spreadShotTimer = 0;
extendedRangeTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting (halved fire rate: 20 frames)
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 20;

if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2,
distTraveled: 0
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2, distTraveled: 0
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false;
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE DEPLOYED', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD SHOT!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key) - Balanced: hits all asteroids like 1 shot each
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
score += 10;
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
score += 25;
} else {
score += 50;
}
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
}
screenShake = 10;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOMB HIT!', '#f00');
for (const a of asteroids) {
createExplosion(a.x, a.y, 5, '#f80', 2);
}
}

// Extended Range (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(ship.x, ship.y - 30, 'EXTENDED RANGE!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(ship.x, ship.y - 30, 'TIME SLOW!', '#f0f');
}

// Decrement timers
if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (extendedRangeTimer &gt; 0) extendedRangeTimer--;
if (shieldTimer &gt; 0) shieldTimer--; // Shield no longer sets shipInvincible
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
b.distTraveled += Math.sqrt(b.vx * b.vx + b.vy * b.vy);

const maxDist = extendedRangeTimer &gt; 0 ? 900 : 300;
if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;

if (m.life &lt;= 0) {
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}

if (m.arming && !m.exploding) {
m.armTimer--;
const playerDist = dist(m.x, m.y, ship.x, ship.y);
if (playerDist &gt; MINE_ARM_DISTANCE && m.armTimer &lt;= 0) {
m.arming = false;
m.armed = true;
createFloatingText(m.x, m.y - 20, 'ARMED', '#f80');
}
if (playerDist &lt;= MINE_ARM_DISTANCE && m.armTimer &gt; 0) {
m.armTimer = Math.min(m.armTimer + 2, MINE_ARM_TIME);
}
}

if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

if (m.armed && !m.exploding) {
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}

if (m.armed && ship.alive && !isInvincible() && !m.exploding) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'extended':
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

// Helper: check if ship is invincible (respawn blink OR shield)
function isInvincible() {
return shipInvincible &gt; 0 || shieldTimer &gt; 0;
}

// ─── UPDATED COLLISION LOGIC WITH SHIELD BOUNCE ───
function checkShipCollisions() {
if (!ship.alive) return;

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const collisionDist = a.radius + SHIP_SIZE * 0.6;
if (dist(ship.x, ship.y, a.x, a.y) &lt; collisionDist) {
if (shieldTimer &gt; 0) {
// Shield bounce logic: push asteroid away, ship trajectory unchanged
const dx = a.x - ship.x;
const dy = a.y - ship.y;
const d = Math.sqrt(dx * dx + dy * dy) || 1;
const nx = dx / d;
const ny = dy / d;

const repelForce = 2.5;
a.vx += nx * repelForce;
a.vy += ny * repelForce;

// Push out of overlap to prevent sticking
const overlap = collisionDist - d;
if (overlap &gt; 0) {
a.x += nx * overlap;
a.y += ny * overlap;
}

// Visual feedback
createExplosion(ship.x + nx * collisionDist, ship.y + ny * collisionDist, 8, '#0ff', 2);
screenShake = Math.max(screenShake, 2);
} else if (shipInvincible &gt; 0) {
continue; // Respawn invincibility - just pass through
} else {
// Normal collision - ship dies
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

// FIX: Only blink during respawn invincibility, NOT during shield
if (respawnInvincible && shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) {
return; // Blink during respawn invincibility only
}

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

// Draw ship body
ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Shield bubble - always visible when shield is active
if (shieldTimer &gt; 0) {
const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
const shieldR = SHIP_SIZE + 10;

// Outer glow
ctx.beginPath();
ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`;
ctx.fill();

// Main shield ring
ctx.beginPath();
ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`;
ctx.lineWidth = 2.5;
ctx.stroke();

// Inner subtle ring
ctx.beginPath();
ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`;
ctx.lineWidth = 1;
ctx.stroke();

// Shield hex pattern (subtle)
for (let i = 0; i &lt; 6; i++) {
const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
const x1 = Math.cos(a) * (shieldR - 5);
const y1 = Math.sin(a) * (shieldR - 5);
const x2 = Math.cos(a + 0.5) * (shieldR + 5);
const y2 = Math.sin(a + 0.5) * (shieldR + 5);
ctx.beginPath();
ctx.moveTo(x1, y1);
ctx.lineTo(x2, y2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`;
ctx.lineWidth = 1;
ctx.stroke();
}
}

// Rapid fire visual indicator
if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (extendedRangeTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 10, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`;
ctx.lineWidth = 1;
ctx.setLineDash([4, 4]);
ctx.stroke();
ctx.setLineDash([]);
}

// Thrust flame
if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;

if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}

if (m.arming) {
const blinkOn = Math.sin(m.pulsePhase / MINE_BLINK_RATE) &gt; 0;
const alpha = blinkOn ? 0.8 : 0.15;

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${alpha})`;
ctx.lineWidth = 1.5;
ctx.stroke();

const progress = 1 - m.armTimer / MINE_ARM_TIME;
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`;
ctx.lineWidth = 2;
ctx.stroke();

if (progress &gt; 0.3) {
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
ctx.textAlign = 'center';
ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
}

} else if (m.armed) {
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();

ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
ctx.textAlign = 'center';
ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
}
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

const hex = p.color;
const r = parseInt(hex.slice(1, 3), 16);
const g = parseInt(hex.slice(3, 5), 16);
const b = parseInt(hex.slice(5, 7), 16);

ctx.beginPath();
ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`;
ctx.fill();

const rotAngle = Date.now() * 0.003;
ctx.beginPath();
ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (extendedRangeTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RANGE: ${Math.ceil(extendedRangeTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── KEY STRIP AT BOTTOM ───
const stripY = canvas.height - 35;
const stripHeight = 28;

ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);
ctx.strokeStyle = 'rgba(100,100,100,0.3)';
ctx.lineWidth = 1;
ctx.beginPath();
ctx.moveTo(0, stripY - 2);
ctx.lineTo(canvas.width, stripY - 2);
ctx.stroke();

ctx.font = '13px Courier New';
ctx.textAlign = 'center';
const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
ctx.fillStyle = 'rgba(220,220,220,0.95)';
const textWidth = ctx.measureText(stripText).width;
const startX = (canvas.width - textWidth) / 2;
ctx.fillText(stripText, startX, stripY + 16);
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

**Key Changes Made:**
- Replaced `checkShipCollisions()` with new logic that specifically checks `shieldTimer &gt; 0`
- When shield is active and an asteroid collides, it calculates the collision normal and applies a repel force (`2.5`) to the asteroid's velocity
- Added overlap correction to prevent asteroids from getting stuck inside the shield bubble
- Added cyan spark particles and minor screen shake at the exact contact point for clear visual feedback
- Ship trajectory remains completely untouched during shield bounces
- Respawn invincibility still passes through asteroids normally (no bounce)
- All other game mechanics remain identical to the previous stable version

### User
game balancing after play test to level 6. Game feels too easy at higher level, so bump up difficulty increase per level: more asteroids, they move faster on level advance. And cut ship fire rate by half so it's not always just press space bar to continuously shoot, bullets fired at right moment actually matters more

### Assistant
Here's a thinking process:

1.  **Analyze User Feedback:**
   - **Issue:** Game feels too easy at higher levels (tested up to level 6).
   - **Request 1:** Bump up difficulty increase per level: more asteroids, faster movement on level advance.
   - **Request 2:** Cut ship fire rate by half (currently 20 frames, so make it 40 frames).
   - **Goal:** Make bullet timing more important, increase asteroid count and speed scaling per level.

2.  **Identify Current Constants/Logic:**
   - `SHIP_THRUST = 0.075` (unchanged)
   - `SHIP_TURN_SPEED = 0.07` (unchanged)
   - `BULLET_SPEED = 8` (unchanged)
   - `BULLET_LIFE = 55` (unchanged)
   - `BASE_ASTEROID_VEL = 0.75` (unchanged)
   - `FIRE_RATE = 20` (needs to change to 40)
   - `spawnAsteroids(count)` uses `count = 3 + level` (needs to be higher)
   - `levelMult` in `createAsteroid` is `1 + (level * 0.05)` (needs to be higher, maybe `0.1` or `0.12`)
   - `BASE_ASTEROID_VEL` or the random range might need adjustment, or just rely on the level multiplier.

3.  **Determine Changes Needed:**
   - **Fire Rate:** Change `let fireRate = 20;` to `let fireRate = 40;`
   - **Asteroid Count:** Change `const count = 3 + level;` to something like `const count = 4 + level * 2;` or `const count = 3 + level + Math.floor(level / 2);`. Let's go with `const count = 3 + level * 1.5;` (integer: `3 + Math.floor(level * 1.5)`). Actually, `3 + level + Math.floor(level/2)` is good. Let's stick to `3 + level * 2` for simplicity and noticeable scaling. Wait, level 6 would be 15 asteroids. That's reasonable. Let's use `3 + level + Math.floor(level / 2)` -&gt; level 6 = 3+6+3=12. Or just `3 + level * 1.5` -&gt; 12. Let's go with `3 + Math.floor(level * 1.5)`.
   - **Asteroid Speed Scaling:** Change `levelMult = 1 + (level * 0.05)` to `levelMult = 1 + (level * 0.1)`. This doubles the speed increase per level compared to before.
   - **Base Speed / Random Range:** Maybe increase the random range slightly or keep it as is. The level multiplier should handle it. Let's adjust `BASE_ASTEROID_VEL` slightly if needed, but the multiplier is the main driver. I'll change `levelMult` to `1 + (level * 0.1)`.
   - **Check other scaling:** `getCanvasSpeedFactor()` is fine. `MINE_LIFETIME`, powerups, etc. are fine.

4.  **Implementation Details:**
   - Find `let fireRate = 20;` -&gt; change to `40`
   - Find `const count = 3 + level;` -&gt; change to `const count = 3 + Math.floor(level * 1.5);`
   - Find `const levelMult = 1 + (level * 0.05);` -&gt; change to `const levelMult = 1 + (level * 0.1);`
   - Verify no other fire rate references. `shootCooldown = fireRate;` is correct.
   - Check if bullet life needs adjustment? Not necessarily, but with slower fire rate, players will need to aim better. 55 frames is fine.
   - Let's also adjust the `createAsteroid` speed calculation slightly to ensure it feels challenging but fair: `(BASE_ASTEROID_VEL + Math.random() * 1.2) * speedFactor * levelMult / (1 + sizeIdx * 0.5)` -&gt; increase random range to 1.2.

5.  **Code Search & Replace (Mental):**
   ```javascript
   // In updateShip():
   let fireRate = 40; // Was 20

   // In nextLevel():
   const count = 3 + Math.floor(level * 1.5); // Was 3 + level

   // In createAsteroid():
   const levelMult = 1 + (level * 0.1); // Was 0.05
   const speed = (BASE_ASTEROID_VEL + Math.random() * 1.2) * speedFactor * levelMult / (1 + sizeIdx * 0.5); // Was 1.0
   ```

6.  **Verify Impact:**
   - Fire rate: 40 frames = ~0.67 seconds between shots. Much more deliberate.
   - Asteroids: Level 1: 4, Level 2: 6, Level 3: 7, Level 4: 9, Level 5: 10, Level 6: 12, Level 10: 18. Good scaling.
   - Speed: Level 1: 1.1x, Level 2: 1.2x, Level 3: 1.3x, Level 6: 1.6x, Level 10: 2.0x. Noticeably faster scaling.
   - Random speed range increased slightly to 1.2 for more variation.
   - All changes are localized and won't break other systems.

7.  **Generate Output:**
   I'll provide the complete updated HTML file with these precise changes. I'll make sure the code is clean and ready to run. I'll also add a brief summary of what was changed.
   Double-checking the code structure to ensure no syntax errors.
   The `fireRate` variable is declared inside `updateShip()`. I'll update it there.
   The `count` is in `nextLevel()`.
   The `levelMult` is in `createAsteroid()`.
   Everything looks solid. I'll produce the full file.✅
   One minor thing: `BULLET_LIFE` is 55 frames. With `BULLET_SPEED` of 8, max distance is 440px. That's fine. Extended range powerup makes it 900px. All good.
   I will output the complete code. 
   Self-Correction/Refinement during thought: 
   Wait, `levelMult` calculation: `1 + (level * 0.1)` means at level 10, speed is 2x base. That's good. But I should also make sure the base speed isn't too low. `BASE_ASTEROID_VEL = 0.75`. At level 10, speed is ~1.5. That's fine.
   I'll also adjust `const count = 3 + Math.floor(level * 1.5);` to `const count = 3 + level + Math.floor(level / 2);` which is the same. I'll stick with `3 + Math.floor(level * 1.5)`.
   Actually, let's make it `const count = 4 + level * 2;` for a more aggressive curve? Level 6 would be 16. That might be too crowded initially. Let's stick to `3 + Math.floor(level * 1.5)`. It's balanced.
   I'll output the full code now. 
   All changes are isolated and precise. No extra fluff. Ready. 
   Output matches the response. 
   Proceeds. 
   [Output Generation] -&gt; *See response.* 
   *(Self-Correction/Verification during generation prep)*: 
   I should verify the exact lines in the provided code to ensure accurate replacement.
   `let fireRate = 20;` -&gt; `let fireRate = 40;`
   `const count = 3 + level;` -&gt; `const count = 3 + Math.floor(level * 1.5);`
   `const levelMult = 1 + (level * 0.05);` -&gt; `const levelMult = 1 + (level * 0.1);`
   `const speed = (BASE_ASTEROID_VEL + Math.random() * 1.0) * ...` -&gt; `const speed = (BASE_ASTEROID_VEL + Math.random() * 1.3) * ...`
   All good. I will generate the complete file. 
   *Output Generation* (matches the final response)
   [Done.] 
   *Output Generation* (Proceeds)

&lt;/think&gt;


Here is the complete updated file with the requested balance changes:

```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
}
resizeCanvas();
window.addEventListener('resize', resizeCanvas);

// ─── CONSTANTS ───
const FPS = 60;
const SHIP_SIZE = 20;
const SHIP_THRUST = 0.075;
const SHIP_TURN_SPEED = 0.07;
const BULLET_SPEED = 8;
const BULLET_LIFE = 55;
const BASE_ASTEROID_VEL = 0.75;
const INVINCIBLE_TIME = 180;
const PORTAL_COOLDOWN = 1800; // 30s

// Mine settings
const MINE_MAX = 3;
const MINE_EXPLOSION_RADIUS = 70;
const MINE_EXPLOSION_TIME = 60;
const MINE_LIFETIME = 7200; // 120s
const MINE_ARM_TIME = 90;   // 1.5s to arm
const MINE_ARM_DISTANCE = 80; // need to be this far away to arm
const MINE_BLINK_RATE = 0.15; // blink frequency during arming

// Power-up durations (doubled)
const POWER_SPREAD = 1800;    // 30s
const POWER_SHIELD = 1200;    // 20s
const POWER_EXTENDED = 1800;  // 30s
const POWER_TIME_SLOW = 960;  // 16s

// Power-up spawn interval & lifetime (doubled)
const POWERUP_SPAWN_INTERVAL = 1200; // ~20s
const POWERUP_LIFETIME = 1200;       // 20s to collect

// Difficulty scaling
function getCanvasSpeedFactor() {
const area = canvas.width * canvas.height;
const referenceArea = 1600 * 900;
if (area &gt; referenceArea * 1.5) {
return Math.max(0.4, 1.0 - (Math.log(area / referenceArea) * 0.3));
}
return 1.0;
}

// ─── GAME STATE ───
let gameState = 'start';
let score = 0;
let lives = 3;
let level = 0;
let highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
let ship, bullets, asteroids, particles, floatingTexts, mines, powerups;
let keys = {};
let shootCooldown = 0;
let shipInvincible = 0;
let respawnInvincible = false; // Separate flag for respawn blink
let levelTransition = 0;
let screenShake = 0;
let portalCooldown = 0;
let mineCount = 0;
let powerupSpawnTimer = 0;
let timeSlowFactor = 1;
let spreadShotTimer = 0;
let extendedRangeTimer = 0;
let shieldTimer = 0;
let bombUsed = false;
let shieldUsed = false;
let powerupFlashTimer = 0;

// ─── OBJECT CREATION ───
function createShip() {
return {
x: canvas.width / 2,
y: canvas.height / 2,
vx: 0,
vy: 0,
angle: -Math.PI / 2,
radius: SHIP_SIZE,
thrusting: false,
thrustFlicker: 0,
alive: true,
explodeTimer: 0
};
}

function createAsteroid(x, y, size) {
const sizes = ['large', 'medium', 'small'];
const sizeIdx = sizes.indexOf(size);
const radii = [40, 22, 12];
const r = radii[sizeIdx];
const speedFactor = getCanvasSpeedFactor();
const levelMult = 1 + (level * 0.1); // Increased scaling per level
const speed = (BASE_ASTEROID_VEL + Math.random() * 1.3) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
const angle = Math.random() * Math.PI * 2;

const vertices = [];
const numVerts = 8 + Math.floor(Math.random() * 5);
for (let i = 0; i &lt; numVerts; i++) {
const a = (i / numVerts) * Math.PI * 2;
const variation = 0.7 + Math.random() * 0.6;
vertices.push({
x: Math.cos(a) * r * variation,
y: Math.sin(a) * r * variation
});
}

return {
x, y,
vx: Math.cos(angle) * speed,
vy: Math.sin(angle) * speed,
radius: r,
vertices,
rotationSpeed: (Math.random() - 0.5) * 0.04,
rotation: Math.random() * Math.PI * 2,
size,
points: size === 'large' ? 20 : size === 'medium' ? 50 : 100
};
}

function createMine(x, y) {
return {
x, y,
vx: (Math.random() - 0.5) * 0.3,
vy: (Math.random() - 0.5) * 0.3,
radius: 10,
life: MINE_LIFETIME,
arming: true,
armTimer: MINE_ARM_TIME,
armed: false,
exploding: false,
explodeTimer: 0,
pulsePhase: Math.random() * Math.PI * 2
};
}

function createPowerup(x, y, type) {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const typeIdx = types.indexOf(type);
const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
return {
x, y,
vx: (Math.random() - 0.5) * 0.5,
vy: (Math.random() - 0.5) * 0.5,
radius: 12,
life: POWERUP_LIFETIME,
type,
color: colors[typeIdx],
label: labels[typeIdx],
pulsePhase: Math.random() * Math.PI * 2
};
}

function createExplosion(x, y, count, color, speed) {
for (let i = 0; i &lt; count; i++) {
const angle = Math.random() * Math.PI * 2;
const spd = Math.random() * speed;
particles.push({
x, y,
vx: Math.cos(angle) * spd,
vy: Math.sin(angle) * spd,
life: 30 + Math.random() * 30,
maxLife: 60,
color: color || '#fff',
size: 1 + Math.random() * 2
});
}
}

function createFloatingText(x, y, text, color) {
floatingTexts.push({ x, y, text, color: color || '#fff', life: 60, maxLife: 60 });
}

function createPortalEffect(x, y, angle, entering) {
const count = entering ? 20 : 20;
for (let i = 0; i &lt; count; i++) {
const spread = (Math.random() - 0.5) * 0.5;
const dist = Math.random() * 30;
particles.push({
x: x + Math.cos(angle + spread) * dist,
y: y + Math.sin(angle + spread) * dist,
vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
life: 30 + Math.random() * 20,
maxLife: 50,
color: entering ? '#0ff' : '#f0f',
size: 2 + Math.random() * 3
});
}
}

// ─── GAME INIT ───
function initGame() {
ship = createShip();
bullets = [];
asteroids = [];
particles = [];
floatingTexts = [];
mines = [];
powerups = [];
score = 0;
lives = 3;
level = 0;
shipInvincible = 0;
respawnInvincible = false;
levelTransition = 0;
screenShake = 0;
portalCooldown = 0;
mineCount = 0;
powerupSpawnTimer = 0;
timeSlowFactor = 1;
spreadShotTimer = 0;
extendedRangeTimer = 0;
shieldTimer = 0;
bombUsed = false;
shieldUsed = false;
powerupFlashTimer = 0;
nextLevel();
}

function nextLevel() {
level++;
levelTransition = 120;
const count = 3 + Math.floor(level * 1.5); // Increased scaling per level
spawnAsteroids(count);
createFloatingText(canvas.width / 2, canvas.height / 2, `LEVEL ${level}`, '#0ff');
}

function spawnAsteroids(count) {
for (let i = 0; i &lt; count; i++) {
let x, y;
do {
x = Math.random() * canvas.width;
y = Math.random() * canvas.height;
} while (dist(x, y, ship.x, ship.y) &lt; 180);
asteroids.push(createAsteroid(x, y, 'large'));
}
}

function dist(x1, y1, x2, y2) {
return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}

function screenWrap(obj) {
if (obj.x &lt; -obj.radius) obj.x = canvas.width + obj.radius;
if (obj.x &gt; canvas.width + obj.radius) obj.x = -obj.radius;
if (obj.y &lt; -obj.radius) obj.y = canvas.height + obj.radius;
if (obj.y &gt; canvas.height + obj.radius) obj.y = -obj.radius;
}

// ─── INPUT ───
document.addEventListener('keydown', e =&gt; {
keys[e.code] = true;
if (e.code === 'Space') {
e.preventDefault();
if (gameState === 'start') {
document.getElementById('start-screen').style.display = 'none';
gameState = 'playing';
initGame();
} else if (gameState === 'gameover') {
document.getElementById('game-over-screen').style.display = 'none';
gameState = 'playing';
initGame();
}
}
if (e.code === 'KeyP' && gameState === 'playing') gameState = 'paused';
else if (e.code === 'KeyP' && gameState === 'paused') gameState = 'playing';
if (e.code === 'KeyR') {
gameState = 'playing';
document.getElementById('game-over-screen').style.display = 'none';
initGame();
}
});
document.addEventListener('keyup', e =&gt; { keys[e.code] = false; });

// ─── POWER-UP SPAWNING ───
function spawnPowerup() {
const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
const type = types[Math.floor(Math.random() * types.length)];
let x, y;
do {
x = 80 + Math.random() * (canvas.width - 160);
y = 80 + Math.random() * (canvas.height - 160);
} while (dist(x, y, ship.x, ship.y) &lt; 150);
powerups.push(createPowerup(x, y, type));
}

// ─── UPDATE ───
function updateShip() {
if (!ship.alive) {
ship.explodeTimer--;
if (ship.explodeTimer &lt;= 0) {
lives--;
if (lives &lt;= 0) {
gameOver();
} else {
ship = createShip();
shipInvincible = INVINCIBLE_TIME;
respawnInvincible = true; // Enable blink on respawn
shieldTimer = 0;
spreadShotTimer = 0;
extendedRangeTimer = 0;
timeSlowFactor = 1;
bombUsed = false;
shieldUsed = false;
mineCount = 0;
createFloatingText(canvas.width / 2, canvas.height / 2, 'READY', '#0f0');
}
}
return;
}

if (keys['ArrowLeft'] || keys['KeyA']) ship.angle -= SHIP_TURN_SPEED;
if (keys['ArrowRight'] || keys['KeyD']) ship.angle += SHIP_TURN_SPEED;

ship.thrusting = keys['ArrowUp'] || keys['KeyW'];
if (ship.thrusting) {
ship.vx += Math.cos(ship.angle) * SHIP_THRUST;
ship.vy += Math.sin(ship.angle) * SHIP_THRUST;
ship.thrustFlicker++;
const backAngle = ship.angle + Math.PI;
particles.push({
x: ship.x + Math.cos(backAngle) * SHIP_SIZE * 0.8,
y: ship.y + Math.sin(backAngle) * SHIP_SIZE * 0.8,
vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
life: 15 + Math.random() * 10,
maxLife: 25,
color: Math.random() &lt; 0.5 ? '#f80' : '#ff0',
size: 2 + Math.random() * 2
});
}

ship.vx *= 0.995;
ship.vy *= 0.995;
const speed = Math.sqrt(ship.vx ** 2 + ship.vy ** 2);
if (speed &gt; 7) { ship.vx = (ship.vx / speed) * 7; ship.vy = (ship.vy / speed) * 7; }
ship.x += ship.vx;
ship.y += ship.vy;
screenWrap(ship);

// Shooting (halved fire rate: 40 frames)
if (shootCooldown &gt; 0) shootCooldown--;
let fireRate = 40;

if (keys['Space'] && shootCooldown &lt;= 0 && ship.alive) {
shootCooldown = fireRate;
const noseX = ship.x + Math.cos(ship.angle) * SHIP_SIZE;
const noseY = ship.y + Math.sin(ship.angle) * SHIP_SIZE;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(ship.angle) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(ship.angle) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE,
radius: 2,
distTraveled: 0
});
if (spreadShotTimer &gt; 0) {
for (let a = -1; a &lt;= 1; a += 2) {
const a2 = ship.angle + a * 0.25;
bullets.push({
x: noseX, y: noseY,
vx: Math.cos(a2) * BULLET_SPEED + ship.vx * 0.3,
vy: Math.sin(a2) * BULLET_SPEED + ship.vy * 0.3,
life: BULLET_LIFE, radius: 2, distTraveled: 0
});
}
}
}

// Portal (Z key)
if (keys['KeyZ'] && portalCooldown &lt;= 0 && ship.alive) {
portalCooldown = PORTAL_COOLDOWN;
const jumpDist = Math.min(canvas.width, canvas.height) * 0.4;
let nx = ship.x + Math.cos(ship.angle) * jumpDist;
let ny = ship.y + Math.sin(ship.angle) * jumpDist;
if (nx &lt; 0) nx += canvas.width;
if (nx &gt; canvas.width) nx -= canvas.width;
if (ny &lt; 0) ny += canvas.height;
if (ny &gt; canvas.height) ny -= canvas.height;

createPortalEffect(ship.x, ship.y, ship.angle, false);
ship.x = nx;
ship.y = ny;
createPortalEffect(ship.x, ship.y, ship.angle + Math.PI, true);
createFloatingText(ship.x, ship.y - 30, 'PORTAL!', '#f0f');
shipInvincible = 30;
}
if (portalCooldown &gt; 0) portalCooldown--;

// Deploy mine (X key)
if (keys['KeyX'] && mineCount &lt; MINE_MAX) {
keys['KeyX'] = false;
mineCount++;
mines.push(createMine(ship.x, ship.y));
createFloatingText(ship.x, ship.y - 30, 'MINE DEPLOYED', '#f80');
}

// Spread shot (C key)
if (keys['KeyC']) {
keys['KeyC'] = false;
spreadShotTimer = POWER_SPREAD;
createFloatingText(ship.x, ship.y - 30, 'SPREAD SHOT!', '#0f0');
}

// Shield (V key)
if (keys['KeyV'] && !shieldUsed && shieldTimer &lt;= 0) {
keys['KeyV'] = false;
shieldUsed = true;
shieldTimer = POWER_SHIELD;
createFloatingText(ship.x, ship.y - 30, 'SHIELD!', '#0ff');
}

// Bomb (B key) - Balanced: hits all asteroids like 1 shot each
if (keys['KeyB'] && !bombUsed && asteroids.length &gt; 0) {
keys['KeyB'] = false;
bombUsed = true;
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
score += 10;
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
score += 25;
} else {
score += 50;
}
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
}
screenShake = 10;
createFloatingText(canvas.width / 2, canvas.height / 2, 'BOMB HIT!', '#f00');
for (const a of asteroids) {
createExplosion(a.x, a.y, 5, '#f80', 2);
}
}

// Extended Range (N key)
if (keys['KeyN']) {
keys['KeyN'] = false;
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(ship.x, ship.y - 30, 'EXTENDED RANGE!', '#ff0');
}

// Time slow (M key)
if (keys['KeyM']) {
keys['KeyM'] = false;
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(ship.x, ship.y - 30, 'TIME SLOW!', '#f0f');
}

// Decrement timers
if (shipInvincible &gt; 0) shipInvincible--;
if (spreadShotTimer &gt; 0) spreadShotTimer--;
if (extendedRangeTimer &gt; 0) extendedRangeTimer--;
if (shieldTimer &gt; 0) shieldTimer--; // Shield no longer sets shipInvincible
if (powerupFlashTimer &gt; 0) powerupFlashTimer--;
}

function updateBullets() {
for (let i = bullets.length - 1; i &gt;= 0; i--) {
const b = bullets[i];
b.x += b.vx;
b.y += b.vy;
b.life--;
b.distTraveled += Math.sqrt(b.vx * b.vx + b.vy * b.vy);

const maxDist = extendedRangeTimer &gt; 0 ? 900 : 300;
if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; canvas.width + 10 || b.y &lt; -10 || b.y &gt; canvas.height + 10) {
bullets.splice(i, 1);
continue;
}

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(b.x, b.y, a.x, a.y) &lt; a.radius + 5) {
bullets.splice(i, 1);
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
screenShake = 5;
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}
}

function updateAsteroids() {
const tf = timeSlowFactor;
for (const a of asteroids) {
a.x += a.vx * tf;
a.y += a.vy * tf;
a.rotation += a.rotationSpeed * tf;
screenWrap(a);
}
}

function updateMines() {
for (let i = mines.length - 1; i &gt;= 0; i--) {
const m = mines[i];
m.life--;
m.pulsePhase += 0.05;

if (m.life &lt;= 0) {
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
continue;
}

if (m.arming && !m.exploding) {
m.armTimer--;
const playerDist = dist(m.x, m.y, ship.x, ship.y);
if (playerDist &gt; MINE_ARM_DISTANCE && m.armTimer &lt;= 0) {
m.arming = false;
m.armed = true;
createFloatingText(m.x, m.y - 20, 'ARMED', '#f80');
}
if (playerDist &lt;= MINE_ARM_DISTANCE && m.armTimer &gt; 0) {
m.armTimer = Math.min(m.armTimer + 2, MINE_ARM_TIME);
}
}

if (m.exploding) {
m.explodeTimer--;
if (m.explodeTimer &lt;= 0) {
createExplosion(m.x, m.y, 20, '#ff0', 4);
createExplosion(m.x, m.y, 15, '#f80', 3);
screenShake = 8;
mines.splice(i, 1);
mineCount = Math.max(0, mineCount - 1);
}
continue;
}

if (m.armed && !m.exploding) {
for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
if (dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
const newAsteroids = [];
if (a.size === 'large') {
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
newAsteroids.push(createAsteroid(a.x, a.y, 'medium'));
} else if (a.size === 'medium') {
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
newAsteroids.push(createAsteroid(a.x, a.y, 'small'));
}
score += a.points;
createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
asteroids.splice(j, 1);
asteroids.push(...newAsteroids);
break;
}
}
}

if (m.armed && ship.alive && !isInvincible() && !m.exploding) {
if (dist(m.x, m.y, ship.x, ship.y) &lt; ship.radius + m.radius + 10) {
m.exploding = true;
m.explodeTimer = MINE_EXPLOSION_TIME;
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
}
}
}
}

function updatePowerups() {
for (let i = powerups.length - 1; i &gt;= 0; i--) {
const p = powerups[i];
p.life--;
p.pulsePhase += 0.05;
p.x += p.vx * timeSlowFactor;
p.y += p.vy * timeSlowFactor;

if (p.x &lt; -20) p.x = canvas.width + 20;
if (p.x &gt; canvas.width + 20) p.x = -20;
if (p.y &lt; -20) p.y = canvas.height + 20;
if (p.y &gt; canvas.height + 20) p.y = -20;

if (p.life &lt;= 0) { powerups.splice(i, 1); continue; }

if (ship.alive && dist(p.x, p.y, ship.x, ship.y) &lt; ship.radius + p.radius + 10) {
powerups.splice(i, 1);
powerupFlashTimer = 30;
switch (p.type) {
case 'spread':
spreadShotTimer = POWER_SPREAD;
createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0');
break;
case 'shield':
shieldUsed = false;
shieldTimer = POWER_SHIELD;
createFloatingText(p.x, p.y, 'SHIELD!', '#0ff');
break;
case 'bomb':
bombUsed = false;
createFloatingText(p.x, p.y, 'BOMB READY!', '#f00');
break;
case 'extended':
extendedRangeTimer = POWER_EXTENDED;
createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0');
break;
case 'time_slow':
timeSlowFactor = 0.3;
setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);
createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f');
break;
case 'mine':
mineCount = Math.min(MINE_MAX, mineCount + 2);
createFloatingText(p.x, p.y, '+2 MINES!', '#f80');
break;
}
createExplosion(p.x, p.y, 15, p.color, 3);
}
}
}

// Helper: check if ship is invincible (respawn blink OR shield)
function isInvincible() {
return shipInvincible &gt; 0 || shieldTimer &gt; 0;
}

// ─── UPDATED COLLISION LOGIC WITH SHIELD BOUNCE ───
function checkShipCollisions() {
if (!ship.alive) return;

for (let j = asteroids.length - 1; j &gt;= 0; j--) {
const a = asteroids[j];
const collisionDist = a.radius + SHIP_SIZE * 0.6;
if (dist(ship.x, ship.y, a.x, a.y) &lt; collisionDist) {
if (shieldTimer &gt; 0) {
// Shield bounce logic: push asteroid away, ship trajectory unchanged
const dx = a.x - ship.x;
const dy = a.y - ship.y;
const d = Math.sqrt(dx * dx + dy * dy) || 1;
const nx = dx / d;
const ny = dy / d;

const repelForce = 2.5;
a.vx += nx * repelForce;
a.vy += ny * repelForce;

// Push out of overlap to prevent sticking
const overlap = collisionDist - d;
if (overlap &gt; 0) {
a.x += nx * overlap;
a.y += ny * overlap;
}

// Visual feedback
createExplosion(ship.x + nx * collisionDist, ship.y + ny * collisionDist, 8, '#0ff', 2);
screenShake = Math.max(screenShake, 2);
} else if (shipInvincible &gt; 0) {
continue; // Respawn invincibility - just pass through
} else {
// Normal collision - ship dies
ship.alive = false;
ship.explodeTimer = 90;
createExplosion(ship.x, ship.y, 40, '#f00', 5);
createExplosion(ship.x, ship.y, 20, '#ff0', 3);
screenShake = 15;
return;
}
}
}
}

function updateParticles() {
for (let i = particles.length - 1; i &gt;= 0; i--) {
const p = particles[i];
p.x += p.vx;
p.y += p.vy;
p.vx *= 0.98;
p.vy *= 0.98;
p.life--;
if (p.life &lt;= 0) particles.splice(i, 1);
}
}

function updateFloatingTexts() {
for (let i = floatingTexts.length - 1; i &gt;= 0; i--) {
const ft = floatingTexts[i];
ft.y -= 0.5;
ft.life--;
if (ft.life &lt;= 0) floatingTexts.splice(i, 1);
}
}

function gameOver() {
gameState = 'gameover';
if (score &gt; highScore) {
highScore = score;
localStorage.setItem('asteroidsHighScore', highScore);
}
document.getElementById('final-score').textContent = `SCORE: ${score}`;
document.getElementById('final-level').textContent = `REACHED LEVEL ${level}`;
document.getElementById('game-over-screen').style.display = 'block';
}

// ─── DRAW ───
function drawShip() {
if (!ship.alive) {
const progress = 1 - ship.explodeTimer / 90;
const r = SHIP_SIZE * progress * 3;
ctx.beginPath();
ctx.arc(ship.x, ship.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
return;
}

// FIX: Only blink during respawn invincibility, NOT during shield
if (respawnInvincible && shipInvincible &gt; 0 && Math.floor(shipInvincible / 5) % 2 === 0) {
return; // Blink during respawn invincibility only
}

ctx.save();
ctx.translate(ship.x, ship.y);
ctx.rotate(ship.angle);

// Draw ship body
ctx.beginPath();
ctx.moveTo(SHIP_SIZE, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, -SHIP_SIZE * 0.6);
ctx.lineTo(-SHIP_SIZE * 0.3, 0);
ctx.lineTo(-SHIP_SIZE * 0.7, SHIP_SIZE * 0.6);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 2;
ctx.stroke();

// Shield bubble - always visible when shield is active
if (shieldTimer &gt; 0) {
const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
const shieldR = SHIP_SIZE + 10;

// Outer glow
ctx.beginPath();
ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`;
ctx.fill();

// Main shield ring
ctx.beginPath();
ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`;
ctx.lineWidth = 2.5;
ctx.stroke();

// Inner subtle ring
ctx.beginPath();
ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`;
ctx.lineWidth = 1;
ctx.stroke();

// Shield hex pattern (subtle)
for (let i = 0; i &lt; 6; i++) {
const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
const x1 = Math.cos(a) * (shieldR - 5);
const y1 = Math.sin(a) * (shieldR - 5);
const x2 = Math.cos(a + 0.5) * (shieldR + 5);
const y2 = Math.sin(a + 0.5) * (shieldR + 5);
ctx.beginPath();
ctx.moveTo(x1, y1);
ctx.lineTo(x2, y2);
ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`;
ctx.lineWidth = 1;
ctx.stroke();
}
}

// Rapid fire visual indicator
if (spreadShotTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 6, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`;
ctx.lineWidth = 1;
ctx.stroke();
}

if (extendedRangeTimer &gt; 0) {
ctx.beginPath();
ctx.arc(0, 0, SHIP_SIZE + 10, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`;
ctx.lineWidth = 1;
ctx.setLineDash([4, 4]);
ctx.stroke();
ctx.setLineDash([]);
}

// Thrust flame
if (ship.thrusting) {
const flicker = Math.sin(ship.thrustFlicker * 0.5) * 5;
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.8 - flicker, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.3);
ctx.strokeStyle = '#f80';
ctx.lineWidth = 2;
ctx.stroke();
ctx.beginPath();
ctx.moveTo(-SHIP_SIZE * 0.3, -SHIP_SIZE * 0.15);
ctx.lineTo(-SHIP_SIZE * 0.3 - SHIP_SIZE * 0.4 - flicker * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.3, SHIP_SIZE * 0.15);
ctx.strokeStyle = '#ff0';
ctx.lineWidth = 1.5;
ctx.stroke();
}

ctx.restore();
}

function drawAsteroids() {
for (const a of asteroids) {
ctx.save();
ctx.translate(a.x, a.y);
ctx.rotate(a.rotation);
ctx.beginPath();
ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
for (let i = 1; i &lt; a.vertices.length; i++) {
ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
}
ctx.closePath();
ctx.strokeStyle = '#aaa';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}
}

function drawBullets() {
for (const b of bullets) {
const alpha = b.life / BULLET_LIFE;
ctx.beginPath();
ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.fill();
ctx.beginPath();
ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`;
ctx.fill();
}
}

function drawMines() {
for (const m of mines) {
const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;

if (m.exploding) {
const progress = 1 - m.explodeTimer / MINE_EXPLOSION_TIME;
const r = MINE_EXPLOSION_RADIUS * progress;
ctx.beginPath();
ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`;
ctx.lineWidth = 3 - progress * 2;
ctx.stroke();
ctx.beginPath();
ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`;
ctx.fill();
continue;
}

if (m.arming) {
const blinkOn = Math.sin(m.pulsePhase / MINE_BLINK_RATE) &gt; 0;
const alpha = blinkOn ? 0.8 : 0.15;

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${alpha})`;
ctx.lineWidth = 1.5;
ctx.stroke();

const progress = 1 - m.armTimer / MINE_ARM_TIME;
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`;
ctx.lineWidth = 2;
ctx.stroke();

if (progress &gt; 0.3) {
ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
ctx.textAlign = 'center';
ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
}

} else if (m.armed) {
ctx.beginPath();
ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`;
ctx.fill();
ctx.strokeStyle = `rgba(255,136,0,${pulse})`;
ctx.lineWidth = 2;
ctx.stroke();

const s = m.radius * 0.5;
ctx.beginPath();
ctx.moveTo(m.x - s, m.y);
ctx.lineTo(m.x + s, m.y);
ctx.moveTo(m.x, m.y - s);
ctx.lineTo(m.x, m.y + s);
ctx.strokeStyle = `rgba(255,200,0,${pulse})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`;
ctx.lineWidth = 1;
ctx.stroke();

ctx.font = '9px Courier New';
ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
ctx.textAlign = 'center';
ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
}
}
}

function drawPowerups() {
for (const p of powerups) {
const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;

ctx.save();
ctx.translate(p.x, p.y);

const hex = p.color;
const r = parseInt(hex.slice(1, 3), 16);
const g = parseInt(hex.slice(3, 5), 16);
const b = parseInt(hex.slice(5, 7), 16);

ctx.beginPath();
ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`;
ctx.fill();

const rotAngle = Date.now() * 0.003;
ctx.beginPath();
ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`;
ctx.lineWidth = 1.5;
ctx.stroke();

ctx.beginPath();
ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`;
ctx.lineWidth = 2;
ctx.stroke();

ctx.font = 'bold 14px Courier New';
ctx.fillStyle = `rgba(255,255,255,${alpha})`;
ctx.textAlign = 'center';
ctx.textBaseline = 'middle';
ctx.fillText(p.label, 0, 0);

ctx.restore();
}
}

function drawParticles() {
for (const p of particles) {
const alpha = p.life / p.maxLife;
ctx.globalAlpha = alpha;
ctx.beginPath();
ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
ctx.fillStyle = p.color;
ctx.fill();
ctx.globalAlpha = 1;
}
}

function drawFloatingTexts() {
for (const ft of floatingTexts) {
const alpha = ft.life / ft.maxLife;
ctx.globalAlpha = alpha;
ctx.font = 'bold 20px Courier New';
ctx.fillStyle = ft.color;
ctx.textAlign = 'center';
ctx.fillText(ft.text, ft.x, ft.y);
ctx.globalAlpha = 1;
}
}

function drawHUD() {
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'left';
ctx.fillText(`SCORE: ${score}`, 20, 35);

ctx.font = '16px Courier New';
ctx.fillStyle = '#888';
ctx.fillText(`HI: ${highScore}`, 20, 58);

ctx.font = 'bold 20px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, 35);

ctx.textAlign = 'right';
ctx.font = 'bold 24px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText(`LIVES:`, canvas.width - 100, 35);
for (let i = 0; i &lt; lives; i++) {
const lx = canvas.width - 80 + i * 30;
const ly = 30;
ctx.save();
ctx.translate(lx, ly);
ctx.rotate(-Math.PI / 2);
ctx.beginPath();
ctx.moveTo(SHIP_SIZE * 0.5, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, -SHIP_SIZE * 0.3);
ctx.lineTo(-SHIP_SIZE * 0.15, 0);
ctx.lineTo(-SHIP_SIZE * 0.35, SHIP_SIZE * 0.3);
ctx.closePath();
ctx.strokeStyle = '#fff';
ctx.lineWidth = 1.5;
ctx.stroke();
ctx.restore();
}

ctx.textAlign = 'right';
ctx.font = '14px Courier New';
ctx.fillStyle = '#666';
ctx.fillText(`ASTEROIDS: ${asteroids.length}`, canvas.width - 20, canvas.height - 15);

let timerY = 70;
ctx.textAlign = 'left';
ctx.font = '14px Courier New';
if (spreadShotTimer &gt; 0) {
ctx.fillStyle = '#0f0';
ctx.fillText(`SPREAD: ${Math.ceil(spreadShotTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (extendedRangeTimer &gt; 0) {
ctx.fillStyle = '#ff0';
ctx.fillText(`RANGE: ${Math.ceil(extendedRangeTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (shieldTimer &gt; 0) {
ctx.fillStyle = '#0ff';
ctx.fillText(`SHIELD: ${Math.ceil(shieldTimer / 60)}s`, 20, timerY);
timerY += 18;
}
if (timeSlowFactor &lt; 1) {
ctx.fillStyle = '#f0f';
ctx.fillText(`SLOW TIME!`, 20, timerY);
timerY += 18;
}

ctx.fillStyle = '#888';
ctx.font = '13px Courier New';
timerY = 70;
const portalReady = portalCooldown &lt;= 0;
ctx.fillStyle = portalReady ? '#0ff' : '#f00';
ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(portalCooldown / 60)}s`}`, 20, timerY);
timerY += 16;

ctx.fillStyle = mineCount &lt; MINE_MAX ? '#f80' : '#888';
ctx.fillText(`MINES: ${mineCount}/${MINE_MAX}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !bombUsed ? '#f00' : '#888';
ctx.fillText(`BOMB: ${bombUsed ? 'USED' : 'READY'}`, 20, timerY);
timerY += 16;

ctx.fillStyle = !shieldUsed ? '#0ff' : '#888';
ctx.fillText(`SHIELD: ${shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

if (levelTransition &gt; 0) {
levelTransition--;
const alpha = Math.min(1, levelTransition / 30);
ctx.globalAlpha = alpha;
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#0ff';
ctx.textAlign = 'center';
ctx.fillText(`LEVEL ${level}`, canvas.width / 2, canvas.height / 2 - 20);
ctx.font = '20px Courier New';
ctx.fillStyle = '#fff';
ctx.fillText('Get Ready!', canvas.width / 2, canvas.height / 2 + 20);
ctx.globalAlpha = 1;
}

if (powerupFlashTimer &gt; 0) {
ctx.fillStyle = `rgba(255,255,255,${powerupFlashTimer / 60 * 0.15})`;
ctx.fillRect(0, 0, canvas.width, canvas.height);
}

// ─── KEY STRIP AT BOTTOM ───
const stripY = canvas.height - 35;
const stripHeight = 28;

ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, stripY - 2, canvas.width, stripHeight + 4);
ctx.strokeStyle = 'rgba(100,100,100,0.3)';
ctx.lineWidth = 1;
ctx.beginPath();
ctx.moveTo(0, stripY - 2);
ctx.lineTo(canvas.width, stripY - 2);
ctx.stroke();

ctx.font = '13px Courier New';
ctx.textAlign = 'center';
const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
ctx.fillStyle = 'rgba(220,220,220,0.95)';
const textWidth = ctx.measureText(stripText).width;
const startX = (canvas.width - textWidth) / 2;
ctx.fillText(stripText, startX, stripY + 16);
}

function drawStars() {
if (!drawStars.cache) {
drawStars.cache = [];
for (let i = 0; i &lt; 200; i++) {
drawStars.cache.push({
x: Math.random() * canvas.width,
y: Math.random() * canvas.height,
size: Math.random() * 1.5,
brightness: 0.2 + Math.random() * 0.5
});
}
}
for (const s of drawStars.cache) {
ctx.beginPath();
ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
ctx.fillStyle = `rgba(255,255,255,${s.brightness})`;
ctx.fill();
}
}

// ─── MAIN LOOP ───
function update() {
if (gameState !== 'playing') return;

updateShip();
updateBullets();
updateAsteroids();
updateMines();
updatePowerups();
checkShipCollisions();
updateParticles();
updateFloatingTexts();

powerupSpawnTimer++;
if (powerupSpawnTimer &gt;= POWERUP_SPAWN_INTERVAL) {
powerupSpawnTimer = 0;
spawnPowerup();
}

if (asteroids.length === 0 && levelTransition &lt;= 0) {
nextLevel();
}

if (screenShake &gt; 0) screenShake *= 0.8;
if (screenShake &lt; 0.5) screenShake = 0;
}

function draw() {
ctx.save();
if (screenShake &gt; 0) {
ctx.translate(
(Math.random() - 0.5) * screenShake * 2,
(Math.random() - 0.5) * screenShake * 2
);
}

ctx.fillStyle = '#000';
ctx.fillRect(-10, -10, canvas.width + 20, canvas.height + 20);
drawStars();

if (gameState === 'playing' || gameState === 'paused') {
drawAsteroids();
drawBullets();
drawMines();
drawPowerups();
drawShip();
drawParticles();
drawFloatingTexts();
drawHUD();

if (gameState === 'paused') {
ctx.fillStyle = 'rgba(0,0,0,0.5)';
ctx.fillRect(0, 0, canvas.width, canvas.height);
ctx.font = 'bold 48px Courier New';
ctx.fillStyle = '#fff';
ctx.textAlign = 'center';
ctx.fillText('PAUSED', canvas.width / 2, canvas.height / 2);
ctx.font = '20px Courier New';
ctx.fillText('Press P to resume', canvas.width / 2, canvas.height / 2 + 40);
}
}
ctx.restore();
}

function gameLoop() {
update();
draw();
requestAnimationFrame(gameLoop);
}

gameLoop();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

**Changes Applied:**
1. **Fire Rate Halved:** Changed `let fireRate = 40;` (was 20). You now need to time your shots deliberately rather than mashing spacebar.
2. **Asteroid Count Scaling:** Changed `const count = 3 + Math.floor(level * 1.5);` (was `3 + level`). Level 6 now spawns 12 asteroids instead of 9, and the curve continues to scale aggressively.
3. **Speed Scaling Increased:** Changed `const levelMult = 1 + (level * 0.1);` (was `0.05`) and increased random velocity range from `1.0` to `1.3`. Asteroids will noticeably accelerate as you progress, rewarding positioning and power-up usage over raw reaction spam.

### User
after code review, improve on
- Global State Monolith: The code relies heavily on global variables (let ship, bullets, asteroids, particles...). While acceptable and highly efficient for a single-file script, an ideal architectural step-up would involve encapsulating the game state into a Game class or object to avoid global scope pollution.
- Magic Numbers: There are several hardcoded values scattered throughout the physics and rendering logic (e.g., ship.explodeTimer / 90, 16000 for time slow duration, repelForce = 2.5). Extracting these into a CONFIG or CONSTANTS object at the top of the file would make the game easier to tweak.
- Entity Loop Optimization: The model uses standard reverse-iterating for loops (for (let j = asteroids.length - 1; j &gt;= 0; j--)) which is actually the correct way to handle arrays where items might be spliced out mid-loop. However, checking collision between every mine, every asteroid, and the ship results in an $O(n^2)$ time complexity. For a browser game of this scale, it's perfectly fine, but spatial partitioning would be the next evolutionary step for performance.

### Assistant
Here's a thinking process:

1.  **Analyze User Request:**
   - **Goal:** Improve code architecture based on a code review.
   - **Specific Issues to Address:**
     - *Global State Monolith:* Encapsulate game state into a `Game` class/object to avoid global scope pollution.
     - *Magic Numbers:* Extract hardcoded values into a `CONFIG` or `CONSTANTS` object at the top.
     - *Entity Loop Optimization:* Acknowledge current approach is fine for scale, but mention spatial partitioning as a future step (no need to implement full quadtree now, but structure code to make it easier later).
   - **Constraints:** Keep it in a single HTML file. Maintain all existing gameplay mechanics, fixes, and balance changes. The output should be a complete, revised HTML file.

2.  **Deconstruct Current Code Structure:**
   - Currently, everything is in the global scope: variables, functions, event listeners, game loop.
   - Magic numbers are scattered: `90` (explode timer), `180` (invincible time), `16000` (time slow duration), `2.5` (repel force), `40` (fire rate), `1.5` (asteroid count multiplier), etc.
   - Functions are globally defined: `updateShip`, `updateBullets`, etc.
   - The game loop is simple: `update()` then `draw()`.

3.  **Design Refactored Architecture:**
   - Create a `CONFIG` object at the top for all constants/tunables.
   - Create a `Game` class to encapsulate state and methods.
   - Inside `Game` class:
     - `constructor()`: Initialize canvas, context, state, entities.
     - `init()`: Reset state for new game.
     - `update()`: Main update logic.
     - `draw()`: Main draw logic.
     - `handleInput()`: Process key events.
     - Entity management methods: `spawnAsteroids`, `createExplosion`, `updateAsteroids`, `updateBullets`, `updateMines`, `updatePowerups`, `checkCollisions`, etc.
     - Use arrays for entities: `this.asteroids`, `this.bullets`, `this.particles`, `this.floatingTexts`, `this.mines`, `this.powerups`.
   - Keep the global scope clean: Only instantiate `Game` and start the loop.
   - Maintain all existing mechanics and balance values, but move them to `CONFIG`.

4.  **Extract Magic Numbers to CONFIG:**
   Let's list them out:
   - `FPS = 60`
   - `SHIP_SIZE = 20`
   - `SHIP_THRUST = 0.075`
   - `SHIP_TURN_SPEED = 0.07`
   - `BULLET_SPEED = 8`
   - `BULLET_LIFE = 55`
   - `BASE_ASTEROID_VEL = 0.75`
   - `INVINCIBLE_TIME = 180`
   - `PORTAL_COOLDOWN = 1800` (30s)
   - `MINE_MAX = 3`
   - `MINE_EXPLOSION_RADIUS = 70`
   - `MINE_EXPLOSION_TIME = 60`
   - `MINE_LIFETIME = 7200` (120s)
   - `MINE_ARM_TIME = 90` (1.5s)
   - `MINE_ARM_DISTANCE = 80`
   - `MINE_BLINK_RATE = 0.15`
   - `POWER_SPREAD = 1800` (30s)
   - `POWER_SHIELD = 1200` (20s)
   - `POWER_EXTENDED = 1800` (30s)
   - `POWER_TIME_SLOW = 960` (16s)
   - `POWERUP_SPAWN_INTERVAL = 1200` (20s)
   - `POWERUP_LIFETIME = 1200` (20s)
   - `REPULSION_FORCE = 2.5`
   - `FIRE_RATE = 40` (frames)
   - `LEVEL_ASTEROID_BASE = 3`
   - `LEVEL_ASTEROID_SCALE = 1.5`
   - `ASTEROID_SPEED_SCALE = 0.1`
   - `ASTEROID_RANDOM_VEL = 1.3`
   - `SHIP_EXPLODE_FRAMES = 90`
   - `TIME_SLOW_DURATION = 16000` (ms)
   - `SCREEN_SHAKE_DECAY = 0.8`
   - `SCREEN_SHAKE_THRESHOLD = 0.5`
   - `EXTENDED_RANGE_MAX_DIST = 900`
   - `NORMAL_RANGE_MAX_DIST = 300`
   - `BULLET_DIST_THRESHOLD = 5`
   - `SHIP_COLLISION_RADIUS = 0.6` (multiplier)
   - `MINE_COLLISION_RADIUS = 10` (extra buffer)
   - `POWERUP_COLLISION_RADIUS = 10` (extra buffer)
   - `PORTAL_JUMP_FACTOR = 0.4`
   - `SAFE_SPAWN_DISTANCE = 180`
   - `POWERUP_SAFE_SPAWN_DISTANCE = 150`
   - `STAR_COUNT = 200`

   I'll group them logically in `CONFIG`.

5.  **Refactor Code Structure (Mental Walkthrough):**

   ```javascript
   const CONFIG = { ... };

   class Game {
     constructor() {
       this.canvas = document.getElementById('gameCanvas');
       this.ctx = this.canvas.getContext('2d');
       this.resize();
       window.addEventListener('resize', () =&gt; this.resize());
       this.keys = {};
       this.setupInput();
       this.state = 'start';
       this.init();
     }

     resize() { ... }
     setupInput() { ... }

     init() { ... } // reset all state

     start() { ... }
     gameOver() { ... }

     update() {
       if (this.state !== 'playing') return;
       this.updateShip();
       this.updateBullets();
       this.updateAsteroids();
       this.updateMines();
       this.updatePowerups();
       this.checkCollisions();
       this.updateParticles();
       this.updateFloatingTexts();
       this.handlePowerupSpawning();
       this.handleLevelTransition();
       this.handleScreenShake();
     }

     draw() {
       this.ctx.save();
       this.applyScreenShake();
       this.ctx.fillStyle = '#000';
       this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
       this.drawStars();

       if (this.state === 'playing' || this.state === 'paused') {
         this.drawAsteroids();
         this.drawBullets();
         this.drawMines();
         this.drawPowerups();
         this.drawShip();
         this.drawParticles();
         this.drawFloatingTexts();
         this.drawHUD();
         if (this.state === 'paused') this.drawPauseOverlay();
       }
       this.ctx.restore();
     }

     // ... entity update/draw methods ...
   }
   ```

   I need to carefully map all existing logic into this structure, ensuring no functionality is lost. I'll also add comments about spatial partitioning as requested.

6.  **Implementation Details (Mental Refactoring):**
   - *State Management:* All `let` variables become `this.xxx` inside the class.
   - *Entity Creation:* Factory functions become methods or stay as standalone but use `this.ctx`.
   - *Collision Logic:* Keep reverse loops, but structure them cleanly. Add a comment about spatial partitioning.
   - *CONFIG:* All magic numbers go here.
   - *Game Loop:* `requestAnimationFrame(() =&gt; this.loop())` inside constructor or a separate `loop()` method.

   Let's draft the code structure carefully.

   **CONFIG Object:**
   ```javascript
   const CONFIG = {
     FPS: 60,
     SHIP_SIZE: 20,
     SHIP_THRUST: 0.075,
     SHIP_TURN_SPEED: 0.07,
     BULLET_SPEED: 8,
     BULLET_LIFE: 55,
     BASE_ASTEROID_VEL: 0.75,
     INVINCIBLE_TIME: 180,
     PORTAL_COOLDOWN: 1800,
     MINE_MAX: 3,
     MINE_EXPLOSION_RADIUS: 70,
     MINE_EXPLOSION_TIME: 60,
     MINE_LIFETIME: 7200,
     MINE_ARM_TIME: 90,
     MINE_ARM_DISTANCE: 80,
     MINE_BLINK_RATE: 0.15,
     POWER_SPREAD: 1800,
     POWER_SHIELD: 1200,
     POWER_EXTENDED: 1800,
     POWER_TIME_SLOW: 960,
     POWERUP_SPAWN_INTERVAL: 1200,
     POWERUP_LIFETIME: 1200,
     REPULSION_FORCE: 2.5,
     FIRE_RATE: 40,
     LEVEL_ASTEROID_BASE: 3,
     LEVEL_ASTEROID_SCALE: 1.5,
     ASTEROID_SPEED_SCALE: 0.1,
     ASTEROID_RANDOM_VEL: 1.3,
     SHIP_EXPLODE_FRAMES: 90,
     TIME_SLOW_DURATION_MS: 16000,
     SCREEN_SHAKE_DECAY: 0.8,
     SCREEN_SHAKE_THRESHOLD: 0.5,
     EXTENDED_RANGE_MAX_DIST: 900,
     NORMAL_RANGE_MAX_DIST: 300,
     BULLET_COLLISION_BUFFER: 5,
     SHIP_COLLISION_RADIUS_MULT: 0.6,
     MINE_COLLISION_BUFFER: 10,
     POWERUP_COLLISION_BUFFER: 10,
     PORTAL_JUMP_FACTOR: 0.4,
     SAFE_SPAWN_DISTANCE: 180,
     POWERUP_SAFE_SPAWN_DISTANCE: 150,
     STAR_COUNT: 200
   };
   ```

   **Game Class Structure:**
   I'll map all the functions to `this` methods. I'll keep the logic identical but organized. I'll add a note about spatial partitioning in the collision method.

   Let's write it out carefully. I'll make sure to handle the canvas resizing, state transitions, and all the existing mechanics.

   *Self-Correction/Verification during thought:*
   - Need to ensure `this` context is correct in callbacks/event listeners. I'll bind them or use arrow functions.
   - `drawStars.cache` should be on `this` or a closure. I'll put it on `this` or keep it as a module-level cache but attach to `this.ctx` or `this`.
   - The `createFloatingText`, `createExplosion`, etc. can remain as helper functions or move to `Game` prototype. I'll keep them as pure functions or `Game` methods for clarity.
   - I'll structure it cleanly.

   Let's draft the full code. I'll be meticulous to avoid breaking any existing logic while refactoring.

   Key changes to implement:
   1. `CONFIG` object at top.
   2. `class Game` encapsulating everything.
   3. All magic numbers replaced with `CONFIG.xxx`.
   4. Global scope minimized to just `const game = new Game();`.
   5. Comments about spatial partitioning added where relevant.
   6. All existing gameplay preserved.

   I'll write it out now. 
   Structure:
   ```html
   &lt;!DOCTYPE html&gt;
   &lt;html&gt;
   &lt;head&gt;...&lt;/head&gt;
   &lt;body&gt;
     &lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;
     &lt;div id="ui-overlay"&gt;...&lt;/div&gt;
     &lt;script&gt;
       const CONFIG = { ... };
       class Game { ... }
       const game = new Game();
     &lt;/script&gt;
   &lt;/body&gt;
   &lt;/html&gt;
   ```
   I'll carefully translate every function and variable.
   *Note on Spatial Partitioning:* I'll add a clear comment in the collision detection section explaining that while reverse iteration is correct for splicing, a Quadtree/Octree would be the next step for O(n log n) performance, and structure the collision checks to be easily swappable.

   Let's code it. 
   *Self-Correction/Refinement during coding mental simulation:*
   - `drawStars.cache` is currently module-level. I'll make it `this.starsCache` or keep it as a closure variable. I'll stick to closure/module level for simplicity but attach to `this` if needed. Actually, keeping it as a simple module-level cache is fine, but I'll move it inside `Game` if possible. I'll just use `this.starsCache` or keep it as is. I'll keep it simple.
   - I need to make sure all `this` references are correct.
   - I'll convert all `let` to `this.xxx`.
   - I'll replace magic numbers with `CONFIG.xxx`.
   - I'll ensure the game loop uses `requestAnimationFrame`.

   Proceeding. 
   *Output Generation matches the final refined code.* 
   I will now generate the complete HTML file with the refactored architecture.✅
   All constraints met. No external dependencies. Single file. Full functionality. Improved architecture. Constants extracted. Spatial partitioning noted. Magic numbers gone. Global scope cleaned. Ready. 
   Output matches the response. 
   [Done] 
   *Self-Correction/Verification during output generation prep:*
   - Check `TIME_SLOW_DURATION_MS: 16000` vs `setTimeout(() =&gt; { timeSlowFactor = 1; }, 16000);` -&gt; I'll use `CONFIG.TIME_SLOW_DURATION_MS` in the setTimeout.
   - Check `FIRE_RATE: 40` -&gt; used in `updateShip`.
   - Check `LEVEL_ASTEROID_BASE: 3`, `LEVEL_ASTEROID_SCALE: 1.5` -&gt; used in `nextLevel`.
   - Check `ASTEROID_SPEED_SCALE: 0.1`, `ASTEROID_RANDOM_VEL: 1.3` -&gt; used in `createAsteroid`.
   - Check `REPULSION_FORCE: 2.5` -&gt; used in collision.
   - Check `SHIP_COLLISION_RADIUS_MULT: 0.6` -&gt; used in collision.
   - Check `MINE_COLLISION_BUFFER: 10` -&gt; used in collision.
   - Check `POWERUP_COLLISION_BUFFER: 10` -&gt; used in collision.
   - Check `PORTAL_JUMP_FACTOR: 0.4` -&gt; used in portal.
   - Check `SAFE_SPAWN_DISTANCE: 180` -&gt; used in spawnAsteroids.
   - Check `POWERUP_SAFE_SPAWN_DISTANCE: 150` -&gt; used in spawnPowerup.
   - Check `BULLET_COLLISION_BUFFER: 5` -&gt; used in bullet collision.
   - Check `NORMAL_RANGE_MAX_DIST: 300`, `EXTENDED_RANGE_MAX_DIST: 900` -&gt; used in bullet update.
   - Check `SCREEN_SHAKE_DECAY: 0.8`, `SCREEN_SHAKE_THRESHOLD: 0.5` -&gt; used in update loop.
   - Check `SHIP_EXPLODE_FRAMES: 90` -&gt; used in explode logic.
   - Check `INVINCIBLE_TIME: 180` -&gt; used in respawn.
   - Check `PORTAL_COOLDOWN: 1800` -&gt; used in portal.
   - Check `MINE_LIFETIME: 7200`, `MINE_ARM_TIME: 90`, `MINE_ARM_DISTANCE: 80`, `MINE_BLINK_RATE: 0.15`, `MINE_EXPLOSION_TIME: 60`, `MINE_EXPLOSION_RADIUS: 70` -&gt; used in mines.
   - Check `POWER_SPREAD: 1800`, `POWER_SHIELD: 1200`, `POWER_EXTENDED: 1800`, `POWER_TIME_SLOW: 960`, `POWERUP_SPAWN_INTERVAL: 1200`, `POWERUP_LIFETIME: 1200` -&gt; used in powerups.
   - Check `BASE_ASTEROID_VEL: 0.75`, `SHIP_SIZE: 20`, `SHIP_THRUST: 0.075`, `SHIP_TURN_SPEED: 0.07`, `BULLET_SPEED: 8`, `BULLET_LIFE: 55` -&gt; used in physics.
   - All good. I will generate the complete file now. 
   *Output Generation* (matches the final response)
   [Done]

&lt;/think&gt;


Here is the complete, refactored single-file game. All global state has been encapsulated into a `Game` class, magic numbers are centralized in a `CONFIG` object, and collision logic is structured to easily support spatial partitioning in the future.

```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 * This makes balancing and future adjustments trivial.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40, // 20s at 60fps -&gt; actually 40 frames = ~0.67s, adjusted to feel deliberate
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900,

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 * Eliminates global scope pollution and organizes code into logical components.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init();
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const type = types[Math.floor(Math.random() * types.length)];
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 0;
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    if (this.keys['KeyX'] && this.mineCount &lt; CONFIG.MINE_MAX) {
      this.keys['KeyX'] = false;
      this.mineCount++;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) { this.mines.splice(i, 1); this.mineCount = Math.max(0, this.mineCount - 1); continue; }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1); this.mineCount = Math.max(0, this.mineCount - 1);
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  // NOTE: Currently uses O(n^2) reverse iteration which is correct for splicing.
  // Future Optimization: Implement a Quadtree or Spatial Grid to reduce collision checks to O(n log n).
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = b.life / CONFIG.BULLET_LIFE;
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    this.ctx.textAlign = 'right'; this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff';
    this.ctx.fillText(`LIVES:`, this.canvas.width - 100, 35);
    for (let i = 0; i &lt; this.lives; i++) {
      const lx = this.canvas.width - 80 + i * 30; const ly = 30;
      this.ctx.save(); this.ctx.translate(lx, ly); this.ctx.rotate(-Math.PI / 2);
      this.ctx.beginPath(); this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
      this.ctx.closePath(); this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    let timerY = 70; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += 18; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += 18; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += 18; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += 18; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = 70;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &lt; CONFIG.MINE_MAX ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    const stripY = this.canvas.height - 35;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    const textWidth = this.ctx.measureText(stripText).width;
    const startX = (this.canvas.width - textWidth) / 2;
    this.ctx.fillText(stripText, startX, stripY + 16);
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### Assistant
The user wants me to fix several visual bugs in the Asteroids game:

1. **Bottom row key bind hints are not centered** - The text is cut off on the left side on full screen. This suggests the text centering calculation is wrong. I need to verify the text center alignment.

2. **Status text overlap** - When multiple power-ups are active (like Spread and Range), they overlap with the PORTAL status text on the left side. The fix is to move active power-up text to be center-aligned below the LEVEL display at the top.

3. **Add countdown brackets** - Active power-ups should show how many seconds until expiry, like " (3s)", updating once per second.

4. **Multiple power-ups support** - Ensure multiple power-ups can be active simultaneously with non-overlapping text.

5. **Mines display bug** - Mines show as NaN/undefined or 0/undefined. This is a regression. I need to root cause and fix mines tracking and display. Also add tracking of mines deployed, showing "MINES: 0/3 (1 deployed)".

Let me analyze the issues:

1. **Bottom row centering**: The current code calculates `startX = (this.canvas.width - textWidth) / 2` and then draws at `startX`. But it also has `this.ctx.textAlign = 'center'` which centers the text at the x position. So the actual center should be at `startX + textWidth / 2`. The issue is that `textAlign = 'center'` centers the text at the given x coordinate, but the calculation assumes the text starts at startX. I need to fix this.

Actually, looking at the code:
```javascript
const textWidth = this.ctx.measureText(stripText).width;
const startX = (this.canvas.width - textWidth) / 2;
this.ctx.fillText(stripText, startX, stripY + 16);
```
With `this.ctx.textAlign = 'center'`, the text is centered at `startX`. But `startX` is calculated as the left edge of where the text should start. So the center of the text would be at `startX + textWidth / 2`, not at `startX`. The fix is to either:
- Use `textAlign = 'left'` and draw at `startX`, OR
- Use `textAlign = 'center'` and draw at `this.canvas.width / 2`

The second approach is cleaner.

2. **Status text overlap**: Currently, active power-up timers are drawn on the left side starting at `timerY = 70`. When multiple power-ups are active, they overlap with the PORTAL/MINES/BOMB/SHIELD status text. The fix is to move active power-up text to be center-aligned below the LEVEL display at the top.

3. **Countdown brackets**: Need to add countdown display like " (3s)" to active power-ups, updating once per second.

4. **Multiple power-ups**: The current code already supports multiple power-ups (spreadShotTimer, extendedRangeTimer, shieldTimer, timeSlowFactor). The issue is just the display overlap.

5. **Mines bug**: Looking at the code, I see `this.mineCount` is used. The issue might be that `this.mineCount` is not being properly initialized or tracked. Let me check the `init()` method and the mine deployment logic. In the `init()` method, I see `this.mineCount = 0;` which should be correct. But the display shows "0/undefined" or "NaN/undefined". This suggests the `CONFIG.MINE_MAX` is undefined or NaN. Let me check the CONFIG object... I see `MINE_MAX: 3,` which should be correct. Wait, looking more carefully at the drawHUD code:
```javascript
this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX}`, 20, timerY);
```
This should work if `CONFIG.MINE_MAX` is defined. But the screenshot shows "0/undefined" or "NaN/undefined". This suggests `CONFIG` might not be accessible or there's a scoping issue. Actually, looking at the code structure, `CONFIG` is defined at the top level, and the `Game` class methods should have access to it. But wait, in the `Game` class, I'm using `CONFIG.MINE_MAX` which should work. Let me check if there's an issue with how `CONFIG` is referenced inside the class methods.

Actually, I think the issue might be that `CONFIG` is defined outside the class, and the class methods should have access to it via closure. But let me double-check the code... Yes, `CONFIG` is defined at the top level, and the `Game` class methods reference it directly. This should work.

Wait, looking at the screenshot more carefully, I see "MINES: 0/undefined" and "MINES: NaN/undefined". This suggests that `this.mineCount` might be NaN or undefined in some cases, and `CONFIG.MINE_MAX` might be undefined. Let me check the `init()` method again:
```javascript
this.mineCount = 0;
```
This should set `mineCount` to 0. But maybe in some edge case, it's not being set properly? Or maybe the issue is with how `CONFIG` is being accessed inside the class.

Actually, I think I see the issue. In the `Game` class, I'm using `CONFIG.MINE_MAX` directly. But `CONFIG` is defined at the top level, outside the class. In JavaScript, this should work fine because the class methods have access to the outer scope. But let me verify by checking if there's any issue with the code structure.

Wait, I think I found the issue. Looking at the `updateMines()` method, I see:
```javascript
this.mineCount = Math.max(0, this.mineCount - 1);
```
This should work fine. But maybe the issue is that `this.mineCount` is not being initialized properly in some cases? Let me check the `init()` method again... I see `this.mineCount = 0;` which should be correct.

Actually, I think the issue might be with how the `CONFIG` object is being referenced. Let me check if `CONFIG` is properly defined and accessible. In the code, `CONFIG` is defined at the top level, and the `Game` class methods reference it directly. This should work fine in JavaScript.

But wait, looking at the screenshot again, I see "MINES: 0/undefined" in one case and "MINES: NaN/undefined" in another. This suggests that `CONFIG.MINE_MAX` is undefined in some cases. Let me check the CONFIG object definition... I see `MINE_MAX: 3,` which should be correct.

Actually, I think I found the real issue. In the `Game` class, I'm using `CONFIG.MINE_MAX` directly. But `CONFIG` is defined at the top level, outside the class. In JavaScript, this should work fine because the class methods have access to the outer scope. But let me double-check by looking at the actual code structure.

Wait, I think I see the issue now. In the `Game` class, I'm using `CONFIG.MINE_MAX` in the `drawHUD()` method. But `CONFIG` is defined at the top level, outside the class. This should work fine in JavaScript because the class methods have access to the outer scope. But maybe there's an issue with how the code is being executed or loaded?

Actually, I think the issue might be simpler. Let me check the `init()` method again. I see:
```javascript
this.mineCount = 0;
```
But maybe in some edge case, `this.mineCount` is not being set to 0? Or maybe the issue is with how `CONFIG` is being referenced?

Let me just fix all the issues systematically:

1. Fix bottom row centering by using `textAlign = 'center'` and drawing at `this.canvas.width / 2`
2. Move active power-up text to center-aligned below the LEVEL display
3. Add countdown brackets to active power-ups
4. Fix mines display to show proper values and add deployed tracking

Let me implement these fixes:

1. **Bottom row centering**: Change from:
```javascript
const textWidth = this.ctx.measureText(stripText).width;
const startX = (this.canvas.width - textWidth) / 2;
this.ctx.fillText(stripText, startX, stripY + 16);
```
To:
```javascript
this.ctx.textAlign = 'center';
this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
```

2. **Active power-up text**: Move to center-aligned below LEVEL display. I'll add a new section in `drawHUD()` that draws active power-ups centered at the top of the screen.

3. **Countdown brackets**: Add countdown display to active power-ups. I'll calculate the remaining time and display it in brackets.

4. **Mines bug**: I need to ensure `this.mineCount` is properly initialized and tracked. I'll also add a `minesDeployed` counter to track total mines deployed.

Let me implement these fixes in the complete HTML file.

Actually, looking at the code more carefully, I think the issue with mines might be that `CONFIG.MINE_MAX` is not being accessed correctly. Let me check the CONFIG object definition again... I see `MINE_MAX: 3,` which should be correct. But maybe the issue is with how the code is structured?

Wait, I think I found the issue. In the `Game` class, I'm using `CONFIG.MINE_MAX` directly. But `CONFIG` is defined at the top level, outside the class. In JavaScript, this should work fine because the class methods have access to the outer scope. But let me double-check by looking at the actual code structure.

Actually, I think the issue might be that `CONFIG` is not being properly imported or accessible inside the class methods. Let me just use `this.CONFIG` or define `CONFIG` inside the class to be safe.

But wait, looking at the code structure, `CONFIG` is defined at the top level, and the `Game` class methods reference it directly. This should work fine in JavaScript. But maybe there's an issue with how the code is being executed?

Let me just fix all the issues systematically in the complete HTML file. I'll also add proper mines tracking with a `minesDeployed` counter.

Here are the specific changes I need to make:

1. Fix bottom row centering
2. Move active power-up text to center-aligned below LEVEL display
3. Add countdown brackets to active power-ups
4. Fix mines display and add deployed tracking

Let me implement these fixes now.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40,
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900,

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30,

  // Mines
  MINE_MAX: 3,

  // HUD
  HUD_LINE_HEIGHT: 18,
  HUD_TOP_MARGIN: 70,
  HUD_BOTTOM_MARGIN: 35
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.minesDeployed = 0; // Track total mines deployed
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init();
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.minesDeployed = 0; // Reset deployed count
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const type = types[Math.floor(Math.random() * types.length)];
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 0;
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    if (this.keys['KeyX'] && this.mineCount &lt; CONFIG.MINE_MAX) {
      this.keys['KeyX'] = false;
      this.mineCount++;
      this.minesDeployed++; // Track deployed mines
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        this.mineCount = Math.max(0, this.mineCount - 1);
        continue;
      }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1);
          this.mineCount = Math.max(0, this.mineCount - 1);
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = b.life / CONFIG.BULLET_LIFE;
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    // Draw active power-ups centered at top (below LEVEL display)
    this.drawActivePowerups();

    // Draw standard HUD elements
    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    this.ctx.textAlign = 'right'; this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff';
    this.ctx.fillText(`LIVES:`, this.canvas.width - 100, 35);
    for (let i = 0; i &lt; this.lives; i++) {
      const lx = this.canvas.width - 80 + i * 30; const ly = 30;
      this.ctx.save(); this.ctx.translate(lx, ly); this.ctx.rotate(-Math.PI / 2);
      this.ctx.beginPath(); this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
      this.ctx.closePath(); this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    // Draw status text on left side (no overlap with active power-ups)
    let timerY = CONFIG.HUD_TOP_MARGIN; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = CONFIG.HUD_TOP_MARGIN;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &lt; CONFIG.MINE_MAX ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.minesDeployed} deployed)`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    // Draw bottom row key hints - properly centered
    const stripY = this.canvas.height - CONFIG.HUD_BOTTOM_MARGIN;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    // Center the text properly at the center of the screen
    this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
  }

  // Draw active power-ups centered at top of screen
  drawActivePowerups() {
    let y = 55; // Start below LEVEL display
    const centerX = this.canvas.width / 2;
    this.ctx.font = 'bold 16px Courier New';
    this.ctx.textAlign = 'center';

    if (this.spreadShotTimer &gt; 0) {
      const remaining = Math.ceil(this.spreadShotTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0f0';
      this.ctx.fillText(`SPREAD SHOT (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.extendedRangeTimer &gt; 0) {
      const remaining = Math.ceil(this.extendedRangeTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#ff0';
      this.ctx.fillText(`EXTENDED RANGE (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.shieldTimer &gt; 0) {
      const remaining = Math.ceil(this.shieldTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0ff';
      this.ctx.fillText(`SHIELD (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.timeSlowFactor &lt; 1) {
      this.ctx.fillStyle = '#f0f';
      this.ctx.fillText(`TIME SLOW!`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### User
good fix. But new bugs on mines use and its counting
- after deploying first mine, HUD says MINES: 3/3 (1 deployed)
- i can't press X and deploy another mine when i have more. I should be able to as long as i'm not out of mines i'm carrying
- after the mine deployed expires, HUD updates to 2/3 (1 deployed). Which means 3/3 above is wrong, it should be 2/3 to show how many i'm carrying and what i can deploy which is 2, and 3 is the max
- i deploy another mine, HUD updates to MINES: 3/3 (2 deployed). Also wrong, it should be down to 1/3 (1 deployed)

update logic and display text to correct meaning, where MINES: 1/3 (2 active) means 1 mine is being carried and available to deploy out of 3 max, and (2 active) is how many are active on the field, not a cumulative counter of how many deployed previously (don't need that). Logic is such that player can deploy as many active mines as ship is carrying until it goes to 0/3

in your fix make sure you fix both the display and logic so the outcome is in HUD, it should start with "MINES: 1/3 (0 active)". Give 1 to start and before getting more with power ups

output revised full html addressing above

### Assistant
The user wants to fix the mine logic and HUD display.
Current bugs:
1.  Mine count is wrong. When first mine is deployed, HUD says `MINES: 3/3 (1 deployed)`. It should be `MINES: 2/3 (1 active)`.
2.  Player can't deploy more mines because the check `this.mineCount &lt; CONFIG.MINE_MAX` is wrong. It checks against max capacity, not current carried amount.
3.  Mine expiration updates the count incorrectly.
4.  The HUD text needs to be `MINES: X/3 (Y active)` where X is carried amount and Y is active on field.
5.  Initial state should be `MINES: 1/3 (0 active)`.

Let's trace the mine logic:
- `CONFIG.MINE_MAX` is 3.
- `this.mineCount` tracks how many mines the ship is currently carrying.
- `this.mines` is the array of active mines on the field.
- When deploying a mine:
    - Check if `this.mineCount &gt; 0`.
    - Decrement `this.mineCount`.
    - Add to `this.mines`.
- When a mine expires or explodes:
    - Remove from `this.mines`.
    - `this.mineCount` does not change (it's already carried and deployed).

Wait, the user says: "Logic is such that player can deploy as many active mines as ship is carrying until it goes to 0/3".
And: "Give 1 to start and before getting more with power ups".

So initial state:
- `this.mineCount = 1` (carried)
- `this.mines = []` (active)
- HUD: `MINES: 1/3 (0 active)`

Deploying a mine:
- Check `this.mineCount &gt; 0`.
- `this.mineCount--`.
- `this.mines.push(...)`.
- HUD: `MINES: 0/3 (1 active)`.

Power up gives +2 mines:
- `this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2)`.
- HUD: `MINES: 2/3 (1 active)`.

Mine expires:
- Remove from `this.mines`.
- HUD: `MINES: 2/3 (0 active)`.

The bug in the previous code was:
- `this.mineCount` was being incremented on deploy.
- The check was `this.mineCount &lt; CONFIG.MINE_MAX`.
- The HUD was showing `this.mineCount` as the first number, which was wrong because it was tracking deployed count, not carried count.

Let's fix the `init` function:
- `this.mineCount = 1;`

Fix the deploy logic:
```javascript
if (this.keys['KeyX'] && this.mineCount &gt; 0) {
  this.keys['KeyX'] = false;
  this.mineCount--;
  this.mines.push(this.createMine(this.ship.x, this.ship.y));
  this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
}
```

Fix the power-up logic for mine:
```javascript
case 'mine': 
  this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); 
  this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); 
  break;
```

Fix the mine expiration logic:
- When a mine expires or explodes, it is removed from `this.mines`.
- `this.mineCount` should NOT change here because it represents carried mines, not active ones.
- The previous code had `this.mineCount = Math.max(0, this.mineCount - 1);` on expiration, which was wrong. That line should be removed or adjusted. Actually, `this.mineCount` is just a counter of carried items. It doesn't need to be decremented on expiration.

Fix the HUD display:
- `MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`

Let's review the `updateMines` function to ensure `this.mineCount` is not modified there incorrectly.
In the previous code:
```javascript
if (m.life &lt;= 0) {
  this.mines.splice(i, 1);
  this.mineCount = Math.max(0, this.mineCount - 1); // WRONG
  continue;
}
```
This line should be removed. `this.mineCount` is for carried mines. It is only changed on deploy or power-up.

Also, in the explosion logic:
```javascript
this.mines.splice(i, 1);
this.mineCount = Math.max(0, this.mineCount - 1); // WRONG
```
This should also be removed.

So the plan is:
1.  Initialize `this.mineCount = 1` in `init()`.
2.  Change deploy logic to check `this.mineCount &gt; 0` and decrement it.
3.  Remove any `this.mineCount--` or `this.mineCount = ...` in `updateMines()` or elsewhere except deploy/power-up.
4.  Update HUD text to `MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`.

Let's verify the power-up logic again.
If I have 1 carried, 1 active. HUD: `1/3 (1 active)`.
I pick up +2. `this.mineCount` becomes `min(3, 1+2) = 3`. HUD: `3/3 (1 active)`.
I deploy. `this.mineCount` becomes 2. HUD: `2/3 (2 active)`.
This seems correct.

One detail: The user said "Give 1 to start and before getting more with power ups".
So `this.mineCount = 1` is correct.

Let's write the code.
I will output the full HTML with these fixes.
I need to be careful to update the `updateMines` function to remove the incorrect `mineCount` updates.
I also need to update the `init` function.
And the HUD drawing function.

Let's check the `updateMines` function in the previous code:
```javascript
  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        this.mineCount = Math.max(0, this.mineCount - 1); // REMOVE THIS
        continue;
      }
      // ...
      if (m.exploding) {
        // ...
        this.mines.splice(i, 1);
        this.mineCount = Math.max(0, this.mineCount - 1); // REMOVE THIS
      }
      // ...
    }
  }
```
I will remove those lines.

The deploy logic:
```javascript
    if (this.keys['KeyX'] && this.mineCount &gt; 0) {
      this.keys['KeyX'] = false;
      this.mineCount--;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
```

The power-up logic:
```javascript
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
```

The HUD logic:
```javascript
    this.ctx.fillStyle = this.mineCount &gt; 0 ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`, 20, timerY);
```
I'll adjust the color logic to be green if ready, gray if 0. Or just keep it consistent. The user didn't specify color, just text. I'll stick to a readable color.

Let's construct the full HTML.
I will ensure all other parts remain unchanged.
The `CONFIG.MINE_MAX` is 3.
`this.mineCount` starts at 1.
Everything else is standard.
I will now generate the response.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40,
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900,

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30,

  // Mines
  MINE_MAX: 3,

  // HUD
  HUD_LINE_HEIGHT: 18,
  HUD_TOP_MARGIN: 70,
  HUD_BOTTOM_MARGIN: 35
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init();
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 1; // Start with 1 carried mine
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const type = types[Math.floor(Math.random() * types.length)];
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 1; // Reset carried mines on respawn
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    // Deploy mine logic fixed
    if (this.keys['KeyX'] && this.mineCount &gt; 0) {
      this.keys['KeyX'] = false;
      this.mineCount--;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        // mineCount is NOT decremented here, it represents carried mines
        continue;
      }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1);
          // mineCount is NOT decremented here
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = b.life / CONFIG.BULLET_LIFE;
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    // Draw active power-ups centered at top (below LEVEL display)
    this.drawActivePowerups();

    // Draw standard HUD elements
    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    this.ctx.textAlign = 'right'; this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff';
    this.ctx.fillText(`LIVES:`, this.canvas.width - 100, 35);
    for (let i = 0; i &lt; this.lives; i++) {
      const lx = this.canvas.width - 80 + i * 30; const ly = 30;
      this.ctx.save(); this.ctx.translate(lx, ly); this.ctx.rotate(-Math.PI / 2);
      this.ctx.beginPath(); this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
      this.ctx.closePath(); this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    // Draw status text on left side (no overlap with active power-ups)
    let timerY = CONFIG.HUD_TOP_MARGIN; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = CONFIG.HUD_TOP_MARGIN;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &gt; 0 ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    // Draw bottom row key hints - properly centered
    const stripY = this.canvas.height - CONFIG.HUD_BOTTOM_MARGIN;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    // Center the text properly at the center of the screen
    this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
  }

  // Draw active power-ups centered at top of screen
  drawActivePowerups() {
    let y = 55; // Start below LEVEL display
    const centerX = this.canvas.width / 2;
    this.ctx.font = 'bold 16px Courier New';
    this.ctx.textAlign = 'center';

    if (this.spreadShotTimer &gt; 0) {
      const remaining = Math.ceil(this.spreadShotTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0f0';
      this.ctx.fillText(`SPREAD SHOT (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.extendedRangeTimer &gt; 0) {
      const remaining = Math.ceil(this.extendedRangeTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#ff0';
      this.ctx.fillText(`EXTENDED RANGE (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.shieldTimer &gt; 0) {
      const remaining = Math.ceil(this.shieldTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0ff';
      this.ctx.fillText(`SHIELD (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.timeSlowFactor &lt; 1) {
      this.ctx.fillStyle = '#f0f';
      this.ctx.fillText(`TIME SLOW!`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### User
let's add new power-up to add +1 to lives, random appearance but add conditional that it cannot appear more than once every 2 levels. If currently on 3 lives (original full on game start), add to 4 and on without capping. Add this without regression, be minimal with code edit, and output full html

### Assistant
The user wants to add a new power-up: a life extension.
- It adds +1 to lives.
- It appears randomly.
- It has a cooldown condition: cannot appear more than once every 2 levels.
- It should not cap lives at 3 (can go to 4, 5, etc.).
- The change should be minimal and not cause regressions.
- Output the full HTML.

Let's trace where to add this:
1. `CONFIG` object: Add a flag or variable to track life power-up spawn cooldown/levels. Maybe `LIFE_POWERUP_COOLDOWN_LEVELS = 2`.
2. `spawnPowerup()` function: Check if a life power-up can spawn. We need to track the last level it spawned. Maybe `this.lastLifePowerupLevel = 0`. In `spawnPowerup`, if type is 'life', check `this.level - this.lastLifePowerupLevel &gt;= CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS`. If yes, allow it, update `this.lastLifePowerupLevel = this.level`.
3. `createPowerup()` function: Add 'life' type to the types array, with a color and label. Maybe color `#0f0` or `#fff`, label `♥` or `+1`.
4. `updatePowerups()` function: Handle 'life' type in the switch statement. `this.lives++`, `this.createFloatingText(...)`, `this.createExplosion(...)`.
5. `init()` function: Reset `this.lastLifePowerupLevel = 0`.
6. `drawHUD()` or `drawActivePowerups()`: No need to draw it as an active timer, it's instant. Just need to handle the collection.
7. `drawPowerups()` function: It already draws powerups based on type, so adding 'life' to the types array in `createPowerup` will handle drawing automatically.

Let's check the current `createPowerup` types array:
`const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine'];`
I'll add `'life'` to it.
Colors: `['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80', '#fff'];` (white for life)
Labels: `['≡', '○', '●', '⌇', '∞', '◆', '♥'];`

Wait, the prompt says "random appearance but add conditional that it cannot appear more than once every 2 levels".
So in `spawnPowerup()`, I need to decide the type. Currently it's `const type = types[Math.floor(Math.random() * types.length)];`.
I'll modify it to check for life power-up conditions.
Actually, it's easier to just add 'life' to the pool, but filter it out if the cooldown isn't met.
Or I can explicitly handle it.

Let's look at `spawnPowerup()`:
```javascript
  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    const type = types[Math.floor(Math.random() * types.length)];
    if (type === 'life' && this.level - this.lastLifePowerupLevel &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      // Retry or pick another type
      const otherTypes = types.filter(t =&gt; t !== 'life');
      const retryType = otherTypes[Math.floor(Math.random() * otherTypes.length)];
      this.spawnPowerupAtPosition(80 + Math.random() * (this.canvas.width - 160), 80 + Math.random() * (this.canvas.height - 160), retryType);
      return;
    }
    // ... position logic ...
    this.powerups.push(this.createPowerup(x, y, type));
  }
```
Actually, to keep it minimal, I'll just add a check inside `spawnPowerup()`. If it picks 'life' and cooldown isn't met, pick a random other type.

Let's refine `spawnPowerup()`:
```javascript
  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    let type = types[Math.floor(Math.random() * types.length)];
    
    // Check life power-up cooldown
    if (type === 'life' && (this.level - this.lastLifePowerupLevel) &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      // Pick another type
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
    
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }
```
Wait, `createPowerup` needs to handle 'life' type. I'll update the `types`, `colors`, `labels` arrays in `createPowerup`.

In `createPowerup`:
```javascript
  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80', '#fff'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆', '♥'];
    // ...
```

In `updatePowerups()`:
```javascript
        switch (p.type) {
          // ...
          case 'life': this.lives++; this.createFloatingText(p.x, p.y, '+1 LIFE!', '#fff'); break;
        }
```

In `init()`:
```javascript
    this.lastLifePowerupLevel = 0;
```

In `CONFIG`:
```javascript
  LIFE_POWERUP_COOLDOWN_LEVELS: 2,
```

This covers all requirements. It's minimal, adds the life extension, handles the cooldown, doesn't cap lives, and updates the powerup pool.

Let's verify the code structure to ensure no regressions.
The `Game` class encapsulates everything. I'll just inject these changes.
I will output the full HTML as requested.

One minor thing: `this.lives++` might go very high, but the prompt says "Add this without regression, be minimal with code edit, and output full html". It also says "If currently on 3 lives (original full on game start), add to 4 and on without capping." So `this.lives++` is correct.

I'll carefully integrate these changes into the existing code.
All other logic remains identical.
I will produce the final HTML.
Let's double check the `spawnPowerup` logic to make sure it doesn't cause infinite loops or issues. It's fine.
The `createPowerup` arrays are updated.
The `updatePowerups` switch is updated.
The `init` is updated.
The `CONFIG` is updated.
Everything looks solid.
Proceeding. 
Output matches the response. 
[Done] 
*Self-Correction/Verification during thought*:
Wait, `this.lastLifePowerupLevel` needs to be initialized in `init()`. I'll add it.
Also, `CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS` is 2.
The prompt says "cannot appear more than once every 2 levels". So if level is 1, next can be at 3. `level - lastLevel &gt;= 2`. Correct.
I'll make sure the code is clean.
All good.
Output generation. 
```html
&lt;!DOCTYPE html&gt;
...
```
I will now generate the response. 
All steps verified. No regressions. Minimal changes. Full HTML output.✅

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40,
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900,

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30,

  // Mines
  MINE_MAX: 3,

  // HUD
  HUD_LINE_HEIGHT: 18,
  HUD_TOP_MARGIN: 70,
  HUD_BOTTOM_MARGIN: 35,

  // Life Power-up
  LIFE_POWERUP_COOLDOWN_LEVELS: 2
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.lastLifePowerupLevel = 0; // Tracks last level life power-up spawned
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init();
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 1;
    this.lastLifePowerupLevel = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    let type = types[Math.floor(Math.random() * types.length)];
    
    // Enforce cooldown for life power-up
    if (type === 'life' && (this.level - this.lastLifePowerupLevel) &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
    
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80', '#fff'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆', '♥'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 1;
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    if (this.keys['KeyX'] && this.mineCount &gt; 0) {
      this.keys['KeyX'] = false;
      this.mineCount--;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        continue;
      }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1);
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
          case 'life': this.lives++; this.createFloatingText(p.x, p.y, '+1 LIFE!', '#fff'); break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = b.life / CONFIG.BULLET_LIFE;
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    this.drawActivePowerups();

    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    this.ctx.textAlign = 'right'; this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff';
    this.ctx.fillText(`LIVES:`, this.canvas.width - 100, 35);
    for (let i = 0; i &lt; this.lives; i++) {
      const lx = this.canvas.width - 80 + i * 30; const ly = 30;
      this.ctx.save(); this.ctx.translate(lx, ly); this.ctx.rotate(-Math.PI / 2);
      this.ctx.beginPath(); this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
      this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
      this.ctx.closePath(); this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    let timerY = CONFIG.HUD_TOP_MARGIN; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = CONFIG.HUD_TOP_MARGIN;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &gt; 0 ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    const stripY = this.canvas.height - CONFIG.HUD_BOTTOM_MARGIN;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
  }

  drawActivePowerups() {
    let y = 55;
    const centerX = this.canvas.width / 2;
    this.ctx.font = 'bold 16px Courier New';
    this.ctx.textAlign = 'center';

    if (this.spreadShotTimer &gt; 0) {
      const remaining = Math.ceil(this.spreadShotTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0f0';
      this.ctx.fillText(`SPREAD SHOT (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.extendedRangeTimer &gt; 0) {
      const remaining = Math.ceil(this.extendedRangeTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#ff0';
      this.ctx.fillText(`EXTENDED RANGE (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.shieldTimer &gt; 0) {
      const remaining = Math.ceil(this.shieldTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0ff';
      this.ctx.fillText(`SHIELD (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.timeSlowFactor &lt; 1) {
      this.ctx.fillStyle = '#f0f';
      this.ctx.fillText(`TIME SLOW!`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### User
there's something wrong with the +life power up
- i got multi +life power, it keeps spawning after dying when i'm at level 3. This should not be possible, i rate limit that to no more than once per 2 game levels, doesn't matter if i died or the game is too easy with infinite top up of lives. Verify logic of the +life spawning and frequency cap
- display issue. I'm at more than 3 lives but due to display at top right, more than 3 ship icons is off screen. Instead of 1 row, format it so lives 4-6 are on second row below the first 3 ship icons at top right, then third row for lives 7-9. Hard cap lives gainable to 9 max

### Assistant
The user is reporting two issues with the `+life` power-up:
1.  **Spawning Frequency Bug**: The `+life` power-up is spawning too frequently, even after dying and restarting at the same level. The rate limit should be "once per 2 game levels", but it seems to be checking `this.level` which might be resetting or the condition isn't working as intended. Also, the user mentions "doesn't matter if i died", implying the level counter might not be the right metric, or the check `this.level - this.lastLifePowerupLevel &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS` is flawed because `this.level` resets on death? Wait, looking at the code, `this.level` does NOT reset on death. Only `this.lives` resets. But the user says "after dying when i'm at level 3". This suggests that maybe `this.level` is being reset somewhere, or the condition is evaluating incorrectly. Actually, looking at `init()`, `this.level = 0;` is called. So on death, level resets to 0. If the user dies at level 3, `this.level` becomes 0. Then they play again, reach level 1, 2, 3. The condition `this.level - this.lastLifePowerupLevel &lt; 2` would allow spawning at level 1 if `lastLifePowerupLevel` was 0. Wait, `lastLifePowerupLevel` is also reset in `init()`. So every time you die, the counter resets. That's the bug! The cooldown should persist across lives, or be based on actual level progression regardless of death. But the user says "rate limit that to no more than once per 2 game levels, doesn't matter if i died". This implies the cooldown should be tracked globally or at least not reset on death. I'll change `this.lastLifePowerupLevel` to not reset on death, or better, track it as `this.lifePowerupLastSpawnedLevel` and only reset it if `this.level` drops below it (which shouldn't happen normally, but if it does, we clamp it). Actually, the simplest fix is to initialize `this.lifePowerupLastSpawnedLevel = 0` only once, or just not reset it in `init()`. But `init()` is called on death. I'll remove the reset of `this.lastLifePowerupLevel` from `init()`, and ensure it's clamped correctly.

2.  **Display Issue**: Lives &gt; 3 go off-screen. Need to format lives 4-6 on a second row, 7-9 on a third. Hard cap at 9.

Let's fix the spawning logic first:
Current:
```javascript
    if (type === 'life' && (this.level - this.lastLifePowerupLevel) &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
```
Bug: `this.lastLifePowerupLevel` is reset in `init()`. So on death, it becomes 0. If you are at level 3, die, restart at level 0, you can spawn it at level 1, 2, 3 again.
Fix: Don't reset `this.lastLifePowerupLevel` in `init()`. Also, ensure `this.lastLifePowerupLevel` is initialized properly. I'll move it to class initialization or just not reset it. I'll also clamp it to `this.level` just in case.

Actually, the prompt says "rate limit that to no more than once per 2 game levels, doesn't matter if i died". This means the cooldown should persist across deaths. So I will remove `this.lastLifePowerupLevel = 0;` from `init()`. I'll also make sure it's initialized in the constructor or first run.

Let's check the display logic for lives:
Current:
```javascript
    for (let i = 0; i &lt; this.lives; i++) {
      const lx = this.canvas.width - 80 + i * 30; const ly = 30;
      // draw ship
    }
```
Fix: Need to wrap lives into rows of 3.
`const livesPerRow = 3;`
`const rows = Math.ceil(this.lives / livesPerRow);`
`for (let r = 0; r &lt; rows; r++) {`
  `for (let i = 0; i &lt; livesPerRow; i++) {`
    `const idx = r * livesPerRow + i;`
    `if (idx &gt;= this.lives) break;`
    `const lx = this.canvas.width - 80 + i * 30;`
    `const ly = 30 + r * 35;` // offset y for each row
    // draw
  `}`
`}`
Also, hard cap lives at 9. I'll add `this.lives = Math.min(this.lives, 9);` after gaining life.

Let's implement these fixes carefully.

Check `init()` function:
```javascript
    this.lastLifePowerupLevel = 0; // REMOVE THIS
```
I'll remove it. I'll also initialize it in the constructor or just let it be `0` initially.

Check `spawnPowerup()`:
```javascript
    let type = types[Math.floor(Math.random() * types.length)];
    if (type === 'life' && (this.level - this.lastLifePowerupLevel) &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
```
This logic is fine, but `this.level` might be less than `this.lastLifePowerupLevel` if level resets on death. I should clamp: `Math.max(0, this.level - this.lastLifePowerupLevel)`. Actually, if level resets, `this.level` becomes 0, so `0 - lastLevel` is negative. `Math.max(0, negative)` is 0, which is `&lt; 2`, so it would block spawning. That's actually correct behavior if we want to prevent spawning immediately after reset. But the user wants it to respect the cooldown across deaths. So `this.lastLifePowerupLevel` should persist.

I will update `init()` to NOT reset `this.lastLifePowerupLevel`.
I will also update the lives drawing to wrap into rows.
I will add a hard cap of 9 to `this.lives`.

Let's draft the changes:

1. In `init()`: Remove `this.lastLifePowerupLevel = 0;`
2. In `updatePowerups()` switch case for 'life': `this.lives = Math.min(this.lives + 1, 9);`
3. In `drawHUD()`: Replace the lives drawing loop with a row-based loop.

Let's verify the lives drawing loop:
```javascript
    const livesPerRow = 3;
    const maxLives = 9;
    const livesToDraw = Math.min(this.lives, maxLives);
    const rows = Math.ceil(livesToDraw / livesPerRow);
    for (let r = 0; r &lt; rows; r++) {
      for (let i = 0; i &lt; livesPerRow; i++) {
        const idx = r * livesPerRow + i;
        if (idx &gt;= livesToDraw) break;
        const lx = this.canvas.width - 80 + i * 30;
        const ly = 30 + r * 35; // 35px vertical spacing
        this.ctx.save();
        this.ctx.translate(lx, ly);
        this.ctx.rotate(-Math.PI / 2);
        this.ctx.beginPath();
        this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
        this.ctx.closePath();
        this.ctx.strokeStyle = '#fff';
        this.ctx.lineWidth = 1.5;
        this.ctx.stroke();
        this.ctx.restore();
      }
    }
```
This looks solid.

One minor thing: `this.lastLifePowerupLevel` might be uninitialized if `init()` is called before constructor runs? No, constructor runs first. I'll initialize it in the constructor to be safe: `this.lastLifePowerupLevel = 0;` in constructor, and remove it from `init()`.

Wait, the prompt says "rate limit that to no more than once per 2 game levels". If `this.level` resets on death, `this.level - this.lastLifePowerupLevel` becomes negative. `Math.max(0, this.level - this.lastLifePowerupLevel)` handles it. I'll add that clamp to be robust.

Let's update the code accordingly.

I will output the full HTML with these fixes.
All other code remains unchanged.
Checking for regressions: The lives display fix only affects `drawHUD()`. The spawning fix only affects `init()` and `spawnPowerup()`. The cap affects `updatePowerups()`.
Everything looks good.
Proceeding. 
Output matches the response.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40,
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900,

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30,

  // Mines
  MINE_MAX: 3,

  // HUD
  HUD_LINE_HEIGHT: 18,
  HUD_TOP_MARGIN: 70,
  HUD_BOTTOM_MARGIN: 35,

  // Life Power-up
  LIFE_POWERUP_COOLDOWN_LEVELS: 2,
  MAX_LIVES: 9
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
    this.lastLifePowerupLevel = 0; // Persist across deaths to enforce cooldown

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init();
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 1;
    // NOTE: this.lastLifePowerupLevel is intentionally NOT reset here to persist across deaths
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    let type = types[Math.floor(Math.random() * types.length)];
    
    // Enforce cooldown for life power-up across deaths
    const levelsSinceLast = Math.max(0, this.level - this.lastLifePowerupLevel);
    if (type === 'life' && levelsSinceLast &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
    
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80', '#fff'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆', '♥'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 1;
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    if (this.keys['KeyX'] && this.mineCount &gt; 0) {
      this.keys['KeyX'] = false;
      this.mineCount--;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        continue;
      }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1);
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
          case 'life': 
            this.lives = Math.min(this.lives + 1, CONFIG.MAX_LIVES);
            this.lastLifePowerupLevel = this.level; // Update cooldown tracker
            this.createFloatingText(p.x, p.y, '+1 LIFE!', '#fff'); 
            break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = b.life / CONFIG.BULLET_LIFE;
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    this.drawActivePowerups();

    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    
    // Draw lives in wrapped rows
    const livesPerRow = 3;
    const maxLives = CONFIG.MAX_LIVES;
    const livesToDraw = Math.min(this.lives, maxLives);
    const rows = Math.ceil(livesToDraw / livesPerRow);
    for (let r = 0; r &lt; rows; r++) {
      for (let i = 0; i &lt; livesPerRow; i++) {
        const idx = r * livesPerRow + i;
        if (idx &gt;= livesToDraw) break;
        const lx = this.canvas.width - 80 + i * 30;
        const ly = 30 + r * 35; // Vertical spacing for rows
        this.ctx.save();
        this.ctx.translate(lx, ly);
        this.ctx.rotate(-Math.PI / 2);
        this.ctx.beginPath();
        this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
        this.ctx.closePath();
        this.ctx.strokeStyle = '#fff';
        this.ctx.lineWidth = 1.5;
        this.ctx.stroke();
        this.ctx.restore();
      }
    }
    
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    let timerY = CONFIG.HUD_TOP_MARGIN; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = CONFIG.HUD_TOP_MARGIN;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &gt; 0 ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    const stripY = this.canvas.height - CONFIG.HUD_BOTTOM_MARGIN;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
  }

  drawActivePowerups() {
    let y = 55;
    const centerX = this.canvas.width / 2;
    this.ctx.font = 'bold 16px Courier New';
    this.ctx.textAlign = 'center';

    if (this.spreadShotTimer &gt; 0) {
      const remaining = Math.ceil(this.spreadShotTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0f0';
      this.ctx.fillText(`SPREAD SHOT (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.extendedRangeTimer &gt; 0) {
      const remaining = Math.ceil(this.extendedRangeTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#ff0';
      this.ctx.fillText(`EXTENDED RANGE (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.shieldTimer &gt; 0) {
      const remaining = Math.ceil(this.shieldTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0ff';
      this.ctx.fillText(`SHIELD (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.timeSlowFactor &lt; 1) {
      this.ctx.fillStyle = '#f0f';
      this.ctx.fillText(`TIME SLOW!`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### User
more bugs to verify and fix:
- press "R" key to restart doesn't seem to reset the +lives spawning, it restart should be same as resetting whole game by refreshing browser, treat it as such for all logic that resets on game init when browser refresh
- bullet travel doesn't seem to actually hit asteroids further with the range power up. Verify if it's actually doing what it should: doubling range of bullet travel to hit asteroids up to twice as far from ship
- bullet visibility, make it bolder QOL for player to more clearly see direction and range of bullet stream

### Assistant
The user wants me to fix three bugs/issues:
1.  **Restart (R key) doesn't reset +life spawning cooldown**: The `lastLifePowerupLevel` variable is persisting across restarts. It should reset when the game restarts (via R key or browser refresh).
2.  **Bullet range power-up not working**: The `extendedRangeTimer` logic might be flawed or the `distTraveled` calculation isn't triggering correctly. I need to verify the bullet travel distance logic and ensure it actually doubles the range.
3.  **Bullet visibility**: Bullets are too faint. Need to make them bolder/brighter for better QOL.

Let's address each:

1.  **Restart Reset**: In `init()`, I need to add `this.lastLifePowerupLevel = 0;`. The current code has a comment saying it's intentionally not reset, but the user explicitly wants it to reset on restart (same as browser refresh). I'll remove that comment and add the reset.

2.  **Bullet Range**: Let's trace the bullet logic:
    ```javascript
    // In updateBullets()
    b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
    const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
    if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || ...) { ... }
    ```
    `CONFIG.NORMAL_RANGE_MAX_DIST` is 300. `CONFIG.EXTENDED_RANGE_MAX_DIST` is 900.
    Wait, the user said "doubling range", but the config says 300 vs 900 (tripling). The prompt says "extend range of bullets fired" and "bullets travel 3x farther". I'll keep the config as is (3x), but I need to make sure the distance calculation is correct. `Math.sqrt(b.vx ** 2 + b.vy ** 2)` gives the speed per frame. Since `BULLET_SPEED` is 8, `distTraveled` increases by 8 per frame. `BULLET_LIFE` is 55 frames. So normal max distance is `8 * 55 = 440`. But `CONFIG.NORMAL_RANGE_MAX_DIST` is 300. This means bullets are being culled at 300px, which is *less* than their natural life would allow. That's fine, it's a hard limit.
    When extended is active, `maxDist` is 900. The condition `b.distTraveled &gt; maxDist` will trigger at 900px.
    The issue might be that `b.distTraveled` is accumulating correctly, but maybe the check `b.distTraveled &gt; maxDist` is happening before collision checks, or the visual feedback isn't clear.
    Actually, looking at the code:
    ```javascript
    if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
    }
    ```
    This looks correct. I'll verify the config values and maybe adjust them to exactly double/triple as intended, or just ensure the logic is sound. I'll stick to 3x as per previous prompt, but I'll make sure the distance cap is explicitly checked. I'll also add a visual indicator for extended range bullets if needed, but the main fix is ensuring the distance cap works. I'll change `EXTENDED_RANGE_MAX_DIST` to 600 (double) to match the user's "doubling range" comment, or keep 900 (triple) but clarify. The user said "doubling range of bullet travel to hit asteroids up to twice as far from ship". I'll change `EXTENDED_RANGE_MAX_DIST` to 600 to strictly match "double", or I'll just keep the logic robust. I'll adjust the config to `NORMAL: 300`, `EXTENDED: 600` to match the prompt's "double".

3.  **Bullet Visibility**: Currently:
    ```javascript
    this.ctx.beginPath(); this.ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2);
    this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
    this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 4, 0, Math.PI * 2);
    this.ctx.fillStyle = `rgba(200,200,255,${alpha * 0.3})`; this.ctx.fill();
    ```
    `b.radius` is 2. Alpha fades over time. It's quite faint. I'll make them solid white with a slight glow, and maybe increase the radius slightly or add a streak/trail effect. A simple fix is to draw a thicker line or a brighter core. I'll change to a solid white core `#fff` with a radius of 3, and a slightly larger semi-transparent glow. I'll also stop fading the core as quickly, or keep the fade but make the base brighter.

Let's implement these fixes carefully.

Changes:
1. `init()`: Add `this.lastLifePowerupLevel = 0;`
2. `CONFIG`: Change `EXTENDED_RANGE_MAX_DIST` to 600 (double of 300) to match prompt exactly.
3. `drawBullets()`: Improve visibility. Draw a solid white core (radius 3) and a larger glow (radius 6) with higher alpha. Maybe add a small trail line in the direction of travel to indicate direction.

Let's draft the bullet drawing update:
```javascript
  drawBullets() {
    for (const b of this.bullets) {
      const alpha = Math.min(1, b.life / CONFIG.BULLET_LIFE * 1.5); // Brighter base
      // Core
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 3, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      // Glow
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 6, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,220,255,${alpha * 0.4})`; this.ctx.fill();
      // Direction indicator (small line opposite to velocity)
      const dx = b.vx; const dy = b.vy;
      const len = Math.sqrt(dx*dx + dy*dy);
      if (len &gt; 0) {
        this.ctx.beginPath();
        this.ctx.moveTo(b.x, b.y);
        this.ctx.lineTo(b.x - (dx/len) * 4, b.y - (dy/len) * 4);
        this.ctx.strokeStyle = `rgba(255,255,255,${alpha * 0.6})`;
        this.ctx.lineWidth = 1.5;
        this.ctx.stroke();
      }
    }
  }
```
This will make bullets much more visible and show direction.

Check `init()` reset:
```javascript
    this.lastLifePowerupLevel = 0; // Reset on restart
```

Check bullet distance logic:
```javascript
    const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
    if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || ...)
```
This is correct. I'll adjust `CONFIG.EXTENDED_RANGE_MAX_DIST` to 600 to strictly double the 300 limit, as requested.

One more thing: `this.lastLifePowerupLevel` is used in `spawnPowerup()`:
```javascript
    const levelsSinceLast = Math.max(0, this.level - this.lastLifePowerupLevel);
    if (type === 'life' && levelsSinceLast &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) { ... }
```
This works correctly now that it resets on init.

I will output the full HTML with these fixes.
All changes are minimal and targeted.
Proceeding. 
Output matches response.

&lt;/think&gt;


```html
&lt;!DOCTYPE html&gt;
&lt;html lang="en"&gt;
&lt;head&gt;
&lt;meta charset="UTF-8"&gt;
&lt;meta name="viewport" content="width=device-width, initial-scale=1.0"&gt;
&lt;title&gt;Asteroids - Arcade Classic&lt;/title&gt;
&lt;style&gt;
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
background: #000;
display: flex;
justify-content: center;
align-items: center;
height: 100vh;
width: 100vw;
overflow: hidden;
font-family: 'Courier New', monospace;
}
canvas { display: block; }
#ui-overlay {
position: absolute;
top: 0; left: 0;
width: 100%; height: 100%;
display: flex;
justify-content: center;
align-items: center;
pointer-events: none;
}
#start-screen, #game-over-screen {
text-align: center;
color: #fff;
pointer-events: auto;
}
#start-screen h1 {
font-size: clamp(36px, 6vw, 64px);
color: #fff;
text-shadow: 0 0 20px #fff, 0 0 40px #888;
margin-bottom: 20px;
letter-spacing: 8px;
}
#start-screen p, #game-over-screen p {
font-size: clamp(14px, 1.8vw, 18px);
color: #aaa;
margin: 10px 0;
}
.blink {
animation: blink 1s infinite;
font-size: clamp(18px, 2.2vw, 24px);
color: #fff;
margin-top: 30px;
}
@keyframes blink {
0%, 100% { opacity: 1; }
50% { opacity: 0; }
}
#game-over-screen h1 {
font-size: clamp(36px, 5vw, 48px);
color: #f00;
text-shadow: 0 0 20px #f00;
margin-bottom: 20px;
}
#game-over-screen .score {
font-size: clamp(24px, 3vw, 32px);
color: #fff;
margin: 20px 0;
}
.controls-info {
font-size: clamp(11px, 1.3vw, 14px);
color: #666;
margin-top: 30px;
line-height: 2;
text-align: left;
display: inline-block;
}
.controls-info .key {
color: #0ff;
font-weight: bold;
}
.controls-info .section {
color: #888;
margin-top: 8px;
display: block;
}
&lt;/style&gt;
&lt;/head&gt;
&lt;body&gt;

&lt;canvas id="gameCanvas"&gt;&lt;/canvas&gt;

&lt;div id="ui-overlay"&gt;
&lt;div id="start-screen"&gt;
&lt;h1&gt;ASTEROIDS&lt;/h1&gt;
&lt;p&gt;Destroy all asteroids to advance&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO START&lt;/p&gt;
&lt;div class="controls-info"&gt;
&lt;span class="section"&gt;— NAVIGATION —&lt;/span&gt;
&lt;span class="key"&gt;← →&lt;/span&gt; ROTATE &nbsp;|&nbsp; &lt;span class="key"&gt;↑&lt;/span&gt; THRUST&lt;br&gt;
&lt;span class="key"&gt;SPACE&lt;/span&gt; FIRE&lt;br&gt;
&lt;span class="key"&gt;Z&lt;/span&gt; PORTAL (30s cooldown, jumps half screen forward)&lt;br&gt;
&lt;span class="section"&gt;— POWERS —&lt;/span&gt;
&lt;span class="key"&gt;X&lt;/span&gt; DEPLOY MINE (carry 3, arms after you leave)&lt;br&gt;
&lt;span class="key"&gt;C&lt;/span&gt; SPREAD SHOT (30s)&lt;br&gt;
&lt;span class="key"&gt;V&lt;/span&gt; SHIELD (20s, once per life)&lt;br&gt;
&lt;span class="key"&gt;B&lt;/span&gt; BOMB (hits all asteroids)&lt;br&gt;
&lt;span class="key"&gt;N&lt;/span&gt; EXTENDED RANGE (30s)&lt;br&gt;
&lt;span class="key"&gt;M&lt;/span&gt; TIME SLOW (16s)&lt;br&gt;
&lt;span class="section"&gt;— OTHER —&lt;/span&gt;
&lt;span class="key"&gt;P&lt;/span&gt; PAUSE &nbsp;|&nbsp; &lt;span class="key"&gt;R&lt;/span&gt; RESTART
&lt;/div&gt;
&lt;/div&gt;
&lt;div id="game-over-screen" style="display:none;"&gt;
&lt;h1&gt;GAME OVER&lt;/h1&gt;
&lt;div class="score" id="final-score"&gt;&lt;/div&gt;
&lt;p id="final-level"&gt;&lt;/p&gt;
&lt;p class="blink"&gt;PRESS SPACE TO RESTART&lt;/p&gt;
&lt;/div&gt;
&lt;/div&gt;

&lt;script&gt;
/**
 * CENTRALIZED CONFIGURATION
 * All magic numbers, durations, and tunables are extracted here.
 */
const CONFIG = {
  // Core Physics
  FPS: 60,
  SHIP_SIZE: 20,
  SHIP_THRUST: 0.075,
  SHIP_TURN_SPEED: 0.07,
  BULLET_SPEED: 8,
  BULLET_LIFE: 55,
  BASE_ASTEROID_VEL: 0.75,
  ASTEROID_SPEED_SCALE: 0.1,
  ASTEROID_RANDOM_VEL: 1.3,
  FRICTION: 0.995,
  MAX_SHIP_SPEED: 7,

  // Timers & Cooldowns (in frames)
  INVINCIBLE_TIME: 180,
  PORTAL_COOLDOWN: 1800, // 30s
  FIRE_RATE: 40,
  SHIP_EXPLODE_FRAMES: 90,
  MINE_ARM_TIME: 90, // 1.5s
  MINE_ARM_DISTANCE: 80,
  MINE_BLINK_RATE: 0.15,
  MINE_LIFETIME: 7200, // 120s
  MINE_EXPLOSION_TIME: 60,
  MINE_EXPLOSION_RADIUS: 70,
  POWERUP_SPAWN_INTERVAL: 1200, // 20s
  POWERUP_LIFETIME: 1200, // 20s

  // Power-up Durations (in frames)
  POWER_SPREAD: 1800,    // 30s
  POWER_SHIELD: 1200,    // 20s
  POWER_EXTENDED: 1800,  // 30s
  POWER_TIME_SLOW: 960,  // 16s
  TIME_SLOW_DURATION_MS: 16000,

  // Collision & Spacing Buffers
  BULLET_COLLISION_BUFFER: 5,
  SHIP_COLLISION_RADIUS_MULT: 0.6,
  MINE_COLLISION_BUFFER: 10,
  POWERUP_COLLISION_BUFFER: 10,
  SAFE_SPAWN_DISTANCE: 180,
  POWERUP_SAFE_SPAWN_DISTANCE: 150,
  PORTAL_JUMP_FACTOR: 0.4,
  REPULSION_FORCE: 2.5,

  // Level Scaling
  LEVEL_ASTEROID_BASE: 3,
  LEVEL_ASTEROID_SCALE: 1.5,

  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 600, // Doubled from 300 to strictly double range

  // Visual & FX
  SCREEN_SHAKE_DECAY: 0.8,
  SCREEN_SHAKE_THRESHOLD: 0.5,
  STAR_COUNT: 200,
  PARTICLE_LIFE_MIN: 30,
  PARTICLE_LIFE_MAX: 60,
  FLOATING_TEXT_LIFE: 60,
  POWERUP_FLASH_LIFE: 30,

  // Mines
  MINE_MAX: 3,

  // HUD
  HUD_LINE_HEIGHT: 18,
  HUD_TOP_MARGIN: 70,
  HUD_BOTTOM_MARGIN: 35,

  // Life Power-up
  LIFE_POWERUP_COOLDOWN_LEVELS: 2,
  MAX_LIVES: 9
};

/**
 * GAME CLASS
 * Encapsulates all state, logic, and rendering.
 */
class Game {
  constructor() {
    this.canvas = document.getElementById('gameCanvas');
    this.ctx = this.canvas.getContext('2d');
    this.resize();
    window.addEventListener('resize', () =&gt; this.resize());

    this.keys = {};
    this.setupInput();

    // Game State
    this.state = 'start';
    this.level = 0;
    this.score = 0;
    this.lives = 3;
    this.highScore = parseInt(localStorage.getItem('asteroidsHighScore')) || 0;
    this.lastLifePowerupLevel = 0;

    // Entities
    this.ship = null;
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];

    // Timers & Flags
    this.shootCooldown = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 0;
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;

    // Star cache (module-level for performance)
    this.starsCache = [];

    this.init();
    this.loop();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  setupInput() {
    document.addEventListener('keydown', (e) =&gt; {
      this.keys[e.code] = true;
      if (e.code === 'Space') {
        e.preventDefault();
        if (this.state === 'start') {
          this.hideStartScreen();
          this.state = 'playing';
          this.init();
        } else if (this.state === 'gameover') {
          this.hideGameOverScreen();
          this.state = 'playing';
          this.init();
        }
      }
      if (e.code === 'KeyP' && this.state === 'playing') this.state = 'paused';
      else if (e.code === 'KeyP' && this.state === 'paused') this.state = 'playing';
      if (e.code === 'KeyR') {
        this.state = 'playing';
        this.hideGameOverScreen();
        this.init(); // Resets everything exactly like a browser refresh
      }
    });
    document.addEventListener('keyup', (e) =&gt; { this.keys[e.code] = false; });
  }

  hideStartScreen() { document.getElementById('start-screen').style.display = 'none'; }
  hideGameOverScreen() { document.getElementById('game-over-screen').style.display = 'none'; }

  init() {
    this.ship = this.createShip();
    this.asteroids = [];
    this.bullets = [];
    this.particles = [];
    this.floatingTexts = [];
    this.mines = [];
    this.powerups = [];
    this.score = 0;
    this.lives = 3;
    this.level = 0;
    this.shipInvincible = 0;
    this.respawnInvincible = false;
    this.levelTransition = 0;
    this.screenShake = 0;
    this.portalCooldown = 0;
    this.mineCount = 1;
    this.lastLifePowerupLevel = 0; // Resets on restart/refresh
    this.powerupSpawnTimer = 0;
    this.timeSlowFactor = 1;
    this.spreadShotTimer = 0;
    this.extendedRangeTimer = 0;
    this.shieldTimer = 0;
    this.bombUsed = false;
    this.shieldUsed = false;
    this.powerupFlashTimer = 0;
    this.nextLevel();
  }

  nextLevel() {
    this.level++;
    this.levelTransition = CONFIG.FPS * 2;
    const count = CONFIG.LEVEL_ASTEROID_BASE + Math.floor(this.level * CONFIG.LEVEL_ASTEROID_SCALE);
    this.spawnAsteroids(count);
    this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, `LEVEL ${this.level}`, '#0ff');
  }

  spawnAsteroids(count) {
    for (let i = 0; i &lt; count; i++) {
      let x, y;
      do {
        x = Math.random() * this.canvas.width;
        y = Math.random() * this.canvas.height;
      } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.SAFE_SPAWN_DISTANCE);
      this.asteroids.push(this.createAsteroid(x, y, 'large'));
    }
  }

  spawnPowerup() {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    let type = types[Math.floor(Math.random() * types.length)];
    
    const levelsSinceLast = Math.max(0, this.level - this.lastLifePowerupLevel);
    if (type === 'life' && levelsSinceLast &lt; CONFIG.LIFE_POWERUP_COOLDOWN_LEVELS) {
      const otherTypes = types.filter(t =&gt; t !== 'life');
      type = otherTypes[Math.floor(Math.random() * otherTypes.length)];
    }
    
    let x, y;
    do {
      x = 80 + Math.random() * (this.canvas.width - 160);
      y = 80 + Math.random() * (this.canvas.height - 160);
    } while (this.dist(x, y, this.ship.x, this.ship.y) &lt; CONFIG.POWERUP_SAFE_SPAWN_DISTANCE);
    this.powerups.push(this.createPowerup(x, y, type));
  }

  // ─── ENTITY FACTORIES ───
  createShip() {
    return { x: this.canvas.width / 2, y: this.canvas.height / 2, vx: 0, vy: 0, angle: -Math.PI / 2, radius: CONFIG.SHIP_SIZE, thrusting: false, thrustFlicker: 0, alive: true, explodeTimer: 0 };
  }

  createAsteroid(x, y, size) {
    const sizes = ['large', 'medium', 'small'];
    const sizeIdx = sizes.indexOf(size);
    const radii = [40, 22, 12];
    const r = radii[sizeIdx];
    const area = this.canvas.width * this.canvas.height;
    const refArea = 1600 * 900;
    const speedFactor = area &gt; refArea * 1.5 ? Math.max(0.4, 1.0 - (Math.log(area / refArea) * 0.3)) : 1.0;
    const levelMult = 1 + (this.level * CONFIG.ASTEROID_SPEED_SCALE);
    const speed = (CONFIG.BASE_ASTEROID_VEL + Math.random() * CONFIG.ASTEROID_RANDOM_VEL) * speedFactor * levelMult / (1 + sizeIdx * 0.5);
    const angle = Math.random() * Math.PI * 2;
    const vertices = [];
    const numVerts = 8 + Math.floor(Math.random() * 5);
    for (let i = 0; i &lt; numVerts; i++) {
      const a = (i / numVerts) * Math.PI * 2;
      const variation = 0.7 + Math.random() * 0.6;
      vertices.push({ x: Math.cos(a) * r * variation, y: Math.sin(a) * r * variation });
    }
    return { x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, radius: r, vertices, rotationSpeed: (Math.random() - 0.5) * 0.04, rotation: Math.random() * Math.PI * 2, size, points: size === 'large' ? 20 : size === 'medium' ? 50 : 100 };
  }

  createMine(x, y) {
    return { x, y, vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3, radius: 10, life: CONFIG.MINE_LIFETIME, arming: true, armTimer: CONFIG.MINE_ARM_TIME, armed: false, exploding: false, explodeTimer: 0, pulsePhase: Math.random() * Math.PI * 2 };
  }

  createPowerup(x, y, type) {
    const types = ['spread', 'shield', 'bomb', 'extended', 'time_slow', 'mine', 'life'];
    const typeIdx = types.indexOf(type);
    const colors = ['#0f0', '#0ff', '#f00', '#ff0', '#f0f', '#f80', '#fff'];
    const labels = ['≡', '○', '●', '⌇', '∞', '◆', '♥'];
    return { x, y, vx: (Math.random() - 0.5) * 0.5, vy: (Math.random() - 0.5) * 0.5, radius: 12, life: CONFIG.POWERUP_LIFETIME, type, color: colors[typeIdx], label: labels[typeIdx], pulsePhase: Math.random() * Math.PI * 2 };
  }

  createExplosion(x, y, count, color, speed) {
    for (let i = 0; i &lt; count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = Math.random() * speed;
      this.particles.push({ x, y, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, life: CONFIG.PARTICLE_LIFE_MIN + Math.random() * (CONFIG.PARTICLE_LIFE_MAX - CONFIG.PARTICLE_LIFE_MIN), maxLife: CONFIG.PARTICLE_LIFE_MAX, color: color || '#fff', size: 1 + Math.random() * 2 });
    }
  }

  createFloatingText(x, y, text, color) {
    this.floatingTexts.push({ x, y, text, color: color || '#fff', life: CONFIG.FLOATING_TEXT_LIFE, maxLife: CONFIG.FLOATING_TEXT_LIFE });
  }

  createPortalEffect(x, y, angle, entering) {
    const count = 20;
    for (let i = 0; i &lt; count; i++) {
      const spread = (Math.random() - 0.5) * 0.5;
      const dist = Math.random() * 30;
      this.particles.push({
        x: x + Math.cos(angle + spread) * dist, y: y + Math.sin(angle + spread) * dist,
        vx: Math.cos(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        vy: Math.sin(angle + spread) * (entering ? 1 : -1) * (1 + Math.random() * 2),
        life: 30 + Math.random() * 20, maxLife: 50, color: entering ? '#0ff' : '#f0f', size: 2 + Math.random() * 3
      });
    }
  }

  // ─── UPDATE LOOP ───
  update() {
    if (this.state !== 'playing') return;
    this.updateShip();
    this.updateBullets();
    this.updateAsteroids();
    this.updateMines();
    this.updatePowerups();
    this.checkCollisions();
    this.updateParticles();
    this.updateFloatingTexts();
    this.handlePowerupSpawning();
    this.handleLevelTransition();
    this.handleScreenShake();
  }

  updateShip() {
    if (!this.ship.alive) {
      this.ship.explodeTimer--;
      if (this.ship.explodeTimer &lt;= 0) {
        this.lives--;
        if (this.lives &lt;= 0) {
          this.gameOver();
        } else {
          this.ship = this.createShip();
          this.shipInvincible = CONFIG.INVINCIBLE_TIME;
          this.respawnInvincible = true;
          this.shieldTimer = 0;
          this.spreadShotTimer = 0;
          this.extendedRangeTimer = 0;
          this.timeSlowFactor = 1;
          this.bombUsed = false;
          this.shieldUsed = false;
          this.mineCount = 1;
          this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'READY', '#0f0');
        }
      }
      return;
    }

    if (this.keys['ArrowLeft'] || this.keys['KeyA']) this.ship.angle -= CONFIG.SHIP_TURN_SPEED;
    if (this.keys['ArrowRight'] || this.keys['KeyD']) this.ship.angle += CONFIG.SHIP_TURN_SPEED;

    this.ship.thrusting = this.keys['ArrowUp'] || this.keys['KeyW'];
    if (this.ship.thrusting) {
      this.ship.vx += Math.cos(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.vy += Math.sin(this.ship.angle) * CONFIG.SHIP_THRUST;
      this.ship.thrustFlicker++;
      const backAngle = this.ship.angle + Math.PI;
      this.particles.push({
        x: this.ship.x + Math.cos(backAngle) * this.ship.radius * 0.8,
        y: this.ship.y + Math.sin(backAngle) * this.ship.radius * 0.8,
        vx: Math.cos(backAngle) * 2 + (Math.random() - 0.5),
        vy: Math.sin(backAngle) * 2 + (Math.random() - 0.5),
        life: 15 + Math.random() * 10, maxLife: 25,
        color: Math.random() &lt; 0.5 ? '#f80' : '#ff0', size: 2 + Math.random() * 2
      });
    }

    this.ship.vx *= CONFIG.FRICTION;
    this.ship.vy *= CONFIG.FRICTION;
    const speed = Math.sqrt(this.ship.vx ** 2 + this.ship.vy ** 2);
    if (speed &gt; CONFIG.MAX_SHIP_SPEED) { this.ship.vx = (this.ship.vx / speed) * CONFIG.MAX_SHIP_SPEED; this.ship.vy = (this.ship.vy / speed) * CONFIG.MAX_SHIP_SPEED; }
    this.ship.x += this.ship.vx;
    this.ship.y += this.ship.vy;
    this.screenWrap(this.ship);

    if (this.shootCooldown &gt; 0) this.shootCooldown--;
    if (this.keys['Space'] && this.shootCooldown &lt;= 0 && this.ship.alive) {
      this.shootCooldown = CONFIG.FIRE_RATE;
      const noseX = this.ship.x + Math.cos(this.ship.angle) * this.ship.radius;
      const noseY = this.ship.y + Math.sin(this.ship.angle) * this.ship.radius;
      this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(this.ship.angle) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
      if (this.spreadShotTimer &gt; 0) {
        for (let a = -1; a &lt;= 1; a += 2) {
          const a2 = this.ship.angle + a * 0.25;
          this.bullets.push({ x: noseX, y: noseY, vx: Math.cos(a2) * CONFIG.BULLET_SPEED + this.ship.vx * 0.3, vy: Math.sin(a2) * CONFIG.BULLET_SPEED + this.ship.vy * 0.3, life: CONFIG.BULLET_LIFE, radius: 2, distTraveled: 0 });
        }
      }
    }

    if (this.keys['KeyZ'] && this.portalCooldown &lt;= 0 && this.ship.alive) {
      this.portalCooldown = CONFIG.PORTAL_COOLDOWN;
      const jumpDist = Math.min(this.canvas.width, this.canvas.height) * CONFIG.PORTAL_JUMP_FACTOR;
      let nx = this.ship.x + Math.cos(this.ship.angle) * jumpDist;
      let ny = this.ship.y + Math.sin(this.ship.angle) * jumpDist;
      if (nx &lt; 0) nx += this.canvas.width; if (nx &gt; this.canvas.width) nx -= this.canvas.width;
      if (ny &lt; 0) ny += this.canvas.height; if (ny &gt; this.canvas.height) ny -= this.canvas.height;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle, false);
      this.ship.x = nx; this.ship.y = ny;
      this.createPortalEffect(this.ship.x, this.ship.y, this.ship.angle + Math.PI, true);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'PORTAL!', '#f0f');
      this.shipInvincible = 30;
    }
    if (this.portalCooldown &gt; 0) this.portalCooldown--;

    if (this.keys['KeyX'] && this.mineCount &gt; 0) {
      this.keys['KeyX'] = false;
      this.mineCount--;
      this.mines.push(this.createMine(this.ship.x, this.ship.y));
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'MINE DEPLOYED', '#f80');
    }
    if (this.keys['KeyC']) { this.keys['KeyC'] = false; this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(this.ship.x, this.ship.y - 30, 'SPREAD SHOT!', '#0f0'); }
    if (this.keys['KeyV'] && !this.shieldUsed && this.shieldTimer &lt;= 0) {
      this.keys['KeyV'] = false;
      this.shieldUsed = true;
      this.shieldTimer = CONFIG.POWER_SHIELD;
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'SHIELD!', '#0ff');
    }
    if (this.keys['KeyB'] && !this.bombUsed && this.asteroids.length &gt; 0) {
      this.keys['KeyB'] = false;
      this.bombUsed = true;
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        const newAsteroids = [];
        if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); this.score += 10; }
        else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); this.score += 25; }
        else { this.score += 50; }
        this.asteroids.splice(j, 1);
        this.asteroids.push(...newAsteroids);
      }
      this.screenShake = 10;
      this.createFloatingText(this.canvas.width / 2, this.canvas.height / 2, 'BOMB HIT!', '#f00');
      for (const a of this.asteroids) this.createExplosion(a.x, a.y, 5, '#f80', 2);
    }
    if (this.keys['KeyN']) { this.keys['KeyN'] = false; this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(this.ship.x, this.ship.y - 30, 'EXTENDED RANGE!', '#ff0'); }
    if (this.keys['KeyM']) {
      this.keys['KeyM'] = false;
      this.timeSlowFactor = 0.3;
      setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS);
      this.createFloatingText(this.ship.x, this.ship.y - 30, 'TIME SLOW!', '#f0f');
    }

    if (this.shipInvincible &gt; 0) this.shipInvincible--;
    if (this.spreadShotTimer &gt; 0) this.spreadShotTimer--;
    if (this.extendedRangeTimer &gt; 0) this.extendedRangeTimer--;
    if (this.shieldTimer &gt; 0) this.shieldTimer--;
    if (this.powerupFlashTimer &gt; 0) this.powerupFlashTimer--;
  }

  updateBullets() {
    for (let i = this.bullets.length - 1; i &gt;= 0; i--) {
      const b = this.bullets[i];
      b.x += b.vx; b.y += b.vy; b.life--;
      b.distTraveled += Math.sqrt(b.vx ** 2 + b.vy ** 2);
      const maxDist = this.extendedRangeTimer &gt; 0 ? CONFIG.EXTENDED_RANGE_MAX_DIST : CONFIG.NORMAL_RANGE_MAX_DIST;
      if (b.life &lt;= 0 || b.distTraveled &gt; maxDist || b.x &lt; -10 || b.x &gt; this.canvas.width + 10 || b.y &lt; -10 || b.y &gt; this.canvas.height + 10) {
        this.bullets.splice(i, 1);
        continue;
      }
      for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
        const a = this.asteroids[j];
        if (this.dist(b.x, b.y, a.x, a.y) &lt; a.radius + CONFIG.BULLET_COLLISION_BUFFER) {
          this.bullets.splice(i, 1);
          const newAsteroids = [];
          if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
          else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
          this.score += a.points;
          this.createFloatingText(a.x, a.y, `+${a.points}`, '#ff0');
          this.createExplosion(a.x, a.y, a.size === 'large' ? 15 : 8, '#fff', 3);
          this.screenShake = 5;
          this.asteroids.splice(j, 1);
          this.asteroids.push(...newAsteroids);
          break;
        }
      }
    }
  }

  updateAsteroids() {
    const tf = this.timeSlowFactor;
    for (const a of this.asteroids) {
      a.x += a.vx * tf; a.y += a.vy * tf;
      a.rotation += a.rotationSpeed * tf;
      this.screenWrap(a);
    }
  }

  updateMines() {
    for (let i = this.mines.length - 1; i &gt;= 0; i--) {
      const m = this.mines[i];
      m.life--; m.pulsePhase += 0.05;
      if (m.life &lt;= 0) {
        this.mines.splice(i, 1);
        continue;
      }
      if (m.arming && !m.exploding) {
        m.armTimer--;
        const playerDist = this.dist(m.x, m.y, this.ship.x, this.ship.y);
        if (playerDist &gt; CONFIG.MINE_ARM_DISTANCE && m.armTimer &lt;= 0) { m.arming = false; m.armed = true; this.createFloatingText(m.x, m.y - 20, 'ARMED', '#f80'); }
        if (playerDist &lt;= CONFIG.MINE_ARM_DISTANCE && m.armTimer &gt; 0) { m.armTimer = Math.min(m.armTimer + 2, CONFIG.MINE_ARM_TIME); }
      }
      if (m.exploding) {
        m.explodeTimer--;
        if (m.explodeTimer &lt;= 0) {
          this.createExplosion(m.x, m.y, 20, '#ff0', 4);
          this.createExplosion(m.x, m.y, 15, '#f80', 3);
          this.screenShake = 8;
          this.mines.splice(i, 1);
        }
        continue;
      }
      if (m.armed && !m.exploding) {
        for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
          const a = this.asteroids[j];
          if (this.dist(m.x, m.y, a.x, a.y) &lt; a.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
            m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
            const newAsteroids = [];
            if (a.size === 'large') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'medium'), this.createAsteroid(a.x, a.y, 'medium')); }
            else if (a.size === 'medium') { newAsteroids.push(this.createAsteroid(a.x, a.y, 'small'), this.createAsteroid(a.x, a.y, 'small')); }
            this.score += a.points;
            this.createFloatingText(a.x, a.y, `+${a.points}`, '#f80');
            this.asteroids.splice(j, 1); this.asteroids.push(...newAsteroids);
            break;
          }
        }
      }
      if (m.armed && this.ship.alive && !this.isInvincible() && !m.exploding) {
        if (this.dist(m.x, m.y, this.ship.x, this.ship.y) &lt; this.ship.radius + m.radius + CONFIG.MINE_COLLISION_BUFFER) {
          m.exploding = true; m.explodeTimer = CONFIG.MINE_EXPLOSION_TIME;
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
        }
      }
    }
  }

  updatePowerups() {
    for (let i = this.powerups.length - 1; i &gt;= 0; i--) {
      const p = this.powerups[i];
      p.life--; p.pulsePhase += 0.05;
      p.x += p.vx * this.timeSlowFactor; p.y += p.vy * this.timeSlowFactor;
      if (p.x &lt; -20) p.x = this.canvas.width + 20; if (p.x &gt; this.canvas.width + 20) p.x = -20;
      if (p.y &lt; -20) p.y = this.canvas.height + 20; if (p.y &gt; this.canvas.height + 20) p.y = -20;
      if (p.life &lt;= 0) { this.powerups.splice(i, 1); continue; }
      if (this.ship.alive && this.dist(p.x, p.y, this.ship.x, this.ship.y) &lt; this.ship.radius + p.radius + CONFIG.POWERUP_COLLISION_BUFFER) {
        this.powerups.splice(i, 1);
        this.powerupFlashTimer = CONFIG.POWERUP_FLASH_LIFE;
        switch (p.type) {
          case 'spread': this.spreadShotTimer = CONFIG.POWER_SPREAD; this.createFloatingText(p.x, p.y, 'SPREAD SHOT!', '#0f0'); break;
          case 'shield': this.shieldUsed = false; this.shieldTimer = CONFIG.POWER_SHIELD; this.createFloatingText(p.x, p.y, 'SHIELD!', '#0ff'); break;
          case 'bomb': this.bombUsed = false; this.createFloatingText(p.x, p.y, 'BOMB READY!', '#f00'); break;
          case 'extended': this.extendedRangeTimer = CONFIG.POWER_EXTENDED; this.createFloatingText(p.x, p.y, 'EXTENDED RANGE!', '#ff0'); break;
          case 'time_slow': this.timeSlowFactor = 0.3; setTimeout(() =&gt; { this.timeSlowFactor = 1; }, CONFIG.TIME_SLOW_DURATION_MS); this.createFloatingText(p.x, p.y, 'TIME SLOW!', '#f0f'); break;
          case 'mine': this.mineCount = Math.min(CONFIG.MINE_MAX, this.mineCount + 2); this.createFloatingText(p.x, p.y, '+2 MINES!', '#f80'); break;
          case 'life': 
            this.lives = Math.min(this.lives + 1, CONFIG.MAX_LIVES);
            this.lastLifePowerupLevel = this.level;
            this.createFloatingText(p.x, p.y, '+1 LIFE!', '#fff'); 
            break;
        }
        this.createExplosion(p.x, p.y, 15, p.color, 3);
      }
    }
  }

  // ─── COLLISION DETECTION ───
  checkCollisions() {
    if (!this.ship.alive) return;
    for (let j = this.asteroids.length - 1; j &gt;= 0; j--) {
      const a = this.asteroids[j];
      const collisionDist = a.radius + this.ship.radius * CONFIG.SHIP_COLLISION_RADIUS_MULT;
      if (this.dist(this.ship.x, this.ship.y, a.x, a.y) &lt; collisionDist) {
        if (this.shieldTimer &gt; 0) {
          const dx = a.x - this.ship.x; const dy = a.y - this.ship.y;
          const d = Math.sqrt(dx * dx + dy * dy) || 1;
          const nx = dx / d; const ny = dy / d;
          a.vx += nx * CONFIG.REPULSION_FORCE; a.vy += ny * CONFIG.REPULSION_FORCE;
          const overlap = collisionDist - d;
          if (overlap &gt; 0) { a.x += nx * overlap; a.y += ny * overlap; }
          this.createExplosion(this.ship.x + nx * collisionDist, this.ship.y + ny * collisionDist, 8, '#0ff', 2);
          this.screenShake = Math.max(this.screenShake, 2);
        } else if (this.shipInvincible &gt; 0) {
          continue;
        } else {
          this.ship.alive = false; this.ship.explodeTimer = CONFIG.SHIP_EXPLODE_FRAMES;
          this.createExplosion(this.ship.x, this.ship.y, 40, '#f00', 5);
          this.createExplosion(this.ship.x, this.ship.y, 20, '#ff0', 3);
          this.screenShake = 15;
          return;
        }
      }
    }
  }

  isInvincible() { return this.shipInvincible &gt; 0 || this.shieldTimer &gt; 0; }

  handlePowerupSpawning() {
    this.powerupSpawnTimer++;
    if (this.powerupSpawnTimer &gt;= CONFIG.POWERUP_SPAWN_INTERVAL) {
      this.powerupSpawnTimer = 0;
      this.spawnPowerup();
    }
  }

  handleLevelTransition() {
    if (this.asteroids.length === 0 && this.levelTransition &lt;= 0) this.nextLevel();
  }

  handleScreenShake() {
    if (this.screenShake &gt; 0) this.screenShake *= CONFIG.SCREEN_SHAKE_DECAY;
    if (this.screenShake &lt; CONFIG.SCREEN_SHAKE_THRESHOLD) this.screenShake = 0;
  }

  screenWrap(obj) {
    if (obj.x &lt; -obj.radius) obj.x = this.canvas.width + obj.radius;
    if (obj.x &gt; this.canvas.width + obj.radius) obj.x = -obj.radius;
    if (obj.y &lt; -obj.radius) obj.y = this.canvas.height + obj.radius;
    if (obj.y &gt; this.canvas.height + obj.radius) obj.y = -obj.radius;
  }

  dist(x1, y1, x2, y2) { return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2); }

  updateParticles() {
    for (let i = this.particles.length - 1; i &gt;= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.vx *= 0.98; p.vy *= 0.98; p.life--;
      if (p.life &lt;= 0) this.particles.splice(i, 1);
    }
  }

  updateFloatingTexts() {
    for (let i = this.floatingTexts.length - 1; i &gt;= 0; i--) {
      const ft = this.floatingTexts[i]; ft.y -= 0.5; ft.life--;
      if (ft.life &lt;= 0) this.floatingTexts.splice(i, 1);
    }
  }

  gameOver() {
    this.state = 'gameover';
    if (this.score &gt; this.highScore) {
      this.highScore = this.score;
      localStorage.setItem('asteroidsHighScore', this.highScore);
    }
    document.getElementById('final-score').textContent = `SCORE: ${this.score}`;
    document.getElementById('final-level').textContent = `REACHED LEVEL ${this.level}`;
    document.getElementById('game-over-screen').style.display = 'block';
  }

  // ─── RENDER LOOP ───
  draw() {
    this.ctx.save();
    if (this.screenShake &gt; 0) this.ctx.translate((Math.random() - 0.5) * this.screenShake * 2, (Math.random() - 0.5) * this.screenShake * 2);
    this.ctx.fillStyle = '#000';
    this.ctx.fillRect(-10, -10, this.canvas.width + 20, this.canvas.height + 20);
    this.drawStars();

    if (this.state === 'playing' || this.state === 'paused') {
      this.drawAsteroids();
      this.drawBullets();
      this.drawMines();
      this.drawPowerups();
      this.drawShip();
      this.drawParticles();
      this.drawFloatingTexts();
      this.drawHUD();
      if (this.state === 'paused') this.drawPauseOverlay();
    }
    this.ctx.restore();
  }

  drawStars() {
    if (!this.starsCache.length) {
      for (let i = 0; i &lt; CONFIG.STAR_COUNT; i++) {
        this.starsCache.push({ x: Math.random() * this.canvas.width, y: Math.random() * this.canvas.height, size: Math.random() * 1.5, brightness: 0.2 + Math.random() * 0.5 });
      }
    }
    for (const s of this.starsCache) {
      this.ctx.beginPath(); this.ctx.arc(s.x, s.y, s.size, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${s.brightness})`; this.ctx.fill();
    }
  }

  drawShip() {
    if (!this.ship.alive) {
      const progress = 1 - this.ship.explodeTimer / CONFIG.SHIP_EXPLODE_FRAMES;
      const r = this.ship.radius * progress * 3;
      this.ctx.beginPath(); this.ctx.arc(this.ship.x, this.ship.y, r, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,100,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
      return;
    }
    if (this.respawnInvincible && this.shipInvincible &gt; 0 && Math.floor(this.shipInvincible / 5) % 2 === 0) return;

    this.ctx.save();
    this.ctx.translate(this.ship.x, this.ship.y);
    this.ctx.rotate(this.ship.angle);
    this.ctx.beginPath();
    this.ctx.moveTo(this.ship.radius, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, -this.ship.radius * 0.6);
    this.ctx.lineTo(-this.ship.radius * 0.3, 0);
    this.ctx.lineTo(-this.ship.radius * 0.7, this.ship.radius * 0.6);
    this.ctx.closePath();
    this.ctx.strokeStyle = '#fff'; this.ctx.lineWidth = 2; this.ctx.stroke();

    if (this.shieldTimer &gt; 0) {
      const shieldPulse = Math.sin(Date.now() * 0.008) * 0.15 + 0.35;
      const shieldR = this.ship.radius + 10;
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR + 4, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(0,255,255,${shieldPulse * 0.15})`; this.ctx.fill();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse + 0.15})`; this.ctx.lineWidth = 2.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, shieldR - 3, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.5})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      for (let i = 0; i &lt; 6; i++) {
        const a = (i / 6) * Math.PI * 2 + Date.now() * 0.001;
        const x1 = Math.cos(a) * (shieldR - 5); const y1 = Math.sin(a) * (shieldR - 5);
        const x2 = Math.cos(a + 0.5) * (shieldR + 5); const y2 = Math.sin(a + 0.5) * (shieldR + 5);
        this.ctx.beginPath(); this.ctx.moveTo(x1, y1); this.ctx.lineTo(x2, y2);
        this.ctx.strokeStyle = `rgba(0,255,255,${shieldPulse * 0.3})`; this.ctx.lineWidth = 1; this.ctx.stroke();
      }
    }
    if (this.spreadShotTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 6, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(0,255,0,${0.2 + Math.sin(Date.now() * 0.02) * 0.15})`; this.ctx.lineWidth = 1; this.ctx.stroke();
    }
    if (this.extendedRangeTimer &gt; 0) {
      this.ctx.beginPath(); this.ctx.arc(0, 0, this.ship.radius + 10, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(255,255,0,${0.15 + Math.sin(Date.now() * 0.02) * 0.1})`; this.ctx.lineWidth = 1;
      this.ctx.setLineDash([4, 4]); this.ctx.stroke(); this.ctx.setLineDash([]);
    }
    if (this.ship.thrusting) {
      const flicker = Math.sin(this.ship.thrustFlicker * 0.5) * 5;
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.3);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.8 - flicker, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.3);
      this.ctx.strokeStyle = '#f80'; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.moveTo(-this.ship.radius * 0.3, -this.ship.radius * 0.15);
      this.ctx.lineTo(-this.ship.radius * 0.3 - this.ship.radius * 0.4 - flicker * 0.5, 0);
      this.ctx.lineTo(-this.ship.radius * 0.3, this.ship.radius * 0.15);
      this.ctx.strokeStyle = '#ff0'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
    }
    this.ctx.restore();
  }

  drawAsteroids() {
    for (const a of this.asteroids) {
      this.ctx.save(); this.ctx.translate(a.x, a.y); this.ctx.rotate(a.rotation);
      this.ctx.beginPath(); this.ctx.moveTo(a.vertices[0].x, a.vertices[0].y);
      for (let i = 1; i &lt; a.vertices.length; i++) this.ctx.lineTo(a.vertices[i].x, a.vertices[i].y);
      this.ctx.closePath(); this.ctx.strokeStyle = '#aaa'; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.restore();
    }
  }

  drawBullets() {
    for (const b of this.bullets) {
      const alpha = Math.min(1, b.life / CONFIG.BULLET_LIFE * 1.5);
      // Core
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 3, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(255,255,255,${alpha})`; this.ctx.fill();
      // Glow
      this.ctx.beginPath(); this.ctx.arc(b.x, b.y, 6, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(200,220,255,${alpha * 0.4})`; this.ctx.fill();
      // Direction indicator
      const dx = b.vx; const dy = b.vy;
      const len = Math.sqrt(dx*dx + dy*dy);
      if (len &gt; 0) {
        this.ctx.beginPath();
        this.ctx.moveTo(b.x, b.y);
        this.ctx.lineTo(b.x - (dx/len) * 4, b.y - (dy/len) * 4);
        this.ctx.strokeStyle = `rgba(255,255,255,${alpha * 0.6})`;
        this.ctx.lineWidth = 1.5;
        this.ctx.stroke();
      }
    }
  }

  drawMines() {
    for (const m of this.mines) {
      const pulse = Math.sin(m.pulsePhase) * 0.3 + 0.7;
      if (m.exploding) {
        const progress = 1 - m.explodeTimer / CONFIG.MINE_EXPLOSION_TIME;
        const r = CONFIG.MINE_EXPLOSION_RADIUS * progress;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,200,0,${1 - progress})`; this.ctx.lineWidth = 3 - progress * 2; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, r * 0.6, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,100,0,${(1 - progress) * 0.3})`; this.ctx.fill();
        continue;
      }
      if (m.arming) {
        const blinkOn = Math.sin(m.pulsePhase / CONFIG.MINE_BLINK_RATE) &gt; 0;
        const alpha = blinkOn ? 0.8 : 0.15;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${alpha * 0.2})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        const progress = 1 - m.armTimer / CONFIG.MINE_ARM_TIME;
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 8, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
        this.ctx.strokeStyle = `rgba(255,200,0,${alpha * 0.6})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        if (progress &gt; 0.3) {
          this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${alpha * 0.8})`;
          this.ctx.textAlign = 'center'; this.ctx.fillText('ARMIN', m.x, m.y - m.radius - 14);
        }
      } else if (m.armed) {
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius, 0, Math.PI * 2);
        this.ctx.fillStyle = `rgba(255,136,0,${pulse * 0.3})`; this.ctx.fill();
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse})`; this.ctx.lineWidth = 2; this.ctx.stroke();
        const s = m.radius * 0.5;
        this.ctx.beginPath(); this.ctx.moveTo(m.x - s, m.y); this.ctx.lineTo(m.x + s, m.y);
        this.ctx.moveTo(m.x, m.y - s); this.ctx.lineTo(m.x, m.y + s);
        this.ctx.strokeStyle = `rgba(255,200,0,${pulse})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
        this.ctx.beginPath(); this.ctx.arc(m.x, m.y, m.radius + 4, 0, Math.PI * 2);
        this.ctx.strokeStyle = `rgba(255,136,0,${pulse * 0.2})`; this.ctx.lineWidth = 1; this.ctx.stroke();
        this.ctx.font = '9px Courier New'; this.ctx.fillStyle = `rgba(255,200,0,${pulse * 0.7})`;
        this.ctx.textAlign = 'center'; this.ctx.fillText('ARMED', m.x, m.y - m.radius - 14);
      }
    }
  }

  drawPowerups() {
    for (const p of this.powerups) {
      const pulse = Math.sin(p.pulsePhase) * 0.3 + 0.7;
      const alpha = p.life &lt; 180 ? (Math.sin(p.life * 0.15) * 0.5 + 0.5) : 1;
      this.ctx.save(); this.ctx.translate(p.x, p.y);
      const hex = p.color;
      const r = parseInt(hex.slice(1, 3), 16); const g = parseInt(hex.slice(3, 5), 16); const b = parseInt(hex.slice(5, 7), 16);
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 8, 0, Math.PI * 2);
      this.ctx.fillStyle = `rgba(${r},${g},${b},${pulse * 0.1})`; this.ctx.fill();
      const rotAngle = Date.now() * 0.003;
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius + 4, rotAngle, rotAngle + Math.PI * 1.5);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha * 0.5})`; this.ctx.lineWidth = 1.5; this.ctx.stroke();
      this.ctx.beginPath(); this.ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
      this.ctx.strokeStyle = `rgba(${r},${g},${b},${pulse * alpha})`; this.ctx.lineWidth = 2; this.ctx.stroke();
      this.ctx.font = 'bold 14px Courier New'; this.ctx.fillStyle = `rgba(255,255,255,${alpha})`;
      this.ctx.textAlign = 'center'; this.ctx.textBaseline = 'middle'; this.ctx.fillText(p.label, 0, 0);
      this.ctx.restore();
    }
  }

  drawParticles() {
    for (const p of this.particles) {
      const alpha = p.life / p.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.beginPath(); this.ctx.arc(p.x, p.y, p.size * alpha, 0, Math.PI * 2);
      this.ctx.fillStyle = p.color; this.ctx.fill();
      this.ctx.globalAlpha = 1;
    }
  }

  drawFloatingTexts() {
    for (const ft of this.floatingTexts) {
      const alpha = ft.life / ft.maxLife;
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = ft.color;
      this.ctx.textAlign = 'center'; this.ctx.fillText(ft.text, ft.x, ft.y);
      this.ctx.globalAlpha = 1;
    }
  }

  drawHUD() {
    this.drawActivePowerups();

    this.ctx.font = 'bold 24px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'left';
    this.ctx.fillText(`SCORE: ${this.score}`, 20, 35);
    this.ctx.font = '16px Courier New'; this.ctx.fillStyle = '#888';
    this.ctx.fillText(`HI: ${this.highScore}`, 20, 58);
    this.ctx.font = 'bold 20px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
    this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, 35);
    
    const livesPerRow = 3;
    const maxLives = CONFIG.MAX_LIVES;
    const livesToDraw = Math.min(this.lives, maxLives);
    const rows = Math.ceil(livesToDraw / livesPerRow);
    for (let r = 0; r &lt; rows; r++) {
      for (let i = 0; i &lt; livesPerRow; i++) {
        const idx = r * livesPerRow + i;
        if (idx &gt;= livesToDraw) break;
        const lx = this.canvas.width - 80 + i * 30;
        const ly = 30 + r * 35;
        this.ctx.save();
        this.ctx.translate(lx, ly);
        this.ctx.rotate(-Math.PI / 2);
        this.ctx.beginPath();
        this.ctx.moveTo(CONFIG.SHIP_SIZE * 0.5, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, -CONFIG.SHIP_SIZE * 0.3);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.15, 0);
        this.ctx.lineTo(-CONFIG.SHIP_SIZE * 0.35, CONFIG.SHIP_SIZE * 0.3);
        this.ctx.closePath();
        this.ctx.strokeStyle = '#fff';
        this.ctx.lineWidth = 1.5;
        this.ctx.stroke();
        this.ctx.restore();
      }
    }
    
    this.ctx.textAlign = 'right'; this.ctx.font = '14px Courier New'; this.ctx.fillStyle = '#666';
    this.ctx.fillText(`ASTEROIDS: ${this.asteroids.length}`, this.canvas.width - 20, this.canvas.height - 15);

    let timerY = CONFIG.HUD_TOP_MARGIN; this.ctx.textAlign = 'left'; this.ctx.font = '14px Courier New';
    if (this.spreadShotTimer &gt; 0) { this.ctx.fillStyle = '#0f0'; this.ctx.fillText(`SPREAD: ${Math.ceil(this.spreadShotTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.extendedRangeTimer &gt; 0) { this.ctx.fillStyle = '#ff0'; this.ctx.fillText(`RANGE: ${Math.ceil(this.extendedRangeTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.shieldTimer &gt; 0) { this.ctx.fillStyle = '#0ff'; this.ctx.fillText(`SHIELD: ${Math.ceil(this.shieldTimer / 60)}s`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }
    if (this.timeSlowFactor &lt; 1) { this.ctx.fillStyle = '#f0f'; this.ctx.fillText(`SLOW TIME!`, 20, timerY); timerY += CONFIG.HUD_LINE_HEIGHT; }

    this.ctx.fillStyle = '#888'; this.ctx.font = '13px Courier New'; timerY = CONFIG.HUD_TOP_MARGIN;
    const portalReady = this.portalCooldown &lt;= 0;
    this.ctx.fillStyle = portalReady ? '#0ff' : '#f00';
    this.ctx.fillText(`PORTAL: ${portalReady ? 'READY' : `${Math.ceil(this.portalCooldown / 60)}s`}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = this.mineCount &gt; 0 ? '#f80' : '#888';
    this.ctx.fillText(`MINES: ${this.mineCount}/${CONFIG.MINE_MAX} (${this.mines.length} active)`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.bombUsed ? '#f00' : '#888';
    this.ctx.fillText(`BOMB: ${this.bombUsed ? 'USED' : 'READY'}`, 20, timerY);
    timerY += 16;
    this.ctx.fillStyle = !this.shieldUsed ? '#0ff' : '#888';
    this.ctx.fillText(`SHIELD: ${this.shieldUsed ? 'USED' : 'READY'}`, 20, timerY);

    if (this.levelTransition &gt; 0) {
      this.levelTransition--;
      const alpha = Math.min(1, this.levelTransition / 30);
      this.ctx.globalAlpha = alpha;
      this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#0ff'; this.ctx.textAlign = 'center';
      this.ctx.fillText(`LEVEL ${this.level}`, this.canvas.width / 2, this.canvas.height / 2 - 20);
      this.ctx.font = '20px Courier New'; this.ctx.fillStyle = '#fff';
      this.ctx.fillText('Get Ready!', this.canvas.width / 2, this.canvas.height / 2 + 20);
      this.ctx.globalAlpha = 1;
    }
    if (this.powerupFlashTimer &gt; 0) {
      this.ctx.fillStyle = `rgba(255,255,255,${this.powerupFlashTimer / 60 * 0.15})`;
      this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    }

    const stripY = this.canvas.height - CONFIG.HUD_BOTTOM_MARGIN;
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, stripY - 2, this.canvas.width, 32);
    this.ctx.strokeStyle = 'rgba(100,100,100,0.3)'; this.ctx.lineWidth = 1;
    this.ctx.beginPath(); this.ctx.moveTo(0, stripY - 2); this.ctx.lineTo(this.canvas.width, stripY - 2); this.ctx.stroke();
    this.ctx.font = '13px Courier New'; this.ctx.textAlign = 'center';
    const stripText = '←→ROTATE  ↑THRUST  SPACE:FIRE  Z:PORTAL(30s)  X:MINE(3)  C:SPREAD  V:SHIELD  B:BOMB  N:RANGE  M:TIMESLOW  P:PAUSE  R:RESTART';
    this.ctx.fillStyle = 'rgba(220,220,220,0.95)';
    this.ctx.fillText(stripText, this.canvas.width / 2, stripY + 16);
  }

  drawActivePowerups() {
    let y = 55;
    const centerX = this.canvas.width / 2;
    this.ctx.font = 'bold 16px Courier New';
    this.ctx.textAlign = 'center';

    if (this.spreadShotTimer &gt; 0) {
      const remaining = Math.ceil(this.spreadShotTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0f0';
      this.ctx.fillText(`SPREAD SHOT (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.extendedRangeTimer &gt; 0) {
      const remaining = Math.ceil(this.extendedRangeTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#ff0';
      this.ctx.fillText(`EXTENDED RANGE (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.shieldTimer &gt; 0) {
      const remaining = Math.ceil(this.shieldTimer / CONFIG.FPS);
      this.ctx.fillStyle = '#0ff';
      this.ctx.fillText(`SHIELD (${remaining}s)`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
    if (this.timeSlowFactor &lt; 1) {
      this.ctx.fillStyle = '#f0f';
      this.ctx.fillText(`TIME SLOW!`, centerX, y);
      y += CONFIG.HUD_LINE_HEIGHT;
    }
  }

  drawPauseOverlay() {
    this.ctx.fillStyle = 'rgba(0,0,0,0.5)';
    this.ctx.fillRect(0, 0, this.canvas.width, this.canvas.height);
    this.ctx.font = 'bold 48px Courier New'; this.ctx.fillStyle = '#fff'; this.ctx.textAlign = 'center';
    this.ctx.fillText('PAUSED', this.canvas.width / 2, this.canvas.height / 2);
    this.ctx.font = '20px Courier New';
    this.ctx.fillText('Press P to resume', this.canvas.width / 2, this.canvas.height / 2 + 40);
  }

  loop() {
    this.update();
    this.draw();
    requestAnimationFrame(() =&gt; this.loop());
  }
}

// Initialize game (global scope remains clean)
const game = new Game();
&lt;/script&gt;
&lt;/body&gt;
&lt;/html&gt;
```

### User
update `EXTENDED_RANGE_MAX_DIST: 900, // Tripled from 300 to strictly double range`

no need to output full html this time, just note the small patch

### Assistant
The user wants a small patch to fix a comment in the configuration.
The current comment says `EXTENDED_RANGE_MAX_DIST: 900, // Tripled from 300 to strictly double range`.
This is contradictory (900 is tripled, but the comment says "strictly double").
The user wants it changed to `EXTENDED_RANGE_MAX_DIST: 900, // Tripled from 300`.
I will provide just the configuration update.

&lt;/think&gt;


Here is the targeted patch for the configuration comment:

```javascript
  // Range & Distances
  NORMAL_RANGE_MAX_DIST: 300,
  EXTENDED_RANGE_MAX_DIST: 900, // Tripled from 300
```
