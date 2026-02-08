<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <title>Aunt Rosa</title>
    <style>
        :root { --pink: #d81b60; --light-pink: #fff0f5; }
        body { font-family: -apple-system, sans-serif; background-color: var(--light-pink); display: flex; justify-content: center; padding: 40px 20px; margin: 0; }
        .card { background: white; padding: 25px; border-radius: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); width: 100%; max-width: 420px; }
        h1 { color: var(--pink); text-align: center; margin-bottom: 20px; }
        .prediction { background: linear-gradient(135deg, #ffebee, #fce4ec); border: 2px solid var(--pink); padding: 20px; border-radius: 20px; text-align: center; margin-bottom: 20px; }
        .prediction p { margin: 0; color: #ad1457; font-weight: bold; }
        .prediction h2 { margin: 10px 0 10px 0; font-size: 1.4rem; color: var(--pink); }
        
        .input-box { background: #fdfdfd; padding: 20px; border-radius: 20px; border: 1px solid #eee; text-align: center; }
        label { display: block; margin-bottom: 12px; font-weight: bold; color: #555; }
        input[type="date"] { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 12px; font-size: 1rem; margin-bottom: 10px; box-sizing: border-box; -webkit-appearance: none; background: white; }
        
        button { width: 100%; padding: 15px; border: none; border-radius: 12px; font-size: 1rem; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn-main { background-color: var(--pink); color: white; margin-top: 10px; }
        .btn-today { background-color: #f06292; color: white; padding: 10px; font-size: 0.9rem; margin-bottom: 5px; }

        .history { margin-top: 30px; }
        table { width: 100%; border-collapse: collapse; font-size: 0.8rem; }
        th { text-align: left; color: #888; font-size: 0.7rem; text-transform: uppercase; padding-bottom: 10px; }
        td { border-bottom: 1px solid #f9f9f9; padding: 12px 2px; height: 50px; }
        
        .row-virtual { font-style: italic; color: #888; }
        .btn-edit { background: none; font-size: 1.2rem; cursor: pointer; border: none; padding: 5px; }
        .btn-del { color: #ff1744; background: none; border: none; font-size: 1.1rem; cursor: pointer; }
        .duration-tag { color: #888; font-style: italic; }
    </style>
</head>
<body>

<div class="card">
    <h1 id="main-title">Aunt Rosa 🍆</h1>

    <div class="prediction">
        <p>Nächste Periode erwartet:</p>
        <h2 id="next-range">Berechnung läuft...</h2>
    </div>

    <div class="input-box">
        <label id="input-label">Wann ging es los?</label>
        <input type="date" id="date-field">
        <button class="btn-today" onclick="setToday()">Heute</button>
        <button class="btn-main" id="save-btn" onclick="handleSave()">Anfang speichern</button>
    </div>

    <div class="history">
        <table id="history-table">
            <thead>
                <tr>
                    <th>Von</th>
                    <th>Bis</th>
                    <th>Dauer</th>
                    <th></th>
                </tr>
            </thead>
            <tbody></tbody>
        </table>
    </div>
</div>

<script>
    let isWaitingForEnd = false;
    let cachedAvgCycle = 28;
    let cachedAvgDuration = 5;

    document.addEventListener('DOMContentLoaded', () => {
        refreshUI();
    });

    function getHistory() { return JSON.parse(localStorage.getItem('periodHistory')) || []; }

    function setMode(waiting) {
        isWaitingForEnd = waiting;
        document.getElementById('input-label').innerText = isWaitingForEnd ? "Wann war es zu Ende?" : "Wann ging es los?";
        document.getElementById('save-btn').innerText = isWaitingForEnd ? "Ende speichern" : "Anfang speichern";
        document.getElementById('main-title').innerText = isWaitingForEnd ? "Aunt Rosa 🌸" : "Aunt Rosa 🍆";
    }

    function setToday() { document.getElementById('date-field').valueAsDate = new Date(); }

    function handleSave() {
        const dateVal = document.getElementById('date-field').value;
        if (!dateVal) return alert("Datum wählen!");
        let history = getHistory();
        
        if (!isWaitingForEnd) {
            history.unshift({ id: Date.now(), start: dateVal, end: null });
        } else {
            if (new Date(dateVal) < new Date(history[0].start)) return alert("Fehler: Ende vor Anfang!");
            history[0].end = dateVal;
        }
        
        localStorage.setItem('periodHistory', JSON.stringify(history));
        document.getElementById('date-field').value = "";
        refreshUI();
    }

    function refreshUI() {
        calculateStats();
        const history = getHistory();
        setMode(history.length > 0 && !history[0].end);
        renderHistory();
    }

    function calculateStats() {
        const history = getHistory().filter(e => e.start && e.end);
        if (history.length < 2) {
            cachedAvgCycle = 28;
            cachedAvgDuration = 5;
            return;
        }
        const sorted = [...history].sort((a,b) => new Date(a.start) - new Date(b.start));
        let cycleLengths = [];
        for (let i = 0; i < sorted.length - 1; i++) {
            const diff = (new Date(sorted[i+1].start) - new Date(sorted[i].start)) / 86400000;
            if (diff >= 20 && diff <= 45) cycleLengths.push(diff);
        }
        let count = cycleLengths.length > 10 ? 12 : (cycleLengths.length > 4 ? 6 : 3);
        const recentCycles = cycleLengths.slice(-count);
        cachedAvgCycle = recentCycles.length > 0 ? recentCycles.reduce((a,b)=>a+b,0)/recentCycles.length : 28;
        const durations = history.map(e => (new Date(e.end)-new Date(e.start))/86400000 + 1);
        cachedAvgDuration = durations.reduce((a,b)=>a+b,0)/durations.length;
    }

    function renderHistory() {
        const history = getHistory();
        const tbody = document.querySelector('#history-table tbody');
        tbody.innerHTML = '';
        const today = new Date();
        today.setHours(0,0,0,0);

        if (history.length > 0) {
            let lastStart = new Date(history[0].start);
            let nextPredictedStart = new Date(lastStart);
            while (nextPredictedStart <= today) {
                nextPredictedStart.setDate(nextPredictedStart.getDate() + Math.round(cachedAvgCycle));
            }
            let nextPredictedEnd = new Date(nextPredictedStart);
            nextPredictedEnd.setDate(nextPredictedStart.getDate() + Math.round(cachedAvgDuration) - 1);
            document.getElementById('next-range').innerText = `${formatDateShort(nextPredictedStart)} – ${formatDateShort(nextPredictedEnd)}`;

            // VORSCHLAG-ZEILE (Wird nur angezeigt, wenn kein aktueller Zyklus läuft)
            let expectedStart = new Date(lastStart);
            expectedStart.setDate(lastStart.getDate() + Math.round(cachedAvgCycle));
            
            if (!isWaitingForEnd && expectedStart <= today) {
                let virtualEnd = new Date(expectedStart);
                virtualEnd.setDate(expectedStart.getDate() + Math.round(cachedAvgDuration) - 1);
                
                tbody.innerHTML += `<tr class="row-virtual">
                    <td>${formatDateLong(expectedStart)}</td>
                    <td>${formatDateLong(virtualEnd)}</td>
                    <td>Vorschlag</td>
                    <td align="right">
                        <button class="btn-edit" onclick="triggerEdit('${expectedStart.toISOString().split('T')[0]}')">✏️</button>
                    </td>
                </tr>`;
            }
        }

        history.forEach(entry => {
            let durationText = "";
            if(entry.start && entry.end) {
                const diff = Math.round((new Date(entry.end) - new Date(entry.start)) / 86400000) + 1;
                durationText = diff + " Tage";
            }
            tbody.innerHTML += `<tr>
                <td>${formatDateLong(entry.start)}</td>
                <td>${entry.end ? formatDateLong(entry.end) : '...'}</td>
                <td class="duration-tag">${durationText}</td>
                <td align="right"><button class="btn-del" onclick="deleteItem(${entry.id})">✕</button></td>
            </tr>`;
        });
    }

    function triggerEdit(dateStr) {
        const field = document.getElementById('date-field');
        field.value = dateStr;
        // Fokus und Klick simulieren, um Kalender auf Mobilgeräten zu öffnen
        field.focus();
        field.click(); 
    }

    function formatDateShort(d) { return new Date(d).toLocaleDateString('de-DE', {day:'2-digit', month:'2-digit'}); }
    function formatDateLong(d) { return new Date(d).toLocaleDateString('de-DE', {day:'2-digit', month:'2-digit', year:'numeric'}); }

    function deleteItem(id) {
        if(confirm("Eintrag löschen?")) {
            let history = getHistory().filter(e => e.id !== id);
            localStorage.setItem('periodHistory', JSON.stringify(history));
            refreshUI();
        }
    }
</script>
</body>
</html>
