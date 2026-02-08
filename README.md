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
        body { font-family: -apple-system, sans-serif; background-color: var(--light-pink); display: flex; justify-content: center; padding: 40px 20px; margin: 0; }
        .card { background: white; padding: 25px; border-radius: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.1); width: 100%; max-width: 420px; }
        h1 { color: var(--pink); text-align: center; margin-bottom: 20px; }
        .prediction { background: linear-gradient(135deg, #ffebee, #fce4ec); border: 2px solid var(--pink); padding: 20px; border-radius: 20px; text-align: center; margin-bottom: 20px; }
        .prediction p { margin: 0; color: #ad1457; font-weight: bold; }
        .prediction h2 { margin: 10px 0 10px 0; font-size: 1.4rem; color: var(--pink); }
        
        .btn-reminder { background-color: white; color: var(--pink); border: 1px solid var(--pink); padding: 10px; border-radius: 12px; font-size: 0.85rem; font-weight: bold; cursor: pointer; display: none; margin: 10px auto 0 auto; width: 100%; }
        
        .input-box { background: #fdfdfd; padding: 20px; border-radius: 20px; border: 1px solid #eee; text-align: center; }
        label { display: block; margin-bottom: 12px; font-weight: bold; color: #555; }
        input[type="date"] { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 12px; font-size: 1rem; margin-bottom: 10px; box-sizing: border-box; -webkit-appearance: none; }
        button { width: 100%; padding: 15px; border: none; border-radius: 12px; font-size: 1rem; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn-main { background-color: var(--pink); color: white; margin-top: 10px; }
        .btn-today { background-color: #f06292; color: white; padding: 10px; font-size: 0.9rem; margin-bottom: 5px; }

        .history { margin-top: 30px; }
        table { width: 100%; border-collapse: collapse; font-size: 0.8rem; }
        th { text-align: left; color: #888; font-size: 0.7rem; text-transform: uppercase; padding-bottom: 10px; }
        td { border-bottom: 1px solid #f9f9f9; padding: 12px 2px; }
        .btn-del { color: #ff1744; background: none; width: auto; padding: 5px; font-size: 1.1rem; font-weight: bold; }
        .duration-tag { color: #888; font-style: italic; }
    </style>
</head>
<body>

<div class="card">
    <h1>Tante Rosa 🌸</h1>

    <div class="prediction">
        <p>Voraussichtlicher Zeitraum:</p>
        <h2 id="next-range">Berechnung läuft...</h2>
        <button id="remind-btn" class="btn-reminder" onclick="addToCalendar()">📅 Zeitraum im Kalender speichern</button>
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
    let globalNextStart = null;
    let globalNextEnd = null;

    document.addEventListener('DOMContentLoaded', () => {
        const history = getHistory();
        setMode(history.length > 0 && !history[0].end);
        refreshUI();
    });

    function getHistory() { return JSON.parse(localStorage.getItem('periodHistory')) || []; }

    function setMode(waiting) {
        isWaitingForEnd = waiting;
        document.getElementById('input-label').innerText = isWaitingForEnd ? "Wann war es zu Ende?" : "Wann ging es los?";
        document.getElementById('save-btn').innerText = isWaitingForEnd ? "Ende speichern" : "Anfang speichern";
    }

    function setToday() { document.getElementById('date-field').valueAsDate = new Date(); }

    function handleSave() {
        const dateVal = document.getElementById('date-field').value;
        if (!dateVal) return alert("Datum wählen!");
        let history = getHistory();
        if (!isWaitingForEnd) {
            history.unshift({ id: Date.now(), start: dateVal, end: null });
            setMode(true);
        } else {
            if (new Date(dateVal) < new Date(history[0].start)) return alert("Fehler: Ende vor Anfang!");
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

    function formatDateShort(d) { return new Date(d).toLocaleDateString('de-DE', {day:'2-digit', month:'2-digit'}); }
    function formatDateLong(d) { return new Date(d).toLocaleDateString('de-DE', {day:'2-digit', month:'2-digit', year:'numeric'}); }

    function calculate() {
        const history = getHistory().filter(e => e.start && e.end);
        if (history.length < 2) return;

        const sorted = [...history].sort((a,b) => new Date(a.start) - new Date(b.start));
        let intervals = [];
        for (let i = 0; i < sorted.length - 1; i++) {
            const diff = (new Date(sorted[i+1].start) - new Date(sorted[i].start)) / 86400000;
            if (diff >= 20 && diff <= 35) intervals.push(diff);
        }
        const avgCycle = intervals.length > 0 ? intervals.reduce((a,b)=>a+b,0)/intervals.length : 28;
        const avgDur = history.map(e => (new Date(e.end)-new Date(e.start))/86400000).reduce((a,b)=>a+b,0)/history.length;

        let nextStart = new Date(sorted[sorted.length-1].start);
        const today = new Date(); today.setHours(0,0,0,0);
        while (nextStart <= today) { nextStart.setDate(nextStart.getDate() + Math.round(avgCycle)); }
        
        globalNextStart = new Date(nextStart);
        let nextEnd = new Date(nextStart);
        nextEnd.setDate(nextStart.getDate() + Math.round(avgDur));
        globalNextEnd = new Date(nextEnd);

        document.getElementById('next-range').innerText = `${formatDateShort(nextStart)} – ${formatDateShort(nextEnd)}`;
        document.getElementById('remind-btn').style.display = 'block';
    }

    function addToCalendar() {
        if (!globalNextStart || !globalNextEnd) return;
        const formatDateICS = (date) => date.toISOString().split('T')[0].replace(/-/g, "");
        const start = formatDateICS(globalNextStart);
        const calendarEnd = new Date(globalNextEnd);
        calendarEnd.setDate(calendarEnd.getDate() + 1);
        const end = formatDateICS(calendarEnd);

        const icsContent = [
            "BEGIN:VCALENDAR", "VERSION:2.0", "BEGIN:VEVENT",
            `DTSTART;VALUE=DATE:${start}`, `DTEND;VALUE=DATE:${end}`,
            "SUMMARY:Tante Rosa 🌸", "DESCRIPTION:Voraussichtlicher Zeitraum",
            "BEGIN:VALARM", "TRIGGER:-P1D", "ACTION:DISPLAY", "DESCRIPTION:Tante Rosa kommt morgen", "END:VALARM",
            "END:VEVENT", "END:VCALENDAR"
        ].join("%0A");

        window.open("data:text/calendar;charset=utf8," + icsContent);
    }

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
