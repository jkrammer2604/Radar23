<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Calliope Einpark-Assistent</title>
    <style>
        /* Grunddesign */
        body { 
            margin: 0; background: #000; color: white; 
            font-family: -apple-system, sans-serif; text-align: center; 
            overflow: hidden; height: 100vh;
        }

        /* Video im Hintergrund */
        #cam { 
            position: absolute; top: 0; left: 0; 
            width: 100%; height: 100%; object-fit: cover; 
            z-index: 1; 
        }

        /* Overlay für die Anzeigen */
        .ui-layer {
            position: relative; z-index: 10;
            height: 100vh; display: flex; flex-direction: column;
            justify-content: space-between; padding: 20px;
            box-sizing: border-box;
            background: rgba(0,0,0,0.2); /* Ganz leichter Schatten fürs Video */
        }

        /* Distanz oben */
        #dist-val { font-size: 7rem; font-weight: bold; color: #4cd964; text-shadow: 3px 3px 10px #000; }
        
        /* Knopf-Nachricht in der Mitte */
        #msg-box { 
            font-size: 2.5rem; font-weight: bold; padding: 20px; 
            border-radius: 20px; display: none; border: 5px solid white;
            box-shadow: 0 0 20px rgba(0,0,0,0.5);
        }
        .color-a { background: #ff3b30; } /* Rot */
        .color-b { background: #007aff; } /* Blau */

        /* Buttons unten */
        .controls { display: flex; gap: 10px; justify-content: center; margin-bottom: 20px; }
        .ui-btn {
            background: rgba(68, 68, 68, 0.8); color: white; border: none; 
            padding: 15px 25px; border-radius: 12px; font-size: 1rem;
        }
        #connect { background: #28a745; font-weight: bold; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="ui-layer">
        <div>
            <div id="dist-val">---</div>
            <div style="font-size: 1.5rem; text-shadow: 2px 2px 5px #000;">cm</div>
        </div>

        <div id="msg-box"></div>

        <div class="controls">
            <button id="connect" class="ui-btn">VERBINDEN</button>
            <button id="switchCam" class="ui-btn">📷 CAM WECHSELN</button>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('dist-val');
        const msgBox = document.getElementById('msg-box');
        const videoEl = document.getElementById('cam');
        let currentStream, useFront = false;

        // Kamera Logik
        async function startCam(front) {
            if (currentStream) currentStream.getTracks().forEach(t => t.stop());
            currentStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: front ? "user" : "environment" } });
            videoEl.srcObject = currentStream;
        }
        document.getElementById('switchCam').onclick = () => { useFront = !useFront; startCam(useFront); };
        startCam(useFront);

        // Bluetooth Logik
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
                    let raw = new TextDecoder().decode(e.target.value).trim();
                    
                    // Distanz filtern
                    if (raw.includes("D:")) {
                        let m = raw.match(/D:(\d+)/);
                        if (m) dEl.innerText = m[1];
                    }

                    // Knöpfe filtern (Zahlen 1 und 0)
                    if (raw === "1") { showUI("KNOPF A", "color-a"); } 
                    else if (raw === "0") { showUI("KNOPF B", "color-b"); }
                });
                document.getElementById('connect').style.display = "none";
            } catch (e) { alert(e); }
        };

        function showUI(txt, cls) {
            msgBox.innerText = txt;
            msgBox.className = cls;
            msgBox.style.display = "block";
            setTimeout(() => { msgBox.style.display = "none"; }, 1200);
        }
    </script>
</body>
</html>
