<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Calliope Fix Control</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; display: flex; flex-direction: column; height: 100vh; overflow: hidden; touch-action: none; user-select: none; }
        
        /* Obere Hälfte: Video & Distanz */
        .top-section { position: relative; height: 45%; background: #111; border-bottom: 2px solid #333; }
        #cam { width: 100%; height: 100%; object-fit: cover; }
        .hud { position: absolute; top: 10px; width: 100%; pointer-events: none; }
        #d-val { font-size: 5rem; font-weight: bold; color: #4cd964; text-shadow: 2px 2px 10px #000; }
        
        /* Untere Hälfte: Steuerung */
        .bottom-section { height: 55%; display: flex; flex-direction: column; justify-content: space-evenly; padding: 10px; background: #000; }
        
        .row { display: flex; justify-content: center; gap: 10px; width: 100%; }
        #connect { background: #ff3b30; color: white; border: none; padding: 12px; border-radius: 10px; font-weight: bold; flex-grow: 1; max-width: 300px; }
        
        /* Steuerkreuz */
        .grid { display: grid; grid-template-columns: repeat(3, 80px); gap: 10px; justify-content: center; }
        .ctrl { width: 80px; height: 80px; border-radius: 20px; border: none; font-size: 2rem; background: #333; color: white; }
        .ctrl:active { background: #007aff; }
        
        #msg { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); padding: 15px; border-radius: 15px; display: none; z-index: 20; font-weight: bold; }
        .color-a { background: #ff3b30; } .color-b { background: #007aff; }
    </style>
</head>
<body>

    <div class="top-section">
        <video id="cam" autoplay playsinline muted></video>
        <div class="hud">
            <div id="d-val">---</div>
            <div style="font-size: 1rem;">cm</div>
        </div>
        <div id="msg"></div>
    </div>

    <div class="bottom-section">
        <div class="row">
            <button id="connect">CALLIOPE VERBINDEN</button>
            <button onclick="location.reload()" style="background:#444; color:white; border:none; border-radius:10px; padding:10px;">🔄</button>
        </div>

        <div class="grid">
            <div></div>
            <button class="ctrl" onpointerdown="send('W\n')" onpointerup="send('Q\n')">▲</button>
            <div></div>
            <button class="ctrl" onpointerdown="send('A\n')" onpointerup="send('Q\n')">◀</button>
            <button class="ctrl" onpointerdown="send('S\n')" onpointerup="send('Q\n')">▼</button>
            <button class="ctrl" onpointerdown="send('D\n')" onpointerup="send('Q\n')">▶</button>
        </div>
    </div>

    <script>
        let rx; 
        const dEl = document.getElementById('d-val'), msgEl = document.getElementById('msg'), videoEl = document.getElementById('cam');
        
        navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } }).then(s => videoEl.srcObject = s);

        async function send(m) {
            if (rx) { try { await rx.writeValue(new TextEncoder().encode(m)); } catch(e){} }
        }

        document.getElementById('connect').onclick = async () => {
            try {
                const dev = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });
                const gatt = await dev.gatt.connect();
                const serv = await gatt.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                rx = await serv.getCharacteristic('6e400003-b5a3-f393-e0a9-e50e24dcca9e');
                const tx = await serv.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e');

                await tx.startNotifications();
                tx.addEventListener('characteristicvaluechanged', (e) => {
                    let r = new TextDecoder().decode(e.target.value).trim();
                    if (r.includes("D:")) { 
                        let m = r.match(/D:(\d+)/); 
                        if(m) dEl.innerText = m[1];
                    }
                    if (r === "1") show("A", "color-a");
                    if (r === "0") show("B", "color-b");
                });
                document.getElementById('connect').style.background = "#28a745";
                document.getElementById('connect').innerText = "VERBUNDEN";
            } catch (e) { alert(e); }
        };

        function show(t, c) {
            msgEl.innerText = "KNOPF " + t; msgEl.className = c; msgEl.style.display = "block";
            setTimeout(() => msgEl.style.display = "none", 1000);
        }
    </script>
</body>
</html>
