<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Calliope Full Control</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; overflow: hidden; touch-action: none; user-select: none; -webkit-user-select: none; }
        #cam { position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; z-index: 1; }
        .ui { position: relative; z-index: 10; height: 100vh; display: flex; flex-direction: column; justify-content: space-between; padding: 15px; box-sizing: border-box; pointer-events: none; }
        .ui * { pointer-events: auto; }
        
        /* Oben: Verbindung & Distanz */
        #connect { background: #ff3b30; color: white; border: 2px solid white; padding: 15px; border-radius: 12px; font-weight: bold; width: 100%; margin-bottom: 5px; }
        #d-val { font-size: 5rem; font-weight: bold; color: #4cd964; text-shadow: 2px 2px 10px #000; }

        /* Mitte: Knopf-Nachricht */
        #msg { font-size: 2rem; font-weight: bold; padding: 15px; border-radius: 15px; display: none; border: 3px solid white; margin: 10px auto; width: 70%; }
        .color-a { background: #ff3b30; } .color-b { background: #007aff; }

        /* Unten: Steuerkreuz */
        .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 10px; width: 240px; margin: 0 auto 20px auto; }
        .ctrl { padding: 25px; border-radius: 15px; border: none; font-size: 1.5rem; background: rgba(255,255,255,0.2); color: white; backdrop-filter: blur(5px); }
        .ctrl:active { background: #007aff; }
        #switchCam { position: absolute; top: 80px; right: 10px; padding: 10px; font-size: 0.8rem; background: rgba(0,0,0,0.5); color: white; border: none; border-radius: 5px; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="ui">
        <div>
            <button id="connect">CALLIOPE KOPPELN</button>
            <div id="d-val">---</div>
            <div style="text-shadow: 2px 2px 5px #000;">cm</div>
        </div>

        <div id="msg"></div>
        <button id="switchCam">📷 CAM</button>

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
        let stream, front = false;

        async function startCam(f) {
            if (stream) stream.getTracks().forEach(t => t.stop());
            stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: f ? "user" : "environment" } });
            videoEl.srcObject = stream;
        }
        document.getElementById('switchCam').onclick = () => { front = !front; startCam(front); };
        startCam(front);

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
                    if (r.includes("D:")) { let m = r.match(/D:(\d+)/); if(m) dEl.innerText = m[1]; }
                    if (r === "1") show("KNOPF A", "color-a");
                    if (r === "0") show("KNOPF B", "color-b");
                });
                document.getElementById('connect').style.background = "#28a745";
                document.getElementById('connect').innerText = "VERBUNDEN";
            } catch (e) { alert(e); }
        };

        function show(t, c) {
            msgEl.innerText = t; msgEl.className = c; msgEl.style.display = "block";
            setTimeout(() => msgEl.style.display = "none", 1500);
        }
    </script>
</body>
</html>
