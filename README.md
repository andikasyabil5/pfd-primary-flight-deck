<!DOCTYPE html>
<html lang="id" class="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Primary Flight Display (PFD) - Kokpit Modern</title>
    <!-- Tailwind CSS untuk tata letak dan UI -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Font Monospace dan Sans-serif berpresisi tinggi ala Avionik -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&family=JetBrains+Mono:wght@500;700;800&display=swap" rel="stylesheet">
    
    <style>
        /* Pengaturan dasar dan pencegahan scroll/pull-to-refresh saat memiringkan HP */
        html, body {
            overscroll-behavior: none;
            touch-action: none;
            user-select: none;
            -webkit-user-select: none;
            font-family: 'Inter', sans-serif;
            background-color: #070a0f;
            color: #e2e8f0;
        }

        .font-avionics {
            font-family: 'JetBrains Mono', monospace;
            font-variant-numeric: tabular-nums;
        }

        /* Bingkai kokpit / Bezel layar DU (Display Unit) */
        .du-bezel {
            background: linear-gradient(145deg, #1a202c, #0d1117);
            box-shadow: 
                inset 0 2px 4px rgba(255,255,255,0.08),
                inset 0 -3px 8px rgba(0,0,0,0.8),
                0 15px 35px rgba(0,0,0,0.9);
            border: 2px solid #2d3748;
        }

        .du-screen-recess {
            box-shadow: inset 0 0 20px rgba(0, 0, 0, 0.95);
            background-color: #05070a;
        }

        /* Custom Slider untuk Throttle */
        input[type=range].throttle-slider {
            -webkit-appearance: none;
            background: transparent;
        }
        input[type=range].throttle-slider::-webkit-slider-thumb {
            -webkit-appearance: none;
            height: 28px;
            width: 18px;
            border-radius: 4px;
            background: linear-gradient(180deg, #38bdf8, #0284c7);
            border: 1px solid #bae6fd;
            cursor: pointer;
            margin-top: -10px;
            box-shadow: 0 2px 6px rgba(0,0,0,0.6);
        }
        input[type=range].throttle-slider::-webkit-slider-runnable-track {
            width: 100%;
            height: 8px;
            cursor: pointer;
            background: #1e293b;
            border-radius: 4px;
            border: 1px solid #334155;
        }

        /* Animasi berkedip untuk peringatan avionik */
        @keyframes pulse-amber {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.35; }
        }
        .animate-amber {
            animation: pulse-amber 1s infinite;
        }
    </style>
</head>
<body class="h-screen w-screen overflow-hidden flex flex-col justify-between p-1.5 sm:p-3">

    <!-- Header Ringkas Kokpit -->
    <header class="flex items-center justify-between px-2 py-1 bg-slate-900/90 border border-slate-800 rounded-lg text-xs sm:text-sm shrink-0">
        <div class="flex items-center gap-2">
            <span class="inline-block w-2.5 h-2.5 rounded-full bg-emerald-400 shadow-[0_0_8px_#34d399]"></span>
            <span class="font-bold tracking-wider text-slate-200">CAPT PFD <span class="text-sky-400 font-avionics text-xs">B738/A320</span></span>
        </div>
        
        <!-- Indikator Status Sensor & Kontrol -->
        <div class="flex items-center gap-2 sm:gap-3 font-avionics text-[11px] sm:text-xs">
            <div id="sensor-badge" class="px-2 py-0.5 rounded bg-slate-800 text-amber-400 border border-amber-500/30">
                MODE: SENTUH/MOUSE
            </div>
            <div class="hidden sm:block text-slate-400">
                PITCH: <span id="debug-pitch" class="text-white">0.0°</span> | ROLL: <span id="debug-roll" class="text-white">0.0°</span>
            </div>
        </div>
    </header>

    <!-- Area Utama Layar PFD (Display Unit) -->
    <main class="relative flex-1 my-1.5 sm:my-2 du-bezel rounded-2xl p-2 sm:p-3 flex flex-col overflow-hidden min-h-0">
        <!-- Sekrup sudut dekoratif ala panel instrumen pesawat -->
        <div class="hidden sm:block absolute top-1.5 left-1.5 w-2 h-2 rounded-full bg-slate-700 border border-slate-900"></div>
        <div class="hidden sm:block absolute top-1.5 right-1.5 w-2 h-2 rounded-full bg-slate-700 border border-slate-900"></div>
        <div class="hidden sm:block absolute bottom-1.5 left-1.5 w-2 h-2 rounded-full bg-slate-700 border border-slate-900"></div>
        <div class="hidden sm:block absolute bottom-1.5 right-1.5 w-2 h-2 rounded-full bg-slate-700 border border-slate-900"></div>

        <!-- Kontainer Canvas PFD Responsif -->
        <div id="canvas-wrapper" class="relative w-full h-full du-screen-recess rounded-xl overflow-hidden border border-slate-800/90 cursor-grab active:cursor-grabbing">
            <canvas id="pfdCanvas" class="block w-full h-full"></canvas>

            <!-- Kotak Pesan Notifikasi (Pengganti alert()) -->
            <div id="toast-box" class="pointer-events-none absolute top-12 left-1/2 -translate-x-1/2 z-30 transition-all duration-300 opacity-0 transform -translate-y-2">
                <div id="toast-content" class="bg-slate-900/95 border border-sky-500/60 text-sky-300 px-3.5 py-1.5 rounded-lg text-xs sm:text-sm font-semibold shadow-lg backdrop-blur-sm text-center whitespace-nowrap">
                    Pesan Sistem
                </div>
            </div>

            <!-- Petunjuk Interaksi Singkat di Sudut Bawah Layar -->
            <div id="hint-overlay" class="pointer-events-none absolute bottom-2 left-2 right-2 flex justify-between items-center text-[10px] sm:text-xs text-slate-400/80 font-avionics px-1">
                <span>GESER LAYAR / TILT HP UNTUK KENDALI</span>
                <span id="g-force-readout">1.00 G</span>
            </div>
        </div>
    </main>

    <!-- Panel Kontrol Kokpit Bawah (MCP / Pedestal Ringkas) -->
    <footer class="bg-slate-900/95 border border-slate-800 rounded-xl p-2 sm:p-2.5 shrink-0 flex flex-col gap-2">
        <!-- Baris 1: Kontrol Throttle & Aksi Utama -->
        <div class="grid grid-cols-12 gap-2 items-center">
            <!-- Kontrol Throttle (N1 %) -->
            <div class="col-span-12 sm:col-span-5 bg-slate-950/80 border border-slate-800 rounded-lg px-2.5 py-1.5 flex items-center gap-2.5">
                <div class="flex flex-col leading-tight shrink-0">
                    <span class="text-[10px] font-bold text-slate-400 tracking-wider">THROTTLE</span>
                    <span id="throttle-val" class="font-avionics text-xs sm:text-sm font-bold text-emerald-400">72% N1</span>
                </div>
                <input id="throttle-slider" type="range" min="20" max="100" value="72" class="throttle-slider w-full" aria-label="Kontrol Throttle">
            </div>

            <!-- Tombol Kontrol Sensor & Kalibrasi -->
            <div class="col-span-12 sm:col-span-7 grid grid-cols-3 gap-1.5 sm:gap-2">
                <button id="btn-gyro" type="button" class="flex items-center justify-center gap-1.5 bg-sky-600 hover:bg-sky-500 active:bg-sky-700 text-white font-semibold text-xs py-2.5 px-2 rounded-lg transition shadow-md border border-sky-400/30">
                    <svg class="w-4 h-4 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 18h.01M8 21h8a2 2 0 002-2V5a2 2 0 00-2-2H8a2 2 0 00-2 2v14a2 2 0 002 2z"></path>
                    </svg>
                    <span class="truncate">Sensor Tilt HP</span>
                </button>

                <button id="btn-calibrate" type="button" class="flex items-center justify-center gap-1 bg-slate-800 hover:bg-slate-700 active:bg-slate-600 text-slate-200 font-semibold text-xs py-2.5 px-2 rounded-lg transition border border-slate-700">
                    <svg class="w-4 h-4 shrink-0 text-amber-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"></path>
                    </svg>
                    <span class="truncate">Kalibrasi Nol</span>
                </button>

                <button id="btn-mode" type="button" class="flex items-center justify-center gap-1 bg-slate-800 hover:bg-slate-700 active:bg-slate-600 text-emerald-400 font-semibold text-xs py-2.5 px-2 rounded-lg transition border border-slate-700">
                    <svg class="w-4 h-4 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path>
                    </svg>
                    <span id="btn-mode-text" class="truncate">Manual Flight</span>
                </button>
            </div>
        </div>
    </footer>

    <script>
        /**
         * ============================================================================
         * PRIMARY FLIGHT DISPLAY (PFD) - SIMULATOR AVIONIK INTERAKTIF
         * ============================================================================
         * Arsitektur:
         * 1. FlightState: Menyimpan parameter fisik pesawat (pitch, roll, heading,
         *    airspeed, altitude, verticalSpeed, throttle, dll.) serta kalibrasi sensor.
         * 2. InputController: Mengelola DeviceOrientation API (iOS & Android),
         *    Mouse/Touch Drag fallback, dan tombol kontrol.
         * 3. PhysicsEngine: Menghitung aerodinamika sederhana (perubahan kecepatan saat
         *    menanjak/menukik, laju belok berdasarkan sudut bank, dan VSI).
         * 4. PFDRenderer: Menggambar seluruh instrumen ke HTML5 Canvas dengan skala
         *    responsif (DPI-aware) dan estetika kokpit kaca modern (Glass Cockpit).
         */

        // --- 1. KONFIGURASI & STATE PENERBANGAN ---
        const flightState = {
            // Sikap Pesawat (Attitude) dalam derajat
            pitch: 2.5,           // Positif = hidung naik, Negatif = hidung turun
            roll: 0.0,            // Positif = miring kanan, Negatif = miring kiri
            targetPitch: 2.5,
            targetRoll: 0.0,

            // Parameter Penerbangan
            airspeed: 250.0,      // Knots (IAS - Indicated Airspeed)
            altitude: 14500.0,    // Feet (MSL)
            heading: 340.0,       // Derajat (0 - 360)
            verticalSpeed: 0.0,   // Feet per minute (fpm)
            throttle: 72,         // Persentase N1 (20% - 100%)
            gForce: 1.0,          // Load factor (G)

            // Batas Operasional
            vStall: 135,          // Batas kecepatan stall (pita merah bawah)
            vMax: 340,            // Batas kecepatan maksimum Vmo (pita merah atas)
            targetAlt: 15000,     // Target ketinggian MCP
            targetSpd: 250,       // Target kecepatan MCP

            // Status Sensor & Mode
            gyroActive: false,
            demoMode: false,
            rawBeta: 45,          // Sudut beta mentah dari sensor HP
            rawGamma: 0,          // Sudut gamma mentah dari sensor HP
            zeroBeta: 45,         // Offset kalibrasi pitch HP (default 45° nyaman digenggam)
            zeroGamma: 0,         // Offset kalibrasi roll HP
            hasReceivedSensorData: false,

            // Interaksi Drag (Fallback Desktop / Sentuh)
            isDragging: false,
            dragStartX: 0,
            dragStartY: 0,
            dragStartPitch: 0,
            dragStartRoll: 0,

            // Waktu simulasi untuk mode demo
            demoTime: 0
        };

        // Referensi Elemen DOM
        const canvas = document.getElementById('pfdCanvas');
        const ctx = canvas.getContext('2d');
        const canvasWrapper = document.getElementById('canvas-wrapper');
        const sensorBadge = document.getElementById('sensor-badge');
        const debugPitch = document.getElementById('debug-pitch');
        const debugRoll = document.getElementById('debug-roll');
        const gForceReadout = document.getElementById('g-force-readout');
        const throttleSlider = document.getElementById('throttle-slider');
        const throttleVal = document.getElementById('throttle-val');
        const btnGyro = document.getElementById('btn-gyro');
        const btnCalibrate = document.getElementById('btn-calibrate');
        const btnMode = document.getElementById('btn-mode');
        const btnModeText = document.getElementById('btn-mode-text');
        const toastBox = document.getElementById('toast-box');
        const toastContent = document.getElementById('toast-content');

        let toastTimeout = null;

        /**
         * Menampilkan notifikasi bergaya avionik tanpa menggunakan alert()
         */
        function showToast(message, type = 'info') {
            toastContent.textContent = message;
            toastContent.className = 'px-3.5 py-1.5 rounded-lg text-xs sm:text-sm font-semibold shadow-lg backdrop-blur-sm text-center border transition-colors ';
            
            if (type === 'success') {
                toastContent.className += 'bg-emerald-950/95 border-emerald-500/70 text-emerald-300';
            } else if (type === 'warn') {
                toastContent.className += 'bg-amber-950/95 border-amber-500/70 text-amber-300';
            } else {
                toastContent.className += 'bg-slate-900/95 border-sky-500/60 text-sky-300';
            }

            toastBox.classList.remove('opacity-0', '-translate-y-2');
            toastBox.classList.add('opacity-100', 'translate-y-0');

            clearTimeout(toastTimeout);
            toastTimeout = setTimeout(() => {
                toastBox.classList.remove('opacity-100', 'translate-y-0');
                toastBox.classList.add('opacity-0', '-translate-y-2');
            }, 3000);
        }

        // --- 2. PENANGANAN UKURAN CANVAS RESPONSIF (HIGH-DPI) ---
        let canvasWidth = 0;
        let canvasHeight = 0;

        function resizeCanvas() {
            const rect = canvasWrapper.getBoundingClientRect();
            const dpr = Math.min(window.devicePixelRatio || 1, 2); // Maksimal 2x demi performa optimal di HP
            canvasWidth = rect.width;
            canvasHeight = rect.height;

            canvas.width = Math.floor(canvasWidth * dpr);
            canvas.height = Math.floor(canvasHeight * dpr);
            ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
        }

        window.addEventListener('resize', resizeCanvas);
        window.addEventListener('orientationchange', () => {
            setTimeout(resizeCanvas, 150);
        });

        // --- 3. KONTROL SENSOR KEMIRINGAN HP (DEVICE ORIENTATION API) ---
        function handleDeviceOrientation(event) {
            if (event.beta === null || event.gamma === null) return;

            flightState.hasReceivedSensorData = true;
            flightState.rawBeta = event.beta;
            flightState.rawGamma = event.gamma;

            if (!flightState.gyroActive || flightState.demoMode) return;

            // Sesuaikan pembacaan berdasarkan orientasi layar (Portrait vs Landscape)
            const screenAngle = (window.screen && window.screen.orientation)
                ? window.screen.orientation.angle
                : (window.orientation || 0);

            let pitchDelta = 0;
            let rollDelta = 0;

            if (screenAngle === 90) {
                // Landscape kiri
                pitchDelta = -(event.gamma - flightState.zeroGamma);
                rollDelta = (event.beta - flightState.zeroBeta);
            } else if (screenAngle === -90 || screenAngle === 270) {
                // Landscape kanan
                pitchDelta = (event.gamma - flightState.zeroGamma);
                rollDelta = -(event.beta - flightState.zeroBeta);
            } else {
                // Portrait standar
                pitchDelta = (event.beta - flightState.zeroBeta);
                rollDelta = (event.gamma - flightState.zeroGamma);
            }

            // Batasi sudut agar realistis (-45° s/d +45° pitch, -75° s/d +75° roll)
            // Memiringkan HP ke depan (beta mengecil) -> hidung pesawat menukik (pitch negatif)
            // Memiringkan HP ke belakang (beta membesar) -> hidung pesawat menanjak (pitch positif)
            flightState.targetPitch = Math.max(-45, Math.min(45, pitchDelta * 0.85));
            flightState.targetRoll = Math.max(-75, Math.min(75, rollDelta * 1.1));
        }

        async function toggleGyroSensor() {
            if (flightState.gyroActive) {
                flightState.gyroActive = false;
                window.removeEventListener('deviceorientation', handleDeviceOrientation);
                updateStatusUI();
                showToast('Sensor Tilt HP dinonaktifkan (Mode Sentuh/Mouse)', 'info');
                return;
            }

            // Matikan mode demo jika sedang aktif
            if (flightState.demoMode) {
                flightState.demoMode = false;
            }

            try {
                // iOS 13+ mewajibkan izin eksplisit via user gesture
                if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
                    const permissionState = await DeviceOrientationEvent.requestPermission();
                    if (permissionState === 'granted') {
                        activateGyroListener();
                    } else {
                        showToast('Izin sensor ditolak oleh perangkat.', 'warn');
                    }
                } else if (typeof DeviceOrientationEvent !== 'undefined') {
                    // Android dan browser desktop yang mendukung
                    activateGyroListener();
                } else {
                    showToast('Perangkat ini tidak mendukung sensor kemiringan.', 'warn');
                }
            } catch (err) {
                console.error('Error meminta izin DeviceOrientation:', err);
                showToast('Gagal mengakses sensor perangkat. Gunakan geser layar.', 'warn');
            }
        }

        function activateGyroListener() {
            flightState.gyroActive = true;
            flightState.hasReceivedSensorData = false;
            window.addEventListener('deviceorientation', handleDeviceOrientation);
            
            // Otomatis kalibrasi titik nol setelah sensor terbaca
            setTimeout(() => {
                if (flightState.hasReceivedSensorData) {
                    calibrateZeroPosition(true);
                } else {
                    showToast('Sensor aktif, tapi tidak ada data gyro (coba di HP).', 'warn');
                }
            }, 300);

            updateStatusUI();
            showToast('Sensor Tilt HP Aktif! Miringkan HP untuk terbang.', 'success');
        }

        function calibrateZeroPosition(silent = false) {
            if (flightState.gyroActive && flightState.hasReceivedSensorData) {
                flightState.zeroBeta = flightState.rawBeta;
                flightState.zeroGamma = flightState.rawGamma;
            }
            flightState.targetPitch = 0;
            flightState.targetRoll = 0;
            if (!silent) {
                showToast('Posisi Tengah (0° Pitch / 0° Roll) Dikalibrasi!', 'success');
            }
        }

        function toggleFlightMode() {
            flightState.demoMode = !flightState.demoMode;
            if (flightState.demoMode) {
                flightState.demoTime = 0;
                showToast('Mode Demo Autopilot Aktif', 'info');
            } else {
                flightState.targetPitch = 0;
                flightState.targetRoll = 0;
                showToast('Kembali ke Kendali Manual', 'success');
            }
            updateStatusUI();
        }

        function updateStatusUI() {
            if (flightState.demoMode) {
                sensorBadge.textContent = 'MODE: AUTO DEMO';
                sensorBadge.className = 'px-2 py-0.5 rounded bg-purple-950 text-purple-300 border border-purple-500/40';
                btnModeText.textContent = 'Auto Demo';
                btnMode.classList.replace('text-emerald-400', 'text-purple-300');
            } else if (flightState.gyroActive) {
                sensorBadge.textContent = 'SENSOR: TILT HP AKTIF';
                sensorBadge.className = 'px-2 py-0.5 rounded bg-emerald-950 text-emerald-300 border border-emerald-500/40';
                btnModeText.textContent = 'Manual Flight';
                btnMode.classList.replace('text-purple-300', 'text-emerald-400');
            } else {
                sensorBadge.textContent = 'MODE: SENTUH/MOUSE';
                sensorBadge.className = 'px-2 py-0.5 rounded bg-slate-800 text-amber-400 border border-amber-500/30';
                btnModeText.textContent = 'Manual Flight';
                btnMode.classList.replace('text-purple-300', 'text-emerald-400');
            }

            if (flightState.gyroActive) {
                btnGyro.classList.replace('bg-sky-600', 'bg-emerald-600');
                btnGyro.classList.replace('hover:bg-sky-500', 'hover:bg-emerald-500');
            } else {
                btnGyro.classList.replace('bg-emerald-600', 'bg-sky-600');
                btnGyro.classList.replace('hover:bg-emerald-500', 'hover:bg-sky-500');
            }
        }

        // --- 4. KONTROL FALLBACK MOUSE & TOUCH DRAG ---
        function onPointerDown(clientX, clientY) {
            if (flightState.demoMode) return;
            flightState.isDragging = true;
            flightState.dragStartX = clientX;
            flightState.dragStartY = clientY;
            flightState.dragStartPitch = flightState.targetPitch;
            flightState.dragStartRoll = flightState.targetRoll;
        }

        function onPointerMove(clientX, clientY) {
            if (!flightState.isDragging || flightState.demoMode) return;
            const dx = clientX - flightState.dragStartX;
            const dy = clientY - flightState.dragStartY;

            // Geser ke atas -> pitch turun (gaya joystick pesawat), geser ke bawah -> pitch naik
            // Atau intuitif layar sentuh: tarik ke bawah untuk menarik hidung pesawat ke atas (pull up)
            const sensitivity = 0.22;
            flightState.targetPitch = Math.max(-45, Math.min(45, flightState.dragStartPitch + dy * sensitivity));
            flightState.targetRoll = Math.max(-75, Math.min(75, flightState.dragStartRoll + dx * sensitivity));
        }

        function onPointerUp() {
            flightState.isDragging = false;
        }

        // Event Listeners Mouse
        canvasWrapper.addEventListener('mousedown', (e) => onPointerDown(e.clientX, e.clientY));
        window.addEventListener('mousemove', (e) => onPointerMove(e.clientX, e.clientY));
        window.addEventListener('mouseup', onPointerUp);

        // Event Listeners Touch
        canvasWrapper.addEventListener('touchstart', (e) => {
            if (e.touches.length === 1) {
                onPointerDown(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, { passive: true });

        window.addEventListener('touchmove', (e) => {
            if (flightState.isDragging && e.touches.length === 1) {
                onPointerMove(e.touches[0].clientX, e.touches[0].clientY);
            }
        }, { passive: true });

        window.addEventListener('touchend', onPointerUp);

        // Kontrol Keyboard (Arrow Keys) untuk pengguna Desktop
        const keysPressed = {};
        window.addEventListener('keydown', (e) => { keysPressed[e.key] = true; });
        window.addEventListener('keyup', (e) => { keysPressed[e.key] = false; });

        // Tombol & Slider Listeners
        btnGyro.addEventListener('click', toggleGyroSensor);
        btnCalibrate.addEventListener('click', () => calibrateZeroPosition(false));
        btnMode.addEventListener('click', toggleFlightMode);

        throttleSlider.addEventListener('input', (e) => {
            flightState.throttle = parseInt(e.target.value, 10);
            throttleVal.textContent = `${flightState.throttle}% N1`;
        });

        // --- 5. MESIN FISIKA & AERODINAMIKA PENERBANGAN ---
        function updateFlightPhysics(dt) {
            // Batasi dt agar tidak melompat saat tab tidak aktif
            dt = Math.min(dt, 0.05);

            // Kontrol keyboard halus jika tombol panah ditekan
            if (!flightState.demoMode) {
                const keyRate = 35 * dt;
                if (keysPressed['ArrowUp']) flightState.targetPitch = Math.max(-45, flightState.targetPitch - keyRate);
                if (keysPressed['ArrowDown']) flightState.targetPitch = Math.min(45, flightState.targetPitch + keyRate);
                if (keysPressed['ArrowLeft']) flightState.targetRoll = Math.max(-75, flightState.targetRoll - keyRate * 1.4);
                if (keysPressed['ArrowRight']) flightState.targetRoll = Math.min(75, flightState.targetRoll + keyRate * 1.4);
            }

            // Mode Demo Autopilot (Simulasi manuver terbang halus)
            if (flightState.demoMode) {
                flightState.demoTime += dt;
                flightState.targetRoll = Math.sin(flightState.demoTime * 0.45) * 28;
                flightState.targetPitch = Math.sin(flightState.demoTime * 0.3) * 8 + 1.5;
            }

            // Interpolasi halus (damping) pada pitch dan roll agar terasa berat & realistis seperti pesawat komersial
            const damping = 1 - Math.exp(-8 * dt);
            flightState.pitch += (flightState.targetPitch - flightState.pitch) * damping;
            flightState.roll += (flightState.targetRoll - flightState.roll) * damping;

            const pitchRad = (flightState.pitch * Math.PI) / 180;
            const rollRad = (flightState.roll * Math.PI) / 180;

            // 1. Airspeed (Kecepatan Udara dalam Knots)
            // Dipengaruhi oleh dorongan mesin (throttle), hambatan udara, dan gravitasi saat menanjak/menukik
            const baseCruiseSpeed = 110 + (flightState.throttle * 2.35); // 20% -> ~157 kts, 100% -> ~345 kts
            const gravityEffect = -Math.sin(pitchRad) * 28.0; // Menukik mempercepat, menanjak memperlambat
            const speedAccel = (baseCruiseSpeed - flightState.airspeed) * 0.25 + gravityEffect;
            flightState.airspeed += speedAccel * dt;
            flightState.airspeed = Math.max(85, Math.min(390, flightState.airspeed));

            // 2. Vertical Speed (VSI dalam Feet per Minute)
            // Kecepatan vertikal proporsional terhadap kecepatan udara * sin(pitch) - efek hilangnya daya angkat saat bank tajam
            const bankLiftLoss = (1 - Math.cos(rollRad)) * 1800;
            const rawVSI = (flightState.airspeed * 101.27) * Math.sin(pitchRad) * 0.42 - bankLiftLoss;
            flightState.verticalSpeed += (rawVSI - flightState.verticalSpeed) * (1 - Math.exp(-5 * dt));

            // 3. Altitude (Ketinggian dalam Feet)
            flightState.altitude += (flightState.verticalSpeed / 60) * dt;
            flightState.altitude = Math.max(0, Math.min(45000, flightState.altitude));

            // 4. Heading & Turn Rate (Laju belok berdasarkan sudut bank/roll & airspeed)
            // Rumus turn rate standar: rate (deg/s) = (1091 * tan(bank)) / TAS
            const clampedRollRad = Math.max(-1.2, Math.min(1.2, rollRad));
            const turnRate = (1091 * Math.tan(clampedRollRad)) / Math.max(120, flightState.airspeed);
            flightState.heading = (flightState.heading + turnRate * dt) % 360;
            if (flightState.heading < 0) flightState.heading += 360;

            // 5. G-Force (Load Factor sederhana)
            const targetG = (1 / Math.max(0.35, Math.cos(clampedRollRad))) + ((flightState.targetPitch - flightState.pitch) * 0.04);
            flightState.gForce += (targetG - flightState.gForce) * 0.15;

            // Update teks telemetry ringan di DOM
            debugPitch.textContent = `${flightState.pitch >= 0 ? '+' : ''}${flightState.pitch.toFixed(1)}°`;
            debugRoll.textContent = `${flightState.roll >= 0 ? '+' : ''}${flightState.roll.toFixed(1)}°`;
            gForceReadout.textContent = `${flightState.gForce.toFixed(2)} G`;
        }

        // --- 6. RENDERER GRAFIS PFD (HTML5 CANVAS) ---
        function renderPFD() {
            const w = canvasWidth;
            const h = canvasHeight;
            if (w <= 0 || h <= 0) return;

            // Bersihkan latar belakang layar
            ctx.fillStyle = '#05070a';
            ctx.fillRect(0, 0, w, h);

            // Tata letak proporsional responsif
            const isCompact = w < 520;
            const fmaHeight = Math.max(32, Math.min(44, h * 0.085));
            const hdgHeight = Math.max(46, Math.min(62, h * 0.12));
            
            const tapeWidth = isCompact ? Math.max(54, w * 0.155) : Math.max(76, w * 0.135);
            const vsiWidth = isCompact ? Math.max(28, w * 0.075) : Math.max(38, w * 0.065);
            const sideMargin = isCompact ? 6 : 14;

            const mainTop = fmaHeight + 6;
            const mainBottom = h - hdgHeight - 6;
            const mainHeight = mainBottom - mainTop;

            // Area ADI (Artificial Horizon) di tengah
            const adiLeft = sideMargin + tapeWidth + (isCompact ? 8 : 16);
            const adiRight = w - sideMargin - vsiWidth - tapeWidth - (isCompact ? 10 : 20);
            const adiWidth = Math.max(100, adiRight - adiLeft);
            const adiCenterX = adiLeft + adiWidth / 2;
            const adiCenterY = mainTop + mainHeight / 2;

            // 1. Gambar Artificial Horizon (Attitude Indicator)
            drawAttitudeIndicator(adiLeft, mainTop, adiWidth, mainHeight, adiCenterX, adiCenterY, isCompact);

            // 2. Gambar Pita Kecepatan Udara (Airspeed Tape - Kiri)
            drawAirspeedTape(sideMargin, mainTop, tapeWidth, mainHeight, isCompact);

            // 3. Gambar Pita Ketinggian (Altimeter Tape - Kanan)
            const altLeft = w - sideMargin - vsiWidth - tapeWidth - 4;
            drawAltimeterTape(altLeft, mainTop, tapeWidth, mainHeight, isCompact);

            // 4. Gambar Vertical Speed Indicator (VSI - Paling Kanan)
            const vsiLeft = w - sideMargin - vsiWidth;
            drawVSIIndicator(vsiLeft, mainTop + mainHeight * 0.1, vsiWidth, mainHeight * 0.8, isCompact);

            // 5. Gambar Kompas Horizontal (Heading Tape - Bawah)
            drawHeadingTape(adiLeft, mainBottom + 4, adiWidth, hdgHeight - 4, isCompact);

            // 6. Gambar Flight Mode Annunciator (FMA - Atas)
            drawFMA(sideMargin, 4, w - sideMargin * 2, fmaHeight - 4, isCompact);
        }

        /**
         * Menggambar Artificial Horizon (Langit, Tanah, Tangga Pitch, Skala Roll, Simbol Pesawat)
         */
        function drawAttitudeIndicator(x, y, w, h, cx, cy, isCompact) {
            ctx.save();

            // Buat clipping region sudut membulat untuk layar ADI
            const radius = isCompact ? 14 : 22;
            roundRectPath(ctx, x, y, w, h, radius);
            ctx.clip();

            // Skala piksel per derajat pitch
            const pxPerDeg = h / 52;
            const rollRad = (-flightState.roll * Math.PI) / 180;

            // Transformasi rotasi (roll) dan translasi (pitch)
            ctx.save();
            ctx.translate(cx, cy);
            ctx.rotate(rollRad);

            const pitchOffset = flightState.pitch * pxPerDeg;
            const diag = Math.hypot(w, h) * 1.6;

            // Langit (Biru Avionik Gradien)
            const skyGrad = ctx.createLinearGradient(0, -diag / 2, 0, pitchOffset);
            skyGrad.addColorStop(0, '#0055a5');
            skyGrad.addColorStop(1, '#0084e8');
            ctx.fillStyle = skyGrad;
            ctx.fillRect(-diag, -diag + pitchOffset, diag * 2, diag);

            // Tanah (Cokelat Hangat Gradien)
            const groundGrad = ctx.createLinearGradient(0, pitchOffset, 0, diag / 2);
            groundGrad.addColorStop(0, '#8c4b19');
            groundGrad.addColorStop(1, '#4a250a');
            ctx.fillStyle = groundGrad;
            ctx.fillRect(-diag, pitchOffset, diag * 2, diag);

            // Garis Horizon Utama (Putih Tebal)
            ctx.strokeStyle = '#ffffff';
            ctx.lineWidth = 2.5;
            ctx.beginPath();
            ctx.moveTo(-diag, pitchOffset);
            ctx.lineTo(diag, pitchOffset);
            ctx.stroke();

            // Tangga Skala Pitch (Pitch Ladder setiap 2.5 / 5 / 10 derajat)
            ctx.font = `700 ${isCompact ? 10 : 12}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';

            for (let deg = -40; deg <= 40; deg += 2.5) {
                if (deg === 0) continue;
                const py = pitchOffset - deg * pxPerDeg;

                // Hanya gambar garis yang terlihat di dalam area ADI
                if (Math.abs(py) > h * 0.65) continue;

                const isMajor = deg % 10 === 0;
                const isMedium = deg % 5 === 0;
                const halfLen = isMajor ? w * 0.21 : (isMedium ? w * 0.11 : w * 0.055);

                ctx.strokeStyle = '#ffffff';
                ctx.lineWidth = isMajor ? 2 : 1.4;

                // Jika pitch negatif, garis putus-putus ala Airbus/Boeing modern
                if (deg < 0 && isMajor) {
                    ctx.setLineDash([5, 4]);
                } else {
                    ctx.setLineDash([]);
                }

                ctx.beginPath();
                ctx.moveTo(-halfLen, py);
                ctx.lineTo(halfLen, py);
                ctx.stroke();
                ctx.setLineDash([]);

                // Ujung kait kecil pada garis mayor
                if (isMajor) {
                    const hookDir = deg > 0 ? 5 : -5;
                    ctx.beginPath();
                    ctx.moveTo(-halfLen, py);
                    ctx.lineTo(-halfLen, py + hookDir);
                    ctx.moveTo(halfLen, py);
                    ctx.lineTo(halfLen, py + hookDir);
                    ctx.stroke();

                    // Label Angka Derajat Pitch di kiri & kanan garis
                    const label = Math.abs(deg).toString();
                    ctx.fillStyle = '#ffffff';
                    ctx.fillText(label, -halfLen - (isCompact ? 14 : 18), py);
                    ctx.fillText(label, halfLen + (isCompact ? 14 : 18), py);
                }
            }

            // Segitiga Penunjuk Sudut Roll yang ikut berputar (Sky Pointer)
            const arcRadius = Math.min(w, h) * 0.43;
            ctx.fillStyle = '#ffffff';
            ctx.beginPath();
            ctx.moveTo(0, -arcRadius + 2);
            ctx.lineTo(-7, -arcRadius + 14);
            ctx.lineTo(7, -arcRadius + 14);
            ctx.closePath();
            ctx.fill();

            // Indikator Slip/Skid kecil di bawah segitiga
            const slipOffset = Math.max(-10, Math.min(10, flightState.roll * 0.12));
            ctx.strokeStyle = '#ffffff';
            ctx.lineWidth = 1.8;
            ctx.strokeRect(-7 + slipOffset, -arcRadius + 16, 14, 4);

            ctx.restore(); // Selesai rotasi pitch & roll

            // Busur Skala Bank/Roll Statis di Bagian Atas ADI
            const arcRadius = Math.min(w, h) * 0.43;
            const bankTicks = [-60, -45, -30, -20, -10, 0, 10, 20, 30, 45, 60];
            
            ctx.save();
            ctx.translate(cx, cy);
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.75)';
            ctx.lineWidth = 1.5;

            // Garis busur tipis
            ctx.beginPath();
            ctx.arc(0, 0, arcRadius, (-150 * Math.PI) / 180, (-30 * Math.PI) / 180);
            ctx.stroke();

            bankTicks.forEach(angle => {
                const rad = ((angle - 90) * Math.PI) / 180;
                const isMajor = Math.abs(angle) === 30 || Math.abs(angle) === 60;
                const tickLen = isMajor ? 11 : 6;

                if (angle === 0) {
                    // Segitiga putih tetap di puncak 0 derajat
                    ctx.fillStyle = '#ffffff';
                    ctx.beginPath();
                    ctx.moveTo(0, -arcRadius - 2);
                    ctx.lineTo(-6, -arcRadius - 11);
                    ctx.lineTo(6, -arcRadius - 11);
                    ctx.closePath();
                    ctx.fill();
                } else if (Math.abs(angle) === 45) {
                    // Segitiga kecil di 45 derajat
                    const tx = Math.cos(rad) * (arcRadius + 4);
                    const ty = Math.sin(rad) * (arcRadius + 4);
                    ctx.fillStyle = '#ffffff';
                    ctx.beginPath();
                    ctx.arc(tx, ty, 2.5, 0, Math.PI * 2);
                    ctx.fill();
                } else {
                    ctx.beginPath();
                    ctx.moveTo(Math.cos(rad) * arcRadius, Math.sin(rad) * arcRadius);
                    ctx.lineTo(Math.cos(rad) * (arcRadius + tickLen), Math.sin(rad) * (arcRadius + tickLen));
                    ctx.stroke();
                }
            });
            ctx.restore();

            // Flight Director (Garis Panduan Magenta ala Boeing/Airbus)
            const fdPitchErr = (flightState.demoMode ? 0 : (1.5 - flightState.pitch)) * (pxPerDeg * 0.5);
            const fdRollErr = (flightState.demoMode ? 0 : (-flightState.roll)) * 0.6;
            ctx.strokeStyle = '#f43f5e'; // Magenta/Pink cerah avionik
            ctx.lineWidth = 2.5;
            ctx.beginPath();
            // Bar horizontal FD
            ctx.moveTo(cx - w * 0.16, cy + Math.max(-35, Math.min(35, fdPitchErr)));
            ctx.lineTo(cx + w * 0.16, cy + Math.max(-35, Math.min(35, fdPitchErr)));
            // Bar vertikal FD
            const fdX = cx + Math.max(-35, Math.min(35, fdRollErr));
            ctx.moveTo(fdX, cy - h * 0.14);
            ctx.lineTo(fdX, cy + h * 0.14);
            ctx.stroke();

            // Simbol Referensi Pesawat Tetap di Tengah (Aircraft Reference Symbol - Kuning/Hitam kontras)
            const wingW = isCompact ? 28 : 42;
            const wingGap = isCompact ? 16 : 22;

            // Sayap Kiri
            ctx.fillStyle = '#090d16';
            ctx.strokeStyle = '#facc15'; // Kuning avionik
            ctx.lineWidth = 2.2;

            ctx.beginPath();
            ctx.moveTo(cx - wingGap - wingW, cy - 3);
            ctx.lineTo(cx - wingGap, cy - 3);
            ctx.lineTo(cx - wingGap, cy + 8);
            ctx.lineTo(cx - wingGap - 6, cy + 8);
            ctx.lineTo(cx - wingGap - 6, cy + 3);
            ctx.lineTo(cx - wingGap - wingW, cy + 3);
            ctx.closePath();
            ctx.fill();
            ctx.stroke();

            // Sayap Kanan
            ctx.beginPath();
            ctx.moveTo(cx + wingGap + wingW, cy - 3);
            ctx.lineTo(cx + wingGap, cy - 3);
            ctx.lineTo(cx + wingGap, cy + 8);
            ctx.lineTo(cx + wingGap + 6, cy + 8);
            ctx.lineTo(cx + wingGap + 6, cy + 3);
            ctx.lineTo(cx + wingGap + wingW, cy + 3);
            ctx.closePath();
            ctx.fill();
            ctx.stroke();

            // Kotak Titik Tengah (Boresight)
            ctx.fillStyle = '#090d16';
            ctx.fillRect(cx - 4, cy - 4, 8, 8);
            ctx.strokeRect(cx - 4, cy - 4, 8, 8);

            // Peringatan Sikap Ekstrem ("BANK ANGLE" / "PITCH")
            if (Math.abs(flightState.roll) > 38) {
                ctx.fillStyle = 'rgba(245, 158, 11, 0.9)';
                ctx.font = `800 ${isCompact ? 11 : 13}px "JetBrains Mono", monospace`;
                ctx.textAlign = 'center';
                ctx.fillText('BANK ANGLE', cx, y + h - 18);
            } else if (flightState.airspeed < flightState.vStall + 8) {
                ctx.fillStyle = 'rgba(239, 68, 68, 0.95)';
                ctx.font = `800 ${isCompact ? 11 : 13}px "JetBrains Mono", monospace`;
                ctx.textAlign = 'center';
                ctx.fillText('STALL WARNING', cx, y + h - 18);
            }

            ctx.restore();

            // Bingkai luar ADI
            ctx.strokeStyle = '#334155';
            ctx.lineWidth = 1.5;
            roundRectPath(ctx, x, y, w, h, radius);
            ctx.stroke();
        }

        /**
         * Menggambar Pita Kecepatan Udara (Airspeed Tape - Kiri)
         */
        function drawAirspeedTape(x, y, w, h, isCompact) {
            const cy = y + h / 2;
            const pxPerKnot = h / 110; // Rentang terlihat ±55 knots

            ctx.save();
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.clip();

            // Latar belakang pita abu-abu gelap khas kokpit
            ctx.fillStyle = '#141b26';
            ctx.fillRect(x, y, w, h);

            // Pita batas Merah Overspeed (Vmo) & Stall (Vs)
            const vMaxY = cy - (flightState.vMax - flightState.airspeed) * pxPerKnot;
            if (vMaxY > y) {
                ctx.fillStyle = '#ef4444';
                ctx.fillRect(x + w - 7, y, 7, Math.min(h, vMaxY - y));
            }

            const vStallY = cy - (flightState.vStall - flightState.airspeed) * pxPerKnot;
            if (vStallY < y + h) {
                ctx.fillStyle = '#ef4444';
                ctx.fillRect(x + w - 7, Math.max(y, vStallY), 7, (y + h) - Math.max(y, vStallY));
            }

            // Garis skala setiap 10 Knots
            const minSpd = Math.floor((flightState.airspeed - 70) / 10) * 10;
            const maxSpd = Math.ceil((flightState.airspeed + 70) / 10) * 10;

            ctx.strokeStyle = '#e2e8f0';
            ctx.fillStyle = '#f8fafc';
            ctx.font = `600 ${isCompact ? 11 : 13}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'right';
            ctx.textBaseline = 'middle';

            for (let spd = minSpd; spd <= maxSpd; spd += 10) {
                if (spd < 40) continue;
                const sy = cy - (spd - flightState.airspeed) * pxPerKnot;
                const isMajor = spd % 20 === 0;

                ctx.lineWidth = isMajor ? 2 : 1.2;
                ctx.beginPath();
                ctx.moveTo(x + w - (isMajor ? 14 : 9), sy);
                ctx.lineTo(x + w - 2, sy);
                ctx.stroke();

                if (isMajor) {
                    ctx.fillText(spd.toString(), x + w - 17, sy);
                }
            }

            // Indikator Tren Kecepatan (Panah hijau naik/turun prediksi 10 detik)
            const accelTrend = -Math.sin((flightState.pitch * Math.PI) / 180) * 28 + ((110 + flightState.throttle * 2.35 - flightState.airspeed) * 0.25);
            if (Math.abs(accelTrend) > 1.5) {
                const trendLen = Math.max(-h * 0.35, Math.min(h * 0.35, accelTrend * 4 * pxPerKnot));
                ctx.strokeStyle = '#34d399';
                ctx.lineWidth = 2.5;
                ctx.beginPath();
                ctx.moveTo(x + w - 4, cy);
                ctx.lineTo(x + w - 4, cy - trendLen);
                ctx.stroke();
            }

            ctx.restore();

            // Bingkai luar pita
            ctx.strokeStyle = '#334155';
            ctx.lineWidth = 1.5;
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.stroke();

            // Kotak Digital Readout di Tengah (Current Airspeed Box)
            const boxH = isCompact ? 30 : 36;
            const boxW = w + 6;
            const bx = x - 2;
            const by = cy - boxH / 2;

            ctx.fillStyle = '#05080f';
            ctx.strokeStyle = flightState.airspeed < flightState.vStall ? '#ef4444' : '#38bdf8';
            ctx.lineWidth = 2;

            ctx.beginPath();
            ctx.moveTo(bx, by);
            ctx.lineTo(bx + boxW - 8, by);
            ctx.lineTo(bx + boxW, cy);
            ctx.lineTo(bx + boxW - 8, by + boxH);
            ctx.lineTo(bx, by + boxH);
            ctx.closePath();
            ctx.fill();
            ctx.stroke();

            ctx.fillStyle = flightState.airspeed < flightState.vStall ? '#f87171' : '#ffffff';
            ctx.font = `800 ${isCompact ? 13 : 16}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(Math.round(flightState.airspeed).toString(), bx + (boxW - 6) / 2, cy + 1);

            // Label "KTS" di bawah pita
            ctx.fillStyle = '#38bdf8';
            ctx.font = `700 ${isCompact ? 9 : 11}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.fillText('KTS', x + w / 2, y + h + 13);
        }

        /**
         * Menggambar Pita Ketinggian (Altimeter Tape - Kanan)
         */
        function drawAltimeterTape(x, y, w, h, isCompact) {
            const cy = y + h / 2;
            const pxPerFoot = h / 1200; // Rentang terlihat ±600 feet

            ctx.save();
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.clip();

            ctx.fillStyle = '#141b26';
            ctx.fillRect(x, y, w, h);

            // Garis skala setiap 100 Feet
            const minAlt = Math.floor((flightState.altitude - 800) / 100) * 100;
            const maxAlt = Math.ceil((flightState.altitude + 800) / 100) * 100;

            ctx.strokeStyle = '#e2e8f0';
            ctx.fillStyle = '#f8fafc';
            ctx.font = `600 ${isCompact ? 10 : 12}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'left';
            ctx.textBaseline = 'middle';

            for (let alt = minAlt; alt <= maxAlt; alt += 100) {
                if (alt < 0) continue;
                const ay = cy - (alt - flightState.altitude) * pxPerFoot;
                const isMajor = alt % 500 === 0;
                const is200 = alt % 200 === 0;

                ctx.lineWidth = isMajor ? 2.2 : 1.2;
                ctx.beginPath();
                ctx.moveTo(x + 2, ay);
                ctx.lineTo(x + (isMajor ? 14 : 8), ay);
                ctx.stroke();

                if (is200) {
                    // Kotak kecil di kelipatan 500
                    if (isMajor) {
                        ctx.strokeStyle = '#38bdf8';
                        ctx.strokeRect(x + 15, ay - 8, w - 18, 16);
                        ctx.strokeStyle = '#e2e8f0';
                    }
                    ctx.fillText(alt.toString(), x + 17, ay);
                }
            }

            ctx.restore();

            ctx.strokeStyle = '#334155';
            ctx.lineWidth = 1.5;
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.stroke();

            // Kotak Readout Digital Ketinggian di Tengah
            const boxH = isCompact ? 30 : 36;
            const boxW = w + 8;
            const bx = x - 6;
            const by = cy - boxH / 2;

            ctx.fillStyle = '#05080f';
            ctx.strokeStyle = '#34d399'; // Hijau avionik
            ctx.lineWidth = 2;

            ctx.beginPath();
            ctx.moveTo(bx + 8, by);
            ctx.lineTo(bx + boxW, by);
            ctx.lineTo(bx + boxW, by + boxH);
            ctx.lineTo(bx + 8, by + boxH);
            ctx.lineTo(bx, cy);
            ctx.closePath();
            ctx.fill();
            ctx.stroke();

            ctx.fillStyle = '#ffffff';
            ctx.font = `800 ${isCompact ? 12 : 15}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            const altFormatted = Math.round(flightState.altitude).toString().padStart(4, '0');
            ctx.fillText(altFormatted, bx + 6 + (boxW - 6) / 2, cy + 1);

            // Label QNH / STD di bawah altimeter
            ctx.fillStyle = '#34d399';
            ctx.font = `700 ${isCompact ? 9 : 11}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.fillText('1013 HPA', x + w / 2, y + h + 13);
        }

        /**
         * Menggambar Vertical Speed Indicator (VSI - Paling Kanan)
         */
        function drawVSIIndicator(x, y, w, h, isCompact) {
            const cy = y + h / 2;
            const halfH = h / 2;

            // Latar belakang trapesium/kotak VSI
            ctx.fillStyle = '#111722';
            ctx.strokeStyle = '#334155';
            ctx.lineWidth = 1.2;
            roundRectPath(ctx, x, y, w, h, 6);
            ctx.fill();
            ctx.stroke();

            // Skala VSI non-linear khas Boeing/Airbus (0, 1, 2, 6 ribu fpm)
            const marks = [
                { val: 6000, pos: 0.92, label: '6' },
                { val: 2000, pos: 0.65, label: '2' },
                { val: 1000, pos: 0.38, label: '1' },
                { val: 0,    pos: 0.0,  label: '0' },
                { val: -1000, pos: -0.38, label: '1' },
                { val: -2000, pos: -0.65, label: '2' },
                { val: -6000, pos: -0.92, label: '6' }
            ];

            ctx.fillStyle = '#94a3b8';
            ctx.font = `700 ${isCompact ? 9 : 10}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'left';
            ctx.textBaseline = 'middle';

            marks.forEach(m => {
                const my = cy - m.pos * halfH;
                ctx.strokeStyle = m.val === 0 ? '#ffffff' : '#64748b';
                ctx.lineWidth = m.val === 0 ? 2 : 1.2;
                ctx.beginPath();
                ctx.moveTo(x + 2, my);
                ctx.lineTo(x + 8, my);
                ctx.stroke();

                if (m.val !== 0) {
                    ctx.fillText(m.label, x + 10, my);
                }
            });

            // Hitung posisi jarum VSI secara non-linear
            const vsi = Math.max(-6000, Math.min(6000, flightState.verticalSpeed));
            const absVsi = Math.abs(vsi);
            let normPos = 0;
            if (absVsi <= 1000) {
                normPos = (absVsi / 1000) * 0.38;
            } else if (absVsi <= 2000) {
                normPos = 0.38 + ((absVsi - 1000) / 1000) * 0.27;
            } else {
                normPos = 0.65 + ((absVsi - 2000) / 4000) * 0.27;
            }
            if (vsi < 0) normPos = -normPos;

            const needleY = cy - normPos * halfH;

            // Gambar Jarum VSI Hijau/Putih
            ctx.strokeStyle = '#34d399';
            ctx.lineWidth = 2.5;
            ctx.beginPath();
            ctx.moveTo(x + w + 4, cy);
            ctx.lineTo(x + 4, needleY);
            ctx.stroke();

            // Angka Digital VSI jika laju naik/turun > 200 fpm
            if (absVsi >= 200) {
                const vsiHundreds = Math.round(vsi / 100);
                const text = (vsiHundreds > 0 ? '+' : '') + vsiHundreds;
                ctx.fillStyle = '#05080f';
                ctx.fillRect(x - 2, vsi > 0 ? y - 16 : y + h + 2, w + 4, 14);
                ctx.fillStyle = '#34d399';
                ctx.font = `700 ${isCompact ? 9 : 10}px "JetBrains Mono", monospace`;
                ctx.textAlign = 'center';
                ctx.fillText(text, x + w / 2, vsi > 0 ? y - 9 : y + h + 9);
            }
        }

        /**
         * Menggambar Pita Kompas Horizontal (Heading Indicator - Bawah)
         */
        function drawHeadingTape(x, y, w, h, isCompact) {
            const cx = x + w / 2;
            const pxPerDeg = w / 75; // Rentang terlihat ±37.5 derajat

            ctx.save();
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.clip();

            ctx.fillStyle = '#141b26';
            ctx.fillRect(x, y, w, h);

            const startDeg = Math.floor((flightState.heading - 45) / 5) * 5;
            const endDeg = Math.ceil((flightState.heading + 45) / 5) * 5;

            ctx.strokeStyle = '#cbd5e1';
            ctx.fillStyle = '#f8fafc';
            ctx.font = `700 ${isCompact ? 11 : 13}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'top';

            const cardinals = { 0: 'N', 90: 'E', 180: 'S', 270: 'W' };

            for (let deg = startDeg; deg <= endDeg; deg += 5) {
                const hx = cx + (deg - flightState.heading) * pxPerDeg;
                let normDeg = deg % 360;
                if (normDeg < 0) normDeg += 360;

                const isMajor = normDeg % 10 === 0;
                ctx.lineWidth = isMajor ? 2 : 1.2;
                ctx.beginPath();
                ctx.moveTo(hx, y + 2);
                ctx.lineTo(hx, y + (isMajor ? 12 : 7));
                ctx.stroke();

                if (isMajor) {
                    let label = cardinals[normDeg] || (normDeg / 10).toString().padStart(2, '0');
                    ctx.fillStyle = cardinals[normDeg] ? '#38bdf8' : '#f8fafc';
                    ctx.fillText(label, hx, y + 15);
                }
            }

            ctx.restore();

            // Bingkai Kompas
            ctx.strokeStyle = '#334155';
            ctx.lineWidth = 1.5;
            roundRectPath(ctx, x, y, w, h, 8);
            ctx.stroke();

            // Segitiga Penunjuk Heading Saat Ini & Kotak Digital di Atas Kompas
            ctx.fillStyle = '#facc15';
            ctx.beginPath();
            ctx.moveTo(cx, y + 4);
            ctx.lineTo(cx - 6, y - 4);
            ctx.lineTo(cx + 6, y - 4);
            ctx.closePath();
            ctx.fill();

            // Kotak Readout Heading 3 Digit
            const boxW = isCompact ? 42 : 50;
            const boxH = isCompact ? 18 : 21;
            ctx.fillStyle = '#05080f';
            ctx.strokeStyle = '#facc15';
            ctx.lineWidth = 1.5;
            ctx.fillRect(cx - boxW / 2, y - boxH - 2, boxW, boxH);
            ctx.strokeRect(cx - boxW / 2, y - boxH - 2, boxW, boxH);

            const hdgStr = Math.round(flightState.heading % 360).toString().padStart(3, '0') + '°';
            ctx.fillStyle = '#ffffff';
            ctx.font = `800 ${isCompact ? 11 : 12}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(hdgStr, cx, y - boxH / 2 - 1);
        }

        /**
         * Menggambar Flight Mode Annunciator (FMA - Baris Status Kokpit Atas)
         */
        function drawFMA(x, y, w, h, isCompact) {
            ctx.fillStyle = '#0d131f';
            ctx.strokeStyle = '#1e293b';
            ctx.lineWidth = 1.5;
            roundRectPath(ctx, x, y, w, h, 6);
            ctx.fill();
            ctx.stroke();

            const colW = w / 3;

            // Garis pemisah vertikal antar kolom FMA
            ctx.strokeStyle = '#334155';
            ctx.beginPath();
            ctx.moveTo(x + colW, y + 4);
            ctx.lineTo(x + colW, y + h - 4);
            ctx.moveTo(x + colW * 2, y + 4);
            ctx.lineTo(x + colW * 2, y + h - 4);
            ctx.stroke();

            // Teks Mode FMA (Autothrottle | Roll Mode | Pitch Mode)
            const thrMode = flightState.throttle >= 92 ? 'THR TOGA' : (flightState.demoMode ? 'SPD' : 'THR REF');
            const rollMode = flightState.demoMode ? 'HDG SEL' : (flightState.gyroActive ? 'TILT ROLL' : 'MAN ROLL');
            const pitchMode = flightState.demoMode ? 'ALT HLD' : (flightState.gyroActive ? 'TILT PTCH' : 'MAN PTCH');

            ctx.font = `800 ${isCompact ? 10 : 13}px "JetBrains Mono", monospace`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';

            // Kolom 1: Thrust Mode
            ctx.fillStyle = '#34d399'; // Hijau aktif
            ctx.fillText(thrMode, x + colW * 0.5, y + h * 0.52);

            // Kolom 2: Lateral / Roll Mode
            ctx.fillStyle = flightState.demoMode ? '#34d399' : '#38bdf8';
            ctx.fillText(rollMode, x + colW * 1.5, y + h * 0.52);

            // Kolom 3: Vertical / Pitch Mode
            ctx.fillStyle = flightState.demoMode ? '#34d399' : '#38bdf8';
            ctx.fillText(pitchMode, x + colW * 2.5, y + h * 0.52);
        }

        /**
         * Helper untuk menggambar path persegi panjang dengan sudut membulat
         */
        function roundRectPath(context, x, y, width, height, radius) {
            context.beginPath();
            context.moveTo(x + radius, y);
            context.lineTo(x + width - radius, y);
            context.quadraticCurveTo(x + width, y, x + width, y + radius);
            context.lineTo(x + width, y + height - radius);
            context.quadraticCurveTo(x + width, y + height, x + width - radius, y + height);
            context.lineTo(x + radius, y + height);
            context.quadraticCurveTo(x, y + height, x, y + height - radius);
            context.lineTo(x, y + radius);
            context.quadraticCurveTo(x, y, x + radius, y);
            context.closePath();
        }

        // --- 7. LOOP ANIMASI UTAMA ---
        let lastTimestamp = 0;

        function mainLoop(timestamp) {
            if (!lastTimestamp) lastTimestamp = timestamp;
            const dt = (timestamp - lastTimestamp) / 1000;
            lastTimestamp = timestamp;

            updateFlightPhysics(dt);
            renderPFD();

            requestAnimationFrame(mainLoop);
        }

        // Inisialisasi saat halaman dimuat
        window.onload = function () {
            resizeCanvas();
            updateStatusUI();
            requestAnimationFrame(mainLoop);
        };
    </script>
</body>
</html>
