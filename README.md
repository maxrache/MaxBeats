<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Kasipro Touch ULTRA MASTER</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <link rel="manifest" href="manifest.json" />
  <script src="https://cdn.jsdelivr.net/npm/tone@next"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
  <style>
    body {
      background: #111; color: #0ff; font-family: monospace;
      margin: 0; padding: 1rem; text-align: center;
    }
    h1 { color: #ff00cc; margin: 0.5rem; }
    .grid { display: grid; grid-template-columns: repeat(16, 1fr); gap: 2px; }
    .step { height: 28px; background: #222; border-radius: 6px; }
    .step.active { background: #0ff; }
    .step.current { border: 2px solid #fff; }
    .pad-row, .controls, .panel, .download-list { margin: 1rem auto; max-width: 800px; }
    button, select, input[type="file"], input[type="range"] {
      font-family: monospace; font-size: 1rem; margin: 0.4rem; padding: 0.6rem;
      border-radius: 6px; border: none; background: #0ff; color: #000;
    }
    audio, canvas { margin-top: 1rem; width: 100%; }
  </style>
</head>
<body>
  <h1>Kasipro Touch ULTRA MASTER</h1>

  <!-- Live Scene Buttons -->
  <div class="controls">
    <button onclick="launchScene('A')">Scene A</button>
    <button onclick="launchScene('B')">Scene B</button>
    <button onclick="launchScene('C')">Scene C</button>
  </div>

  <!-- Soundpack Upload + Selector -->
  <div class="controls">
    <select id="kitSelector">
      <option value="lofi">LoFi Kit</option>
      <option value="trap">Trap Kit</option>
      <option value="custom">Custom WAV Upload</option>
    </select>
    <input type="file" id="customUpload" accept=".wav" multiple style="display:none" />
  </div>

  <!-- Sequencer -->
  <div id="sequencerGrid"></div>

  <!-- Main Controls -->
  <div class="controls">
    <button onclick="start()">Start</button>
    <button onclick="stop()">Stop</button>
    <button onclick="recordMic()">Mic Rec</button>
    <button onclick="stopMic()">Stop + Play</button>
    <button onclick="exportZIP()">Export ZIP</button>
    <button onclick="installPWA()">Install App</button>
  </div>

  <audio id="micPlayer" controls></audio>
  <canvas id="visualizerCanvas" height="80"></canvas>

  <!-- Preview Player -->
  <div class="controls">
    <h2>Preview Sample</h2>
    <input type="file" id="previewInput" accept=".wav" />
    <button onclick="previewSample()">Play Preview</button>
    <audio id="previewAudio" controls></audio>
  </div>

  <!-- Download-Zentrale -->
  <div class="download-list">
    <h3>Download kostenlose Soundpacks:</h3>
    <ul style="text-align: left;">
      <li><a href="https://www.musicradar.com/news/drums/1000-free-drum-samples" target="_blank">MusicRadar – 1000+ Samples</a></li>
      <li><a href="https://soundpacks.io" target="_blank">SoundPacks.io – Free Trap & LoFi Kits</a></li>
      <li><a href="https://99sounds.org" target="_blank">99Sounds – Premium WAV Packs</a></li>
      <li><a href="https://sampleswap.org" target="_blank">SampleSwap – FX + Vintage Kits</a></li>
    </ul>
  </div>

<script>
let deferredPrompt;
window.addEventListener("beforeinstallprompt", e => {
  e.preventDefault(); deferredPrompt = e;
});
function installPWA() {
  if (deferredPrompt) {
    deferredPrompt.prompt();
    deferredPrompt.userChoice.then(() => deferredPrompt = null);
  }
}

const pads = ["kick", "snare", "hat", "clap"];
const stepCount = 16;
const pattern = {};
let currentStep = 0;
const players = {};
const kitUrls = {
  lofi: {
    kick: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/lofi/kick.wav",
    snare: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/lofi/snare.wav",
    hat: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/lofi/hihat.wav",
    clap: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/lofi/clap.wav"
  },
  trap: {
    kick: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/trap/kick.wav",
    snare: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/trap/snare.wav",
    hat: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/trap/hihat.wav",
    clap: "https://cdn.jsdelivr.net/gh/jherr/sfx@main/trap/clap.wav"
  }
};

const grid = document.getElementById("sequencerGrid");

pads.forEach(name => {
  pattern[name] = Array(stepCount).fill(false);
  const row = document.createElement("div");
  row.className = "pad-row";
  for (let i = 0; i < stepCount; i++) {
    const step = document.createElement("div");
    step.className = "step";
    step.onclick = () => {
      pattern[name][i] = !pattern[name][i];
      step.classList.toggle("active");
    };
    row.appendChild(step);
  }
  grid.appendChild(row);
});

kitSelector.onchange = () => {
  if (kitSelector.value === "custom") {
    customUpload.click();
  } else {
    loadKit(kitUrls[kitSelector.value]);
  }
};

customUpload.onchange = () => {
  const files = customUpload.files;
  [...files].forEach((f, i) => {
    if (!pads[i]) return;
    const url = URL.createObjectURL(f);
    players[pads[i]] = new Tone.Player(url).toDestination();
  });
};

function loadKit(kit) {
  pads.forEach(p => {
    players[p] = new Tone.Player(kit[p]).toDestination();
  });
}

Tone.Transport.scheduleRepeat(time => {
  pads.forEach(name => {
    if (pattern[name][currentStep] && players[name]) {
      players[name].start(time);
    }
  });
  highlight(currentStep);
  currentStep = (currentStep + 1) % stepCount;
}, "16n");

function highlight(step) {
  document.querySelectorAll(".pad-row").forEach((row, i) => {
    [...row.children].forEach((el, j) => {
      el.classList.toggle("current", j === step);
    });
  });
}

function start() {
  Tone.start();
  Tone.Transport.start();
}
function stop() {
  Tone.Transport.stop();
  currentStep = 0;
}

let micChunks = [], micStream, mediaRecorder;
function recordMic() {
  navigator.mediaDevices.getUserMedia({ audio: true }).then(stream => {
    micStream = stream;
    mediaRecorder = new MediaRecorder(stream);
    micChunks = [];
    mediaRecorder.ondataavailable = e => micChunks.push(e.data);
    mediaRecorder.onstop = () => {
      micPlayer.src = URL.createObjectURL(new Blob(micChunks, { type: "audio/wav" }));
    };
    mediaRecorder.start();
  });
}
function stopMic() {
  if (mediaRecorder) mediaRecorder.stop();
  if (micStream) micStream.getTracks().forEach(t => t.stop());
}

async function exportZIP() {
  const zip = new JSZip();
  zip.file("pattern.json", JSON.stringify(pattern, null, 2));
  if (micChunks.length > 0) {
    zip.file("recording.wav", new Blob(micChunks, { type: "audio/wav" }));
  }
  const blob = await zip.generateAsync({ type: "blob" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = "kasipro_ultra_session.zip";
  a.click();
}

const analyser = new Tone.Analyser("waveform", 128);
Tone.Destination.connect(analyser);
const canvas = document.getElementById("visualizerCanvas");
const ctx = canvas.getContext("2d");

function draw() {
  requestAnimationFrame(draw);
  const values = analyser.getValue();
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.beginPath();
  values.forEach((v, i) => {
    const x = (i / values.length) * canvas.width;
    const y = 40 + v * 40;
    ctx.lineTo(x, y);
  });
  ctx.strokeStyle = "#0ff";
  ctx.stroke();
}
draw();

function previewSample() {
  const file = document.getElementById("previewInput").files[0];
  if (!file) return;
  const audio = document.getElementById("previewAudio");
  audio.src = URL.createObjectURL(file);
  audio.play();
}

const scenes = {
  A: {
    kick: [1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0],
    snare: [0,0,0,0,1,0,0,0,0,0,0,0,1,0,0,0]
  },
  B: {
    kick: [1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0],
    snare: [0,0,1,0,0,0,1,0,0,0,1,0,0,0,1,0]
  },
  C: {
    kick: [1,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0],
    snare: [0,0,0,0,0,1,0,0,0,0,0,1,0,0,0,0]
  }
};

function launchScene(name) {
  if (!scenes[name]) return;
  pads.forEach(p => {
    pattern[p] = scenes[name][p] || Array(stepCount).fill(false);
  });
  refreshGrid();
}
function refreshGrid() {
  document.querySelectorAll(".pad-row").forEach((row, i) => {
    const pad = pads[i];
    [...row.children].forEach((el, j) => {
      el.classList.toggle("active", !!pattern[pad][j]);
    });
  });
}

if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('service-worker.js');
  });
}
</script>
</body>
</html>
DAW TEST
