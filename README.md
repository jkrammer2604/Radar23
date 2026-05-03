<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Calliope Pro Control</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; overflow: hidden; touch-action: none; user-select: none; }
        #cam { position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; z-index: 1; }
        
        /* UI Container */
        .ui { position: relative; z-index: 10; height: 100vh; display: flex; flex-direction: column; justify-content: space-between; padding: 10px; box-sizing: border-box; pointer-events: none; }
        .ui * { pointer-events: auto; }

        /* Kompakte obere Leiste */
        .top-bar { display: flex; gap: 5px; align-items: center; }
        #connect { background: #ff3b30; color: white; border: 1px solid white; padding: 10px; border-radius: 8px; font-size: 0.9rem; flex-grow: 1; }
        #switchCam { background: rgba(0,0,0,0.6); color: white; border: 1px solid #fff; padding: 10px; border-radius: 8px; }

        /* Mittlere Anzeige */
        #d-display { margin-top: 5px; }
        #d-val { font-size: 4.5rem; font-weight: bold; color: #4cd964; text-shadow: 2px 2px 8px #000; }
        #msg { font-size: 1.5rem; font-weight: bold; padding: 10px; border-radius: 12px; display: none; border: 2px solid white; margin: 5px auto; width: 60%; }
        .color-a { background: #ff3b30; } .color-b { background: #007aff; }

        /* Kompaktes Steuerkreuz unten */
        .controls { margin-bottom: 15px; }
        .grid { display: grid; grid-template-columns: repeat(3, 70px); gap: 8px; justify-content: center; }
        .ctrl { width: 70px; height: 70px; border-radius: 15px; border: none; font-size: 1.5rem; background: rgba(255,255,255,0.25); color: white; backdrop-filter: blur(5px); display: flex; align-items: center; justify-content: center; }
        .ctrl:active { background: #007aff; scale: 0.9; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="ui">
        <div class="top-bar">
            <button id="connect">KOPPELN</button>
            <button id="switchCam">📷</button>
            <div id="d-display">
                <span id="d-val">---</span><span style="font-size: 1rem; text-shadow: 1px 1px 3px #000;">cm</span>
            </div>
        </div>

        <div id="msg"></div>

        <div class="controls">
            <div class="grid">
                <div></div>
                <button class="ctrl" onpointerdown="send('W\n')" onpointerup="send('Q\n')">▲</button>
                <div></div>
                <button class="ctrl" onpointerdown="send('A\n')" onpointerup="send('Q\n')">◀</button>
                <button class="ctrl" onpointerdown="send('S\n')" onpointerup="send('Q\n')">▼</button>
                <button class="ctrl" onpointerdown="send('D\n')" onpointerup="send('Q\n')">▶</button>
            </div>
        </div>
    </div>

    <script>
        let rx; 
        const dEl = document.getElementById('d-val'), msgEl = document.getElementById('msg'), videoEl = document.getElementById('cam');
        let stream, front = false;
        let lastDist = 0;

        async function startCam(f) {
            if (stream) stream.getTracks().forEach(t => t.stop());
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: f ? "user" : "environment" } });
                videoEl.srcObject = stream;
            } catch(e) {}
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
                    
                    // Filter für Ultraschall (Nur anzeigen wenn Änderung > 1cm)
                    if (r.includes("D:")) { 
                        let m = r.match(/D:(\d+)/); 
                        if(m) {
                            let newDist = parseInt(m[1]);
                            if (Math.abs(newDist - lastDist) > 1) { 
                                dEl.innerText = newDist; 
                                lastDist = newDist;
                            }
                        } 
                    }
                    if (r === "1") show("KNOPF A", "color-a");
                    if (r === "0") show("KNOPF B", "color-b");
                });
                document.getElementById('connect').style.background = "#28a745";
                document.getElementById('connect').innerText = "VBD.";
            } catch (e) { alert(e); }
        };

        function show(t, c) {
            msgEl.innerText = t; msgEl.className = c; msgEl.style.display = "block";
            setTimeout(() => msgEl.style.display = "none", 1500);
        }
    </script>
</body>
</html>
