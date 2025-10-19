<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>UOG Shuttle Web App</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            display: flex;
            flex-direction: column;
            height: 100vh;
        }
        #map {
            flex: 1;
            width: 100%;
        }
        .controls {
            padding: 10px;
            background-color: #f0f0f0;
            display: flex;
            justify-content: space-between;
            align-items: center;
            flex-wrap: wrap; /* For mobile responsiveness */
        }
        .shuttle-list {
            display: flex;
            flex-direction: column;
            margin-top: 10px;
            width: 100%;
        }
        .shuttle-item {
            padding: 5px;
            margin: 2px 0;
            background-color: #e9ecef;
            border-radius: 5px;
            cursor: pointer;
        }
        .shuttle-item:hover {
            background-color: #d1ecf1;
        }
        .selected {
            background-color: #007bff;
            color: white;
        }
        button {
            padding: 10px 15px;
            background-color: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover {
            background-color: #0056b3;
        }
        #status {
            font-weight: bold;
        }
        #info {
            margin-left: 20px;
            font-size: 14px;
        }
        /* Modal styles */
        .modal {
            display: none;
            position: fixed;
            z-index: 1;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            overflow: auto;
            background-color: rgba(0, 0, 0, 0.4);
        }
        .modal-content {
            background-color: #fefefe;
            margin: 15% auto;
            padding: 20px;
            border: 1px solid #888;
            width: 80%;
            max-width: 500px;
            border-radius: 5px;
        }
        .close {
            color: #aaa;
            float: right;
            font-size: 28px;
            font-weight: bold;
            cursor: pointer;
        }
        .close:hover,
        .close:focus {
            color: black;
            text-decoration: none;
        }
        .shuttle-detail {
            margin-bottom: 10px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        @media (max-width: 768px) {
            .controls {
                flex-direction: column;
                align-items: stretch;
            }
            #info {
                margin-left: 0;
                margin-top: 10px;
            }
            .modal-content {
                width: 90%;
                margin: 20% auto;
            }
        }
    </style>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js"></script>
