<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>변리사 인생 RPG</title>
  <style>
    body { font-family: 'Arial'; background: #111; color: #fff; text-align: center; padding: 20px; }
    input, select, button { font-size: 16px; margin: 5px; padding: 6px 10px; }
    .section { background: #222; margin: 15px auto; padding: 15px; border-radius: 10px; width: 90%; max-width: 500px; }
    .monster-img { height: 100px; margin: 10px; }
    .progress-bar { height: 15px; background: #444; border-radius: 5px; overflow: hidden; margin-top: 10px; }
    .progress-fill { height: 100%; background: limegreen; width: 0%; transition: width 0.3s; }
    #log { background: #111; margin-top: 15px; padding: 10px; text-align: left; white-space: pre-line; border-left: 4px solid #0f0; }
  </style>
</head>
<body>

<h1>📚 변리사 인생 RPG</h1>

<div class="section">
  <label>사용자 선택: 
    <select id="userSelect" onchange="switchUser(this.value)">
      <option value="A">사용자 A</option>
      <option value="B">사용자 B</option>
      <option value="C">사용자 C</option>
    </select>
  </label>
</div>

<div class="section" id="status">
  <h2>상태창</h2>
  <p>레벨: <span id="level">1</span></p>
  <p>XP: <span id="xp">0</span> / <span id="nextXp">100</span></p>
  <div class="progress-bar"><div class="progress-fill" id="xpBar"></div></div>
  <p>골드: <span id="gold">0</span> G</p>
  <p>오늘 획득: <span id="todayXp">0</span> XP / <span id="todayGold">0</span> G</p>
</div>

<div class="section">
  <select id="subject">
    <option value="civil">민법</option>
    <option value="chem">화학</option>
    <option value="procedure">민사소송법</option>
  </select>
  <input type="number" id="lectures" placeholder="강의 수 (강)">
  <button onclick="completeStudy(false)">수강 완료</button><br>
  <input type="number" id="selfstudy" placeholder="자습 시간 (분)">
  <button onclick="completeStudy(true)">자습 완료</button>
</div>

<div class="section" id="mission">
  <h2>🎯 오늘의 미션</h2>
  <input id="missionLecture" type="number" placeholder="목표 강의 수">
  <input id="missionStudy" type="number" placeholder="목표 자습 시간">
  <button onclick="setMission()">설정</button>
  <p id="missionStatus">설정된 목표 없음</p>
</div>

<div class="section">
  <h2>🎉 보상 사용</h2>
  <button onclick="useGold(100,'밥약')">🍽 밥약 (100G)</button>
  <button onclick="useGold(120,'노래방')">🎤 노래방 (120G)</button>
  <button onclick="useGold(150,'축구')">⚽ 축구 (150G)</button>
  <button onclick="useGold(200,'풋살')">🥅 풋살 (200G)</button>
  <button onclick="useGold(80,'낮잠')">😴 낮잠 (80G)</button>
  <button onclick="useGold(500,'하루 쉬기')">🛌 하루 휴식 (500G)</button>
</div>

<div id="log"></div>

<script>
let user = "A";
let users = { A: {}, B: {}, C: {} };

const subjects = {
  civil: { xp: 2.5, gold: 5 },
  chem: { xp: 3.8, gold: 7 },
  procedure: { xp: 1.85, gold: 4 }
};

function levelCurve(n) {
  return 100 + (n - 1) * 20;
}

function loadUser() {
  const saved = localStorage.getItem("rpg_" + user);
  if (saved) users[user] = JSON.parse(saved);
  else users[user] = { xp: 0, gold: 0, level: 1, todayXp: 0, todayGold: 0, mission: {} };
}

function saveUser() {
  localStorage.setItem("rpg_" + user, JSON.stringify(users[user]));
}

function switchUser(u) {
  user = u;
  loadUser();
  updateDisplay();
  log(`👤 [${user}] 계정으로 전환`);
}

function completeStudy(isSelf) {
  let count = 0;
  let sub = document.getElementById("subject").value;
  if (isSelf) {
    const min = parseInt(document.getElementById("selfstudy").value);
    if (!min || min <= 0) return alert("시간 입력");
    count = min / 30;
  } else {
    count = parseInt(document.getElementById("lectures").value);
    if (!count || count <= 0) return alert("강의 수 입력");
  }
  const mult = isSelf ? 0.5 : 1;
  const gainedXp = subjects[sub].xp * count * mult;
  const gainedGold = subjects[sub].gold * count * mult;
  let u = users[user];
  u.xp += gainedXp;
  u.gold += gainedGold;
  u.todayXp = (u.todayXp || 0) + gainedXp;
  u.todayGold = (u.todayGold || 0) + gainedGold;

  while (u.xp >= levelCurve(u.level)) {
    u.xp -= levelCurve(u.level);
    u.level++;
  }

  checkMission();
  saveUser();
  updateDisplay();
  log(`📚 ${isSelf ? "자습" : "수강"} 완료 → +${gainedXp.toFixed(1)} XP, +${gainedGold.toFixed(1)} G`);
}

function useGold(cost, name) {
  let u = users[user];
  if (u.gold < cost) return alert("골드 부족");
  u.gold -= cost;
  saveUser();
  updateDisplay();
  log(`🎉 [${name}] 보상 사용 (-${cost}G)`);
}

function setMission() {
  const lec = parseInt(document.getElementById("missionLecture").value);
  const stu = parseInt(document.getElementById("missionStudy").value);
  users[user].mission = { lecture: lec || 0, study: stu || 0, done: false };
  saveUser();
  checkMission();
  log(`🎯 미션 설정 완료`);
}

function checkMission() {
  const u = users[user];
  const mission = u.mission || {};
  const done = (mission.lecture ? u.todayXp >= mission.lecture * 2.5 : true) &&
               (mission.study ? u.todayXp >= mission.study / 30 * 2.5 * 0.5 : true);
  if (done && !mission.done) {
    u.gold += 50;
    u.mission.done = true;
    log("🎉 미션 완료! 보너스 50G 지급");
  }
  saveUser();
  updateDisplay();
}

function updateDisplay() {
  const u = users[user];
  document.getElementById("level").textContent = u.level;
  document.getElementById("xp").textContent = u.xp.toFixed(1);
  document.getElementById("nextXp").textContent = levelCurve(u.level).toFixed(0);
  document.getElementById("gold").textContent = u.gold.toFixed(0);
  document.getElementById("todayXp").textContent = u.todayXp?.toFixed(1) || "0";
  document.getElementById("todayGold").textContent = u.todayGold?.toFixed(1) || "0";
  const fill = Math.min(100, u.xp / levelCurve(u.level) * 100);
  document.getElementById("xpBar").style.width = fill + "%";

  const mission = u.mission || {};
  document.getElementById("missionStatus").textContent = mission.lecture || mission.study
    ? `목표: 강의 ${mission.lecture || 0} / 자습 ${mission.study || 0}분 (${mission.done ? "완료됨 ✅" : "진행중"})`
    : "설정된 목표 없음";
}

function log(msg) {
  const time = new Date().toLocaleTimeString();
  const logDiv = document.getElementById("log");
  logDiv.textContent = `[${time}] ${msg}\n` + logDiv.textContent;
}

switchUser("A");
</script>

</body>
</html>
