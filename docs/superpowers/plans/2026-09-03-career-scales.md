# 生涯测评接入可作答量表（首轮 MBTI / 霍兰德SDS / 舒伯WVI）实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `zhijian-app.html` 中的「性格测评」「兴趣测评」「价值观测评」三个占位入口，改成"量表选择页 → 一屏一题作答器 → 结果报告页"两级流程，首轮内置 MBTI、霍兰德 SDS、舒伯 WVI 三套可作答量表。

**Architecture:** 数据驱动的通用测评引擎，单一文件内扩展。复用现有 `.prediction-overlay`（全屏滑入容器）承载新增的三个 overlay；新增少量 `qz-*` 样式；量表数据、计分（`scoring`）与报告渲染（`renderer`）以统一 `SCALES` 结构声明，作答器只依赖该结构。完成结果写入 `localStorage`，并把测评页静态「测评报告」卡片区改成动态列表。

**Tech Stack:** 原生 HTML/CSS/JS（无构建链、无依赖、单文件）。验证用浏览器/Playwright 实测 + 计分手算抽查（spec §7）。设计文档见 `docs/superpowers/specs/2026-09-03-career-scales-design.md`。

---

## 0. 契约（全局，后续任务引用）

**改动文件（唯一目标文件）：** `zhijian-app.html`

**插入锚点：**
| 位置 | 锚点（当前行号，可能随编辑偏移） |
|---|---|
| CSS | `</style>` 之前（~L692） |
| 新 overlay HTML | `</div>`(timeechoPage 闭合, ~L2107) 之后、`<!-- Toast -->`(L2109) 之前 |
| 入口接线 | 首页 grid: `showToast('性格测评')` ~L758 / 兴趣 ~L764 / 价值观 ~L770；测评页 feat-card ~L998 / ~L1008 / ~L1018 |
| 测评报告区 | `page-assess` 内 `.section-title "测评报告"` + 静态卡片 ~L1088-1097 |
| JS（引擎+题库） | `<script>` IIFE 内、`setTimeout(openVersionDialog,500);`(~L2331) 之前 |

**新增 JS 全局（IIFE 内 `window.*`，命名空间 `qz*` 前缀）：**
- `window.openScales(cat)` / `window.closeScales()`：量表选择层开合；`cat ∈ 'personality'|'interest'|'value'`。
- `window.openQuiz(id)` / `window.qzAnswer(optIdx)` / `window.qzPrev()` / `window.qzNext()` / `window.qzSubmit()` / `window.qzClose()` / `window.qzRestart()`：作答器。
- `window.closeQuizResult()`：结果页关闭（回量表选择层）。
- `window.renderAssessReports()`：渲染测评页报告列表（内部函数，openAssess 时调用）。

**作答会话状态（内存）** `qzSession = { id, qs:[...], answers:{}, idx:0 }`。`answers[qIndex] = 选项下标`。

**量表数据统一结构：**
```js
{
  id:'mbti'|'holland'|'wvi', cat:'personality'|'interest'|'value',
  name:'…', short:'…', color:'fi-orange'|'fi-blue'|'fi-green', icon:'<svg…>',
  desc:'…', minutes:8,
  questions:[ { text:'题干', opts:['…','…'] } ],   // 全单选；无 dim 字段，维度隐含在题序
  scoring: function(answers){ /* 纯函数：answers->{dims:{…}, …} */ },
  renderer: function(res){ /* -> result 渲染对象（见下） */ },
  metaOf: function(res){ return {summary:'…', meta:'…'} }  // 供报告列表
}
```

**结果渲染对象契约（renderer 返回）：**
```js
{
  primary:'主结果文字(如 INTP)', primarySub:'…', badge:'…',
  bars:[ {name, a, b, aLabel, bLabel} ],          // 成对维度条
  list:[ {title, value, note} ],                  // 或六型/十五维列表
  sections:[ {title, html} ],                     // 逐块解读（内联样式字符串）
  advice:[ '…' ]
}
```
渲染到结果页时统一用现有 `te-skill-bar*`/`te-anchor-tag`/卡片样式（实现时参照 `careerResultPage` 与 `timeechoPage` 内真实类名，如 `te-skill-bar-fill`）。

**报告记录（localStorage，key `zj_assess_reports`，数组，倒序）：**
`{ id, scaleId, name, summary, meta, doneAt:'YYYY-MM-DD' }`。`summary`：MBTI=4字母码；SDS=Top3 字母码；WVI=Top3 价值观名（用 `、` 连接）。

**必答/边界行为：** 未答当前题点“下一题/提交”→ `showToast('请先作答本题')` 并阻止；全答完才可提交；上一题可改答覆盖；中途关闭（返回）丢弃会话，不落库；“重新测试”重置该套。

---

## 1. 文件结构

唯一被修改的文件：`zhijian-app.html`。新增内容四块（CSS、三个 overlay HTML、入口接线/测评报告区、引擎与题库 JS）。不新建其它源码文件（保持单文件便携特性）；设计/计划文档在 `docs/superpowers/` 下已建。

---

## 2. 任务列表

### Task 1: 三个 overlay 骨架 + 入口接线（选择层可开合）

