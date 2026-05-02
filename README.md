<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Calliope Sensor-Monitor</title>
    <style>
        body { 
            margin: 0; 
            background: #000; 
            color: white; 
            font-family: -apple-system, sans-serif; 
            text-align: center; 
            overflow: hidden;
        }
        /* Das Kamerabild füllt den Hintergrund */
        #cam { 
            width: 100vw; 
            height: 100vh; 
            object-fit: cover; 
            position: absolute; 
            top: 0; 
            left: 0; 
            z-index: 1;
        }
        /* Die Overlay-Elemente liegen über der Kamera */
        .overlay {
            position: relative;
            z-index: 10;
            background: rgba(0, 0, 0, 0.5); /* Halbtransparenter Hintergrund */
            padding: 20px;
        }
        #dist-container {
            margin-top: 50px;
        }
        #d { 
            font-size: 8rem; 
            font-weight: bold; 
            color: #4cd964; 
            text-shadow: 2px 2px 10px rgba(0,0,0,1);
        }
        .unit { font-size: 2rem; color: white; }
        
        button {
            background: #007aff;
            color: white;
            border: none;
            padding: 15px 30px;
            border-radius: 50px;
            font-size: 1.2rem;
            margin-top: 20px;
            cursor: pointer;
        }
        #status { font-size: 0.9rem; color: #aaa; margin-top: 10px; }
    </style>
</head>
<body>

    <video id="cam" autoplay playsinline muted></video>

    <div class="overlay">
        <button id="connect">CALLIOPE VERBINDEN</button>
        <div id="status">Nicht verbunden</div>
        
        <div id="dist-container">
            <span id="d">---</span> <span class="unit">cm</span>
        </div>
    </div>

    <script>
        const dEl = document.getElementById('d');
        const sEl = document.getElementById('status');
        const connBtn = document.getElementById('connect');

        // 1. Kamera aktivieren
        navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
            .then(stream => {
                document.getElementById('cam').srcObject = stream;
            })
            .catch(err => console.error("Kamerafehler:", err));

        // 2. Bluetooth Verbindung
        connBtn.onclick = async () => {
            try {
                const device = await navigator.bluetooth.requestDevice({
                    filters: [{ namePrefix: 'BBC' }, { namePrefix: 'Calliope' }],
                    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
                });
                
                const server = await device.gatt.connect();
                const service = await server.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e');
                const txChar = await service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e');

                await txChar.startNotifications();
                
                txChar.addEventListener('characteristicvaluechanged', (event) => {
                    let text = new TextDecoder().decode(event.target.value);
                    
                    // Sucht nach "D:" gefolgt von Zahlen
                    if (text.includes("D:")) {
                        let match = text.match(/D:(\d+)/);
                        if (match) {
                            dEl.innerText = match[1];
                        }
                    }
                });

                sEl.innerText = "Verbunden mit " + device.name;
                connBtn.style.display = "none"; // Button verschwindet nach Verbindung
            } catch (error) {
                alert("Fehler: " + error);
            }
        };
    </script>
</body>
</html>
