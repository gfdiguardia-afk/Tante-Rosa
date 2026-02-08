<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, viewport-fit=cover">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <title>Tante Rosa</title>
    <style>
        :root { --pink: #d81b60; --light-pink: #fff0f5; }
        body { 
            font-family: -apple-system, BlinkMacSystemFont, sans-serif; 
            background-color: var(--light-pink); 
            display: flex; 
            justify-content: center; 
            padding: 40px 20px; 
            margin: 0; 
        }
        .card { 
            background: white; 
            padding: 25px; 
            border-radius: 25px; 
            box-shadow: 0 10px 30px rgba(0,0,0,0.1); 
            width: 100%; 
            max-width: 400px; 
        }
        h1 { color: var(--pink); text-align: center; margin-bottom: 20px; font-size: 1.8rem; }
        
        .prediction { 
            background: linear-gradient(135deg, #ffebee, #fce4ec); 
            border: 2px solid var(--pink); 
            padding: 20px; 
            border-radius: 20px; 
            text-align: center; 
            margin-bottom: 25px; 
        }
        .prediction p { margin: 0; color: #ad1457; font-weight: bold; font-size: 0.9rem; }
        .prediction h2 { margin: 10px 0 0 0; font-size: 1.4rem; color: var(--pink); }

        .input-box { 
            background: #fdfdfd; 
            padding: 20px; 
            border-radius: 20px; 
            border: 1px solid #eee; 
            text-align: center; 
            box-shadow: inset 0 2px 4px rgba(0,0,0,0.02);
        }
        label { display: block; margin-bottom: 12px; font-weight: bold; color: #555; font-size: 1.1rem; }
        input[type="date"] { 
            width: 100%; 
            padding: 12px; 
            border: 1px solid #ddd; 
            border-radius: 12px; 
            font-size: 1rem; 
            margin-bottom: 10px; 
            box-sizing: border-box;
            -webkit-appearance: none;
        }
        button { width: 100%; padding: 15px; border: none; border-radius: 12px; font-size: 1rem; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn-main { background-color: var(--pink); color: white; margin-top: 10px; box-shadow: 0 4px 10px rgba(216, 27, 96, 0.3); }
        .btn-main:active { transform: scale(0.98); }
        .btn-today { background-color: #f06292; color: white; padding: 10px; font-size: 0.9rem; }

        .history { margin-top: 35px; }
        .history h3 { font-size: 1rem; color: #888; border-bottom: 1px solid #eee; padding-bottom: 10px; }
        table { width: 100%; border-collapse: collapse; font-size: 0.9rem; }
        th, td { border-bottom: 1px solid #f9f9f9; padding: 12px 5px; text-align: left; }
        .btn-del { color: #ccc; background: none; font-size: 1.2rem; width: auto; padding: 0; font-weight: normal; }
        .btn-del:hover { color: var(--pink); }
    </style>
</head>
<body>

<div class="card">
    <h1>Tante Rosa 🌸</h1>

    <div class="prediction">
        <p id="predict-label">Voraussichtlicher Zeitraum:</p>
        <h2 id="next-range">Berechnung läuft...</h2>
    </div>

    <div class="input-box">
        <label id="input-label">Wann ging es los?</label>
        <input type="date" id="date-field">
        <button class="btn-today" onclick="setToday()">Heute</button>
        <button class="btn-main" id="save-btn" onclick="handleSave()">Anfang speichern</button>
    </div>

    <div class="history">
        <h3>Verlauf</h3>
        <table id="history-table">
            <thead><tr><th>Von</th><th>Bis</th><th></th></tr></thead>
            <tbody></tbody>
        </table>
    </div>
    
    <button onclick="clearAll()" style="background:none; color:#ddd; font-size:0.7rem; margin-top:30px; font-weight: normal;">Daten zurücksetzen</button>
</div>

<script>
    let isWaitingForEnd = false;

    // App Start
    document.addEventListener('DOMContentLoaded', () => {
        const history = getHistory();
        // Modus-Check: Fehlt beim letzten Eintrag das Enddatum?
        if (history.length > 0 && !history[0].end) {
            setMode(true);
        } else {
            setMode(false);
        }
        refreshUI();
    });

    function getHistory() {
        return JSON.parse(localStorage.getItem('periodHistory')) || [];
    }

    function setMode(waitingForEnd) {
        isWaitingForEnd = waitingForEnd;
        const label = document.getElementById('input-label');
        const btn = document.getElementById('save-btn');
        if (isWaitingForEnd) {
            label.innerText = "Wann war es zu Ende?";
            btn.innerText = "Ende speichern";
        } else {
            label.innerText = "Wann ging es los?";
            btn.innerText = "Anfang speichern";
        }
    }

    function setToday() {
        document.getElementById('date-field').valueAsDate = new Date();
    }

    function handleSave() {
        const dateVal = document.getElementById('date-field').value;
        if (!dateVal) return alert("Bitte wähle ein Datum!");

        let history = getHistory();

        if (!isWaitingForEnd) {
            // Neuer Eintrag: Startdatum
            history.unshift({ id: Date.now(), start: dateVal, end: null });
            setMode(true);
        } else {
            // Bestehender Eintrag: Enddatum hinzufügen
            if (new Date(dateVal) < new Date(history[0].start)) {
                return alert("Das Ende kann nicht vor dem Anfang liegen!");
            }
            history[0].end = dateVal;
            setMode(false);
        }

        localStorage.setItem('periodHistory', JSON.stringify(history));
        document.getElementById('date-field').value = "";
        refreshUI();
    }

    function refreshUI() {
        renderHistory();
        calculate();
    }

    function renderHistory() {
        const history = getHistory();
        const tbody = document.querySelector('#history-table tbody');
        tbody.innerHTML = '';
        history.forEach(entry => {
            tbody.innerHTML += `<tr>
                <td>${formatDate(entry.start)}</td>
                <td>${entry.end ? formatDate(entry.end) : '<i>läuft...</i>'}</td>
                <td style="text-align:right"><button class="btn-del" onclick="deleteItem(${entry.id})">✕</button></td>
            </tr>`;
        });
    }

    function formatDate(d) {
        return new Date(d).toLocaleDateString('de-DE', {day:'2-digit', month:'2-digit'});
    }

    function calculate() {
        const history = getHistory().filter(e => e.start && e.end);
        const display = document.getElementById('next-range');
        
        if (history.length < 2) {
            display.innerText = "Mehr Daten nötig";
            return;
        }

        // Chronologisch sortieren (alt nach neu)
        const sorted = [...history].sort((a,b) => new Date(a.start) - new Date(b.start));
        
        // 1. Durchschnittliche Zykluslänge (Start zu Start)
        let intervals = [];
        for (let i = 0; i < sorted.length - 1; i++) {
            const diff = (new Date(sorted[i+1].start) - new Date(sorted[i].start)) / (1000*60*60*24);
            // FILTER: Nur Intervalle zwischen 20 und 35 Tagen berücksichtigen
            if (diff >= 20 && diff <= 35) {
                intervals.push(diff);
            }
        }
        
        // Fallback: Wenn keine Intervalle im 35-Tage-Fenster, nutze 28 als Standard
        const avgCycle = intervals.length > 0 ? intervals.reduce((a,b)=>a+b,0)/intervals.length : 28;

        // 2. Durchschnittliche Periodendauer (Tage der Blutung)
        let durations = history.map(e => (new Date(e.end)-new Date(e.start))/(1000*60*60*24));
        const avgDuration = durations.reduce((a,b)=>a+b,0)/durations.length;

        // 3. Vorhersage-Logik
        let lastKnownStart = new Date(sorted[sorted.length-1].start);
        let nextStart = new Date(lastKnownStart);
        const today = new Date();
        today.setHours(0,0,0,0);

        // Falls das Datum in der Vergangenheit liegt, Zyklus addieren (Schutz vor Lücken)
        while (nextStart < today) {
            nextStart.setDate(nextStart.getDate() + Math.round(avgCycle));
        }

        let nextEnd = new Date(nextStart);
        nextEnd.setDate(nextStart.getDate() + Math.round(avgDuration));

        display.innerText = `${formatDate(nextStart)} – ${formatDate(nextEnd)}`;
    }

    function deleteItem(id) {
        if(confirm("Eintrag löschen?")) {
            let history = getHistory().filter(e => e.id !== id);
            localStorage.setItem('periodHistory', JSON.stringify(history));
            location.reload();
        }
    }

    function clearAll() {
        if(confirm("Bist du sicher? Alle Daten gehen verloren!")) {
            localStorage.clear();
            location.reload();
        }
    }
</script>
</body>
</html>
