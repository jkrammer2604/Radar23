<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calliope HUD mit Button-Check</title>
    <style>
        body { 
            margin: 0; background: #000; color: white; 
            font-family: sans-serif; text-align: center; overflow: hidden;
        }
        #cam { 
            width: 100vw; height: 100vh; object-fit: cover; 
            position: absolute; top: 0; left: 0; z-index: 1;
        }
        .overlay {
            position: relative; z-index: 10;
            background: rgba(0, 0, 0, 0.5); padding: 15px;
        }
        #d { font-size: 6rem; font-weight: bold; color: #4cd964; display: block; }
        
        /* Das Feld für die Knopf-Nachricht */
        #button-msg { 
            font-size: 1.5rem; background: #ff3b30; color: white; 
            padding: 10px; border-radius: 10px; display: none; margin: 10px auto; width: 80%;
        }

        .ui-btn {
            background: #444; color: white; border: 1px solid #666;
            padding: 12px 20px; border-radius: 10px; margin: 5px; font-size: 1rem;
        }
        #connect { background: #007aff; border: none; font-weight: bold; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="overlay">
        <button id="connect" class="ui-btn">1. CALLIOPE VERBINDEN</button>
        <button id="switchCam" class="ui-btn">📷 KAMERA WECHSELN</button>
        
        <div id="button-msg">Knopf A wurde gedrückt!</div>
        
        <div style="margin-top: 20px;">
            <span id="d">---</span> <span style="font-size: 1.5rem;">cm</span>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('d');
        const msgEl = document.getElementById('button-msg');
        const videoEl = document.getElementById('cam');
        let currentStream, useFrontCamera = false;

        // --- KAMERA ---
        async function startCamera(front) {
            if (currentStream) currentStream.getTracks().forEach(t => t.stop());
            try {
                currentStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: front ? "user" : "environment" } });
                videoEl.srcObject = currentStream;
            } catch (e) { console.log(e); }
        }
        document.getElementById('switchCam').onclick = () => { useFrontCamera = !useFrontCamera; startCamera(useFrontCamera); };
        startCamera(useFrontCamera);

        // --- BLUETOOTH ---
        document.getElementById('connect').onclick = async () => {
            try {
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });
                const server = await device.gatt.connect();
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                const txChar = await service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e');

                await txChar.startNotifications();
                txChar.addEventListener('characteristicvaluechanged', (e) => {
                    let text = new TextDecoder().decode(e.target.value);
                    
                    // 1. Distanz erkennen (D:...)
                    if (text.includes("D:")) {
                        let match = text.match(/D:(\d+)/);
                        if (match) dEl.innerText = match[1];
                    }

                    // 2. Knopf erkennen (Wenn die "1" vom Calliope kommt)
                    if (text.trim() === "1") {
                        msgEl.style.display = "block";
                        // Nachricht nach 2 Sekunden wieder verstecken
                        setTimeout(() => { msgEl.style.display = "none"; }, 2000);
                    }
                });

                document.getElementById('connect').innerText = "VERBUNDEN";
                document.getElementById('connect').style.background = "#28a745";
            } catch (e) { alert(e); }
        };
    </script>
</body>
</html>
