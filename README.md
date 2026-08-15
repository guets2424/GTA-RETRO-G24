# GTA-RETRO-G24
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Retro Top-Down GTA</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <style>
        body {
            touch-action: none;
            user-select: none;
            -webkit-user-select: none;
            background-color: #111;
            color: #fff;
            overflow: hidden;
        }
        canvas {
            display: block;
            background: #444;
        }
    </style>
</head>
<body class="flex flex-col h-screen justify-between items-center select-none">

    <!-- En-tête / HUD -->
    <header class="w-full max-w-md flex justify-between items-center p-4 bg-gray-900 border-b border-gray-800 text-sm font-bold">
        <div class="flex items-center space-x-2">
            <span class="text-red-500 uppercase tracking-widest" id="status-text">À pied</span>
        </div>
        <div class="text-yellow-400 font-mono text-lg" id="score-display">$0</div>
        <div class="flex space-x-1" id="wanted-stars">
            <span class="text-gray-600">★</span>
            <span class="text-gray-600">★</span>
            <span class="text-gray-600">★</span>
        </div>
    </header>

    <!-- Zone de jeu (Canvas) -->
    <main class="relative flex-grow w-full flex justify-center items-center bg-black overflow-hidden">
        <canvas id="gameCanvas" class="shadow-2xl"></canvas>
        
        <!-- Écran de message / notification -->
        <div id="msg-box" class="absolute top-4 bg-black/80 border border-yellow-500 text-yellow-300 px-4 py-2 rounded-lg text-xs tracking-wide uppercase transition-opacity duration-300 opacity-0 pointer-events-none">
            Message système
        </div>
    </main>

    <!-- Contrôles Tactiles (Optimisés iPhone / Mobile) -->
    <footer class="w-full max-w-md p-4 bg-gray-900 border-t border-gray-800 flex justify-between items-center">
        <!-- Joystick virtuel / Directionnel -->
        <div class="relative w-36 h-36 bg-gray-800/80 rounded-full border border-gray-700 flex items-center justify-center" id="joystick-base">
            <div class="absolute w-14 h-14 bg-red-600 rounded-full shadow-lg transition-transform duration-75 pointer-events-none" id="joystick-handle" style="transform: translate(0px, 0px);"></div>
        </div>

        <!-- Boutons d'Action -->
        <div class="flex flex-col space-y-3">
            <button id="btn-enter" class="w-20 h-14 bg-blue-600 active:bg-blue-700 text-white font-bold rounded-xl shadow-lg border-b-4 border-blue-800 active:border-b-0 active:translate-y-1 transition-all text-xs uppercase">Monter</button>
            <button id="btn-action" class="w-20 h-14 bg-red-600 active:bg-red-700 text-white font-bold rounded-xl shadow-lg border-b-4 border-red-800 active:border-b-0 active:translate-y-1 transition-all text-xs uppercase">Tirer / Frapper</button>
        </div>
    </footer>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Redimensionnement dynamique adapté aux écrans mobiles
        function resizeCanvas() {
            const maxWidth = Math.min(window.innerWidth, 600);
            const maxHeight = window.innerHeight - 220;
            canvas.width = maxWidth;
            canvas.height = maxHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // Variables du Monde et de la Caméra
        const worldWidth = 2000;
        const worldHeight = 2000;
        
        let score = 0;
        let inVehicle = false;

        // Joueur
        let player = {
            x: 1000,
            y: 1000,
            radius: 12,
            angle: 0,
            speed: 3,
            color: '#3b82f6'
        };

        // Véhicule actuel
        let currentCar = null;

        // Génération de décors (Bâtiments et Voitures garées)
        let buildings = [];
        for (let i = 0; i < 25; i++) {
            buildings.push({
                x: Math.random() * (worldWidth - 300) + 150,
                y: Math.random() * (worldHeight - 300) + 150,
                w: 120 + Math.random() * 120,
                h: 120 + Math.random() * 120,
                color: '#1f2937'
            });
        }

        let cars = [];
        for (let i = 0; i < 10; i++) {
            cars.push({
                x: Math.random() * (worldWidth - 200) + 100,
                y: Math.random() * (worldHeight - 200) + 100,
                w: 40,
                h: 22,
                angle: Math.random() * Math.PI * 2,
                speed: 0,
                color: ['#ef4444', '#10b981', '#f59e0b', '#8b5cf6'][Math.floor(Math.random() * 4)]
            });
        }

        // États des touches / Joystick virtuel
        let joystickActive = false;
        let joystickVector = { x: 0, y: 0 };
        let touchOrigin = { x: 0, y: 0 };

        const joystickBase = document.getElementById('joystick-base');
        const joystickHandle = document.getElementById('joystick-handle');

        joystickBase.addEventListener('touchstart', (e) => {
            joystickActive = true;
            const rect = joystickBase.getBoundingClientRect();
            touchOrigin.x = rect.left + rect.width / 2;
            touchOrigin.y = rect.top + rect.height / 2;
            handleTouchMove(e.touches[0]);
        });

        window.addEventListener('touchmove', (e) => {
            if (!joystickActive) return;
            for (let i = 0; i < e.touches.length; i++) {
                const touch = e.touches[i];
                // Vérifier si le toucher concerne la zone gauche
                if (touch.clientX < window.innerWidth / 2) {
                    handleTouchMove(touch);
                }
            }
        });

        window.addEventListener('touchend', (e) => {
            // Si plus aucun toucher à gauche, désactiver le joystick
            let leftTouchActive = false;
            for (let i = 0; i < e.touches.length; i++) {
                if (e.touches[i].clientX < window.innerWidth / 2) leftTouchActive = true;
            }
            if (!leftTouchActive) {
                joystickActive = false;
                joystickVector = { x: 0, y: 0 };
                joystickHandle.style.transform = `translate(0px, 0px)`;
            }
        });

        function handleTouchMove(touch) {
            const maxDist = 45;
            let dx = touch.clientX - touchOrigin.x;
            let dy = touch.clientY - touchOrigin.y;
            const dist = Math.sqrt(dx * dx + dy * dy);

            if (dist > maxDist) {
                dx = (dx / dist) * maxDist;
                dy = (dy / dist) * maxDist;
            }

            joystickHandle.style.transform = `translate(${dx}px, ${dy}px)`;
            joystickVector = { x: dx / maxDist, y: dy / maxDist };
        }

        // Boutons d'action
        const btnEnter = document.getElementById('btn-enter');
        const btnAction = document.getElementById('btn-action');
        const msgBox = document.getElementById('msg-box');

        function showMessage(text) {
            msgBox.textContent = text;
            msgBox.classList.remove('opacity-0');
            setTimeout(() => {
                msgBox.classList.add('opacity-0');
            }, 2000);
        }

        btnEnter.addEventListener('click', () => {
            if (!inVehicle) {
                // Chercher une voiture proche
                let nearestCar = null;
                let minDist = 50;
                cars.forEach(car => {
                    const d = Math.hypot(player.x - car.x, player.y - car.y);
                    if (d < minDist) {
                        minDist = d;
                        nearestCar = car;
                    }
                });

                if (nearestCar) {
                    currentCar = nearestCar;
                    inVehicle = true;
                    document.getElementById('status-text').textContent = "En voiture";
                    document.getElementById('status-text').className = "text-green-500 uppercase tracking-widest";
                    showMessage("Véhicule volé !");
                } else {
                    showMessage("Aucun véhicule à proximité");
                }
            } else {
                // Sortir de la voiture
                player.x = currentCar.x + Math.cos(currentCar.angle + Math.PI/2) * 35;
                player.y = currentCar.y + Math.sin(currentCar.angle + Math.PI/2) * 35;
                inVehicle = false;
                currentCar = null;
                document.getElementById('status-text').textContent = "À pied";
                document.getElementById('status-text').className = "text-red-500 uppercase tracking-widest";
                showMessage("Sortie du véhicule");
            }
        });

        btnAction.addEventListener('click', () => {
            if (inVehicle) {
                showMessage("Klaxon / Boost activé !");
                score += 100;
            } else {
                showMessage("Coup de poing !");
                score += 10;
            }
            document.getElementById('score-display').textContent = `$${score}`;
        });

        // Boucle principale du jeu
        function update() {
            if (joystickVector.x !== 0 || joystickVector.y !== 0) {
                const moveAngle = Math.atan2(joystickVector.y, joystickVector.x);
                const intensity = Math.hypot(joystickVector.x, joystickVector.y);

                if (!inVehicle) {
                    player.angle = moveAngle;
                    player.x += Math.cos(player.angle) * player.speed * intensity;
                    player.y += Math.sin(player.angle) * player.speed * intensity;
                } else {
                    currentCar.angle = moveAngle;
                    currentCar.speed = 4 * intensity;
                    currentCar.x += Math.cos(currentCar.angle) * currentCar.speed;
                    currentCar.y += Math.sin(currentCar.angle) * currentCar.speed;
                }
            } else if (inVehicle && currentCar) {
                currentCar.speed *= 0.95; // Friction
                currentCar.x += Math.cos(currentCar.angle) * currentCar.speed;
                currentCar.y += Math.sin(currentCar.angle) * currentCar.speed;
            }

            // Limites de la carte
            const margin = 30;
            if (!inVehicle) {
                player.x = Math.max(margin, Math.min(worldWidth - margin, player.x));
                player.y = Math.max(margin, Math.min(worldHeight - margin, player.y));
            } else {
                currentCar.x = Math.max(margin, Math.min(worldWidth - margin, currentCar.x));
                currentCar.y = Math.max(margin, Math.min(worldHeight - margin, currentCar.y));
            }
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Centrer la caméra sur le joueur ou la voiture active
            const targetX = inVehicle ? currentCar.x : player.x;
            const targetY = inVehicle ? currentCar.y : player.y;

            const camX = canvas.width / 2 - targetX;
            const camY = canvas.height / 2 - targetY;

            ctx.save();
            ctx.translate(camX, camY);

            // Dessiner le sol (Rues et trottoirs)
            ctx.fillStyle = '#2b2b2b';
            ctx.fillRect(0, 0, worldWidth, worldHeight);

            // Quadrillage routier décoratif
            ctx.strokeStyle = '#383838';
            ctx.lineWidth = 4;
            for (let x = 0; x < worldWidth; x += 250) {
                ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, worldHeight); ctx.stroke();
            }
            for (let y = 0; y < worldHeight; y += 250) {
                ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(worldWidth, y); ctx.stroke();
            }

            // Dessiner les bâtiments
            buildings.forEach(b => {
                ctx.fillStyle = b.color;
                ctx.fillRect(b.x, b.y, b.w, b.h);
                ctx.strokeStyle = '#374151';
                ctx.lineWidth = 2;
                ctx.strokeRect(b.x, b.y, b.w, b.h);
            });

            // Dessiner les voitures
            cars.forEach(car => {
                ctx.save();
                ctx.translate(car.x, car.y);
                ctx.rotate(car.angle);
                ctx.fillStyle = car.color;
                ctx.fillRect(-car.w / 2, -car.h / 2, car.w, car.h);
                // Pare-brise
                ctx.fillStyle = '#93c5fd';
                ctx.fillRect(-5, -car.h / 2 + 4, 10, 6);
                ctx.restore();
            });

            // Dessiner le joueur si non véhiculé
            if (!inVehicle) {
                ctx.save();
                ctx.translate(player.x, player.y);
                ctx.rotate(player.angle);
                ctx.fillStyle = player.color;
                ctx.beginPath();
                ctx.arc(0, 0, player.radius, 0, Math.PI * 2);
                ctx.fill();
                // Direction du regard / bras
                ctx.fillStyle = '#1e3a8a';
                ctx.fillRect(4, -3, 8, 6);
                ctx.restore();
            }

            ctx.restore();
        }

        function loop() {
            update();
            draw();
            requestAnimationFrame(loop);
        }

        loop();
    </script>
</body>
</html>