**Files:** `zhijian-app.html`（CSS 锚点 L692 前；HTML 锚点 L2107 后；JS IIFE 内）

- [ ] **Step 1: 追加 CSS（`</style>` 前）**，新增少量 `qz-*` 样式：

```css
/* ==== Quiz Engine (通用测评) ==== */
.qz-cat-desc { font-size: 13px; color: #666; line-height: 1.6; background:#fff; border-radius:12px; padding:12px 14px; margin-bottom:12px; }
.qz-progress-row { display:flex; align-items:center; gap:10px; padding:12px 16px; background:#fff; border-bottom:0.5px solid #E5E5E5; }
.qz-progress-track { flex:1; height:6px; background:#EFEFEF; border-radius:3px; overflow:hidden; }
.qz-progress-fill { height:100%; background:#1a1a1a; border-radius:3px; transition:width .25s; }
.qz-progress-text { font-size:11px; color:#999; flex-shrink:0; }
.qz-question-num { font-size:12px; color:#5B6EF5; font-weight:600; margin-bottom:6px; }
.qz-question-text { font-size:16px; font-weight:600; color:#1a1a1a; line-height:1.6; }
.qz-opt { display:flex; align-items:center; gap:10px; width:100%; text-align:left; background:#fff; border:1px solid #E5E5E5; border-radius:10px; padding:13px 14px; margin-top:10px; font-size:14px; color:#333; line-height:1.5; cursor:pointer; }
.qz-opt:active { background:#F7F7F7; }
.qz-opt.active { border-color:#1a1a1a; background:#1a1a1a; color:#fff; font-weight:500; }
.qz-opt .qz-opt-mark { width:18px; height:18px; border:1.5px solid #ccc; border-radius:50%; flex-shrink:0; box-sizing:border-box; }
.qz-opt.active .qz-opt-mark { border-color:#fff; background: radial-gradient(circle at center, #fff 45%, transparent 50%); }
.qz-foot { display:flex; gap:10px; padding:12px 16px; background:#fff; border-top:0.5px solid #E5E5E5; flex-shrink:0; }
.qz-btn { flex:1; text-align:center; border-radius:22px; font-size:15px; font-weight:600; padding:11px 0; cursor:pointer; }
.qz-btn.ghost { background:#F2F2F2; color:#333; }
.qz-btn.primary { background:#1a1a1a; color:#fff; }
.qz-btn.primary:active,.qz-btn.ghost:active { opacity:.8; }
.qz-btn.disabled, .qz-btn[disabled] { opacity:.35; pointer-events:none; }
```

- [ ] **Step 2: 在 HTML 锚点追加三个 overlay 骨架**（占位内容由后续任务填充；quizPage/resultPage 内部先留空容器）：

```html
  <!-- ===== 量表选择层（通用测评）===== -->
  <div class="prediction-overlay" id="qzScalePage">
    <div class="status-bar"><span class="time">9:41</span><div class="icons"><svg viewBox="0 0 24 24" fill="currentColor"><path d="M1 9l2 2c4.97-4.97 13.03-4.97 18 0l2-2C16.93 2.93 7.08 2.93 1 9zm8 8l3 3 3-3c-1.65-1.66-4.34-1.66-6 0zm-4-4l2 2c2.76-2.76 7.24-2.76 10 0l2-2C15.14 9.14 8.87 9.14 5 13z"/></svg><div class="battery"><div class="battery-fill"></div></div></div></div>
    <div class="pred-header">
      <div class="pred-header-back" onclick="qzCloseScales()"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="15 18 9 12 15 6"/></svg></div>
      <span class="pred-header-title" id="qzScaleTitle">测评</span>
      <div class="pred-header-close" style="opacity:0;pointer-events:none;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18"/><path d="M6 6l12 12"/></svg></div>
    </div>
    <div class="pred-body" id="qzScaleBody"></div>
  </div>

  <!-- ===== 逐题作答器（通用测评）===== -->
  <div class="prediction-overlay" id="qzQuizPage">
    <div class="status-bar"><span class="time">9:41</span><div class="icons"><svg viewBox="0 0 24 24" fill="currentColor"><path d="M1 9l2 2c4.97-4.97 13.03-4.97 18 0l2-2C16.93 2.93 7.08 2.93 1 9zm8 8l3 3 3-3c-1.65-1.66-4.34-1.66-6 0zm-4-4l2 2c2.76-2.76 7.24-2.76 10 0l2-2C15.14 9.14 8.87 9.14 5 13z"/></svg><div class="battery"><div class="battery-fill"></div></div></div></div>
    <div class="pred-header">
      <div class="pred-header-back" onclick="qzClose()"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="15 18 9 12 15 6"/></svg></div>
      <span class="pred-header-title" id="qzQuizTitle">测评</span>
      <div class="pred-header-close" style="opacity:0;pointer-events:none;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18"/><path d="M6 6l12 12"/></svg></div>
    </div>
    <div class="qz-progress-row">
      <div class="qz-progress-track"><div class="qz-progress-fill" id="qzProgressFill"></div></div>
      <div class="qz-progress-text" id="qzProgressText"></div>
    </div>
    <div class="pred-body" id="qzQuizBody"></div>
    <div class="qz-foot">
      <div class="qz-btn ghost" id="qzPrevBtn" onclick="qzPrev()">上一题</div>
      <div class="qz-btn primary" id="qzNextBtn" onclick="qzNext()">下一题</div>
    </div>
  </div>

  <!-- ===== 结果报告页（通用测评）===== -->
  <div class="timeecho-overlay" id="qzResultPage">
    <div class="status-bar"><span class="time">9:41</span><div class="icons"><svg viewBox="0 0 24 24" fill="currentColor"><path d="M1 9l2 2c4.97-4.97 13.03-4.97 18 0l2-2C16.93 2.93 7.08 2.93 1 9zm8 8l3 3 3-3c-1.65-1.66-4.34-1.66-6 0zm-4-4l2 2c2.76-2.76 7.24-2.76 10 0l2-2C15.14 9.14 8.87 9.14 5 13z"/></svg><div class="battery"><div class="battery-fill"></div></div></div></div>
    <div class="pred-header">
      <div class="pred-header-back" onclick="qzCloseResult()"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="15 18 9 12 15 6"/></svg></div>
      <span class="pred-header-title" id="qzResultTitle">测评结果</span>
      <div class="pred-header-close" style="opacity:0;pointer-events:none;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M18 6 6 18"/><path d="M6 6l12 12"/></svg></div>
    </div>
    <div class="pred-body" id="qzResultBody"></div>
    <div class="qz-foot">
      <div class="qz-btn ghost" onclick="qzRestart()">重新测试</div>
      <div class="qz-btn primary" onclick="qzCloseResult()">完成</div>
    </div>
  </div>
```

