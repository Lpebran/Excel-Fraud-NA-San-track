# Excel-Fraud-NA-San-track
Agent will be able to just click the action and the tool will save it in to an excel with the time,action, task id and notes if needed


// ==UserScript==
// @name         Paragon Tracker v4.5 Final (Minimizable, Transparente, CSV, Meta Diario)
// @namespace    http://tampermonkey.net/
// @version      4.5
// @description  Panel para seguimiento manual con contador, notas, duración, minimizable y exportable. Hecho por Lpebran.
// @match        https://paragon-na.amazon.com/hz/investigation*
// @grant        none
// ==/UserScript==

(function () {
    'use strict';

    const STORAGE_KEY = 'paragonManualTracker';
    const UI_STATE_KEY = 'paragonManualUIState';
    const NOTE_KEY = 'paragonNotes';
    const OFFLINE_STATE_KEY = 'paragonOfflineState';
    const TASK_START_KEY = 'paragonTaskStartTimes';

    const ACTIONS = [
        { label: "Fraud", key: "Fraud", color: "crimson" },
        { label: "No Action", key: "NoAction", color: "orange" },
        { label: "Sanitize", key: "Sanitize", color: "seagreen" },
    ];

    const TOTAL_WORK_MINUTES = (9 * 60) - (60 + 30);
    const CASES_PER_MINUTE = 18 / 60;

    function todayStr() {
        return new Date().toISOString().slice(0, 10);
    }

    function load(key, fallback = {}) {
        return JSON.parse(localStorage.getItem(key) || JSON.stringify(fallback));
    }

    function save(key, value) {
        localStorage.setItem(key, JSON.stringify(value));
    }

    function getCID() {
        return new URLSearchParams(window.location.search).get('caseId') || 'unknown';
    }

    function markTaskStart(cid) {
        const tasks = load(TASK_START_KEY);
        tasks[cid] = Date.now();
        save(TASK_START_KEY, tasks);
    }

    function getTaskDuration(cid) {
        const tasks = load(TASK_START_KEY);
        const start = tasks[cid];
        if (!start) return null;
        const duration = Math.round((Date.now() - start) / 1000);
        delete tasks[cid];
        save(TASK_START_KEY, tasks);
        return duration;
    }

    function registerAction(actionKey, btn) {
        const cid = getCID();
        const notes = load(NOTE_KEY);
        const note = notes[cid] || '';
        const duration = getTaskDuration(cid) ?? '';
        const data = load(STORAGE_KEY, { date: todayStr(), counts: {}, offlineMinutes: 0 });
        if (!data.counts[actionKey]) data.counts[actionKey] = [];
        data.counts[actionKey].push({ caseId: cid, time: new Date().toISOString(), note, duration });
        save(STORAGE_KEY, data);
        flash(btn);
        updateStats();
    }

    function flash(btn) {
        if (!btn) return;
        const original = btn.style.backgroundColor;
        btn.style.backgroundColor = 'limegreen';
        setTimeout(() => btn.style.backgroundColor = original, 200);
    }

    function updateStats() {
        const data = load(STORAGE_KEY);
        let total = 0;
        ACTIONS.forEach(a => total += (data.counts[a.key] || []).length);
        const state = load(OFFLINE_STATE_KEY);
        let offlineMin = data.offlineMinutes || 0;
        if (state.isOffline && state.offlineStart) {
            offlineMin += Math.round((Date.now() - new Date(state.offlineStart)) / 60000);
        }

        const adjustedMin = TOTAL_WORK_MINUTES - offlineMin;
        const adjustedGoal = Math.round(CASES_PER_MINUTE * adjustedMin);
        const remaining = Math.max(0, adjustedGoal - total);
        const percent = Math.min((total / adjustedGoal) * 100, 100);

        const barColor = percent >= 100 ? '#2ecc71' : percent >= 70 ? '#f1c40f' : '#e74c3c';

        const el = document.getElementById('metaBox');
        if (el) {
            el.innerHTML = `
                <div style="color:black;font-weight:bold;font-size:13px;">🎯 Meta diaria: ${adjustedGoal}</div>
                <div style="color:black;font-weight:bold;font-size:13px;">📌 Hechos: ${total}</div>
                <div style="color:black;font-weight:bold;font-size:13px;">🧮 Faltan: ${remaining}</div>
                <div style="margin-top:5px; background:#ccc; border-radius:6px; height:10px;">
                    <div style="width:${percent}%; height:100%; background:${barColor}; transition: width 0.5s;"></div>
                </div>`;
        }
    }

    function saveNote(cid, note) {
        const notes = load(NOTE_KEY);
        notes[cid] = note;
        save(NOTE_KEY, notes);
    }

    function updateOfflineBtn() {
        const state = load(OFFLINE_STATE_KEY, { isOffline: false });
        const btn = document.getElementById('statusBtn');
        btn.textContent = state.isOffline ? '🛑 Offline' : '✅ Available';
        btn.style.backgroundColor = state.isOffline ? '#aa0000' : '#007700';
    }

    function toggleOffline() {
        const state = load(OFFLINE_STATE_KEY, { isOffline: false });
        const now = new Date();
        if (!state.isOffline) {
            state.isOffline = true;
            state.offlineStart = now.toISOString();
        } else {
            const started = new Date(state.offlineStart);
            const diffMin = Math.round((now - started) / 60000);
            const data = load(STORAGE_KEY);
            data.offlineMinutes = (data.offlineMinutes || 0) + diffMin;
            save(STORAGE_KEY, data);
            state.isOffline = false;
            state.offlineStart = null;
        }
        save(OFFLINE_STATE_KEY, state);
        updateOfflineBtn();
        updateStats();
    }

    function resetAll() {
        localStorage.removeItem(STORAGE_KEY);
        localStorage.removeItem(OFFLINE_STATE_KEY);
        updateStats();
        updateOfflineBtn();
    }

    function deleteLast() {
        const data = load(STORAGE_KEY);
        for (let i = ACTIONS.length - 1; i >= 0; i--) {
            const key = ACTIONS[i].key;
            if (data.counts[key] && data.counts[key].length > 0) {
                data.counts[key].pop();
                break;
            }
        }
        save(STORAGE_KEY, data);
        updateStats();
    }

    function downloadCSV() {
        const data = load(STORAGE_KEY);
        let csv = `Acción,Case ID,Fecha,Nota,Duración (s)\n`;
        ACTIONS.forEach(a => {
            (data.counts[a.key] || []).forEach(e => {
                csv += `"${a.label}","${e.caseId}","${e.time}","${(e.note || '').replace(/"/g, '""')}","${e.duration ?? ''}"\n`;
            });
        });
        const blob = new Blob([csv], { type: 'text/csv' });
        const a = document.createElement('a');
        a.href = URL.createObjectURL(blob);
        a.download = 'paragon_tracker.csv';
        document.body.appendChild(a);
        a.click();
        a.remove();
    }

    function createUI() {
        const saved = load(UI_STATE_KEY);
        const wrapper = document.createElement('div');
        wrapper.id = 'trackerPanel';
        wrapper.style = `
            position:fixed;top:${saved.top || '20px'};left:${saved.left || '20px'};
            background:white;border:1px solid #888;border-radius:8px;
            font-family:Arial;font-size:12px;z-index:99999;padding:8px;
            box-shadow:2px 2px 5px rgba(0,0,0,0.2);width:300px;
            display:flex;flex-direction:column;gap:5px;
        `;

        // Botones de acción
        const buttons = ACTIONS.map(a => {
            const btn = document.createElement('button');
            btn.textContent = a.label;
            btn.style = `background:${a.color};color:white;padding:6px;border:none;border-radius:6px;cursor:pointer`;
            btn.onclick = () => registerAction(a.key, btn);
            return btn;
        });

        // Meta + nota
        const metaBox = document.createElement('div');
        metaBox.id = 'metaBox';
        metaBox.style = 'font-size:12px; color:black;';

        const textarea = document.createElement('textarea');
        textarea.id = 'noteBox';
        textarea.style = 'width:100%;height:60px;border-radius:6px;padding:5px;resize:none';
        textarea.placeholder = 'Escribe tu nota aquí...';
        const currentCID = getCID();
        textarea.value = load(NOTE_KEY)[currentCID] || '';
        textarea.oninput = () => saveNote(currentCID, textarea.value);

        // Offline toggle
        const statusBtn = document.createElement('button');
        statusBtn.id = 'statusBtn';
        statusBtn.style = 'padding:5px;background:#007700;color:white;border:none;border-radius:6px;cursor:pointer';
        statusBtn.onclick = toggleOffline;

        // Herramientas
        const tools = document.createElement('div');
        tools.style = 'display:flex;gap:5px;flex-wrap:wrap;margin-top:5px;';
        const delBtn = btn('⛔ Eliminar Último', '#c0392b', deleteLast);
        const resetBtn = btn('🗑️ Reset', '#999', resetAll);
        const csvBtn = btn('📥 CSV', '#444', downloadCSV);
        tools.append(delBtn, resetBtn, csvBtn);

        // Minimizar y transparente
        const topBar = document.createElement('div');
        topBar.style = 'display:flex;justify-content:space-between;align-items:center';
        const dragHandle = document.createElement('div');
        dragHandle.textContent = '📌 Tracker';
        dragHandle.style = 'cursor:move;font-weight:bold';
        const toggleBox = document.createElement('div');
        toggleBox.innerHTML = `
            <button id="minBtn" style="font-size:10px;margin-right:4px;">−</button>
            <button id="transBtn" style="font-size:10px;">☀️</button>
        `;
        topBar.append(dragHandle, toggleBox);

        wrapper.append(topBar, ...buttons, statusBtn, metaBox, textarea, tools);
        document.body.appendChild(wrapper);

        // Botón minimizar y transparente
        const minBtn = document.getElementById('minBtn');
        const transBtn = document.getElementById('transBtn');
        let minimized = saved.minimized || false;
        let transparent = saved.transparent || false;

        const toggleUI = () => {
            const show = !minimized;
            [...buttons, statusBtn, metaBox, textarea, tools].forEach(e => e.style.display = show ? '' : 'none');
            minBtn.textContent = show ? '−' : '+';
        };

        minBtn.onclick = () => {
            minimized = !minimized;
            toggleUI();
            save(UI_STATE_KEY, { ...load(UI_STATE_KEY), minimized });
        };

        transBtn.onclick = () => {
            transparent = !transparent;
            wrapper.style.background = transparent ? 'rgba(255,255,255,0.5)' : 'white';
            save(UI_STATE_KEY, { ...load(UI_STATE_KEY), transparent });
        };

        // Drag
        let offsetX, offsetY, dragging = false;
        dragHandle.onmousedown = e => {
            dragging = true;
            offsetX = e.clientX - wrapper.offsetLeft;
            offsetY = e.clientY - wrapper.offsetTop;
        };
        document.onmousemove = e => {
            if (!dragging) return;
            wrapper.style.left = `${e.clientX - offsetX}px`;
            wrapper.style.top = `${e.clientY - offsetY}px`;
        };
        document.onmouseup = () => {
            if (dragging) {
                dragging = false;
                save(UI_STATE_KEY, { ...load(UI_STATE_KEY), top: wrapper.style.top, left: wrapper.style.left });
            }
        };

        toggleUI();
        updateOfflineBtn();
        updateStats();
    }

    function btn(label, bg, onclick) {
        const b = document.createElement('button');
        b.textContent = label;
        b.style = `padding:5px;flex:1;border:none;border-radius:6px;background:${bg};color:white;cursor:pointer`;
        b.onclick = () => {
            onclick();
            flash(b);
        };
        return b;
    }

    window.addEventListener('load', () => {
        setTimeout(() => {
            createUI();
            markTaskStart(getCID());
        }, 1000);
    });
})();
