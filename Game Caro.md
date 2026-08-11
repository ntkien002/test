[Chơi cờ caro](https://papergames.io/vi/c%E1%BB%9D-caro)

---
- copy script và dán vào Console thực hiện chương trình

---

bash
```
// ==UserScript==
// @name         PaperGames Caro Threat Engine
// @namespace    local.codex
// @version      3.5.0
// @description  Canh bao de doa, goi y nuoc tot (X cheo), du doan nuoc tiep theo, panel legend.
// @match        https://papergames.io/*/r/*
// @grant        none
// ==/UserScript==

(() => {
  'use strict';

  const SIZE = 15;
  const DIRECTIONS = [[1, 0], [0, 1], [1, 1], [1, -1]];
  const MARKS = [
    'caro-danger', 'caro-fork', 'caro-best',
    'caro-double-opp', 'caro-double-me',
    'caro-three-four-opp', 'caro-three-four-me',
    'caro-two-three-opp', 'caro-two-three-me', 'caro-attack',
    'caro-open-three-opp', 'caro-open-three-me',
    'caro-two-one-one-opp', 'caro-two-one-one-me',
    'caro-two-two-one-opp', 'caro-two-two-one-me',
    // v3.4: hint = engine best move (X chéo xanh lá),
    //        predict-me = dự đoán nước mình (hình thoi ◆),
    //        predict-opp = dự đoán nước đối thủ (hình tròn ○)
    'caro-hint', 'caro-predict-me', 'caro-predict-opp',
  ];
  let observer;
  let observedBoard;
  let lastWarning = '';
  let scanTimer;

  const ensureUi = () => {
    if (!document.querySelector('#caro-threat-style')) {
      const style = document.createElement('style');
      style.id = 'caro-threat-style';
      style.textContent = `
        #caro-threat-warning {
          position: fixed; top: 76px; left: 50%; transform: translateX(-50%);
          z-index: 2147483647; max-width: 90vw; padding: 5px 9px;
          border: 1px solid rgba(255,255,255,.7); border-radius: 6px;
          color: #fff; background: rgba(33,37,41,.9);
          font: 700 12px/1.25 sans-serif; text-align: center;
          box-shadow: 0 3px 10px rgba(0,0,0,.3); display: none;
          pointer-events: none;
        }
        table.table-board td { position: relative; }
        .caro-danger::after, .caro-fork::after, .caro-best::after,
        .caro-double-opp::after, .caro-double-me::after,
        .caro-three-four-opp::after, .caro-three-four-me::after,
        .caro-two-three-opp::after, .caro-two-three-me::after,
        .caro-attack::after, .caro-open-three-opp::after,
        .caro-open-three-me::after, .caro-two-one-one-opp::after,
        .caro-two-one-one-me::after, .caro-two-two-one-opp::after,
        .caro-two-two-one-me::after {
          content: ''; position: absolute; left: 50%; top: 50%; z-index: 9;
          width: 7px; height: 7px; transform: translate(-50%, -50%);
          border: 1px solid #fff; border-radius: 50%;
          box-shadow: 0 1px 3px rgba(0,0,0,.6); pointer-events: none;
        }

        /* ── HINT: dấu X chéo xanh lá – nước tốt nhất của mình ── */
        .caro-hint { z-index: 10; }
        .caro-hint::before,
        .caro-hint::after {
          content: ''; position: absolute; left: 50%; top: 50%;
          z-index: 10; pointer-events: none;
          width: 18px; height: 3px; border-radius: 2px;
          background: #2ecc40;
          box-shadow: 0 0 5px rgba(46,204,64,.8), 0 0 10px rgba(46,204,64,.4);
        }
        .caro-hint::before { transform: translate(-50%,-50%) rotate(45deg); }
        .caro-hint::after  { transform: translate(-50%,-50%) rotate(-45deg); }

        /* ── PREDICT-ME: hình thoi ◆ – dự đoán nước tiếp của mình ── */
        .caro-predict-me::after {
          content: ''; position: absolute; left: 50%; top: 50%;
          z-index: 9; pointer-events: none;
          width: 12px; height: 12px;
          background: rgba(52,152,219,.85);
          border: 1.5px solid #fff;
          box-shadow: 0 0 6px rgba(52,152,219,.7);
          transform: translate(-50%,-50%) rotate(45deg);
          border-radius: 2px;
        }

        /* ── PREDICT-OPP: hình tròn rỗng – dự đoán nước tiếp đối thủ ── */
        .caro-predict-opp::after {
          content: ''; position: absolute; left: 50%; top: 50%;
          z-index: 9; pointer-events: none;
          width: 14px; height: 14px;
          background: transparent;
          border: 2.5px solid #ff6b35;
          border-radius: 50%;
          box-shadow: 0 0 6px rgba(255,107,53,.7);
          transform: translate(-50%,-50%);
        }
        .caro-danger::after, .caro-fork::after,
        .caro-double-opp::after, .caro-three-four-opp::after,
        .caro-two-three-opp::after,
        .caro-open-three-opp::after,
        .caro-two-one-one-opp::after,
        .caro-two-two-one-opp::after { background: #ffd43b; }
        .caro-best::after, .caro-double-me::after,
        .caro-three-four-me::after, .caro-two-three-me::after,
        .caro-attack::after, .caro-open-three-me::after,
        .caro-two-one-one-me::after,
        .caro-two-two-one-me::after { background: #f5222d; }
        .caro-danger.caro-best::after,
        .caro-fork.caro-best::after,
        .caro-double-opp.caro-best::after,
        .caro-two-one-one-opp.caro-best::after,
        .caro-two-two-one-opp.caro-best::after,
        .caro-three-four-opp.caro-best::after,
        .caro-open-three-opp.caro-best::after {
          width: 14px; height: 14px; background: #ffd43b;
          box-shadow: 0 0 0 4px #f5222d, 0 2px 5px rgba(0,0,0,.6);
        }
        .caro-double-me.caro-best::after,
        .caro-three-four-me.caro-best::after,
        .caro-two-three-me.caro-best::after,
        .caro-two-one-one-me.caro-best::after,
        .caro-two-two-one-me.caro-best::after,
        .caro-attack.caro-best::after,
        .caro-open-three-me.caro-best::after {
          background: #f5222d; box-shadow: 0 0 0 2px #fff;
        }
        .caro-open-three-opp::after,
        .caro-open-three-opp.caro-best::after {
          width: 14px; height: 14px; background: #f5222d;
          box-shadow: 0 0 0 4px #ffd43b, 0 2px 5px rgba(0,0,0,.6);
        }
        .caro-open-three-me::after,
        .caro-open-three-me.caro-best::after {
          width: 14px; height: 14px; background: #f5222d;
          box-shadow: 0 0 0 4px #111, 0 2px 5px rgba(0,0,0,.6);
        }
        .caro-attack::after, .caro-best::after {
          width: 8px !important; height: 8px !important;
        }
        .caro-two-three-opp::after, .caro-two-three-me::after,
        .caro-two-one-one-opp::after, .caro-two-one-one-me::after {
          width: 10px !important; height: 10px !important;
        }
        .caro-open-three-opp::after, .caro-open-three-me::after {
          width: 12px !important; height: 12px !important;
        }
        .caro-fork::after, .caro-double-opp::after, .caro-double-me::after,
        .caro-two-two-one-opp::after, .caro-two-two-one-me::after {
          width: 14px !important; height: 14px !important;
        }
        .caro-three-four-opp::after, .caro-three-four-me::after {
          width: 16px !important; height: 16px !important;
        }
        .caro-danger::after {
          width: 18px !important; height: 18px !important;
        }

        /* ── PANEL ──────────────────────────────────────────────────────── */
        #caro-panel {
          position: fixed; bottom: 16px; right: 16px;
          z-index: 2147483646;
          background: rgba(18,20,26,.96);
          border: 1px solid rgba(255,255,255,.13);
          border-radius: 12px;
          color: #e9ecef;
          font: 12px/1.5 'Segoe UI', system-ui, sans-serif;
          box-shadow: 0 8px 32px rgba(0,0,0,.55), 0 0 0 1px rgba(255,255,255,.04);
          min-width: 210px; max-width: 240px;
          user-select: none;
          transition: opacity .2s;
        }
        #caro-panel.collapsed #caro-panel-body { display: none; }
        #caro-panel-header {
          display: flex; align-items: center; gap: 7px;
          padding: 9px 12px 8px;
          border-bottom: 1px solid rgba(255,255,255,.08);
          cursor: pointer;
        }
        #caro-panel-header:hover { background: rgba(255,255,255,.04); border-radius: 12px 12px 0 0; }
        #caro-panel-title {
          font-weight: 700; font-size: 12px; letter-spacing: .6px;
          color: #ced4da; flex: 1;
        }
        #caro-panel-chevron {
          font-size: 10px; color: #868e96;
          transition: transform .2s;
        }
        #caro-panel.collapsed #caro-panel-chevron { transform: rotate(-90deg); }
        #caro-panel-body { padding: 10px 12px 12px; }

        /* turn badge */
        #caro-turn-badge {
          text-align: center; font-size: 11px; font-weight: 700;
          padding: 4px 0 8px; letter-spacing: .4px;
        }
        #caro-turn-badge.my-turn   { color: #51cf66; }
        #caro-turn-badge.opp-turn  { color: #ffd43b; }
        #caro-turn-badge.waiting   { color: #868e96; }

        /* divider */
        .caro-divider {
          border: none; border-top: 1px solid rgba(255,255,255,.07);
          margin: 7px 0;
        }

        /* section label */
        .caro-section-label {
          font-size: 9px; font-weight: 700; letter-spacing: 1.2px;
          color: #868e96; text-transform: uppercase; margin-bottom: 5px;
        }

        /* legend rows */
        .caro-legend-row {
          display: flex; align-items: center; gap: 8px;
          margin: 4px 0; font-size: 11px; color: #ced4da;
        }
        .caro-legend-icon {
          flex-shrink: 0; width: 20px; height: 20px;
          display: flex; align-items: center; justify-content: center;
          position: relative;
        }

        /* X cross icon (hint) */
        .cli-x::before, .cli-x::after {
          content: ''; position: absolute;
          width: 14px; height: 2.5px; border-radius: 2px;
          background: #2ecc40;
          box-shadow: 0 0 4px rgba(46,204,64,.7);
        }
        .cli-x::before { transform: rotate(45deg); }
        .cli-x::after  { transform: rotate(-45deg); }

        /* diamond icon (predict-me) */
        .cli-diamond::after {
          content: ''; position: absolute;
          width: 10px; height: 10px; border-radius: 2px;
          background: rgba(52,152,219,.85);
          border: 1.5px solid #fff;
          box-shadow: 0 0 4px rgba(52,152,219,.6);
          transform: rotate(45deg);
        }

        /* circle outline (predict-opp) */
        .cli-circle::after {
          content: ''; position: absolute;
          width: 12px; height: 12px; border-radius: 50%;
          background: transparent;
          border: 2px solid #ff6b35;
          box-shadow: 0 0 4px rgba(255,107,53,.6);
        }

        /* dot icons for threats */
        .cli-dot::after {
          content: ''; position: absolute;
          width: 10px; height: 10px; border-radius: 50%;
          border: 1px solid rgba(255,255,255,.5);
        }
        .cli-dot.yellow::after { background: #ffd43b; }
        .cli-dot.red::after    { background: #f5222d; }
        .cli-dot.orange::after { background: #fd7e14; box-shadow: 0 0 0 2px rgba(253,126,20,.3); }
        .cli-dot.big::after    { width: 14px; height: 14px; }

        /* toggle switch */
        .caro-toggle-row {
          display: flex; align-items: center; justify-content: space-between;
          margin: 5px 0;
        }
        .caro-toggle-label { font-size: 11px; color: #adb5bd; }
        .caro-toggle {
          width: 30px; height: 16px; border-radius: 8px;
          background: #343a40; border: 1px solid #495057;
          cursor: pointer; position: relative; transition: background .2s; flex-shrink: 0;
        }
        .caro-toggle.on { background: #2f9e44; border-color: #2f9e44; }
        .caro-toggle::after {
          content: ''; position: absolute; top: 2px; left: 2px;
          width: 10px; height: 10px; border-radius: 50%;
          background: #fff; transition: left .2s;
        }
        .caro-toggle.on::after { left: 16px; }

        /* pulse animation for hint dot on board */
        @keyframes caro-hint-pulse {
          0%,100% { opacity:1; }
          50%      { opacity:.55; }
        }
        .caro-hint::before, .caro-hint::after { animation: caro-hint-pulse 1.4s ease-in-out infinite; }
        .caro-danger::after, .caro-fork::after,
        .caro-double-opp::after, .caro-three-four-opp::after,
        .caro-two-three-opp::after, .caro-open-three-opp::after,
        .caro-two-one-one-opp::after, .caro-two-two-one-opp::after {
          background: #ffd43b !important;
        }
        .caro-best::after, .caro-double-me::after,
        .caro-three-four-me::after, .caro-two-three-me::after,
        .caro-attack::after, .caro-open-three-me::after,
        .caro-two-one-one-me::after, .caro-two-two-one-me::after {
          background: #f5222d !important;
        }
        .caro-best::after, .caro-open-three-me::after {
          box-shadow: 0 0 0 3px #111, 0 2px 5px rgba(0,0,0,.6) !important;
        }
        .caro-danger.caro-best::after, .caro-fork.caro-best::after,
        .caro-double-opp.caro-best::after,
        .caro-three-four-opp.caro-best::after,
        .caro-two-three-opp.caro-best::after,
        .caro-open-three-opp.caro-best::after,
        .caro-two-one-one-opp.caro-best::after,
        .caro-two-two-one-opp.caro-best::after {
          background: #ffd43b !important;
          box-shadow: 0 0 0 4px #f5222d, 0 2px 5px rgba(0,0,0,.6) !important;
        }
      `;
      document.head.appendChild(style);
    }

    let banner = document.querySelector('#caro-threat-warning');
    if (!banner) {
      banner = document.createElement('div');
      banner.id = 'caro-threat-warning';
      banner.setAttribute('role', 'alert');
      document.body.appendChild(banner);
    }

    // ── PANEL ────────────────────────────────────────────────────────────
    if (!document.querySelector('#caro-panel')) {
      const panel = document.createElement('div');
      panel.id = 'caro-panel';
      panel.innerHTML = `
        <div id="caro-panel-header">
          <span style="font-size:14px">♟</span>
          <span id="caro-panel-title">CARO ENGINE</span>
          <span id="caro-panel-chevron">▾</span>
        </div>
        <div id="caro-panel-body">

          <!-- lượt đi -->
          <div id="caro-turn-badge" class="waiting">⏳ Đang chờ…</div>

          <hr class="caro-divider">

          <!-- ký hiệu trên bàn -->
          <div class="caro-section-label">Ký hiệu trên bàn</div>

          <div class="caro-legend-row">
            <div class="caro-legend-icon cli-x"></div>
            <span>Nước tốt nhất (engine)</span>
          </div>
          <div class="caro-legend-row">
            <div class="caro-legend-icon cli-diamond"></div>
            <span>Dự đoán nước của tôi ◆</span>
          </div>
          <div class="caro-legend-row">
            <div class="caro-legend-icon cli-circle"></div>
            <span>Dự đoán nước đối thủ ○</span>
          </div>
          <div class="caro-legend-row">
            <div class="caro-legend-icon cli-dot yellow big"></div>
            <span>Đe dọa từ đối thủ ⚠</span>
          </div>
          <div class="caro-legend-row">
            <div class="caro-legend-icon cli-dot orange big"></div>
            <span>Fork / nước nguy hiểm</span>
          </div>

          <hr class="caro-divider">

          <!-- cài đặt -->
          <div class="caro-section-label">Cài đặt</div>

          <div class="caro-toggle-row">
            <span class="caro-toggle-label">🟢 Gợi ý nước (X)</span>
            <div class="caro-toggle on" id="caro-toggle-hint"></div>
          </div>
          <div class="caro-toggle-row">
            <span class="caro-toggle-label">◆ Dự đoán của tôi</span>
            <div class="caro-toggle on" id="caro-toggle-pred-me"></div>
          </div>
          <div class="caro-toggle-row">
            <span class="caro-toggle-label">○ Dự đoán đối thủ</span>
            <div class="caro-toggle on" id="caro-toggle-pred-opp"></div>
          </div>
          <div class="caro-toggle-row">
            <span class="caro-toggle-label">🟡 Cảnh báo đe dọa</span>
            <div class="caro-toggle on" id="caro-toggle-warn"></div>
          </div>
          <div class="caro-toggle-row">
            <span class="caro-toggle-label">🔊 Âm thanh</span>
            <div class="caro-toggle on" id="caro-toggle-sound"></div>
          </div>

          <hr class="caro-divider">
          <div id="caro-status-line" style="font-size:10px;color:#868e96;text-align:center">v3.5.0</div>
        </div>
      `;
      document.body.appendChild(panel);

      // collapse toggle
      document.getElementById('caro-panel-header').addEventListener('click', () => {
        panel.classList.toggle('collapsed');
      });

      // setting toggles – store in window.caroSettings
      window.caroSettings = window.caroSettings || {
        showHint: true, showPredMe: true, showPredOpp: true,
        showWarn: true, soundOn: true,
      };
      const bindToggle = (id, key) => {
        const el = document.getElementById(id);
        if (!el) return;
        // sync initial state
        el.classList.toggle('on', window.caroSettings[key] !== false);
        el.addEventListener('click', (e) => {
          e.stopPropagation();
          window.caroSettings[key] = !window.caroSettings[key];
          el.classList.toggle('on', window.caroSettings[key]);
        });
      };
      bindToggle('caro-toggle-hint',     'showHint');
      bindToggle('caro-toggle-pred-me',  'showPredMe');
      bindToggle('caro-toggle-pred-opp', 'showPredOpp');
      bindToggle('caro-toggle-warn',     'showWarn');
      bindToggle('caro-toggle-sound',    'soundOn');
    }

    return banner;
  };

  const playerColor = (panel) => {
    const shape = panel?.querySelector('app-player-symbol .shape');
    if (shape?.classList.contains('circle-dark')) return 'dark';
    if (shape?.classList.contains('circle-light')) return 'light';
    return null;
  };

  const getLocalPlayerName = () => {
    const nameNode = document.querySelector(
      '.user-profile .name-credit > div:first-child'
    );
    return nameNode?.textContent.trim() || null;
  };

  const getLocalPlayerAvatar = () =>
    document.querySelector('.user-profile app-user-avatar img')?.src || null;

  const oppositeColor = (color) =>
    color === 'dark' ? 'light' : color === 'light' ? 'dark' : null;

  const getColors = (table) => {
    const panels = [...document.querySelectorAll('.col-6.d-flex')]
      .filter((panel) => playerColor(panel));
    const localName = getLocalPlayerName();
    const localAvatar = getLocalPlayerAvatar();
    const normalizedName = localName?.toLocaleLowerCase();
    const nameMatches = normalizedName
      ? panels.filter((panel) => [...panel.querySelectorAll('[appprofileopener]')]
        .some((node) => node.textContent.trim().toLocaleLowerCase() === normalizedName))
      : [];
    const avatarMatches = localAvatar
      ? panels.filter((panel) =>
        panel.querySelector('app-user-avatar img')?.src === localAvatar)
      : [];
    const lastMoveColor = getLastMoveColor(table);
    const hasClickableCell = Boolean(table.querySelector('td.clickable'));
    const activePanel = panels.find((panel) =>
      panel.querySelector('app-user-avatar .progress-circle'));
    const activeColor = playerColor(activePanel);
    const signals = [];

    if (lastMoveColor) {
      signals.push({
        source: 'last-move',
        color: hasClickableCell ? oppositeColor(lastMoveColor) : lastMoveColor,
      });
    }
    if (activeColor) {
      signals.push({
        source: 'active-turn',
        color: hasClickableCell ? activeColor : oppositeColor(activeColor),
      });
    }
    if (nameMatches.length === 1) {
      signals.push({ source: 'profile-name', color: playerColor(nameMatches[0]) });
    }
    if (avatarMatches.length === 1) {
      signals.push({ source: 'profile-avatar', color: playerColor(avatarMatches[0]) });
    }

    const detectedColors = new Set(signals.map((signal) => signal.color).filter(Boolean));
    const me = detectedColors.size === 1 ? [...detectedColors][0] : null;
    return {
      me,
      opponent: oppositeColor(me),
      localName,
      lastMoveColor,
      hasClickableCell,
      activeColor,
      identitySource: me
        ? signals.map((signal) => signal.source).join('+')
        : detectedColors.size > 1 ? 'conflict' : 'waiting',
    };
  };

  const readBoard = (table) => {
    const cells = [...table.querySelectorAll('td')];
    if (cells.length !== SIZE * SIZE) return null;
    const grid = Array.from({ length: SIZE }, () => Array(SIZE).fill(null));
    cells.forEach((cell, index) => {
      const row = Math.floor(index / SIZE);
      const col = index % SIZE;
      if (cell.querySelector('.circle-dark')) grid[row][col] = 'dark';
      if (cell.querySelector('.circle-light')) grid[row][col] = 'light';
    });
    return { cells, grid };
  };

  const inBounds = (row, col) =>
    row >= 0 && row < SIZE && col >= 0 && col < SIZE;

  const countSide = (grid, row, col, color, dr, dc) => {
    let count = 0;
    let r = row + dr;
    let c = col + dc;
    while (inBounds(r, c) && grid[r][c] === color) {
      count++;
      r += dr;
      c += dc;
    }
    return { count, open: inBounds(r, c) && !grid[r][c] };
  };

  const lineInfo = (grid, row, col, color, dr, dc) => {
    const forward = countSide(grid, row, col, color, dr, dc);
    const backward = countSide(grid, row, col, color, -dr, -dc);
    return {
      length: 1 + forward.count + backward.count,
      openEnds: Number(forward.open) + Number(backward.open),
    };
  };

  const isWinningMove = (grid, row, col, color) => {
    if (grid[row][col]) return false;
    grid[row][col] = color;
    const wins = DIRECTIONS.some(([dr, dc]) =>
      lineInfo(grid, row, col, color, dr, dc).length >= 5);
    grid[row][col] = null;
    return wins;
  };

  const candidateMoves = (grid, radius = 2) => {
    const stones = [];
    for (let row = 0; row < SIZE; row++) {
      for (let col = 0; col < SIZE; col++) {
        if (grid[row][col]) stones.push([row, col]);
      }
    }
    if (!stones.length) return [[7, 7]];

    const candidates = new Set();
    for (const [stoneRow, stoneCol] of stones) {
      for (let dr = -radius; dr <= radius; dr++) {
        for (let dc = -radius; dc <= radius; dc++) {
          const row = stoneRow + dr;
          const col = stoneCol + dc;
          if (inBounds(row, col) && !grid[row][col]) candidates.add(`${row},${col}`);
        }
      }
    }
    return [...candidates].map((key) => key.split(',').map(Number));
  };

  const immediateWins = (grid, color, candidates = candidateMoves(grid, 1)) =>
    candidates.filter(([row, col]) => isWinningMove(grid, row, col, color));

  const shapeScore = (grid, row, col, color) => {
    if (grid[row][col]) return -Infinity;
    grid[row][col] = color;
    let total = 0;
    for (const [dr, dc] of DIRECTIONS) {
      const { length, openEnds } = lineInfo(grid, row, col, color, dr, dc);
      if (length >= 5) total += 1000000;
      else if (length === 4 && openEnds === 2) total += 120000;
      else if (length === 4 && openEnds === 1) total += 24000;
      else if (length === 3 && openEnds === 2) total += 9000;
      else if (length === 3 && openEnds === 1) total += 900;
      else if (length === 2 && openEnds === 2) total += 300;
      else total += length * 8 + openEnds;
    }
    grid[row][col] = null;
    return total;
  };

  const forkMoves = (grid, color, candidates = candidateMoves(grid, 2)) => {
    const forks = [];
    for (const [row, col] of candidates) {
      if (grid[row][col] || isWinningMove(grid, row, col, color)) continue;
      grid[row][col] = color;
      const nextWins = immediateWins(grid, color);
      grid[row][col] = null;
      if (nextWins.length >= 2) forks.push([row, col]);
    }
    return forks;
  };

  const directionHasOpenThree = (grid, row, col, color, dr, dc) => {
    const values = [];
    for (let offset = -4; offset <= 4; offset++) {
      const r = row + dr * offset;
      const c = col + dc * offset;
      if (!inBounds(r, c)) values.push('#');
      else if (!grid[r][c]) values.push('.');
      else values.push(grid[r][c] === color ? 'X' : 'O');
    }

    const line = values.join('');
    const center = 4;
    const patterns = ['.XXX.', '.XX.X.', '.X.XX.'];
    return patterns.some((pattern) => {
      for (let start = 0; start <= line.length - pattern.length; start++) {
        const centerInPattern = center - start;
        if (centerInPattern < 0 || centerInPattern >= pattern.length ||
            pattern[centerInPattern] !== 'X') continue;
        if (line.slice(start, start + pattern.length) === pattern) return true;
      }
      return false;
    });
  };

  const openThreeDirectionsForMove = (grid, row, col, color) => {
    if (grid[row][col]) return 0;
    grid[row][col] = color;
    const count = DIRECTIONS.filter(([dr, dc]) =>
      directionHasOpenThree(grid, row, col, color, dr, dc)).length;
    grid[row][col] = null;
    return count;
  };

  const doubleThreeMoves = (grid, color, candidates = candidateMoves(grid, 2)) =>
    candidates.filter(([row, col]) =>
      openThreeDirectionsForMove(grid, row, col, color) >= 2);

  const directionHasFour = (grid, row, col, color, dr, dc) => {
    const values = [];
    for (let offset = -4; offset <= 4; offset++) {
      const r = row + dr * offset;
      const c = col + dc * offset;
      if (!inBounds(r, c)) values.push('#');
      else if (!grid[r][c]) values.push('.');
      else values.push(grid[r][c] === color ? 'X' : 'O');
    }

    const line = values.join('');
    const center = 4;
    for (let start = 0; start <= line.length - 5; start++) {
      const window = line.slice(start, start + 5);
      const centerInWindow = center - start;
      if (centerInWindow < 0 || centerInWindow >= 5 ||
          window[centerInWindow] !== 'X') continue;
      if ([...window].filter((value) => value === 'X').length === 4 &&
          [...window].filter((value) => value === '.').length === 1) return true;
    }
    return false;
  };

  const createsThreeFour = (grid, row, col, color) => {
    if (grid[row][col]) return false;
    grid[row][col] = color;
    const threes = [];
    const fours = [];
    DIRECTIONS.forEach(([dr, dc], index) => {
      if (directionHasOpenThree(grid, row, col, color, dr, dc)) threes.push(index);
      if (directionHasFour(grid, row, col, color, dr, dc)) fours.push(index);
    });
    grid[row][col] = null;
    return fours.some((fourDirection) =>
      threes.some((threeDirection) => threeDirection !== fourDirection));
  };

  const threeFourMoves = (grid, color, candidates = candidateMoves(grid, 2)) =>
    candidates.filter(([row, col]) => createsThreeFour(grid, row, col, color));

  const directionHasOpenTwo = (grid, row, col, color, dr, dc) => {
    const values = [];
    for (let offset = -3; offset <= 3; offset++) {
      const r = row + dr * offset;
      const c = col + dc * offset;
      if (!inBounds(r, c)) values.push('#');
      else if (!grid[r][c]) values.push('.');
      else values.push(grid[r][c] === color ? 'X' : 'O');
    }

    const line = values.join('');
    const center = 3;
    const patterns = ['.XX.', '.X.X.', '.X..X.'];
    return patterns.some((pattern) => {
      for (let start = 0; start <= line.length - pattern.length; start++) {
        const centerInPattern = center - start;
        if (centerInPattern < 0 || centerInPattern >= pattern.length ||
            pattern[centerInPattern] !== 'X') continue;
        if (line.slice(start, start + pattern.length) === pattern) return true;
      }
      return false;
    });
  };

  const createsTwoThree = (grid, row, col, color) => {
    if (grid[row][col]) return false;
    grid[row][col] = color;
    const twos = [];
    const threes = [];
    DIRECTIONS.forEach(([dr, dc], index) => {
      if (directionHasOpenTwo(grid, row, col, color, dr, dc)) twos.push(index);
      if (directionHasOpenThree(grid, row, col, color, dr, dc)) threes.push(index);
    });
    grid[row][col] = null;
    return threes.some((threeDirection) =>
      twos.some((twoDirection) => twoDirection !== threeDirection));
  };

  const twoThreeMoves = (grid, color, candidates = candidateMoves(grid, 2)) =>
    candidates.filter(([row, col]) => createsTwoThree(grid, row, col, color));

  const createsTwoOneOne = (grid, row, col, color) => {
    if (grid[row][col]) return false;
    grid[row][col] = color;
    const twos = [];
    const threes = [];
    DIRECTIONS.forEach(([dr, dc], index) => {
      if (directionHasOpenTwo(grid, row, col, color, dr, dc)) twos.push(index);
      if (directionHasOpenThree(grid, row, col, color, dr, dc)) threes.push(index);
    });
    grid[row][col] = null;
    return threes.some((threeDirection) =>
      twos.filter((twoDirection) => twoDirection !== threeDirection).length >= 2);
  };

  const twoOneOneMoves = (grid, color, candidates = candidateMoves(grid, 2)) =>
    candidates.filter(([row, col]) => createsTwoOneOne(grid, row, col, color));

  const createsTwoTwoOne = (grid, row, col, color) => {
    if (grid[row][col]) return false;
    grid[row][col] = color;
    const twos = [];
    const threes = [];
    DIRECTIONS.forEach(([dr, dc], index) => {
      if (directionHasOpenTwo(grid, row, col, color, dr, dc)) twos.push(index);
      if (directionHasOpenThree(grid, row, col, color, dr, dc)) threes.push(index);
    });
    grid[row][col] = null;
    if (threes.length < 2) return false;
    return twos.some((twoDirection) => !threes.includes(twoDirection));
  };

  const twoTwoOneMoves = (grid, color, candidates = candidateMoves(grid, 2)) =>
    candidates.filter(([row, col]) => createsTwoTwoOne(grid, row, col, color));

  const openThreeExtensionMoves = (grid, color) => {
    const extensions = new Set();
    for (const [dr, dc] of DIRECTIONS) {
      for (let row = 0; row < SIZE; row++) {
        for (let col = 0; col < SIZE; col++) {
          const beforeRow = row - dr;
          const beforeCol = col - dc;
          const afterRow = row + dr * 3;
          const afterCol = col + dc * 3;
          if (!inBounds(beforeRow, beforeCol) ||
              !inBounds(afterRow, afterCol) ||
              grid[beforeRow][beforeCol] || grid[afterRow][afterCol]) continue;

          let threeInLine = true;
          for (let index = 0; index < 3; index++) {
            if (grid[row + dr * index][col + dc * index] !== color) {
              threeInLine = false;
              break;
            }
          }
          if (!threeInLine) continue;

          extensions.add(`${beforeRow},${beforeCol}`);
          extensions.add(`${afterRow},${afterCol}`);
        }
      }
    }
    return [...extensions].map((key) => key.split(',').map(Number));
  };

  const topAttackMoves = (grid, color, candidates, limit = 3) =>
    candidates
      .map(([row, col]) => {
        let score = shapeScore(grid, row, col, color);
        if (createsTwoThree(grid, row, col, color)) score += 30000;
        if (createsTwoOneOne(grid, row, col, color)) score += 70000;
        if (createsTwoTwoOne(grid, row, col, color)) score += 260000;
        if (openThreeDirectionsForMove(grid, row, col, color) >= 2) score += 180000;
        if (createsThreeFour(grid, row, col, color)) score += 450000;
        if (isWinningMove(grid, row, col, color)) score += 2000000;
        score -= Math.abs(row - 7) + Math.abs(col - 7);
        return { move: [row, col], score };
      })
      .sort((a, b) => b.score - a.score)
      .slice(0, limit)
      .map((entry) => entry.move);

  const bestMove = (grid, me, opponent) => {
    const candidates = candidateMoves(grid, 2);
    let best = null;
    let bestScore = -Infinity;

    for (const [row, col] of candidates) {
      const myWin = isWinningMove(grid, row, col, me);
      const blocksWin = isWinningMove(grid, row, col, opponent);
      const makesDoubleThree = openThreeDirectionsForMove(grid, row, col, me) >= 2;
      const blocksDoubleThree = openThreeDirectionsForMove(grid, row, col, opponent) >= 2;
      const makesThreeFour = createsThreeFour(grid, row, col, me);
      const blocksThreeFour = createsThreeFour(grid, row, col, opponent);
      const makesTwoThree = createsTwoThree(grid, row, col, me);
      const blocksTwoThree = createsTwoThree(grid, row, col, opponent);
      const makesTwoOneOne = createsTwoOneOne(grid, row, col, me);
      const blocksTwoOneOne = createsTwoOneOne(grid, row, col, opponent);
      const makesTwoTwoOne = createsTwoTwoOne(grid, row, col, me);
      const blocksTwoTwoOne = createsTwoTwoOne(grid, row, col, opponent);
      const attack = shapeScore(grid, row, col, me);
      const defense = shapeScore(grid, row, col, opponent);

      grid[row][col] = me;
      const remainingLosses = immediateWins(grid, opponent).length;
      const myNextWins = immediateWins(grid, me).length;
      grid[row][col] = null;

      let score = attack + defense * 1.15 + myNextWins * 45000;
      score -= remainingLosses * 300000;
      score -= (Math.abs(row - 7) + Math.abs(col - 7)) * 2;
      if (blocksWin) score += 600000;
      if (blocksDoubleThree) score += 360000;
      if (makesDoubleThree) score += 420000;
      if (blocksThreeFour) score += 850000;
      if (makesThreeFour) score += 950000;
      if (blocksTwoThree) score += 90000;
      if (makesTwoThree) score += 110000;
      if (blocksTwoOneOne) score += 180000;
      if (makesTwoOneOne) score += 220000;
      if (blocksTwoTwoOne) score += 520000;
      if (makesTwoTwoOne) score += 620000;
      if (myWin) score += 5000000;

      if (score > bestScore) {
        bestScore = score;
        best = [row, col];
      }
    }
    return best;
  };

  const ENGINE_WIN = 1000000000;
  const ENGINE_TIMEOUT = Symbol('engine-timeout');
  let zobristSeed = 0x6d2b79f5;
  const nextZobrist = () => {
    zobristSeed ^= zobristSeed << 13;
    zobristSeed ^= zobristSeed >>> 17;
    zobristSeed ^= zobristSeed << 5;
    return zobristSeed >>> 0;
  };
  const ZOBRIST = Array.from({ length: SIZE * SIZE }, () =>
    [nextZobrist(), nextZobrist()]);

  const boardHash = (grid, me, opponent) => {
    let hash = 0;
    for (let row = 0; row < SIZE; row++) {
      for (let col = 0; col < SIZE; col++) {
        const value = grid[row][col];
        if (!value) continue;
        hash ^= ZOBRIST[row * SIZE + col][value === me ? 0 : 1];
      }
    }
    return hash >>> 0;
  };

  const hasFiveAt = (grid, row, col, color) =>
    DIRECTIONS.some(([dr, dc]) =>
      lineInfo(grid, row, col, color, dr, dc).length >= 5);

  const engineStaticEvaluation = (grid, me, opponent) => {
    const moves = candidateMoves(grid, 1);
    if (!moves.length) return 0;
    let myBest = 0;
    let opponentBest = 0;
    for (const [row, col] of moves) {
      myBest = Math.max(myBest, shapeScore(grid, row, col, me));
      opponentBest = Math.max(opponentBest, shapeScore(grid, row, col, opponent));
    }
    return myBest - opponentBest * 1.12;
  };

  const engineOrderedMoves = (grid, color, enemy, depth, limit) => {
    const stonesOnBoard = grid.flat().filter(Boolean).length;
    const radius = depth >= 3 && stonesOnBoard > 20 ? 1 : 2;
    return candidateMoves(grid, radius)
      .map(([row, col]) => {
        const wins = isWinningMove(grid, row, col, color);
        const blocksWin = isWinningMove(grid, row, col, enemy);
        let score = shapeScore(grid, row, col, color) +
          shapeScore(grid, row, col, enemy) * 1.1;
        if (createsThreeFour(grid, row, col, color)) score += 900000;
        if (createsThreeFour(grid, row, col, enemy)) score += 800000;
        if (openThreeDirectionsForMove(grid, row, col, color) >= 2) score += 500000;
        if (openThreeDirectionsForMove(grid, row, col, enemy) >= 2) score += 450000;
        if (blocksWin) score += 8000000;
        if (wins) score += 10000000;
        return { move: [row, col], score };
      })
      .sort((a, b) => b.score - a.score)
      .slice(0, limit)
      .map((entry) => entry.move);
  };

  const findEngineMove = (grid, me, opponent, timeLimit = 420, maxDepth = 4) => {
    const startedAt = performance.now();
    const deadline = startedAt + timeLimit;
    const transposition = new Map();
    const initialHash = boardHash(grid, me, opponent);
    let completedMove = null;
    let completedScore = -Infinity;
    let completedDepth = 0;
    let visitedNodes = 0;

    const checkTime = () => {
      if (performance.now() >= deadline) throw ENGINE_TIMEOUT;
    };

    const minimax = (
      depth, maximizing, alpha, beta, hash, lastMove, lastColor, ply
    ) => {
      visitedNodes++;
      if ((visitedNodes & 63) === 0) checkTime();
      if (lastMove && hasFiveAt(grid, lastMove[0], lastMove[1], lastColor)) {
        return lastColor === me ? ENGINE_WIN - ply : -ENGINE_WIN + ply;
      }
      if (depth === 0) return engineStaticEvaluation(grid, me, opponent);

      const cacheKey = `${hash}:${depth}:${maximizing ? 1 : 0}`;
      if (transposition.has(cacheKey)) return transposition.get(cacheKey);

      const color = maximizing ? me : opponent;
      const enemy = maximizing ? opponent : me;
      const moves = engineOrderedMoves(grid, color, enemy, depth, depth >= 3 ? 9 : 12);
      if (!moves.length) return 0;

      let value = maximizing ? -Infinity : Infinity;
      let cutoff = false;
      for (const [row, col] of moves) {
        checkTime();
        grid[row][col] = color;
        const colorIndex = color === me ? 0 : 1;
        const nextHash = (hash ^ ZOBRIST[row * SIZE + col][colorIndex]) >>> 0;
        let score;
        try {
          score = minimax(
            depth - 1, !maximizing, alpha, beta, nextHash,
            [row, col], color, ply + 1
          );
        } finally {
          grid[row][col] = null;
        }

        if (maximizing) {
          value = Math.max(value, score);
          alpha = Math.max(alpha, value);
        } else {
          value = Math.min(value, score);
          beta = Math.min(beta, value);
        }
        if (beta <= alpha) {
          cutoff = true;
          break;
        }
      }
      if (!cutoff) transposition.set(cacheKey, value);
      return value;
    };

    const fallbackMoves = engineOrderedMoves(grid, me, opponent, 1, 12);
    completedMove = fallbackMoves[0] || bestMove(grid, me, opponent);

    for (let depth = 1; depth <= maxDepth; depth++) {
      try {
        checkTime();
        const rootMoves = engineOrderedMoves(grid, me, opponent, depth, 16);
        let depthMove = rootMoves[0] || null;
        let depthScore = -Infinity;
        let alpha = -Infinity;

        for (const [row, col] of rootMoves) {
          checkTime();
          grid[row][col] = me;
          const nextHash = (initialHash ^ ZOBRIST[row * SIZE + col][0]) >>> 0;
          let score;
          try {
            score = minimax(
              depth - 1, false, alpha, Infinity, nextHash,
              [row, col], me, 1
            );
          } finally {
            grid[row][col] = null;
          }
          if (score > depthScore) {
            depthScore = score;
            depthMove = [row, col];
          }
          alpha = Math.max(alpha, depthScore);
        }

        completedMove = depthMove;
        completedScore = depthScore;
        completedDepth = depth;
        if (Math.abs(depthScore) >= ENGINE_WIN - 10) break;
      } catch (error) {
        if (error !== ENGINE_TIMEOUT) throw error;
        break;
      }
    }

    return {
      move: completedMove,
      score: completedScore,
      depth: completedDepth,
      nodes: visitedNodes,
      milliseconds: Math.round(performance.now() - startedAt),
      forced: Math.abs(completedScore) >= ENGINE_WIN - 10,
    };
  };

  const runEngineSelfTests = () => {
    const makeGrid = () =>
      Array.from({ length: SIZE }, () => Array(SIZE).fill(null));
    const isEndpoint = (move) =>
      move?.[0] === 7 && (move[1] === 4 || move[1] === 9);

    const winningGrid = makeGrid();
    for (let col = 5; col <= 8; col++) winningGrid[7][col] = 'me';
    winningGrid[6][6] = 'opponent';
    const winningResult = findEngineMove(
      winningGrid, 'me', 'opponent', 180, 2
    );

    const blockingGrid = makeGrid();
    for (let col = 5; col <= 8; col++) blockingGrid[7][col] = 'opponent';
    blockingGrid[6][6] = 'me';
    const blockingResult = findEngineMove(
      blockingGrid, 'me', 'opponent', 180, 2
    );

    return {
      passed: isEndpoint(winningResult.move) && isEndpoint(blockingResult.move),
      winningMove: winningResult.move,
      blockingMove: blockingResult.move,
    };
  };

  const mark = (state, moves, className) => {
    moves.forEach(([row, col]) =>
      state.cells[row * SIZE + col]?.classList.add(className));
  };

  const getLastMoveColor = (table) => {
    const lastCell = table.querySelector('.last-move')?.closest('td');
    if (lastCell?.querySelector('.circle-dark')) return 'dark';
    if (lastCell?.querySelector('.circle-light')) return 'light';
    return null;
  };

  const beep = (urgent) => {
    try {
      const AudioContext = window.AudioContext || window.webkitAudioContext;
      const context = new AudioContext();
      const oscillator = context.createOscillator();
      const gain = context.createGain();
      oscillator.frequency.value = urgent ? 880 : 560;
      gain.gain.setValueAtTime(0.08, context.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.001, context.currentTime + 0.16);
      oscillator.connect(gain).connect(context.destination);
      oscillator.start();
      oscillator.stop(context.currentTime + 0.16);
    } catch (_) {
      // Sound may be blocked before the first user interaction.
    }
  };

  const scan = () => {
    const table = document.querySelector('table.table-board');
    const state = table && readBoard(table);
    if (!state) return;

    const banner = ensureUi();
    const identity = getColors(table);
    const {
      me, opponent, localName, lastMoveColor,
      hasClickableCell, activeColor, identitySource,
    } = identity;
    const myTurn = Boolean(
      me && hasClickableCell && (!activeColor || activeColor === me)
    );
    const opponentTurn = Boolean(
      me && opponent && !hasClickableCell && lastMoveColor === me &&
      activeColor === opponent
    );
    const refreshThreats = Boolean(
      opponentTurn || (myTurn && lastMoveColor === opponent)
    );
    window.caroThreatIdentity = {
      name: localName, me, opponent, source: identitySource,
    };
    window.caroThreatState = {
      ready: Boolean(me && opponent),
      myTurn,
      opponentTurn,
      lastMoveColor,
      activeColor,
      hasClickableCell,
      refreshThreats,
    };

    // ── Cập nhật turn badge ────────────────────────────────────────────
    const turnBadge = document.getElementById('caro-turn-badge');
    if (turnBadge) {
      if (!me || !opponent) {
        turnBadge.textContent = '⏳ Đang chờ…';
        turnBadge.className = 'waiting';
      } else if (myTurn) {
        turnBadge.textContent = '🟢 Lượt của bạn';
        turnBadge.className = 'my-turn';
      } else {
        turnBadge.textContent = '🟡 Lượt đối thủ';
        turnBadge.className = 'opp-turn';
      }
    }

    const cfg = window.caroSettings || {};

    if (!me || !opponent || !refreshThreats) return;

    state.cells.forEach((cell) => cell.classList.remove(...MARKS));

    // ── ENGINE HINT (X chéo xanh) – nước tốt nhất ────────────────────
    if (cfg.showHint !== false && myTurn) {
      const engineResult = findEngineMove(state.grid, me, opponent, 420, 4);
      window.caroEngineAnalysis = engineResult;
      if (engineResult.move) {
        const [hr, hc] = engineResult.move;
        state.cells[hr * SIZE + hc]?.classList.add('caro-hint');
      }
    } else {
      window.caroEngineAnalysis = null;
    }

    // ── PREDICT-ME (hình thoi ◆) – top-3 nước của mình ───────────────
    if (cfg.showPredMe !== false) {
      const myCandidates = candidateMoves(state.grid, 2);
      const myTopMoves = topAttackMoves(state.grid, me, myCandidates, 3);
      myTopMoves.forEach(([pr, pc]) => {
        const cell = state.cells[pr * SIZE + pc];
        if (cell && !cell.classList.contains('caro-hint')) {
          cell.classList.add('caro-predict-me');
        }
      });
    }

    // ── PREDICT-OPP (hình tròn ○ cam) – top-2 nước đối thủ ──────────
    if (cfg.showPredOpp !== false) {
      const oppCandidates = candidateMoves(state.grid, 2);
      const oppTopMoves = topAttackMoves(state.grid, opponent, oppCandidates, 2);
      oppTopMoves.forEach(([pr, pc]) => {
        state.cells[pr * SIZE + pc]?.classList.add('caro-predict-opp');
      });
    }

    const candidates = candidateMoves(state.grid, 2);
    const opponentWins = immediateWins(state.grid, opponent, candidates);
    const opponentThreeFours = threeFourMoves(state.grid, opponent, candidates);
    const opponentDoubleThrees = doubleThreeMoves(state.grid, opponent, candidates);
    const opponentTwoThrees = twoThreeMoves(state.grid, opponent, candidates);
    const opponentTwoOneOnes = twoOneOneMoves(state.grid, opponent, candidates);
    const opponentTwoTwoOnes = twoTwoOneMoves(state.grid, opponent, candidates);
    const opponentOpenThrees = openThreeExtensionMoves(state.grid, opponent);
    const opponentForks = forkMoves(state.grid, opponent, candidates);
    const opponentThreatMoves = new Set([
      ...opponentWins,
      ...opponentThreeFours,
      ...opponentDoubleThrees,
      ...opponentTwoThrees,
      ...opponentTwoOneOnes,
      ...opponentTwoTwoOnes,
      ...opponentOpenThrees,
      ...opponentForks,
    ].map((move) => move.join(',')));
    mark(state, opponentWins, 'caro-danger');
    mark(state, opponentThreeFours, 'caro-three-four-opp');
    mark(state, opponentDoubleThrees, 'caro-double-opp');
    mark(state, opponentTwoThrees, 'caro-two-three-opp');
    mark(state, opponentTwoOneOnes, 'caro-two-one-one-opp');
    mark(state, opponentTwoTwoOnes, 'caro-two-two-one-opp');
    mark(state, opponentOpenThrees, 'caro-open-three-opp');
    mark(state, opponentForks, 'caro-fork');

    let message = '';
    let urgent = false;
    if (opponentWins.length) {
      message = `CẢNH BÁO: đối thủ có ${opponentWins.length} nước thắng tiếp theo.`;
      urgent = true;
    } else if (
      opponentThreeFours.length || opponentTwoTwoOnes.length ||
      opponentDoubleThrees.length
    ) {
      message = `CẢNH BÁO: đối thủ có ${opponentThreatMoves.size} nước nguy hiểm tiếp theo.`;
      urgent = true;
    } else if (opponentThreatMoves.size) {
      message = `NƯỚC TIẾP THEO CỦA ĐỐI THỦ: ${opponentThreatMoves.size} chấm vàng.`;
    }

    const showWarn = cfg.showWarn !== false;
    banner.textContent = showWarn ? message : '';
    banner.style.display = (showWarn && message) ? 'block' : 'none';
    banner.style.background = urgent
      ? 'rgba(180,35,24,.94)'
      : opponentForks.length ? 'rgba(180,83,9,.94)' : 'rgba(0,91,112,.94)';

    const signature = `${opponentWins.join(';')}|${opponentThreeFours.join(';')}|${opponentTwoTwoOnes.join(';')}|${opponentDoubleThrees.join(';')}|${opponentTwoThrees.join(';')}|${opponentTwoOneOnes.join(';')}|${opponentOpenThrees.join(';')}|${opponentForks.join(';')}`;
    if (cfg.soundOn !== false && message && signature !== lastWarning && (urgent || opponentForks.length)) beep(urgent);
    lastWarning = signature;
  };

  const install = () => {
    const board = document.querySelector('table.table-board');
    if (!board || board === observedBoard) return;
    observer?.disconnect();
    observedBoard = board;
    lastWarning = '';
    const banner = document.querySelector('#caro-threat-warning');
    if (banner) {
      banner.textContent = '';
      banner.style.display = 'none';
    }
    window.caroThreatIdentity = null;
    window.caroThreatState = { ready: false, reason: 'match-loading' };
    const scheduleScan = () => {
      clearTimeout(scanTimer);
      scanTimer = setTimeout(scan, 180);
    };
    const players = document.querySelector('.players-container');
    let observationRoot = board;
    while (
      players && observationRoot.parentElement &&
      !observationRoot.contains(players)
    ) {
      observationRoot = observationRoot.parentElement;
    }
    observer = new MutationObserver(scheduleScan);
    observer.observe(observationRoot, { childList: true, subtree: true });
    scheduleScan();
  };

  install();
  if (location.search.includes('caroDebug')) {
    const tests = runEngineSelfTests();
    console.log(
      '[Caro] Self-test:', tests.passed ? 'PASS ✓' : 'FAIL ✗', tests
    );
  }
  setInterval(install, 1000);
  window.caroThreatScan = scan;
  window.caroFindEngineMove = findEngineMove;
  window.caroEngineSelfTests = runEngineSelfTests;
  window.caroThreatEngineVersion = '3.5.0';
})();
```