- [ ] **Step 3: 替换 6 处入口 onclick**（首页 grid 三处 + 测评页三处）：
  - `showToast('性格测评')` → `openScales('personality')`
  - `showToast('兴趣测评')` → `openScales('interest')`
  - `showToast('价值观测评')` → `openScales('value')`
  （仅替换这三对字符串；情商/领导力等其余 `showToast('…测评')` 不动。）

- [ ] **Step 4: 在 IIFE 内、`setTimeout(openVersionDialog,500);` 前追加引擎开头**（含分类定义与一个临时最小量表桩，供本任务验证开合；后续任务逐步替换）：

```js
  // ================= 通用测评引擎 =================
  var QZ_CATS = {
    personality: { name: '性格测评', tagline: '探索你的性格特质、偏好与行为倾向，找到更自洽的成长与职业方向。' },
    interest:    { name: '兴趣测评', tagline: '发现你真正享受的活动领域，定位与之匹配的职业方向。' },
    value:       { name: '价值观测评', tagline: '看清驱动你的核心价值，找到内在认同的工作与生活重心。' }
  };
  var SCALES = [];
  function qzScaleById(id) { for (var i=0;i<SCALES.length;i++){ if(SCALES[i].id===id) return SCALES[i]; } return null; }

  window.openScales = function(cat) {
    var meta = QZ_CATS[cat]; if (!meta) return;
    document.getElementById('qzScaleTitle').textContent = meta.name;
    var body = document.getElementById('qzScaleBody');
    var html = '<div class="qz-cat-desc">' + meta.tagline + '</div>';
    var list = SCALES.filter(function(s){ return s.cat === cat; });
    if (!list.length) { html += '<div class="card" style="text-align:center;color:#999;font-size:13px;">量表整理中，敬请期待</div>'; }
    list.forEach(function(s){
      html += '<div class="feat-card" onclick="openQuiz(\'' + s.id + '\')">' +
        '<div class="feat-card-icon ' + s.color + '">' + s.icon + '</div>' +
        '<div class="feat-card-body"><div class="feat-card-title">' + s.name + '</div>' +
        '<div class="feat-card-desc">' + s.desc + '</div></div>' +
        '<div class="feat-card-arrow"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"/></svg></div>' +
        '</div>';
    });
    body.innerHTML = html;
    document.getElementById('qzScalePage').classList.add('show');
    document.querySelector('.content-area').scrollTop = 0;
  };
  window.qzCloseScales = function() {
    document.getElementById('qzScalePage').classList.remove('show');
  };

  // 临时桩：让 openQuiz 有可调用目标（Task2 起替换为真实作答器）
  window.openQuiz = function(id){ showToast('作答器开发中'); };
  window.qzCloseResult = function(){ document.getElementById('qzResultPage').classList.remove('show'); };
  window.qzRestart = function(){ qzCloseResult(); };
```

- [ ] **Step 5: 浏览器验证**：首页与测评页两处三个入口各自点击 → 弹出对应分类选择层，标题正确（性格测评/兴趣测评/价值观测评），tagline 显示；返回可关闭并回到原页面。其余占位入口仍只弹 Toast。
- [ ] **Step 6: Commit**（本地，不 push）：`git add zhijian-app.html && git commit -m "feat(assess): 量表选择层骨架与入口接线"`

---

### Task 2: 通用逐题作答器（必答/进度/改答/提交）

**Files:** `zhijian-app.html`（JS 内，替换 Task1 的 `openQuiz` 桩，追加作答器函数）

