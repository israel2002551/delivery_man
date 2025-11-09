<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ESP32 Delivery Robot Control</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f0f0f0;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }
        .panel {
            margin-bottom: 20px;
            padding: 15px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        .panel h3 {
            margin-top: 0;
        }
        .control-btn {
            width: 60px;
            height: 60px;
            font-size: 20px;
            margin: 5px;
            cursor: pointer;
            border: 2px solid #333;
            border-radius: 5px;
            background: #fff;
        }
        .control-btn:hover {
            background: #e0e0e0;
        }
        .control-btn:active {
            background: #ccc;
        }
        .forward-btn { grid-area: 1 / 2; }
        .left-btn { grid-area: 2 / 1; }
        .stop-btn { grid-area: 2 / 2; }
        .right-btn { grid-area: 2 / 3; }
        .backward-btn { grid-area: 3 / 2; }
        .control-grid {
            display: grid;
            grid-template-columns: 1fr 1fr 1fr;
            grid-template-rows: 1fr 1fr 1fr;
            gap: 5px;
            width: 200px;
            margin: 20px auto;
        }
        input, button {
            padding: 8px;
            margin: 5px;
            border-radius: 4px;
            border: 1px solid #ccc;
        }
        .status {
            background: #f9f9f9;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
        }
        .waypoint-list {
            max-height: 200px;
            overflow-y: auto;
            background: #f9f9f9;
            padding: 10px;
            border-radius: 5px;
        }
        .gps-data {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 10px;
        }
        .gps-field {
            background: #f0f8ff;
            padding: 8px;
            border-radius: 3px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚚 ESP32 Delivery Robot</h1>

        <!-- Manual Controls -->
        <div class="panel">
            <h3>🎮 Manual Control</h3>
            <div class="control-grid">
                <button class="control-btn forward-btn" onclick="sendCommand('F')">↑</button>
                <button class="control-btn left-btn" onclick="sendCommand('L')">←</button>
                <button class="control-btn stop-btn" onclick="sendCommand('S')">●</button>
                <button class="control-btn right-btn" onclick="sendCommand('R')">→</button>
                <button class="control-btn backward-btn" onclick="sendCommand('B')">↓</button>
            </div>
        </div>

        <!-- GPS Data -->
        <div class="panel">
            <h3>📍 Live GPS Data</h3>
            <div class="gps-data" id="gpsData">
                <div class="gps-field">Latitude: <span id="latitude">--</span></div>
                <div class="gps-field">Longitude: <span id="longitude">--</span></div>
                <div class="gps-field">Fix: <span id="fix">--</span></div>
                <div class="gps-field">Sats: <span id="sats">--</span></div>
                <div class="gps-field">Speed: <span id="speed">--</span> km/h</div>
                <div class="gps-field">Course: <span id="course">--</span>°</div>
            </div>
        </div>

        <!-- Waypoint Controls -->
        <div class="panel">
            <h3>🗺️ Waypoint Navigation</h3>
            <input type="number" id="latInput" placeholder="Latitude" step="any">
            <input type="number" id="lonInput" placeholder="Longitude" step="any">
            <button onclick="addWaypoint()">Add Waypoint</button>
            <button onclick="clearWaypoints()">Clear All</button>
            <button onclick="startAutonomous()">Start Delivery</button>
            <div class="waypoint-list" id="waypointList">No waypoints added</div>
        </div>

        <!-- Training Controls -->
        <div class="panel">
            <h3>🎬 Training Mode</h3>
            <input type="text" id="trainName" placeholder="Route name">
            <button onclick="startTraining()">Start Training</button>
            <button onclick="stopTraining()">Stop Training</button>
            <button onclick="playbackPath()">Playback Path</button>
        </div>

        <!-- Status -->
        <div class="panel">
            <h3>📊 Status</h3>
            <div class="status" id="status">Connecting to MQTT...</div>
        </div>
    </div>

    <script>
        // MQTT Configuration - Match your ESP32 settings!
        const MQTT_BROKER = 'broker.hivemq.com';
        const MQTT_PORT = 8080; // WebSocket port (for browser) - HiveMQ uses 8080
        const MQTT_TOPIC_COMMANDS = 'my-robot-project-123/commands';
        const MQTT_TOPIC_GPS = 'my-robot-project-123/gps';

        let client;
        let waypoints = [];
        let isConnected = false;

        // Initialize MQTT
        function initMqtt() {
            const clientId = 'web_client_' + Math.random().toString(36).substr(2, 9);
            // For HiveMQ WebSocket, use path '/mqtt'
            client = new Paho.MQTT.Client(MQTT_BROKER, MQTT_PORT, '/mqtt', clientId);

            client.onConnectionLost = onConnectionLost;
            client.onMessageArrived = onMessageArrived;

            client.connect({
                onSuccess: onConnect,
                onFailure: function(error) {
                    console.error("MQTT Connection failed: ", error);
                    updateStatus("❌ MQTT Connection failed: " + error.errorMessage);
                }
            });
        }

        function onConnect() {
            console.log("Connected to MQTT broker via WebSocket");
            updateStatus("✅ Connected to MQTT broker via WebSocket");
            isConnected = true;

            // Subscribe to GPS topic
            client.subscribe(MQTT_TOPIC_GPS);
            updateStatus("✅ Subscribed to GPS topic: " + MQTT_TOPIC_GPS);
        }

        function onConnectionLost(responseObject) {
            if (responseObject.errorCode !== 0) {
                console.log("Connection lost: " + responseObject.errorMessage);
                updateStatus("❌ Connection lost: " + responseObject.errorMessage);
                isConnected = false;
                setTimeout(initMqtt, 5000); // Retry in 5 seconds
            }
        }

        function onMessageArrived(message) {
            const topic = message.destinationName;
            const payload = message.payloadString;

            if (topic === MQTT_TOPIC_GPS) {
                try {
                    const data = JSON.parse(payload);
                    updateGpsData(data);
                } catch (e) {
                    console.error("Error parsing GPS data: ", e);
                }
            }
        }

        function sendCommand(command) {
            if (!isConnected) {
                alert("Not connected to MQTT broker!");
                return;
            }

            const message = new Paho.MQTT.Message(command);
            message.destinationName = MQTT_TOPIC_COMMANDS;
            client.send(message);
            console.log("Sent command:", command);
        }

        function addWaypoint() {
            const lat = parseFloat(document.getElementById('latInput').value);
            const lon = parseFloat(document.getElementById('lonInput').value);

            if (isNaN(lat) || isNaN(lon)) {
                alert("Please enter valid latitude and longitude!");
                return;
            }

            const waypoint = { lat: lat, lon: lon };
            waypoints.push(waypoint);

            const cmd = {
                cmd: "add_waypoint",
                lat: lat,
                lon: lon
            };

            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
            }

            updateWaypointList();
            document.getElementById('latInput').value = '';
            document.getElementById('lonInput').value = '';
        }

        function clearWaypoints() {
            waypoints = [];
            const cmd = { cmd: "clear_waypoints" };
            
            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
            }

            updateWaypointList();
        }

        function startAutonomous() {
            if (waypoints.length === 0) {
                alert("Add at least one waypoint first!");
                return;
            }

            const cmd = { cmd: "start_autonomous" };
            
            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
                updateStatus("🚀 Autonomous mode started!");
            }
        }

        function startTraining() {
            const name = document.getElementById('trainName').value.trim();
            if (!name) {
                alert("Please enter a route name!");
                return;
            }

            const cmd = { cmd: "train_start", name: name };
            
            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
                updateStatus(`🎬 Training started: ${name}`);
            }
        }

        function stopTraining() {
            const cmd = { cmd: "train_stop" };
            
            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
                updateStatus("⏹️ Training stopped");
            }
        }

        function playbackPath() {
            const name = document.getElementById('trainName').value.trim();
            if (!name) {
                alert("Please enter a route name to play back!");
                return;
            }

            const cmd = { cmd: "play_path", name: name };
            
            if (isConnected) {
                const message = new Paho.MQTT.Message(JSON.stringify(cmd));
                message.destinationName = MQTT_TOPIC_COMMANDS;
                client.send(message);
                updateStatus(`▶️ Playing back: ${name}`);
            }
        }

        function updateWaypointList() {
            const listDiv = document.getElementById('waypointList');
            if (waypoints.length === 0) {
                listDiv.innerHTML = 'No waypoints added';
                return;
            }

            let html = '<ul>';
            waypoints.forEach((wp, index) => {
                html += `<li>📍 ${index + 1}: ${wp.lat.toFixed(6)}, ${wp.lon.toFixed(6)}</li>`;
            });
            html += '</ul>';
            listDiv.innerHTML = html;
        }

        function updateGpsData(data) {
            document.getElementById('latitude').textContent = data.lat ? data.lat.toFixed(6) : '--';
            document.getElementById('longitude').textContent = data.lon ? data.lon.toFixed(6) : '--';
            document.getElementById('fix').textContent = data.gpsFix ? '✅ Yes' : '❌ No';
            document.getElementById('sats').textContent = data.sats || '--';
            document.getElementById('speed').textContent = data.speed ? data.speed.toFixed(2) : '--';
            document.getElementById('course').textContent = data.course ? data.course.toFixed(1) : '--';
        }

        function updateStatus(text) {
            document.getElementById('status').textContent = text;
        }

        // Load Paho MQTT library and initialize
        function loadMqttLibrary() {
            const script = document.createElement('script');
            script.src = 'https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.1.0/paho-mqtt.min.js';
            script.onload = function() {
                initMqtt();
            };
            document.head.appendChild(script);
        }

        // Initialize when page loads
        window.onload = function() {
            loadMqttLibrary();
        };
    </script>
</body>
</html>
