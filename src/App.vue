<template>
  <div class="game-container">
    
    <div v-if="gameState === 'login'" class="screen login-screen">
      <h1 class="title">CHAIN HOOK</h1>
      <h2 class="subtitle-game">Tame the Glitch</h2>
      <p class="instruction">Connect your wallet to stabilize the system</p>
      <button @click="connectWallet" class="btn-pixel">
        🦊 CONNECT PROTOCOL
      </button>
      <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
    </div>

    <div v-if="gameState === 'playing'" class="hud">
      <div class="wallet-info">USER: {{ shortAddress }}</div>
      <div class="bars">
        <div class="health-bar">
          <div :style="{ width: player.hp + '%' }" class="hp-fill"></div>
        </div>
        <span class="hp-text">CORE INTEGRITY</span>
      </div>
      <div class="controls-hint">ARROWS: Move | SPACE: Hook/Attack</div>
    </div>

    <div v-if="gameState === 'gameover'" class="screen over-screen">
      <h1 class="red-text">SYSTEM FAILURE</h1>
      <p>Glitch took control.</p>
      <button @click="resetGame" class="btn-pixel">REBOOT</button>
    </div>

    <div v-if="gameState === 'won'" class="screen win-screen">
      <h1 class="gold-text">SYSTEM STABILIZED</h1>
      <p>You tamed the glitches in the system!</p>
      <p>Wallet Proof: {{ shortAddress }}</p>
      <button @click="resetGame" class="btn-pixel">NEW SESSION</button>
    </div>

    <canvas 
      ref="gameCanvas" 
      width="800" 
      height="450"
      v-show="gameState !== 'login'"
    ></canvas>

  </div>
</template>

<script setup>
import { ref, computed, onUnmounted, nextTick } from 'vue';
import { createPublicClient, custom, getContract } from 'viem';
import { mainnet } from 'viem/chains';

const gameState = ref('login');
const walletAddress = ref('');
const errorMsg = ref('');
const gameCanvas = ref(null);
let ctx = null;
let animationFrameId = null;

// --- CONFIG ---
const GRAVITY = 0.8;
const GROUND_Y = 380;
const CANVAS_WIDTH = 800;

const player = ref({
  x: 50, y: GROUND_Y, width: 35, height: 45,
  vx: 0, vy: 0, speed: 5, jumpStrength: -16,
  hp: 100, attacking: false, attackCooldown: 0,
  facingRight: true, color: '#00f2ff' // Cyan Neon
});