- [ ] **Step 1: 追加作答器核心 JS**（替换 `window.openQuiz = function(id){...}` 桩）：

```js
  var qzSession = null;
  window.openQuiz = function(id) {
    var s = qzScaleById(id); if (!s) return;
    qzSession = { s: s, answers: {}, idx: 0 };
    qzRenderQuestion();
    document.getElementById('qzScalePage').classList.remove('show');
    document.getElementById('qzQuizPage').classList.add('show');
  };
  window.qzClose = function() {        // 中途退出：丢弃会话
    qzSession = null;
    document.getElementById('qzQuizPage').classList.remove('show');
  };
  function qzQ() { return qzSession ? qzSession.s.questions : []; }
  function qzQn() { return qzSession && qzSession.answers[qzSession.idx] != null; }
  function qzRenderQuestion() {
    if (!qzSession) return;
    var s = qzSession.s, qs = s.questions, idx = qzSession.idx, q = qs[idx];
    document.getElementById('qzQuizTitle').textContent = s.name;
    document.getElementById('qzProgressText').textContent = '第 ' + (idx+1) + ' / ' + qs.length + ' 题';
    document.getElementById('qzProgressFill').style.width = ((idx+1)/qs.length*100) + '%';
    var body = document.getElementById('qzQuizBody');
    var html = '<div class="qz-question-num">' + (idx+1) + ' / ' + qs.length + '</div>';
    html += '<div class="qz-question-text">' + q.text + '</div>';
    var sel = qzSession.answers[idx];
    q.opts.forEach(function(o, oi){
      html += '<div class="qz-opt' + (sel===oi?' active':'') + '" onclick="qzAnswer(' + oi + ')">' +
        '<span class="qz-opt-mark"></span><span>' + o + '</span></div>';
    });
    body.innerHTML = html;
    body.scrollTop = 0;
    var prevBtn = document.getElementById('qzPrevBtn');
    var nextBtn = document.getElementById('qzNextBtn');
    prevBtn.classList.toggle('ghost', true);
    prevBtn.classList.toggle('disabled', idx === 0);
    nextBtn.textContent = (idx === qs.length - 1) ? '提交并查看结果' : '下一题';
    nextBtn.onclick = (idx === qs.length - 1) ? qzSubmit : qzNext;
  }
  window.qzAnswer = function(oi) { if (!qzSession) return; qzSession.answers[qzSession.idx] = oi; qzRenderQuestion(); };
  window.qzPrev = function() { if (!qzSession || qzSession.idx === 0) return; qzSession.idx--; qzRenderQuestion(); };
  window.qzNext = function() { if (!qzSession) return; if (!qzQn()) { showToast('请先作答本题'); return; } qzSession.idx++; qzRenderQuestion(); };
  window.qzSubmit = function() {
    if (!qzSession) return;
    var qs = qzSession.s.questions;
    if (!qzQn()) { showToast('请先作答本题'); return; }
    for (var k in qzSession.answers) { if (!qzSession.answers.hasOwnProperty(k)) continue; }
    var answeredAll = true;
    qs.forEach(function(_, i){ if (qzSession.answers[i] == null) answeredAll = false; });
    if (!answeredAll) { showToast('还有 ' + (qs.length - Object.keys(qzSession.answers).length) + ' 题未作答'); return; }
    qzFinish(qzSession.s, qzSession.answers);
  };
  function qzFinish(s, answers) {
    var res = s.scoring(answers);
    var out = s.renderer(res);
    qzSession = null;
    renderQzResult(s, res, out);
    qzSaveReport(s, out);
    document.getElementById('qzQuizPage').classList.remove('show');
    document.getElementById('qzResultPage').classList.add('show');
    document.getElementById('qzResultBody').scrollTop = 0;
  }
  window.qzRestart = function() {
    document.getElementById('qzResultPage').classList.remove('show');
    document.getElementById('qzScalePage').classList.add('show');
  };
  window.qzCloseResult = function() {
    document.getElementById('qzResultPage').classList.remove('show');
    document.getElementById('qzScalePage').classList.add('show');
    if (typeof renderAssessReports === 'function') renderAssessReports();
  };
```

- [ ] **Step 2: 追加渲染与报告落库 JS**（`renderQzResult` 用统一样式把 `out` 画到结果页；样式类名实现时对照 `te-*` 现有类，兜底用内联样式，保证美观一致）：