</head>
<body>
    <div class="controls">
        <button id="connect">Connect to Server</button>
        <button id="disconnect">Disconnect</button>
        <button id="viewDetails">View Shuttle Details</button>
        <div id="status">Status: Disconnected</div>
        <div id="info">Select a shuttle for details</div>
    </div>
    <div class="shuttle-list" id="shuttleList">
        <!-- Shuttle items will be added here dynamically -->
    </div>
    <div id="map"></div>

    <!-- Modal for shuttle details -->
    <div id="detailsModal" class="modal">
        <div class="modal-content">
            <span class="close" id="closeModal">&times;</span>
            <h2>Shuttle Details</h2>
            <div id="shuttleDetails">
                <!-- Shuttle details will be populated here -->
            </div>
        </div>
    </div>

    <script>
        // Update this URL to your actual deployed WebSocket server (e.g., wss://your-app.herokuapp.com)
        // For local testing: 'ws://localhost:3000'
        // For user access, use a deployed URL like 'wss://uog-shuttle-server.com'
        const SERVER_URL = 'wss://your-deployed-server.com'; // Replace with your real WebSocket URL

        // Initialize the map centered on University of Ghana (Accra, Ghana)
        const map = L.map('map').setView([5.6500, -0.1869], 16);

        // Add OpenStreetMap tiles
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }).addTo(map);

        // Define bus stop locations (approximate coordinates for University of Ghana and nearby areas)
        const busStops = [
            { name: "Legon Main Gate", lat: 5.653, lng: -0.189 },
            { name: "Balme Library", lat: 5.650, lng: -0.186 },
            { name: "Commonwealth Hall", lat: 5.647, lng: -0.185 },
            { name: "Volta Hall", lat: 5.649, lng: -0.184 },
            { name: "East Legon Junction", lat: 5.640, lng: -0.180 },
            { name: "Madina Junction", lat: 5.668, lng: -0.164 },
            { name: "Akuafo Hall", lat: 5.651, lng: -0.188 },
            { name: "Legon Botanical Gardens", lat: 5.644, lng: -0.182 },
            { name: "Diaspora", lat: 5.642, lng: -0.175 }  // Added bus stop at Diaspora
        ];

        // Create a custom bus stop icon (simple SVG-based, or use a default marker with custom popup)
        const busStopIcon = L.divIcon({
            html: '<div style="background-color: #ff0000; width: 20px; height: 20px; border-radius: 50%; border: 2px solid #fff;"></div>',
            className: 'bus-stop-icon',
            iconSize: [20, 20],
            iconAnchor: [10, 10]
        });

        // Add bus stop markers to the map
        busStops.forEach(stop => {
            L.marker([stop.lat, stop.lng], { icon: busStopIcon })
                .addTo(map)
                .bindPopup(`<b>Bus Stop:</b> ${stop.name}`);
        });

        // Object to store markers, trails, and trail data by shuttle ID
        const shuttleMarkers = new Map();
        const shuttleTrails = new Map(); // Stores polylines for trails
        const shuttleTrailCoords = new Map(); // Stores coordinate arrays for trails
        const shuttleData = new Map(); // Store latest data for each shuttle
        let ws = null;
        let selectedShuttle = null;

        function updateStatus(message) {
            document.getElementById('status').textContent = `Status: ${message}`;
        }

        function updateInfo(shuttleId) {
            const data = shuttleData.get(shuttleId);
            if (data) {
                document.getElementById('info').textContent = `Shuttle ${shuttleId}: Speed ${data.speed || '--'} km/h, Lat ${data.latitude.toFixed(4)}, Lng ${data.longitude.toFixed(4)}`;
            } else {
                document.getElementById('info').textContent = 'Select a shuttle for details';
            }
        }

        function addShuttleToList(shuttleId) {
            const list = document.getElementById('shuttleList');
            const item = document.createElement('div');
            item.className = 'shuttle-item';
            item.textContent = `Shuttle ${shuttleId}`;
            item.onclick = () => selectShuttle(shuttleId);
            list.appendChild(item);
        }

        function selectShuttle(shuttleId) {
            selectedShuttle = shuttleId;
            document.querySelectorAll('.shuttle-item').forEach(el => el.classList.remove('selected'));
            const item = Array.from(document.querySelectorAll('.shuttle-item')).find(el => el.textContent === `Shuttle ${shuttleId}`);
            if (item) item.classList.add('selected');
            updateInfo(shuttleId);
            const marker = shuttleMarkers.get(shuttleId);
            if (marker) map.panTo(marker.getLatLng());
        }

        function showShuttleDetails() {
            const detailsDiv = document.getElementById('shuttleDetails');
            detailsDiv.innerHTML = ''; // Clear previous details

            if (shuttleData.size === 0) {
                detailsDiv.innerHTML = '<p>No shuttle data available. Connect to the server and wait for updates.</p>';
            } else {
                shuttleData.forEach((data, shuttleId) => {
                    const detailDiv = document.createElement('div');
                    detailDiv.className = 'shuttle-detail';
                    detailDiv.innerHTML = `
                        <h3>Shuttle ${shuttleId}</h3>
                        <p><strong>Speed:</strong> ${data.speed || '--'} km/h</p>
                        <p><strong>Latitude:</strong> ${data.latitude.toFixed(4)}</p>
                        <p><strong>Longitude:</strong> ${data.longitude.toFixed(4)}</p>
                        <p><strong>Last Updated:</strong> ${new Date().toLocaleTimeString()}</p>
                    `;
                    detailsDiv.appendChild(detailDiv);
                });
            }

            document.getElementById('detailsModal').style.display = 'block';
        }

        function connectToServer() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                alert('Already connected to the server.');
                return;
            }
            ws = new WebSocket(SERVER_URL);

            ws.onopen = () => {
                updateStatus('Connected');
            };

            ws.onmessage = (event) => {
                try {
                    const data = JSON.parse(event.data);
                    const { shuttleId, latitude, longitude, speed } = data;
                    shuttleData.set(shuttleId, { latitude, longitude, speed });

                    // Update or create marker
                    let marker = shuttleMarkers.get(shuttleId);
                    if (!marker) {
                        marker = L.marker([latitude, longitude]).addTo(map);
                        marker.bindPopup(`Shuttle ${shuttleId}`);
                        shuttleMarkers.set(shuttleId, marker);
                        addShuttleToList(shuttleId);
                        // Initialize trail data
                        shuttleTrailCoords.set(shuttleId, []);
                    } else {
                        marker.setLatLng([latitude, longitude]);
                    }

                    // Update trail
                    const trailCoords = shuttleTrailCoords.get(shuttleId);
                    trailCoords.push([latitude, longitude]);

                    // Remove existing trail polyline if it exists
                    const existingTrail = shuttleTrails.get(shuttleId);
                    if (existingTrail) {
                        map.removeLayer(existingTrail);
                    }

                    // Draw new polyline trail
                    const trail = L.polyline(trailCoords, { color: 'blue', weight: 3, opacity: 0.7 }).addTo(map);
                    shuttleTrails.set(shuttleId, trail);

                    // Optionally pan to the latest shuttle
                    map.panTo([latitude, longitude]);
                    if (selectedShuttle === shuttleId) {
                        updateInfo(shuttleId);
                    }
                } catch (error) {
                    console.error('Error parsing WebSocket message:', error);
                }
            };

            ws.onclose = () => {
                updateStatus('Disconnected');
                shuttleMarkers.clear();
                shuttleTrails.forEach(trail => map.removeLayer(trail));
                shuttleTrails.clear();
                shuttleTrailCoords.clear();
                shuttleData.clear();
                document.getElementById('shuttleList').innerHTML = '';
                // Optional: Auto-reconnect after a delay
                setTimeout(() => {
                    if (!ws || ws.readyState === WebSocket.CLOSED) {
                        connectToServer();
                    }
                }, 5000); // Reconnect after 5 seconds
            };

            ws.onerror = (error) => {
                updateStatus('Connection Failed');
                alert('Failed to connect to server. Check the URL and server status.');
                console.error('WebSocket error:', error);
            };
        }

        function disconnectFromServer() {
            if (ws) {
                ws.close();
                ws = null;
                updateStatus('Disconnected');
            }
        }

        // Modal event listeners
        document.getElementById('viewDetails').addEventListener('click', showShuttleDetails);
        document.getElementById('closeModal').addEventListener('click', () => {
            document.getElementById('detailsModal').style.display = 'none';
        });
        window.addEventListener('click', (event) => {
            if (event.target === document.getElementById('detailsModal')) {
                document.getElementById('detailsModal').style.display = 'none';
            }
        });

        document.getElementById('connect').addEventListener('click', connectToServer);
        document.getElementById('disconnect').addEventListener('click', disconnectFromServer);
    </script>
</body>
</html>