// --- UNISWAP V3 CONSTANTS ---
const UNISWAP_V3_POSITION_MANAGER_ADDRESS = '0xC36442b4a4522E871399CD717aBDD847Ab11FE88';
const UNISWAP_V3_POSITION_MANAGER_ABI = [
  {"inputs":[{"internalType":"address","name":"owner","type":"address"}],"name":"balanceOf","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
  {"inputs":[{"internalType":"uint256","name":"tokenId","type":"uint256"}],"name":"positions","outputs":[{"internalType":"uint96","name":"nonce","type":"uint96"},{"internalType":"address","name":"operator","type":"address"},{"internalType":"address","name":"token0","type":"address"},{"internalType":"address","name":"token1","type":"address"},{"internalType":"uint24","name":"fee","type":"uint24"},{"internalType":"int24","name":"tickLower","type":"int24"},{"internalType":"int24","name":"tickUpper","type":"int24"},{"internalType":"uint128","name":"liquidity","type":"uint128"},{"internalType":"uint256","name":"feeGrowthInside0LastX128","type":"uint256"},{"internalType":"uint256","name":"feeGrowthInside1LastX128","type":"uint256"},{"internalType":"uint128","name":"tokensOwed0","type":"uint128"},{"internalType":"uint128","name":"tokensOwed1","type":"uint128"}],"stateMutability":"view","type":"function"},
  {"inputs":[{"internalType":"address","name":"owner","type":"address"},{"internalType":"uint256","name":"index","type":"uint256"}],"name":"tokenOfOwnerByIndex","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"}
];

// Mostri "Glitch"
const monsterTemplates = [
  { hp: 40, damage: 0.5, color: '#ff00ff', name: 'Buffer Error', size: 30, speed: 1.5 },
  { hp: 70, damage: 0.8, color: '#ffff00', name: 'Logic Leak', size: 50, speed: 2.2 },
  { hp: 120, damage: 1.2, color: '#ff4d00', name: 'Root Virus', size: 75, speed: 3 }
];

let activeMonsters = [];
let cameraX = 0;

const shortAddress = computed(() => {
  if (!walletAddress.value) return '';
  return walletAddress.value.slice(0, 6) + '...' + walletAddress.value.slice(-4);
});

const connectWallet = async () => {
  if (window.ethereum) {
    try {
      console.log("Requesting wallet connection...");
      const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
      console.log("Accounts:", accounts);
      if (accounts.length > 0) {
        walletAddress.value = accounts[0];
        startGame();
      } else {
        errorMsg.value = "No accounts found.";
      }
    } catch (error) {
      console.error("Wallet connection error:", error);
      errorMsg.value = error.message || "Connection Failed.";
    }
  } else {
    errorMsg.value = "MetaMask not found!";
  }
};

const checkUniswapLPs = async (ownerAddress) => {
  console.log("Checking for Uniswap V3 LP positions...");
  try {
    const publicClient = createPublicClient({
      chain: mainnet,
      transport: custom(window.ethereum)
    });

    const positionManager = getContract({
      address: UNISWAP_V3_POSITION_MANAGER_ADDRESS,
      abi: UNISWAP_V3_POSITION_MANAGER_ABI,
      client: publicClient
    });

    const balance = await positionManager.read.balanceOf([ownerAddress]);
    console.log(`Found ${balance} Uniswap V3 LP NFTs.`);

    if (balance > 0n) {
      for (let i = 0; i < Number(balance); i++) {
        const tokenId = await positionManager.read.tokenOfOwnerByIndex([ownerAddress, BigInt(i)]);
        const positionData = await positionManager.read.positions([tokenId]);
        console.log(`--- Position NFT #${tokenId.toString()} ---`);
        console.log({
          token0: positionData[2],
          token1: positionData[3],
          fee: positionData[4].toString(),
          liquidity: positionData[7].toString(),
          tickLower: positionData[5].toString(),
          tickUpper: positionData[6].toString(),
        });
      }
    }
  } catch (error) {
    console.error("Could not fetch Uniswap LP data:", error);
  }
};

const startGame = async () => {
  if (!walletAddress.value) return;

  gameState.value = 'playing';
  player.value.hp = 100;
  player.value.x = 50;
  cameraX = 0;

  await checkUniswapLPs(walletAddress.value);

  activeMonsters = monsterTemplates.map((tpl, index) => ({
    ...tpl,
    x: 700 + (index * 800),
    y: GROUND_Y - tpl.size,
    currentHp: tpl.hp,
    alive: true,
    flash: 0
  }));

  await nextTick();
  if (gameCanvas.value) {
    ctx = gameCanvas.value.getContext('2d');
    loop();
  }
};

const resetGame = () => {
  cancelAnimationFrame(animationFrameId);
  startGame();
};

const keys = {};
const handleKeyDown = (e) => {
  keys[e.code] = true;
  if (e.code === 'Space' && gameState.value === 'playing') attack();
};
const handleKeyUp = (e) => keys[e.code] = false;

window.addEventListener('keydown', handleKeyDown);
window.addEventListener('keyup', handleKeyUp);

const attack = () => {
  if (player.value.attackCooldown > 0) return;
  player.value.attacking = true;
  player.value.attackCooldown = 15;

  const attackRange = 70;
  const attackX = player.value.facingRight ? player.value.x : player.value.x - attackRange;

  activeMonsters.forEach(m => {
    if (!m.alive) return;
    if (Math.abs((player.value.x + player.value.width/2) - (m.x + m.size/2)) < attackRange + m.size/2) {
      m.currentHp -= 15;
      m.flash = 5;
      if (m.currentHp <= 0) {
        m.alive = false;
        if (activeMonsters.every(mon => !mon.alive)) {
          setTimeout(() => gameState.value = 'won', 800);
        }
      }
    }
  });
  setTimeout(() => player.value.attacking = false, 150);
};

const update = () => {
  if (gameState.value !== 'playing') return;
  const p = player.value;

  if (keys['ArrowRight']) { p.vx = p.speed; p.facingRight = true; }
  else if (keys['ArrowLeft']) { p.vx = -p.speed; p.facingRight = false; }
  else { p.vx *= 0.8; }

  if (keys['ArrowUp'] && p.y + p.height >= GROUND_Y) p.vy = p.jumpStrength;

  p.vy += GRAVITY;
  p.x += p.vx;
  p.y += p.vy;

  if (p.y + p.height > GROUND_Y) { p.y = GROUND_Y - p.height; p.vy = 0; }
  if (p.x < 0) p.x = 0;
  if (p.x > cameraX + 400) cameraX = p.x - 400;

  activeMonsters.forEach(m => {
    if (!m.alive) return;
    const dist = p.x - m.x;
    if (Math.abs(dist) < 500) m.x += dist > 0 ? m.speed : -m.speed;

    if (Math.abs(p.x - m.x) < 30 && Math.abs(p.y - m.y) < 30) {
      p.hp -= m.damage;
      if (p.hp <= 0) gameState.value = 'gameover';
    }
    if (m.flash > 0) m.flash--;
  });

  if (p.attackCooldown > 0) p.attackCooldown--;
};

const draw = () => {
  if (!ctx) return;
  ctx.fillStyle = '#0a0a0f';
  ctx.fillRect(0, 0, 800, 450);

  // Background Grid
  ctx.strokeStyle = '#1a1a2e';
  for(let i=0; i<800; i+=40) {
    ctx.beginPath(); ctx.moveTo(i - (cameraX % 40), 0); ctx.lineTo(i - (cameraX % 40), 450); ctx.stroke();
  }

  ctx.save();
  ctx.translate(-cameraX, 0);

  // Floor
  ctx.fillStyle = '#16213e';
  ctx.fillRect(cameraX, GROUND_Y, 800, 70);

  // Monsters
  activeMonsters.forEach(m => {
    if (!m.alive) return;
    ctx.fillStyle = m.flash > 0 ? '#fff' : m.color;
    ctx.fillRect(m.x, m.y, m.size, m.size);
    // Glitch effect (random offset)
    if (Math.random() > 0.8) {
        ctx.fillRect(m.x + Math.random()*10 - 5, m.y, m.size, 2);
    }
  });

  // Player
  const p = player.value;
  ctx.fillStyle = p.color;
  ctx.fillRect(p.x, p.y, p.width, p.height);
  if (p.attacking) {
    ctx.fillStyle = '#fff';
    const swordX = p.facingRight ? p.x + p.width : p.x - 40;
    ctx.fillRect(swordX, p.y + 10, 40, 10);
  }

  ctx.restore();
};

const loop = () => {
  update();
  draw();
  animationFrameId = requestAnimationFrame(loop);
};

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
  cancelAnimationFrame(animationFrameId);
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

.game-container {
  display: flex; justify-content: center; align-items: center;
  height: 100vh; background: #000; font-family: 'Press Start 2P', cursive;
}

.screen {
  position: absolute; width: 800px; height: 450px;
  background: rgba(5, 5, 20, 0.9);
  display: flex; flex-direction: column; justify-content: center; align-items: center;
  z-index: 10; border: 4px solid #00f2ff;
}

.title { color: #00f2ff; font-size: 42px; margin: 0; text-shadow: 4px 4px #ff00ff; }
.subtitle-game { color: #fff; font-size: 18px; margin-top: 10px; letter-spacing: 2px; }
.instruction { font-size: 10px; color: #888; margin-top: 40px; }

.btn-pixel {
  background: transparent; border: 3px solid #00f2ff; color: #00f2ff;
  padding: 15px 25px; font-family: 'Press Start 2P'; cursor: pointer;
  margin-top: 20px; transition: 0.2s;
}
.btn-pixel:hover { background: #00f2ff; color: #000; box-shadow: 0 0 15px #00f2ff; }

.hud {
  position: absolute; top: 20px; width: 760px;
  display: flex; justify-content: space-between; z-index: 5; pointer-events: none;
}

.health-bar { width: 200px; height: 15px; border: 2px solid #fff; background: #222; }
.hp-fill { height: 100%; background: #00f2ff; box-shadow: 0 0 10px #00f2ff; }
.hp-text, .wallet-info { font-size: 10px; margin-top: 5px; color: #fff; }

.red-text { color: #ff0055; text-shadow: 2px 2px #000; }
.gold-text { color: #00ffaa; text-shadow: 2px 2px #000; }
</style>