```js
  function esc(s){ return String(s==null?'':s).replace(/[&<>"]/g,function(c){return{'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c];}); }
  function renderQzResult(s, res, out) {
    document.getElementById('qzResultTitle').textContent = s.name + ' · 测评结果';
    var h = '';
    h += '<div class="card"><div style="font-size:20px;font-weight:800;color:#1a1a1a;' + (out.primaryColor?'color:'+out.primaryColor:'') + '">' + esc(out.primary) + '</div>' +
         (out.primarySub ? '<div style="font-size:13px;color:#666;margin-top:6px;line-height:1.6;">' + esc(out.primarySub) + '</div>' : '') +
         (out.badge ? '<div style="display:inline-block;margin-top:10px;font-size:11px;background:#EEF2FF;color:#5B6EF5;padding:4px 12px;border-radius:12px;">' + esc(out.badge) + '</div>' : '') +
         '</div>';
    (out.bars || []).forEach(function(b){
      var pa = Math.round(b.a/(b.a+b.b)*100) || 50;
      h += '<div class="card">' +
        '<div style="display:flex;justify-content:space-between;align-items:baseline;margin-bottom:8px;">' +
        '<span style="font-size:14px;font-weight:600;color:#1a1a1a;">' + esc(b.name) + '</span>' +
        '<span style="font-size:11px;color:#999;">' + esc(b.aLabel) + ' ' + Math.round(b.a/(b.a+b.b)*100) + '% · ' + Math.round(b.b/(b.a+b.b)*100) + '% ' + esc(b.bLabel) + '</span></div>' +
        '<div style="display:flex;height:8px;border-radius:4px;overflow:hidden;background:#EEF2FF;">' +
        '<div style="width:' + pa + '%;background:#5B6EF5;"></div>' +
        '<div style="width:' + (100-pa) + '%;background:#FFE1D6;"></div></div>' +
        '</div>';
    });
    (out.list || []).forEach(function(it){
      h += '<div class="card"><div style="display:flex;justify-content:space-between;align-items:center;">' +
        '<div><div style="font-size:14px;font-weight:600;color:#1a1a1a;">' + esc(it.title) + '</div>' +
        (it.note ? '<div style="font-size:12px;color:#999;margin-top:3px;">' + esc(it.note) + '</div>' : '') + '</div>' +
        '<span style="font-size:13px;font-weight:700;color:#1a1a1a;">' + esc(it.value) + '</span></div></div>';
    });
    (out.sections || []).forEach(function(sec){
      h += '<div class="card"><div style="font-size:14px;font-weight:700;color:#1a1a1a;margin-bottom:6px;">' + esc(sec.title) + '</div>' + sec.html + '</div>';
    });
    if (out.advice && out.advice.length) {
      var adv = '<div class="card"><div style="font-size:14px;font-weight:700;color:#1a1a1a;margin-bottom:8px;">下一步行动建议</div>';
      out.advice.forEach(function(a){ adv += '<div style="font-size:13px;color:#444;line-height:1.6;padding:6px 0;border-bottom:0.5px solid #F0F0F0;">· ' + esc(a) + '</div>'; });
      adv += '</div>'; h += adv;
    }
    h += '<div style="text-align:center;font-size:11px;color:#bbb;padding:6px 16px 18px;">本结果基于整理版量表，仅供参考，不构成专业测评结论。</div>';
    document.getElementById('qzResultBody').innerHTML = h;
  }
  function qzSaveReport(s, out) {
    try {
      var meta = (typeof s.metaOf === 'function') ? s.metaOf(out) : { summary: out.primary };
      var arr = JSON.parse(localStorage.getItem('zj_assess_reports') || '[]');
      arr.unshift({ scaleId: s.id, name: s.name, summary: meta.summary || '', meta: meta.meta || '', doneAt: (new Date()).toISOString().slice(0,10) });
      if (arr.length > 20) arr.length = 20;
      localStorage.setItem('zj_assess_reports', JSON.stringify(arr));
    } catch(e) {}
  }
  window.qzRestartTmp = null; // no-op guard, remove if unused
```

- [ ] **Step 3: 为验证交互加入临时 fixture**（Task 4 起替换为正式题库；fixture 是 2 题两级维的最小 MBTI 雏形）：

```js
  SCALES.push({
    id:'_fixture', cat:'personality', name:'(临时)MBTI 采样', short:'fixture', color:'fi-orange',
    icon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10"/><path d="M8 14s1.5 2 4 2 4-2 4-2"/><line x1="9" y1="9" x2="9.01" y2="9"/><line x1="15" y1="9" x2="15.01" y2="9"/></svg>',
    desc:'作答器交互验证样例（将移除）', minutes:1,
    questions:[
      { text:'在社交聚会后，你通常感到？（A 更想继续和别人待在一起 / B 需要独处充电）', opts:['A. 更想继续和别人待在一起','B. 需要独处充电'] },
      { text:'做决定时，你更常依据？', opts:['A. 逻辑与分析','B. 价值观与对人的影响'] },
      { text:'你更喜欢？', opts:['A. 有计划、按清单推进','B. 随灵感即兴发挥'] },
      { text:'阅读时，你更关注？', opts:['A. 具体的事实与细节','B. 背后的含义与可能性'] }
    ],
    scoring: function(a){ return { dims:{ EI:a[0]===0?1:-1, TF:a[1]===0?1:-1, JP:a[2]===0?1:-1, NS:a[3]===0?1:-1 } }; },
    renderer: function(res){ return { primary:'样例结果', primarySub:'fixture 用于验证交互，将被真实题库替换。', badge:'整理版', bars:[] }; },
    metaOf: function(){ return { summary:'样例', meta:'' }; }
  });
```

- [ ] **Step 4: 浏览器验证**：进性格→fixture→逐题：未答点“下一题”应 Toast 阻止；作答后可选高亮、能进下一题；上一题回改会覆盖；末题提交 → 结果页出现“样例结果”卡片与脚注；点“完成”→ 回量表选择层。进度条随题号更新。
- [ ] **Step 5: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): 通用逐题作答器与结果渲染骨架"`

