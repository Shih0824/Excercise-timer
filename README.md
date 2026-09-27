<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Tabata 計時器</title>
<style>
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  body {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    background: #1e1e1e;
    color: #fff;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 20px;
    user-select: none;
  }
  h1 { font-size: 22px; margin: 0 0 10px; }
  .phase { font-size: 28px; margin: 5px 0; }
  .time { font-size: 88px; font-weight: bold; margin: 10px 0; font-variant-numeric: tabular-nums; }
  .round { font-size: 16px; color: #aaa; margin-bottom: 20px; }
  .settings { display: grid; grid-template-columns: auto 1fr; gap: 8px 12px; margin-bottom: 20px; }
  .settings label { font-size: 14px; align-self: center; }
  .settings input {
    width: 80px; padding: 6px; font-size: 16px; text-align: center;
    border-radius: 6px; border: 1px solid #444; background: #2a2a2a; color: #fff;
  }
  .buttons { display: flex; gap: 10px; }
  button {
    padding: 12px 24px; font-size: 16px; border: none; border-radius: 8px;
    color: #fff; font-weight: bold; cursor: pointer;
  }
  #startBtn { background: #4CAF50; }
  #pauseBtn { background: #FFC107; color: #000; }
  #resetBtn { background: #f44336; }
  button:disabled { opacity: 0.4; }
  .music-row { margin-top: 20px; font-size: 12px; color: #888; text-align: center; }
</style>
</head>
<body>

<h1>Tabata 訓練</h1>
<div class="phase" id="phase">準備開始</div>
<div class="time" id="time">00:10</div>
<div class="round" id="round">第 0 / 8 組</div>

<div class="settings">
  <label>準備(秒)</label><input type="number" id="prep" value="10" min="0">
  <label>運動(秒)</label><input type="number" id="work" value="20" min="1">
  <label>休息(秒)</label><input type="number" id="rest" value="10" min="1">
  <label>組數</label><input type="number" id="rounds" value="8" min="1">
</div>

<div class="buttons">
  <button id="startBtn">開始</button>
  <button id="pauseBtn" disabled>暫停</button>
  <button id="resetBtn">重置</button>
</div>

<div class="music-row">
  <label>背景音樂（可選）：<input type="file" id="musicFile" accept="audio/*"></label>
</div>

<audio id="music" loop></audio>

<script>
const phaseEl = document.getElementById('phase');
const timeEl = document.getElementById('time');
const roundEl = document.getElementById('round');
const prepInput = document.getElementById('prep');
const workInput = document.getElementById('work');
const restInput = document.getElementById('rest');
const roundsInput = document.getElementById('rounds');
const startBtn = document.getElementById('startBtn');
const pauseBtn = document.getElementById('pauseBtn');
const resetBtn = document.getElementById('resetBtn');
const musicFile = document.getElementById('musicFile');
const music = document.getElementById('music');

let isRunning = false;
let isPaused = false;
let currentPhase = '準備';
let timeLeft = 0;
let currentRound = 0;
let totalRounds = 8;
let timerInterval = null;

let audioCtx = null;
function beep(freq = 1000, duration = 0.15) {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.frequency.value = freq;
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);
  osc.start();
  osc.stop(audioCtx.currentTime + duration);
}

musicFile.addEventListener('change', (e) => {
  const file = e.target.files[0];
  if (file) {
    music.src = URL.createObjectURL(file);
  }
});

function formatTime(sec) {
  const m = Math.floor(sec / 60).toString().padStart(2, '0');
  const s = (sec % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
}

function updateDisplay() {
  timeEl.textContent = formatTime(timeLeft);
  roundEl.textContent = `第 ${currentRound} / ${totalRounds} 組`;
  if (currentPhase === '運動') {
    phaseEl.textContent = '運動！';
    phaseEl.style.color = '#4CAF50';
    timeEl.style.color = '#4CAF50';
  } else if (currentPhase === '休息') {
    phaseEl.textContent = '休息';
    phaseEl.style.color = '#2196F3';
    timeEl.style.color = '#2196F3';
  } else {
    phaseEl.textContent = currentPhase === '完成' ? '完成！' : '準備';
    phaseEl.style.color = '#FFC107';
    timeEl.style.color = '#FFC107';
  }
}

function startTimer() {
  if (isRunning) return;
  isRunning = true;
  isPaused = false;
  startBtn.disabled = true;
  pauseBtn.disabled = false;
  resetBtn.disabled = false; // Keep reset enabled or adjust as preferred

  const prep = parseInt(prepInput.value) || 10;
  const work = parseInt(workInput.value) || 20;
  const rest = parseInt(restInput.value) || 10;
  totalRounds = parseInt(roundsInput.value) || 8;

  currentPhase = '準備';
  timeLeft = prep;
  currentRound = 0;
  updateDisplay();

  if (music.src) {
    music.play().catch(() => {});
  }

  timerInterval = setInterval(() => {
    if (isPaused) return;

    if (timeLeft <= 3 && timeLeft > 0) {
      beep(1200, 0.1);
    }

    if (timeLeft > 0) {
      timeLeft--;
      updateDisplay();
    } else {
      if (currentPhase === '準備') {
        currentPhase = '運動';
        timeLeft = work;
        currentRound = 1;
      } else if (currentPhase === '運動') {
        if (currentRound >= totalRounds) {
          currentPhase = '完成';
          updateDisplay();
          clearInterval(timerInterval);
          stopAll();
          alert('訓練結束！辛苦了！');
          return;
        }
        currentPhase = '休息';
        timeLeft = rest;
      } else if (currentPhase === '休息') {
        currentPhase = '運動';
        timeLeft = work;
        currentRound++;
      }
      beep(800, 0.3);
      updateDisplay();
    }
  }, 1000);
}

function pauseTimer() {
  isPaused = !isPaused;
  pauseBtn.textContent = isPaused ? '繼續' : '暫停';
  if (isPaused) {
    music.pause();
  } else {
    music.play().catch(() => {});
  }
}

function resetTimer() {
  clearInterval(timerInterval);
  timerInterval = null;
  isRunning = false;
  isPaused = false;
  currentPhase = '準備';
  timeLeft = parseInt(prepInput.value) || 10;
  currentRound = 0;
  updateDisplay();
  startBtn.disabled = false;
  pauseBtn.disabled = true;
  pauseBtn.textContent = '暫停';
  resetBtn.disabled = false;
  music.pause();
  music.currentTime = 0;
}

function stopAll() {
  isRunning = false;
  startBtn.disabled = false;
  pauseBtn.disabled = true;
  resetBtn.disabled = false;
  music.pause();
}

startBtn.addEventListener('click', startTimer);
pauseBtn.addEventListener('click', pauseTimer);
resetBtn.addEventListener('click', resetTimer);

updateDisplay();
</script>

</body>
</html>
