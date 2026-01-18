<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>RADICAL BOX-IT</title>
    <style>
        :root { --pink: #FF00FF; --purple: #9D00FF; --teal: #00DBDE; --silver: #C0C0C0; --chartreuse: #7FFF00; --black: #000000; }
        body { background-color: white; font-family: sans-serif; display: flex; flex-direction: column; align-items: center; margin: 0; padding: 0; min-height: 100vh; border: 10px solid; border-image: linear-gradient(45deg, var(--pink), var(--teal), var(--chartreuse), var(--purple)) 1; overflow-x: hidden; }
        header { text-align: center; width: 100%; height: 180px; background: white; border-bottom: 4px solid black; display: flex; flex-direction: column; align-items: center; justify-content: center; }
        h1 { font-size: 1.8rem; margin: 0; font-style: italic; text-shadow: 2px 2px var(--pink); text-transform: uppercase; font-weight: 900; }
        #top-controls { display: none; flex-direction: column; align-items: center; gap: 5px; }
        .dice-area { display: flex; align-items: center; gap: 10px; }
        .die { width: 50px; height: 50px; border: 3px solid black; border-radius: 8px; display: flex; align-items: center; justify-content: center; font-weight: 900; font-size: 0.8rem; background: #eee; }
        #status-bar { background: var(--chartreuse); font-weight: 900; padding: 5px 10px; border: 2px solid black; font-size: 1rem; width: 260px; text-align: center; margin: 5px 0; }
        .btn-group { display: flex; gap: 8px; }
        button { padding: 10px 15px; font-size: 1rem; font-weight: 900; cursor: pointer; background: var(--pink); border: 3px solid black; box-shadow: 3px 3px 0 var(--teal); text-transform: uppercase; color: white; text-shadow: 1px 1px black; }
        button:disabled { background: #bbb; box-shadow: none; opacity: 0.6; }
        #setup-screen { text-align: center; background: white; padding: 20px; border: 5px solid black; box-shadow: 8px 8px 0 var(--purple); width: 85%; max-width: 320px; margin-top: 20px; }
        .p-row { margin: 15px 0; font-weight: 900; display: flex; justify-content: space-between; align-items: center; }
        .p-row input { padding: 10px; width: 75%; border: 3px solid black; font-weight: bold; }
        #game-container { display: none; flex-direction: column; align-items: center; margin-top: 10px; }
        #board { position: relative; display: grid; grid-template-columns: repeat(7, 42px); grid-template-rows: repeat(7, 42px); gap: 12px; padding: 15px; border: 4px solid black; background: white; }
        .dot { width: 20px; height: 20px; border-radius: 50%; border: 2px solid #333; z-index: 10; position: relative; }
        .dot.selected { border: 3px solid black; background-color: var(--black) !important; transform: scale(1.1); }
        .line { position: absolute; background: black; z-index: 5; pointer-events: none; border-radius: 4px; }
        .line.h { height: 8px; margin-top: -4px; }
        .line.v { width: 8px; margin-left: -4px; }
        .box-fill { position: absolute; display: flex; align-items: center; justify-content: center; font-weight: 900; font-size: 1.6rem; color: var(--pink); text-shadow: 2px 2px black; z-index: 2; pointer-events: none; }
    </style>
</head>
<body>

<header>
    <h1>RADICAL BOX-IT</h1>
    <div id="top-controls">
        <div id="status-bar">Ready!</div>
        <div class="dice-area">
            <div id="die1" class="die">?</div>
            <button id="roll-btn" onclick="rollDice()">ROLL</button>
            <div id="die2" class="die">?</div>
        </div>
        <div class="btn-group">
            <button id="reroll-btn" style="visibility:hidden;" onclick="reRollOne()">RE-ROLL</button>
            <button id="pass-btn" style="visibility:hidden;" onclick="endTurn()">PASS</button>
        </div>
    </div>
</header>

<div id="setup-screen">
    <h2 style="margin:0">FAMILY EDITION</h2>
    <div class="p-row">1. <input id="n1" value="Dad"></div>
    <div class="p-row">2. <input id="n2" value="Mom"></div>
    <div class="p-row">3. <input id="n3" value="Ash"></div>
    <button onclick="startGame()">START GAME</button>
</div>

<div id="game-container">
    <div id="board"></div>
</div>

<script>
    var colors = ['pink', 'purple', 'teal', 'silver', 'chartreuse'];
    var css = { pink:'#FF00FF', purple:'#9D00FF', teal:'#00DBDE', silver:'#C0C0C0', chartreuse:'#7FFF00', black:'#000000' };
    var players = [], turnIdx = 0, roll = [], selIdx = null, dots = [], lines = new Set(), dblWild = 0;

    function startGame() {
        players = [];
        for(var i=1; i<=3; i++) {
            var n = document.getElementById('n'+i).value.trim();
            if(n) players.push({n: n, initial: n.charAt(0).toUpperCase(), color: colors[i-1]});
        }
        document.getElementById('setup-screen').style.display = 'none';
        document.getElementById('top-controls').style.display = 'flex';
        document.getElementById('game-container').style.display = 'flex';
        initBoard();
        updateStatus();
    }

    function initBoard() {
        var b = document.getElementById('board'); b.innerHTML = ''; dots = []; lines.clear();
        for(var i=0; i<49; i++) {
            var c = colors[Math.floor(Math.random()*5)];
            var d = document.createElement('div'); d.className = 'dot'; d.style.background = css[c];
            d.onclick = (function(idx) { return function() { handleDot(idx); }; })(i);
            b.appendChild(d); dots.push({c: c, el: d});
        }
    }

    function rollDice() {
        var d1 = Math.floor(Math.random()*6), d2 = Math.floor(Math.random()*6);
        roll = [d1===5?'black':colors[d1], d2===5?'black':colors[d2]];
        updateDie('die1', roll[0]); updateDie('die2', roll[1]);
        document.getElementById('roll-btn').disabled = true;
        document.getElementById('reroll-btn').style.visibility = 'visible';
        document.getElementById('pass-btn').style.visibility = 'visible';
        dblWild = (roll[0]==='black' && roll[1]==='black') ? 2 : 0;
        updateStatus(dblWild ? "DOUBLE WILD!" : "TAP 2 DOTS");
    }

    function handleDot(i) {
        if(roll.length === 0) return;
        if(selIdx === null) { selIdx = i; dots[i].el.classList.add('selected'); }
        else {
            if(selIdx === i) { dots[i].el.classList.remove('selected'); selIdx = null; return; }
            var key = [selIdx, i].sort(function(a,b){return a-b;}).join('-');
            if(isValid(selIdx, i) && !lines.has(key)) {
                lines.add(key); drawLine(selIdx, i);
                var boxes = checkBoxes();
                if(dblWild > 1) { dblWild--; updateStatus("2ND LINE!"); }
                else if(dblWild === 1) { dblWild = 0; if(boxes > 0) resetTurn(false); else endTurn(); }
                else { endTurn(); }
            }
            dots[selIdx].el.classList.remove('selected'); selIdx = null;
        }
    }

    function isValid(a, b) {
        var r1=Math.floor(a/7), c1=a%7, r2=Math.floor(b/7), c2=b%7;
        if(Math.abs(r1-r2)+Math.abs(c1-c2) !== 1) return false;
        if(dblWild > 0) return true;
        var dc1=dots[a].c, dc2=dots[b].c, dr1=roll[0], dr2=roll[1];
        return (((dr1==='black'||dc1===dr1)&&(dr2==='black'||dc2===dr2))||((dr1==='black'||dc2===dr1)&&(dr2==='black'||dc1===dr2)));
    }

    function drawLine(a, b) {
        var d1=dots[a].el, d2=dots[b].el, l=document.createElement('div'); l.className='line';
        var x1=d1.offsetLeft+10, y1=d1.offsetTop+10, x2=d2.offsetLeft+10, y2=d2.offsetTop+10;
        if(y1===y2){ l.className+=' h'; l.style.width='56px'; l.style.left=Math.min(x1,x2)+'px'; l.style.top=y1+'px'; }
        else { l.className+=' v'; l.style.height='56px'; l.style.top=Math.min(y1,y2)+'px'; l.style.left=x1+'px'; }
        document.getElementById('board').appendChild(l);
    }

    function checkBoxes() {
        var count = 0;
        for(var r=0; r<6; r++) { for(var c=0; c<6; c++) {
            var tl=r*7+c, tr=tl+1, bl=(r+1)*7+c, br=bl+1;
            if(lines.has(tl+'-'+tr) && lines.has(bl+'-'+br) && lines.has(tl+'-'+bl) && lines.has(tr+'-'+br) && !document.getElementById('box'+tl)) {
                var f = document.createElement('div'); f.id='box'+tl; f.className='box-fill';
                f.innerText=players[turnIdx].initial; f.style.background=css[players[turnIdx].color];
                f.style.left=(dots[tl].el.offsetLeft+13)+'px'; f.style.top=(dots[tl].el.offsetTop+13)+'px';
                f.style.width='40px'; f.style.height='40px'; document.getElementById('board').appendChild(f); count++;
            }
        }}
        return count;
    }

    function endTurn() { turnIdx = (turnIdx + 1) % players.length; resetTurn(true); }
    function resetTurn(full) { if(full) { roll = []; clearDice(); } document.getElementById('roll-btn').disabled = false; document.getElementById('reroll-btn').style.visibility = 'hidden'; document.getElementById('pass-btn').style.visibility = 'hidden'; updateStatus(); }
    function clearDice() { ['die1','die2'].forEach(function(id){ var e=document.getElementById(id); e.style.background='#eee'; e.innerText='?'; e.style.color='black'; }); }
    function updateDie(id, c) { var e=document.getElementById(id); e.style.background=css[c]; e.innerText=c==='black'?'WILD':c.toUpperCase().substring(0,3); e.style.color=(c==='black'||c==='purple')?'white':'black'; }
    function updateStatus(m) { document.getElementById('status-bar').innerText = m || players[turnIdx].n + "'s Turn"; }
    function reRollOne() { var c = colors[Math.floor(Math.random()*5)]; roll[0] = c; updateDie('die1', c); document.getElementById('reroll-btn').style.visibility='hidden'; }
</script>
</body>
</html>