---

### Task 3: 测评报告列表动态化（替换静态 INTJ 卡片）

**Files:** `zhijian-app.html`

- [ ] **Step 1: 替换静态报告卡片**（~L1088-1097）。把 `page-assess` 内的 `.section-title "测评报告"` 下方的静态卡片，替换为带 id 的动态容器 + 空态默认：

```html
      <div class="section-title" style="margin-top: 8px;">测评报告</div>
      <div id="assessReportList"></div>
```

- [ ] **Step 2: 追加渲染函数 JS**（放引擎区；`openAssess()` 里加一次调用 `renderAssessReports()`）：

```js
  window.renderAssessReports = function() {
    var box = document.getElementById('assessReportList'); if (!box) return;
    var arr = [];
    try { arr = JSON.parse(localStorage.getItem('zj_assess_reports') || '[]'); } catch(e){}
    if (!arr.length) {
      box.innerHTML = '<div class="card" style="text-align:center;color:#999;font-size:13px;line-height:1.8;">完成测评后，你的报告会显示在这里<br>（重新测评会更新记录）</div>';
      return;
    }
    var h = '';
    arr.slice(0,5).forEach(function(r){
      h += '<div class="card" style="cursor:pointer;" onclick="reopenReport(\'' + r.scaleId + '\')">' +
        '<div style="display:flex;align-items:center;justify-content:space-between;">' +
        '<div><div style="font-size:14px;font-weight:600;color:#1a1a1a;">' + r.name + '</div>' +
        '<div style="font-size:12px;color:#999;margin-top:3px;">' + r.doneAt + ' · ' + (r.summary || '') + '</div></div>' +
        '<span style="font-size:11px;background:#EEF2FF;color:#5B6EF5;padding:3px 10px;border-radius:10px;">再测一次</span>' +
        '</div></div>';
    });
    box.innerHTML = h;
  };
  window.reopenReport = function(scaleId) {
    var s = qzScaleById(scaleId); if (!s) { showToast('该量表已下线'); return; }
    // 直接重新作答该量表
    document.getElementById('qzScalePage').classList.add('show');
    openQuiz(scaleId);
  };
```

- [ ] **Step 3: `openAssess()`（~L2130）函数体末尾追加 `renderAssessReports();`**
- [ ] **Step 4: 浏览器验证**：无记录时测评页该区块显示空态文案；完成 Task2 fixture 提交后回到测评页（切 tab 或重进）→ 出现"（临时)MBTI 采样 · 样例"记录，点“再测一次”进入作答。
- [ ] **Step 5: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): 测评报告列表动态化与空态"`

---

### Task 4: MBTI 正式题库 + 16 型库 + 渲染

**Files:** `zhijian-app.html`（JS 数据区；删除 Task2 的 `_fixture`）

- [ ] **Step 1: 删除 `_fixture` 条目，换成正式 MBTI 量表。** 结构要点：

```js
  SCALES.push({
    id:'mbti', cat:'personality', name:'MBTI 性格类型测评', short:'MBTI', color:'fi-orange',
    icon: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10"/><path d="M8 14s1.5 2 4 2 4-2 4-2"/><line x1="9" y1="9" x2="9.01" y2="9"/><line x1="15" y1="9" x2="15.01" y2="9"/></svg>',
    desc: '4 个维度共 ' + (4*15+4) + ' 题 · 约 8 分钟', minutes: 8,
    questions: [ /* 见 Step 2 */ ],
    scoring: [ /* 见 Step 3 */ ],
    renderer: [ /* 见 Step 4 */ ],
    metaOf: ...
  });
```

- [ ] **Step 2: 编写 64 题题库。** 规则：
  - 维度键 `E/I`、`N/S`、`T/F`、`J/P`。每维 **15 个正反各半的两极陈述题** + 每维 **1 道主观倾向 tie-break 题**（"你在 X 维度上更倾向 A 还是 B？"），共 64 题。
  - 题序：E/I 16 题 → N/S 16 题 → T/F 16 题 → J/P 16 题（便于人工核对）。
  - 每题 `opts` 两个：`['A. …','B. …']`，A 对应左极、B 对应右极。
  - 两极陈述素材（据此扩写不同情境的句子，勿整行重复）：
    - E: 人多时更活跃 / 与人相处获得能量；I: 需要独处充电 / 深谈优于泛谈。
    - N: 关注整体与未来可能 / 喜欢琢磨概念隐喻；S: 关注当下事实细节 / 依赖五感经验与步骤。
    - T: 依据逻辑与原则权衡 / 先看对错公正；F: 看重人际和谐与感受 / 用共情做决定。
    - J: 喜欢计划与确定性 / 先决断再调整；P: 保留弹性随机应变 / 享受探索多种可能。
    - 题干句式示例：`在团队讨论中，我通常…`、`准备旅行时，我更…`、`被问到“你怎么看”时，我倾向于…`、`遇到意见分歧，我先…`。
  - tie-break 题 dim 用左极计 +，格式同普通题，仅计分时作为平分局仲裁（见 Step 3）。

