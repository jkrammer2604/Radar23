<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calliope HUD mit Cam-Switch</title>
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
            background: rgba(0, 0, 0, 0.4); padding: 15px;
        }
        #d { 
            font-size: 8rem; font-weight: bold; color: #4cd964; 
            text-shadow: 3px 3px 15px #000; display: block;
        }
        .ui-btn {
            background: #444; color: white; border: 1px solid #666;
            padding: 12px 20px; border-radius: 10px; margin: 5px;
            font-size: 1rem; cursor: pointer;
        }
        #connect { background: #007aff; border: none; font-weight: bold; }
        #status { font-size: 0.8rem; margin-top: 5px; color: #ccc; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="overlay">
        <button id="connect" class="ui-btn">1. CALLIOPE VERBINDEN</button>
        <button id="switchCam" class="ui-btn">📷 KAMERA WECHSELN</button>
        <div id="status">Kamera bereit</div>
        
        <div style="margin-top: 30px;">
            <span id="d">---</span>
            <span style="font-size: 2rem;">cm</span>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('d');
        const sEl = document.getElementById('status');
        const videoEl = document.getElementById('cam');
        let currentStream;
        let useFrontCamera = false; // Startet meist mit der Rückkamera

        // --- FUNKTION: KAMERA STARTEN ---
        async function startCamera(front) {
            if (currentStream) {
                currentStream.getTracks().forEach(track => track.stop());
            }
            
            const constraints = {
                video: { facingMode: front ? "user" : "environment" }
            };

            try {
                currentStream = await navigator.mediaDevices.getUserMedia(constraints);
                videoEl.srcObject = currentStream;
            } catch (err) {
                sEl.innerText = "Kamera-Fehler: " + err;
            }
        }

        // Kamera-Wechsel Button
        document.getElementById('switchCam').onclick = () => {
            useFrontCamera = !useFrontCamera;
            startCamera(useFrontCamera);
        };

        // Initialstart
        startCamera(useFrontCamera);

        // --- BLUETOOTH VERBINDUNG ---
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
                    if (text.includes("D:")) {
                        let match = text.match(/D:(\d+)/);
                        if (match) dEl.innerText = match[1];
                    }
                });

                sEl.innerText = "Verbunden!";
                document.getElementById('connect').style.display = "none";
            } catch (error) {
                alert("BT-Fehler: " + error);
            }
        };
    </script>
</body>
</html>
