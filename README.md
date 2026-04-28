<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Safety Band | Pro Monitor</title>
    <style>
        :root {
            --bg: #020617;
            --card: #1e293b;
            --safe: #22c55e;
            --fall: #ef4444;
            --blue: #38bdf8;
            --text: #f8fafc;
            --dim: #64748b;
        }

        body {
            background-color: var(--bg);
            color: var(--text);
            font-family: 'Inter', -apple-system, sans-serif;
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        /* --- HEADER --- */
        header { text-align: center; margin-bottom: 30px; }
        #clock { font-size: 4.5rem; font-weight: 900; color: var(--blue); margin: 0; letter-spacing: -2px; }
        .date-text { color: var(--dim); font-size: 1.2rem; font-weight: 500; }
        
        #api-status { 
            font-size: 0.8rem; 
            margin-top: 10px; 
            padding: 5px 12px; 
            border-radius: 20px; 
            background: rgba(255,255,255,0.05);
        }

        /* --- DASHBOARD GRID --- */
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 25px;
            width: 100%;
            max-width: 1100px;
            margin-bottom: 30px;
        }

        .box {
            background: var(--card);
            padding: 35px;
            border-radius: 28px;
            text-align: center;
            border: 1px solid rgba(255, 255, 255, 0.08);
            box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.2);
        }

        .label { color: #94a3b8; text-transform: uppercase; font-size: 0.9rem; letter-spacing: 2px; font-weight: 600; margin-bottom: 10px; }
        .val { font-size: 3.2rem; font-weight: 800; margin: 10px 0; }
        .unit { font-size: 1.1rem; color: var(--dim); }

        .status-safe { color: var(--safe); }
        .status-fall { color: var(--fall); animation: pulse 1s infinite; }

        @keyframes pulse { 50% { opacity: 0.6; transform: scale(1.03); } }

        /* --- MAP BOX --- */
        .map-section {
            width: 100%;
            max-width: 1100px;
            height: 450px;
            background: var(--card);
            border-radius: 28px;
            overflow: hidden;
            border: 1px solid rgba(255, 255, 255, 0.1);
        }

        /* --- EMERGENCY MODAL --- */
        #overlay {
            display: none;
            position: fixed;
            inset: 0;
            background: rgba(0, 0, 0, 0.95);
            z-index: 1000;
            justify-content: center;
            align-items: center;
            backdrop-filter: blur(10px);
        }

        .alert-card {
            background: var(--fall);
            padding: 60px;
            border-radius: 40px;
            text-align: center;
            box-shadow: 0 0 80px rgba(239, 68, 68, 0.5);
        }

        .alert-card h1 { font-size: 5rem; margin: 0; font-weight: 900; }
        .btn-safe {
            background: white;
            color: black;
            border: none;
            padding: 20px 50px;
            border-radius: 60px;
            font-weight: 900;
            cursor: pointer;
            margin-top: 40px;
            font-size: 1.4rem;
        }
    </style>
</head>
<body>

    <header>
        <div id="clock">00:00:00</div>
        <div id="date" class="date-text">Syncing...</div>
        <div id="api-status">Connecting to Blynk Cloud...</div>
    </header>

    <div class="stats-grid">
        <div class="box">
            <div class="label">User Identification</div>
            <div class="val" style="color: #cbd5e1;">ZX-233</div>
            <div class="unit">Hardware: ESP32-WROOM</div>
        </div>
<div class="box">
            <div class="label">Vitals: Heart Rate</div>
            <div class="val" id="bpm-display">--</div>
            <div class="unit">BPM</div>
        </div>

        <div class="box">
            <div class="label">Safety Status</div>
            <div id="status-val" class="val status-safe">SAFE</div>
            <div class="unit" id="status-desc">Environment Secure</div>
        </div>
    </div>

    <div class="map-section">
        <iframe 
            width="100%" 
            height="100%" 
            frameborder="0" 
            src="https://www.openstreetmap.org/export/embed.html?bbox=100.4816%2C5.1386%2C100.4996%2C5.1526&amp;layer=mapnik&amp;marker=5.1456%2C100.4906">
        </iframe>
    </div>

    <div id="overlay">
        <div class="alert-card">
            <h1>🚨 FALL!!</h1>
            <p style="font-size: 1.8rem; margin: 20px 0;">Heavy Impact Detected by Safety Band!</p>
            <button class="btn-safe" onclick="closeAlert()">I AM SAFE</button>
        </div>
    </div>

    <script>
        // CONFIGURATION
        const TOKEN = "10VutsRka_hwFWOoH_Qu7jG-zKsSY5aB";
        const REGION = "sgp1"; 
        const BLYNK_API_URL = https://${REGION}.blynk.cloud/external/api/get?token=${TOKEN};

        function runClock() {
            const now = new Date();
            document.getElementById('clock').innerText = now.toLocaleTimeString([], { hour12: false });
            document.getElementById('date').innerText = now.toLocaleDateString([], { 
                weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' 
            });
        }
        setInterval(runClock, 1000);
        runClock();

        async function updateDashboard() {
            const statusIndicator = document.getElementById('api-status');
            
            try {
                // We ask for V0 (BPM) and V2 (Status)
                const res = await fetch(${BLYNK_API_URL}&V0&V2);
                
                if (!res.ok) {
                    throw new Error("Blynk API Unreachable");
                }

                const data = await res.json();
                statusIndicator.innerText = "Blynk Online | Last Sync: " + new Date().toLocaleTimeString();
                statusIndicator.style.color = "#4ade80";

                // 1. Update Heart Rate
                document.getElementById('bpm-display').innerText = data.V0 !== undefined ? data.V0 : "--";

                // 2. Update Safety Logic
                const status = data.V2 ? data.V2.toUpperCase() : "SAFE";
                const display = document.getElementById('status-val');
                const desc = document.getElementById('status-desc');
                const overlay = document.getElementById('overlay');

                if (status.includes("FALL") || status.includes("SOS")) {
                    display.innerText = "FALL!!";
                    display.className = "val status-fall";
                    desc.innerText = "EMERGENCY DETECTED";
                    overlay.style.display = 'flex';
                } else {
                    display.innerText = "SAFE";
                    display.className = "val status-safe";
                    desc.innerText = "Environment Secure";
                    overlay.style.display = 'none';
                }

            } catch (e) {
                console.error("Connection Error:", e);
                statusIndicator.innerText = "Blynk Offline | Check Internet/Token";
                statusIndicator.style.color = "#ef4444";
            }
        }

        function closeAlert() {
            document.getElementById('overlay').style.display = 'none';
        }

        // Fetch data every 2000ms (2 seconds)
        setInterval(updateDashboard, 2000);
        updateDashboard(); // Initial call
    </script>
</body>
</html>
