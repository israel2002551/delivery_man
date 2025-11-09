<html>
<head>
    <title>ESP32 Robot Control</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" crossorigin=""/>
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" crossorigin=""></script>

    <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>

    <style>
        :root {
            --bg-color: #f0f0f0;
            --widget-bg: #ffffff;
            --shadow: 0 4px 8px rgba(0,0,0,0.1);
            --border-rad: 8px;
            --btn-color: #007aff;
            --btn-stop: #ff3b30;
        }
        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; 
            background: var(--bg-color);
            margin: 0;
            padding: 10px;
            -webkit-user-select: none; /* Disable text selection */
            user-select: none;
        }
        .container {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 15px;
            max-width: 1200px;
            margin: auto;
        }
        .widget {
            background: var(--widget-bg);
            padding: 15px;
            border-radius: var(--border-rad);
            box-shadow: var(--shadow);
        }
        h2 { margin-top: 0; border-bottom: 2px solid var(--bg-color); padding-bottom: 5px; }
        #map { height: 350px; width: 100%; border-radius: var(--border-rad); }
        #status-bar { padding: 10px; text-align: center; font-weight: bold; border-radius: 5px; }
        .status-connected { background: #dfffe0; color: #2a8b2e; }
        .status-disconnected { background: #ffdede; color: #c92a2a; }
        
        .controls {
            display: grid;
            grid-template-areas:
                ".    fwd  ."
                "left stop right"
                ".    bwd  .";
            gap: 10px;
            justify-items: center;
        }
        .ctrl-btn {
            width: 80px;
            height: 80px;
            border: none;
            border-radius: 50%;
            background: var(--btn-color);
            color: white;
            font-size: 24px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
            transition: transform 0.1s;
        }
        .ctrl-btn:active {
            transform: scale(0.95);
        }
        #btn-fwd { grid-area: fwd; }
        #btn-bwd { grid-area: bwd; }
        #btn-l { grid-area: left; }
        #btn-r { grid-area: right; }
        #btn-stop { grid-area: stop; background: var(--btn-stop); }
        
        input[type="text"], input[type="number"] {
            width: calc(100% - 20px);
            padding: 10px;
            margin-bottom: 10px;
            border: 1px solid #ccc;
            border-radius: 5px;
        }
        button {
            padding: 10px 15px;
            font-size: 14px;
            background: var(--btn-color);
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin: 5px;
        }
        button:active { transform: scale(0.98); }
        #btn-train-stop { background: #ff9500; }
        #btn-clear-waypoints { background: var(--btn-stop); }
        #path-list { max-height: 150px; overflow-y: auto; background: #f9f9f9; padding: 5px; }
        #path-list button { background: #5856d6; width: calc(100% - 10px); }
    </style>
</head>
<body>

    <div class="container">
        
        <div class="widget">
            <div id="status-bar" class="status-disconnected">DISCONNECTED</div>
            <div id="map"></div>
            <p>Sats: <strong id="sats">0</strong> | Speed: <strong id="speed">0</strong> km/h | Course: <strong id="course">0</strong>°</p>
        </div>

        <div class="widget">
            <h2>Manual Control</h2>
            <div class="controls">
                <button class="ctrl-btn" id="btn-fwd">▲</button>
                <button class="ctrl-btn" id="btn-l">◀</button>
                <button class="ctrl-btn" id="btn-stop">■</button>
                <button class="ctrl-btn" id="btn-r">▶</button>
                <button class="ctrl-btn" id="btn-bwd">▼</button>
            </div>
        </div>

        <div class="widget">
            <h2>Training</h2>
            <input type="text" id="path-name" placeholder="Enter path name">
            <button id="btn-train-start">Start Training</button>
            <button id="btn-train-stop">Stop Training</button>
        </div>

        <div class="widget">
            <h2>Autonomous</h2>
            <button id="btn-auto-gps">Run Waypoint Mission</button>
            <hr>
            <h3>Path Playback</h3>
            <button id="btn-refresh-paths">Refresh Path List (N/A)</button>
            <div id="path-list">(Note: Paths are saved on robot)</div>
        </div>

        <div class="widget">
            <h2>Waypoints</h2>
            <input type="number" id="wp-lat" placeholder="Latitude (e.g., 6.3337)">
            <input type="number" id="wp-lon" placeholder="Longitude (e.g., 5.60015)">
            <button id="btn-add-wp">Add Waypoint</button>
            <button id="btn-clear-waypoints">Clear All Waypoints</button>
        </div>
    </div>

<script>
    // --- NEW: MQTT Config ---
    // IMPORTANT: Make these topics unique and MATCH your ESP32 code!
    const brokerUrl = 'wss://broker.hivemq.com:8884/mqtt'; // Use wss:// for secure web
    const commandTopic = 'my-robot-project-123/commands';
    const sensorTopic = 'my-robot-project-123/gps';

    const client = mqtt.connect(brokerUrl);
    const statusBar = document.getElementById('status-bar');

    client.on('connect', () => {
        console.log('MQTT Connected');
        statusBar.textContent = 'CONNECTED';
        statusBar.className = 'status-connected';
        
        // Subscribe to sensor data
        client.subscribe(sensorTopic, (err) => {
            if (err) console.error("Subscribe failed", err);
            else console.log("Subscribed to " + sensorTopic);
        });
    });

    client.on('close', () => {
        console.log('Connection closed');
        statusBar.textContent = 'DISCONNECTED';
        statusBar.className = 'status-disconnected';
    });
    
    client.on('error', (err) => {
        console.error('MQTT Error:', err);
    });

    // --- Map ---
    const map = L.map('map').setView([6.3337, 5.60015], 13); // Default to UofB
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 19,
        attribution: '© OpenStreetMap'
    }).addTo(map);

    let robotMarker = L.marker([0, 0], { title: "Robot" }).addTo(map);
    let robotPath = L.polyline([], {color: 'blue'}).addTo(map);
    let firstFix = false;

    // --- WebSocket Message Handler ---
    const satsEl = document.getElementById('sats');
    const speedEl = document.getElementById('speed');
    const courseEl = document.getElementById('course');

    // --- NEW: MQTT Message Handler ---
    client.on('message', (topic, message) => {
        if (topic === sensorTopic) {
            let data;
            try {
                data = JSON.parse(message.toString());
            } catch (e) { 
                console.error("Not a JSON message: ", message.toString());
                return;
            }

            if (data.gpsFix) {
                const latLng = [data.lat, data.lon];
                robotMarker.setLatLng(latLng);
                robotPath.addLatLng(latLng);
                
                if (!firstFix) {
                    map.setView(latLng, 18); // Zoom to first fix
                    firstFix = true;
                }
            }
            satsEl.textContent = data.sats || 0;
            speedEl.textContent = data.speed ? data.speed.toFixed(1) : 0;
            courseEl.textContent = data.course ? data.course.toFixed(0) : 0;
        }
    });

    // --- NEW: Send Functions ---
    function sendCommand(message) {
        console.log("Sending: " + message);
        client.publish(commandTopic, message);
    }
    
    function sendJsonCommand(data) {
        sendCommand(JSON.stringify(data));
    }

    // --- Control Buttons ---
    function addControlListeners(element, command) {
        element.addEventListener('mousedown', () => sendCommand(command));
        element.addEventListener('touchstart', (e) => { e.preventDefault(); sendCommand(command); });
        
        element.addEventListener('mouseup', () => sendCommand('S'));
        element.addEventListener('mouseleave', () => sendCommand('S'));
        element.addEventListener('touchend', (e) => { e.preventDefault(); sendCommand('S'); });
    }

    addControlListeners(document.getElementById('btn-fwd'), 'F');
    addControlListeners(document.getElementById('btn-bwd'), 'B');
    addControlListeners(document.getElementById('btn-l'), 'L');
    addControlListeners(document.getElementById('btn-r'), 'R');
    
    document.getElementById('btn-stop').addEventListener('click', () => sendCommand('S'));


    // --- Training Buttons ---
    document.getElementById('btn-train-start').addEventListener('click', () => {
        const name = document.getElementById('path-name').value;
        if (!name) {
            alert('Please enter a path name');
            return;
        }
        sendJsonCommand({ cmd: 'train_start', name: name });
    });
    
    document.getElementById('btn-train-stop').addEventListener('click', () => {
        sendJsonCommand({ cmd: 'train_stop' });
    });
    
    // --- Automation/Path Buttons ---
    // We can't list files from SPIFFS anymore. This is now a "blind" command.
    // The user must remember their path names.
    document.getElementById('btn-refresh-paths').addEventListener('click', () => {
        alert("Path list is stored on the robot. Refresh not available. Please enter the path name you want to run and press 'Start Training' (which is now 'Run Path').");
    });
    
    document.getElementById('btn-auto-gps').addEventListener('click', () => {
        sendJsonCommand({ cmd: 'start_autonomous' });
    });

    // --- Waypoint Buttons ---
    document.getElementById('btn-add-wp').addEventListener('click', () => {
        const lat = parseFloat(document.getElementById('wp-lat').value);
        const lon = parseFloat(document.getElementById('wp-lon').value);
        if (isNaN(lat) || isNaN(lon)) {
            alert("Please enter valid Latitude and Longitude");
            return;
        }
        sendJsonCommand({ cmd: 'add_waypoint', lat: lat, lon: lon });
        
        // Add a visual marker to the map
        L.marker([lat, lon], { title: "Waypoint" }).addTo(map)
            .bindPopup(`Waypoint: ${lat}, ${lon}`).openPopup();
    });

    document.getElementById('btn-clear-waypoints').addEventListener('click', () => {
        sendJsonCommand({ cmd: 'clear_waypoints' });
        alert("Waypoints cleared on robot.");
    });

    // --- Init ---
    // (No 'load' event needed, MQTT client starts connecting immediately)

</script>
</body>
</html>
