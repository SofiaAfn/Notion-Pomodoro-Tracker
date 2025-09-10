<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Notion Pomodoro Logger</title>
  <style>
    :root {
      --bg: #0f1115;
      --panel: #161a22;
      --text: #eef2ff;
      --muted: #9aa4b2;
      --accent: #6ea8fe;
      --danger: #ff6b6b;
      --ok: #22c55e;
      --ring: rgba(110,168,254,.35);
    }
    * { box-sizing: border-box; }
    body {
      margin: 0;
      font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji","Segoe UI Emoji";
      background: var(--bg);
      color: var(--text);
      display: grid;
      place-items: center;
      min-height: 100vh;
      padding: 18px;
    }
    .card {
      width: 100%;
      max-width: 520px;
      background: var(--panel);
      border: 1px solid #1f2430;
      border-radius: 16px;
      padding: 18px;
      box-shadow: 0 10px 30px rgba(0,0,0,.35);
    }
    h1 {
      margin: 0 0 6px;
      font-size: 18px;
      font-weight: 700;
      letter-spacing: .2px;
    }
    p.sub {
      margin: 0 0 14px;
      color: var(--muted);
      font-size: 13px;
    }
    label {
      display: block;
      font-size: 12px;
      color: var(--muted);
      margin-bottom: 6px;
    }
    .row { display: grid; gap: 12px; margin-bottom: 14px; }
    select, input[type="text"], input[type="number"] {
      width: 100%;
      background: #0c0f14;
      color: var(--text);
      border: 1px solid #242b36;
      border-radius: 10px;
      padding: 10px 12px;
      outline: none;
      font-size: 14px;
    }
    select:focus, input:focus { box-shadow: 0 0 0 3px var(--ring); }
    .actions { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 6px; }
    button {
      appearance: none;
      border: 1px solid #2a3240;
      color: var(--text);
      background: linear-gradient(180deg,#1b2330,#141923);
      padding: 12px 14px;
      border-radius: 12px;
      font-weight: 600;
      cursor: pointer;
      transition: transform .04s ease, box-shadow .15s ease, opacity .2s ease;
    }
    button:hover { transform: translateY(-1px); }
    button:disabled { opacity: .5; cursor: not-allowed; }
    .start { border-color: #28406c; }
    .stop { border-color: #5b2731; }
    .pill {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      padding: 6px 10px;
      border-radius: 999px;
      font-size: 12px;
      background: #0c131d;
      border: 1px solid #1d2a3f;
      color: var(--muted);
      margin-top: 8px;
    }
    .pill.ok { border-color: #1f4230; color: #a7f3d0; background: #0b1610; }
    .pill.err { border-color: #4a1f25; color: #fecaca; background: #170c0e; }
    .timer {
      display: grid; place-items: center;
      width: 100%;
      margin: 14px 0 6px;
      padding: 10px;
      border-radius: 12px;
      border: 1px dashed #2a3240;
      color: var(--muted);
      font-variant-numeric: tabular-nums;
    }
    .flex { display:flex; gap:10px; align-items:center; }
    .right { text-align:right; }
    small.hint { color: var(--muted); font-size: 12px; }
    .grid2 { display:grid; grid-template-columns: 1fr auto; gap: 10px; align-items: end; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Pomodoro Logger</h1>
    <p class="sub">Select a task and click <b>Start</b>. When done, click <b>Stop</b> to log to Notion via Make.</p>

    <div class="row">
      <label for="task">Task</label>
      <select id="task"></select>
      <small class="hint">Tip: Use page IDs for reliability. You can switch to live task listing via Make (see code: TASKS_API_URL).</small>
    </div>

    <div class="grid2">
      <div class="row">
        <label for="length">Length (mins)</label>
        <input type="number" id="length" min="1" step="1" value="25" />
      </div>
      <div class="row right">
        <small class="hint">Local time: <span id="loc-time"></span></small>
      </div>
    </div>

    <div class="timer"><span id="elapsed">00:00</span></div>

    <div class="actions">
      <button class="start" id="startBtn">Start</button>
      <button class="stop" id="stopBtn" disabled>Stop & Log</button>
    </div>

    <div id="status" class="pill" style="display:none;"></div>

    <p class="sub" style="margin-top:14px;">
      <span class="flex">
        <small class="hint">Set <code>WEBHOOK_URL</code> & <code>SHARED_SECRET</code> in the code before deploying.</small>
      </span>
    </p>
  </div>

<script>
/** ========== CONFIG ==========
 * Replace with your Make.com webhook URL and a shared secret.
 * Optionally set TASKS to your favorite tasks (name + pageId).
 * Or point TASKS_API_URL to a Make endpoint that returns live tasks:
 *   GET should return JSON: [{ "name": "Task A", "pageId": "xxxxxxxx..." }, ...]
 */
const WEBHOOK_URL = "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"; // <-- REQUIRED
const SHARED_SECRET = "notion-make-api-key"; // <-- REQUIRED
const TASKS_API_URL = ""; // e.g., "https://hook.make.com/list-tasks" (optional)

// Fallback static tasks (edit or empty array). Use real Notion page IDs.
const TASKS = [
  // { name: "Study RL Ch 5", pageId: "YOUR_NOTION_PAGE_ID" },
  // { name: "Thesis: Results Section", pageId: "YOUR_NOTION_PAGE_ID" },
];

// ========== STATE ==========
let ticker = null;
let startAt = null; // Date object
let countdownTarget = null;

const els = {
  task: document.getElementById("task"),
  length: document.getElementById("length"),
  start: document.getElementById("startBtn"),
  stop: document.getElementById("stopBtn"),
  elapsed: document.getElementById("elapsed"),
  status: document.getElementById("status"),
  locTime: document.getElementById("loc-time")
};

function pad(n){ return String(n).padStart(2,"0"); }
function fmtElapsed(ms){
  const m = Math.floor(ms/60000);
  const s = Math.floor((ms%60000)/1000);
  return `${pad(m)}:${pad(s)}`;
}
function showStatus(text, ok=false){
  els.status.style.display = "inline-flex";
  els.status.textContent = text;
  els.status.className = `pill ${ok ? "ok" : "err"}`;
  setTimeout(()=> { els.status.style.display="none"; }, 3500);
}

function populateTasks(list){
  els.task.innerHTML = "";
  if (!list || list.length === 0){
    const opt = document.createElement("option");
    opt.value = "";
    opt.textContent = "Add tasks in code (TASKS) or via TASKS_API_URL";
    els.task.appendChild(opt);
    return;
    }
  for (const t of list){
    const opt = document.createElement("option");
    opt.value = t.pageId;
    opt.textContent = t.name;
    els.task.appendChild(opt);
  }
}

async function loadTasks(){
  if (TASKS_API_URL){
    try{
      const res = await fetch(TASKS_API_URL, { method: "GET" });
      const data = await res.json();
      populateTasks(data);
      return;
    }catch(e){
      console.warn("TASKS_API_URL fetch failed, falling back to static TASKS.", e);
    }
  }
  populateTasks(TASKS);
}

function restoreIfActive(){
  const saved = JSON.parse(localStorage.getItem("pomodoro_active") || "null");
  if (!saved) return;
  // Only restore within 6 hours to avoid stale sessions
  const then = new Date(saved.startAt);
  const now = new Date();
  if (now - then > 6*60*60*1000) {
    localStorage.removeItem("pomodoro_active");
    return;
  }
  startAt = then;
  countdownTarget = new Date(saved.countdownTarget);
  els.task.value = saved.pageId || "";
  els.length.value = saved.length || 25;
  els.start.disabled = true;
  els.stop.disabled = false;
  startTicker();
}

function startTicker(){
  stopTicker();
  ticker = setInterval(()=>{
    const now = new Date();
    const elapsed = now - startAt;
    els.elapsed.textContent = fmtElapsed(elapsed);
  }, 250);
}

function stopTicker(){
  if (ticker){ clearInterval(ticker); ticker = null; }
}

function nowIso(){ return new Date().toISOString(); }

function saveActiveState(){
  localStorage.setItem("pomodoro_active", JSON.stringify({
    startAt: startAt.toISOString(),
    countdownTarget: countdownTarget ? countdownTarget.toISOString() : null,
    pageId: els.task.value,
    length: Number(els.length.value || 25)
  }));
}

function clearActiveState(){
  localStorage.removeItem("pomodoro_active");
}

els.start.addEventListener("click", () => {
  const pageId = els.task.value;
  if (!WEBHOOK_URL || WEBHOOK_URL.includes("REPLACE_ME")) {
    showStatus("Set WEBHOOK_URL & SHARED_SECRET in code.", false);
    return;
  }
  if (!pageId){
    showStatus("Pick a task first.", false);
    return;
  }
  const length = Math.max(1, Number(els.length.value || 25));
  startAt = new Date();
  countdownTarget = new Date(startAt.getTime() + length*60000);
  els.start.disabled = true;
  els.stop.disabled = false;
  startTicker();
  saveActiveState();
  showStatus("Started. Happy focus!", true);
});

els.stop.addEventListener("click", async () => {
  if (!startAt){
    showStatus("No active session.", false);
    return;
  }
  const endAt = new Date();
  const pageId = els.task.value;
  const name = els.task.options[els.task.selectedIndex]?.text || "";
  const payload = {
    task_page_id: pageId,
    task_name: name,
    start_time: startAt.toISOString(),
    end_time: endAt.toISOString(),
    secret: SHARED_SECRET
  };
  try{
    const res = await fetch(WEBHOOK_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });
    if (!res.ok){
      const txt = await res.text();
      showStatus("Make webhook error: " + txt, false);
      return;
    }
    showStatus("Logged to Notion ✅", true);
    // Reset UI
    stopTicker();
    startAt = null;
    els.elapsed.textContent = "00:00";
    els.start.disabled = false;
    els.stop.disabled = true;
    clearActiveState();
  }catch(e){
    showStatus("Network error. Check WEBHOOK_URL.", false);
  }
});

// clock
setInterval(()=>{
  const d = new Date();
  els.locTime.textContent = d.toLocaleString();
}, 1000);

// init
loadTasks().then(restoreIfActive);
</script>
</body>
</html>