- [ ] **Step 3: scoring**（纯函数；`answers`→`dims`，加 tie-break 规则）：

```js
  scoring: function(a){
    function tally(dim, from, to){ var left=0,right=0; for(var i=from;i<=to;i++){ if(a[i]==null)continue; if(a[i]===0)left++; else if(a[i]===1)right++; } return {left:left,right:right}; }
    function letter(pair, leftLetter, rightLetter, tieIdx){ // tieIdx 为 tie-break 题号
      var t = tally(pair, 0, 15);                 // 每维前 15 题为普通题（按题序连续定位）
      if (t.left === t.right) t = tally(pair, 15, 15);   // 平分→用第 16 题(本维末)主观题仲裁
      // 注：维度内题号实际为连续区间，实现时用维起始偏移，这里以函数参数重构
      return t.left > t.right ? leftLetter : rightLetter;
    }
    ...
  }
```

  **更清晰的可执行方案**：题库四段区间各 16 题：`E/I=0-15、N/S=16-31、T/F=32-47、J/P=48-63`；每段最后 1 题为 tie-break。计分：
  - 普通题计该段 0-14（15 题）的 `left/right` 计数。
  - 若平局，用该段第 15 题（tie-break）方向破平；再平取题号较小方向。
  - 返回 `{ dims:{EI:1|-1, NS:1|-1, TF:1|-1, JP:1|-1}, code:'EINT… 拼接' }`。

- [ ] **Step 4: renderer**：输出 `primary = code + ' · ' + 类型名`（如 `INTJ · 建筑师型`）、`primarySub = 该型一句话画像`、16 型库对象映射（键为四字母码：中文名 + 1-2 句画像 + 适合方向/关键词）、四维 `bars`（E-I / N-S / T-F / J-P 双色条，`aLabel/bLabel` 为两极名）、`sections`（每型一段画像与"职业方向参考"）、`advice` 3-4 条。类型库文案精简演示向。
- [ ] **Step 5: 浏览器验证**：进入 MBTI 逐题作答（抽查必答、进度、末题提交）；提交后 result `primary` 呈现四字母 + 类型名；`bars` 长度 4、双色条宽度与手算一致（用一组已知答案验算 code）；报告记录 summary 为四字母码；"再测一次"进入重新作答。
- [ ] **Step 6: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): MBTI 64题题库与16型报告"`

---

### Task 5: 霍兰德 SDS 题库 + RIASEC 渲染

**Files:** `zhijian-app.html`（JS 数据区追加 `id:'holland'`, `cat:'interest'`）

- [ ] **Step 1: 追加霍兰德量表**：`color:'fi-blue'`（兴趣主题色），icon 沿用首页"兴趣测评"星形 SVG。

```js
  SCALES.push({
    id:'holland', cat:'interest', name:'霍兰德职业兴趣测评', short:'SDS', color:'fi-blue',
    icon: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"/></svg>',
    desc: '6 大兴趣类型共 60 项 · 约 5 分钟', minutes: 5,
    questions: [ /* 见 Step 2 */ ],
    ...
  });
```

- [ ] **Step 2: 编写 60 题题库**：RIASEC 六型各 10 项"你是否喜欢…"（喜欢 A / 不喜欢 B），`opts:['A. 喜欢','B. 不喜欢']`。题序按 R,I,A,S,E,C 各 10 题连续。活动素材按型给出（据此扩写，每题一行）：
  - R 现实型：修理/组装电器、种植养护、木工钳工、开机械/车辆、户外作业、制作模型、电脑装机、搬运装卸、电子维修、下厨做菜。
  - I 研究型：解数理谜题、阅读科普、做实验、研究星象/动植物、分析数据、钻研原理、查证资料、学新编程/技能原理、撰写研究报告、思考抽象问题。
  - A 艺术型：画画/素描、写诗写小说、弹奏乐器、摄影、设计海报、跳舞/戏剧、做手账/手工、欣赏艺术展、编曲写歌、布置有美感的空间。
  - S 社会型：帮朋友解决困扰、参加志愿活动、讲解/教学、组织公益、倾听陪伴、调解矛盾、照护老幼、做主持人、团队破冰、分享经验带新人。
  - E 经营型：组织活动/办比赛、说服他人、谈合作谈价、牵头创业点子、负责项目推进、销售推广、竞聘演讲、规划扩张、协调资源、主持商务谈判。
  - C 常规型：整理台账/清单、核对数据、做报表归档、制定日程表、按流程操作、管理账目、录入校对、整理文件柜、报销/行政、维护系统记录。
