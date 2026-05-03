<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calliope Zahl-Checker</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; overflow: hidden; }
        #cam { width: 100vw; height: 100vh; object-fit: cover; position: absolute; top: 0; left: 0; z-index: 1; }
        .overlay { position: relative; z-index: 10; background: rgba(0, 0, 0, 0.5); padding: 15px; }
        #d { font-size: 6rem; font-weight: bold; color: #4cd964; display: block; }
        
        #msg-box { 
            font-size: 2.5rem; font-weight: bold; padding: 20px; border-radius: 15px; 
            display: none; margin: 10px auto; width: 80%; border: 5px solid white;
        }
        .color-a { background: #ff3b30; } /* Rot für Zahl 1 */
        .color-b { background: #007aff; } /* Blau für Zahl 0 */

        .ui-btn { background: #444; color: white; border: none; padding: 12px 20px; border-radius: 10px; margin: 5px; }
        #connect { background: #28a745; font-weight: bold; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="overlay">
        <button id="connect" class="ui-btn">VERBINDEN</button>
        <button id="switchCam" class="ui-btn">📷 CAM</button>
        
        <div id="msg-box"></div>
        
        <div style="margin-top: 20px;">
            <span id="d">---</span> <span style="font-size: 1.5rem;">cm</span>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('d');
        const msgBox = document.getElementById('msg-box');
        const videoEl = document.getElementById('cam');
        let currentStream, useFrontCamera = false;

        async function startCamera(front) {
            if (currentStream) currentStream.getTracks().forEach(t => t.stop());
            currentStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: front ? "user" : "environment" } });
            videoEl.srcObject = currentStream;
        }
        document.getElementById('switchCam').onclick = () => { useFrontCamera = !useFrontCamera; startCamera(useFrontCamera); };
        startCamera(useFrontCamera);

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
                    let rawData = new TextDecoder().decode(e.target.value);
                    
                    // 1. Distanz auslesen (D:...)
                    if (rawData.includes("D:")) {
                        let match = rawData.match(/D:(\d+)/);
                        if (match) dEl.innerText = match[1];
                    }

                    // 2. Knopf-Logik (Prüft auf nackte Zahlen)
                    // Wir säubern den Text von Leerzeichen/Umbrüchen
                    let cleanData = rawData.trim();

                    if (cleanData === "1") {
                        showUI("KNOPF A", "color-a");
                    } 
                    else if (cleanData === "0") {
                        showUI("KNOPF B", "color-b");
                    }
                });
                document.getElementById('connect').innerText = "OK";
            } catch (e) { alert(e); }
        };

        function showUI(txt, cls) {
            msgBox.innerText = txt;
            msgBox.className = cls;
            msgBox.style.display = "block";
            setTimeout(() => { msgBox.style.display = "none"; }, 1000);
        }
    </script>
</body>
</html>
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calliope Zahl-Checker</title>
    <style>
        body { margin: 0; background: #000; color: white; font-family: sans-serif; text-align: center; overflow: hidden; }
        #cam { width: 100vw; height: 100vh; object-fit: cover; position: absolute; top: 0; left: 0; z-index: 1; }
        .overlay { position: relative; z-index: 10; background: rgba(0, 0, 0, 0.5); padding: 15px; }
        #d { font-size: 6rem; font-weight: bold; color: #4cd964; display: block; }
        
        #msg-box { 
            font-size: 2.5rem; font-weight: bold; padding: 20px; border-radius: 15px; 
            display: none; margin: 10px auto; width: 80%; border: 5px solid white;
        }
        .color-a { background: #ff3b30; } /* Rot für Zahl 1 */
        .color-b { background: #007aff; } /* Blau für Zahl 0 */

        .ui-btn { background: #444; color: white; border: none; padding: 12px 20px; border-radius: 10px; margin: 5px; }
        #connect { background: #28a745; font-weight: bold; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="overlay">
        <button id="connect" class="ui-btn">VERBINDEN</button>
        <button id="switchCam" class="ui-btn">📷 CAM</button>
        
        <div id="msg-box"></div>
        
        <div style="margin-top: 20px;">
            <span id="d">---</span> <span style="font-size: 1.5rem;">cm</span>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('d');
        const msgBox = document.getElementById('msg-box');
        const videoEl = document.getElementById('cam');
        let currentStream, useFrontCamera = false;

        async function startCamera(front) {
            if (currentStream) currentStream.getTracks().forEach(t => t.stop());
            currentStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: front ? "user" : "environment" } });
            videoEl.srcObject = currentStream;
        }
        document.getElementById('switchCam').onclick = () => { useFrontCamera = !useFrontCamera; startCamera(useFrontCamera); };
        startCamera(useFrontCamera);

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
                    let rawData = new TextDecoder().decode(e.target.value);
                    
                    // 1. Distanz auslesen (D:...)
                    if (rawData.includes("D:")) {
                        let match = rawData.match(/D:(\d+)/);
                        if (match) dEl.innerText = match[1];
                    }

                    // 2. Knopf-Logik (Prüft auf nackte Zahlen)
                    // Wir säubern den Text von Leerzeichen/Umbrüchen
                    let cleanData = rawData.trim();

                    if (cleanData === "1") {
                        showUI("KNOPF A", "color-a");
                    } 
                    else if (cleanData === "0") {
                        showUI("KNOPF B", "color-b");
                    }
                });
                document.getElementById('connect').innerText = "OK";
            } catch (e) { alert(e); }
        };

        function showUI(txt, cls) {
            msgBox.innerText = txt;
            msgBox.className = cls;
            msgBox.style.display = "block";
            setTimeout(() => { msgBox.style.display = "none"; }, 1000);
        }
    </script>
</body>
</html>