- [ ] **Step 3: scoring**：六段各 10 题；计数每型选"喜欢"的次数（0-10）。返回 `{ dims:{R,I,A,S,E,C:个数}, code:按从高到低取前三字母 }`。并列时字母序优先（R<I<A<S<E<C）。
- [ ] **Step 4: renderer**：`primary = 主型三码`（如 `SEC · 社会·企业·常规`），`primarySub = Top1 型一句话`；六型 `bars` 仅展示单侧值可用 `list` 呈现每型 `title=型名, value=次数/10`；`sections` 按 RIASEC 顺序给每型 1-2 句画像 + 职业关键词；`advice` 2-3 条（含"与主型相邻型的组合提示"）。主型职业参考表随渲染输出。
- [ ] **Step 5: 浏览器验证**：兴趣入口 → SDS → 60 题走通；用已知回答（如 R 全喜欢、其余全不喜欢）验算 code 首字母 R、六型次数正确；报告记录 summary 为三码。
- [ ] **Step 6: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): 霍兰德SDS 60题题库与RIASEC报告"`

---

### Task 6: 舒伯 WVI 题库 + 15 维渲染

**Files:** `zhijian-app.html`（JS 数据区追加 `id:'wvi'`, `cat:'value'`）

- [ ] **Step 1: 追加 WVI**：`color:'fi-green'`（价值观主题色），icon 沿用首页"价值观测评"心形 SVG。

```js
  SCALES.push({
    id:'wvi', cat:'value', name:'舒伯职业价值观测评', short:'WVI', color:'fi-green',
    icon: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/></svg>',
    desc: '15 项价值维度共 45 题 · 约 6 分钟', minutes: 6,
    questions: [ /* 见 Step 2 */ ],
    ...
  });
```

- [ ] **Step 2: 编写 45 题题库**：15 维 × 3 题，`opts:['1 非常不赞同','2 不赞同','3 中立','4 赞同','5 非常赞同']`（值取下标+1）。维度与题面素材（每题写成"在工作中，…对我而言很重要"，围绕价值造句 3 个不同情境）：
  - 成就感 / 物质保障 / 生活方式(工作与生活平衡) / 社会声望 / 工作环境(舒适条件) / 同事关系 / 经济报酬 / 社会交往 / 智力刺激 / 利他主义 / 创造性 / 独立性(自主) / 多样性(变化挑战) / 安全感(稳定) / 管理权力。
  - 每题主题表（维度-情境 A/B/C），例如"成就感"：①能看到自己努力的成果 ②能达成有挑战性的目标 ③所做工作被认可。
- [ ] **Step 3: scoring**：15 段各 3 题，`sum = Σ(选项下标+1)`，得 3-15。返回 `{ dims:{15个:和}, top:[按得分降序前三名维名] }`；并列按题序（值表顺序）优先。
- [ ] **Step 4: renderer**：`primary = Top1 价值观名`，`primarySub = 一句话解释`；`list` 全 15 维（`title=维名, value=得分（最高15）`）；`sections`：Top1/2/3 各自一段解读与"可能偏好的职业特质"；`advice` 结合排序给"在选择 Offer 时优先满足项"提示。
- [ ] **Step 5: 浏览器验证**：价值观入口 → WVI → 45 题走通；用手算组（如把某维 3 题全选 5）验算维和=15 且进入 top；报告记录 summary = Top3 维名。
- [ ] **Step 6: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): 舒伯WVI 45题题库与15维报告"`

---

### Task 7: 回归走查与收尾

**Files:** `zhijian-app.html`

- [ ] **Step 1: 清理与一致性检查**
  - 确认已删除 `_fixture`；确认 6 处入口接线、`renderAssessReports()` 已在 `openAssess` 内调用。
  - 静态"性格测评报告 · INTJ"旧卡片已移除（已被 `#assessReportList` 取代）。
  - 检查不相关入口（情商/领导力/压力/学习/团队/能力）仍是 Toast 占位，未被误改。
  - 检查内联 CSS 无残留临时（如 fixture 的 `qzRestartTmp` 移除）。

- [ ] **Step 2: 浏览器全量走查（Playwright）**
  - 首页 / 测评页 → 三入口映射正确（性格=MBTI、兴趣=SDS、价值观=WVI）。
  - MBTI：末题提交前缺题 Toast；全答提交 → 4 字母+类型名、四维条数值与手算一致；重测、返回选择层正常。
  - SDS / WVI：各自走通与抽样手算校验（按 Task5/6 Step5）。
  - 测评页报告区：三种记录出现（code/三码/Top3）、空态消失后不残留；"再测一次"回写覆盖旧记录。
  - 其余入口与 AI 对话等原功能回归正常。

- [ ] **Step 3: 视觉核对**：选择层卡片、选项高亮、结果主卡与双色条与整体 App 黑白 + fi 主题色风格一致；全屏容器滑入/返回正常，手机宽度滚动正常。

- [ ] **Step 4: Commit**：`git add zhijian-app.html && git commit -m "feat(assess): 回归走查收尾，MBTI/SDS/WVI三套上线"`

---

## 3. 风险与注意

- 文件单 HTML 体积增长显著（题库文案 + 渲染），注意保持 `questions` 内题干不换行、引号转义一致（用 `'` 包裹、内容含 `"` 时避免冲突；题干文案避免反斜杠）。
- 题库为"流传整理版"：题干文案非逐字原版，报告页已有免责脚注。
- MBTI tie-break 段定位以任务 4 Step 3 的可执行方案为准（四段各 16 题，末题仲裁）。
- 所有 `showToast('…测评')` 仅替换性格/兴趣/价值观三对入口字符串，勿全局替换。
