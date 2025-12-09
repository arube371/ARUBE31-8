# ARUBE31-8
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fireworks Generator</title>
    <style>
        /*
        Custom CSS for the game/simulation environment.
        Aesthetics are crucial here for the dark sky and glow effects.
        */
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden; /* Prevent scrollbars */
            background-color: #000;
            font-family: "Inter", sans-serif;
        }

        /* The canvas should fill the entire viewport */
        #fireworksCanvas {
            display: block;
            background-color: #000;
        }

        .controls {
            position: fixed;
            top: 10px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            padding: 10px 20px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
            backdrop-filter: blur(5px);
            text-align: center;
        }
        .controls p {
            margin: 0;
            font-size: 0.9rem;
        }

    </style>
</head>
<body>

    <canvas id="fireworksCanvas"></canvas>

    <div class="controls">
        <p>✨ Click anywhere to launch a firework! ✨</p>
    </div>

    <script>
        // Wrap the entire script in an IIFE to create a local scope and prevent
        // 'Identifier already declared' errors in the global scope.
        (function() {
            // --- Setup Canvas and Constants ---
            const canvas = document.getElementById('fireworksCanvas');
            const ctx = canvas.getContext('2d');
            let fireworks = []; // Array to hold all active firework rockets and explosions
            let hue = 0; // Base color hue for cycling colors
            let W = 0, H = 0; // Canvas dimensions

            // Physics constants
            const GRAVITY = 0.05;
            const PARTICLE_COUNT = 75; // Number of particles per explosion

            // Function to resize canvas to full screen
            function resizeCanvas() {
                W = canvas.width = window.innerWidth;
                H = canvas.height = window.innerHeight;
            }
            window.addEventListener('resize', resizeCanvas);
            resizeCanvas(); // Initial call

            // Helper function for random numbers
            const random = (min, max) => Math.random() * (max - min) + min;

            // --- Particle Class (The Sparks) ---
            class Particle {
                constructor(x, y, hue) {
                    this.x = x;
                    this.y = y;
                    // Random velocity in a circle
                    const angle = random(0, Math.PI * 2);
                    const speed = random(1, 10);
                    this.vx = Math.cos(angle) * speed;
                    this.vy = Math.sin(angle) * speed;

                    this.friction = 0.95; // Slows particle movement slightly
                    this.gravity = GRAVITY;
                    this.alpha = 1;
                    this.decay = random(0.015, 0.03); // Rate of fade
                    this.color = `hsl(${hue}, 100%, 50%)`;
                    this.size = random(1, 3);
                }

                update() {
                    // Apply friction (drag)
                    this.vx *= this.friction;
                    this.vy *= this.friction;
                    // Apply gravity
                    this.vy += this.gravity;

                    this.x += this.vx;
                    this.y += this.vy;

                    // Fade out
                    this.alpha -= this.decay;
                }

                draw() {
                    ctx.beginPath();
                    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
                    ctx.fillStyle = `hsla(${hue}, 100%, 50%, ${this.alpha})`;
                    ctx.fill();
                }
            }

            // --- Firework Class (The Rocket) ---
            class Firework {
                constructor(startX, startY, targetX, targetY) {
                    this.x = startX;
                    this.y = startY;
                    this.sx = startX; // Starting X
                    this.sy = startY; // Starting Y
                    this.targetX = targetX;
                    this.targetY = targetY;
                    this.distanceToTarget = this.calculateDistance(startX, startY, targetX, targetY);
                    this.distanceTraveled = 0;

                    this.hue = hue;
                    this.lineWidth = 2;

                    // Calculate the velocity needed to travel
                    const angle = Math.atan2(targetY - startY, targetX - startX);
                    // Adjust speed based on distance for a more uniform rise time
                    const speed = 10;
                    this.vx = Math.cos(angle) * speed;
                    this.vy = Math.sin(angle) * speed;

                    this.exploded = false;
                    this.particles = [];
                }

                calculateDistance(p1x, p1y, p2x, p2y) {
                    return Math.sqrt(Math.pow(p2x - p1x, 2) + Math.pow(p2y - p1y, 2));
                }

                update() {
                    if (this.exploded) {
                        // Update all particles in the explosion
                        for (let i = this.particles.length - 1; i >= 0; i--) {
                            this.particles[i].update();
                            // Remove dead particles
                            if (this.particles[i].alpha <= this.particles[i].decay) {
                                this.particles.splice(i, 1);
                            }
                        }
                        // Firework is considered done once all particles are gone
                        return this.particles.length === 0;

                    } else {
                        // Check if target is reached
                        const currentDistance = this.calculateDistance(this.sx, this.sy, this.x, this.y);
                        if (currentDistance >= this.distanceToTarget) {
                            this.exploded = true;
                            this.explode();
                            return false; // Still active because particles are flying
                        } else {
                            // Move the rocket
                            this.x += this.vx;
                            this.y += this.vy;
                            // Apply gravity slightly (makes the arc feel more natural)
                            this.vy += GRAVITY * 0.5;
                            return false; // Not done yet
                        }
                    }
                }

                explode() {
                    // Create many particles shooting out in all directions
                    for (let i = 0; i < PARTICLE_COUNT; i++) {
                        this.particles.push(new Particle(this.x, this.y, this.hue));
                    }
                }

                draw() {
                    if (this.exploded) {
                        // Draw all particles
                        this.particles.forEach(p => p.draw());
                    } else {
                        // Draw the ascending rocket (a small, bright circle)
                        ctx.beginPath();
                        ctx.arc(this.x, this.y, this.lineWidth, 0, Math.PI * 2, false);
                        ctx.fillStyle = `hsl(${this.hue}, 100%, 75%)`;
                        ctx.fill();

                        // Optional: Draw a subtle trail
                        const trailLength = 5;
                        const trailX = this.x - this.vx * trailLength;
                        const trailY = this.y - this.vy * trailLength;
                        ctx.beginPath();
                        ctx.moveTo(this.x, this.y);
                        ctx.lineTo(trailX, trailY);
                        ctx.strokeStyle = `hsla(${this.hue}, 100%, 75%, 0.8)`;
                        ctx.lineWidth = 1;
                        ctx.stroke();
                    }
                }
            }

            // --- Animation Loop ---
            function animate() {
                // Request the next frame
                requestAnimationFrame(animate);

                // 1. Draw a semi-transparent black rectangle over the whole canvas
                // This creates the trail/fade effect. Lower alpha for longer trails.
                ctx.fillStyle = 'rgba(0, 0, 0, 0.1)';
                ctx.fillRect(0, 0, W, H);

                // 2. Set blend mode for brighter colors and glow effect
                ctx.globalCompositeOperation = 'lighter';

                // 3. Update and draw fireworks
                for (let i = fireworks.length - 1; i >= 0; i--) {
                    const firework = fireworks[i];
                    firework.draw();
                    const isFinished = firework.update();

                    // If the firework is finished (rocket hit target and all particles are gone)
                    if (isFinished) {
                        fireworks.splice(i, 1);
                    }
                }

                // 4. Reset blend mode
                ctx.globalCompositeOperation = 'source-over';

                // 5. Cycle the base color hue slowly
                hue += 0.5;
                if (hue > 360) hue = 0;
            }

            // --- Interaction ---

            function launchFirework(e) {
                // Handle both mouse and touch events
                const clientX = e.clientX || (e.touches && e.touches[0] ? e.touches[0].clientX : null);
                const clientY = e.clientY || (e.touches && e.touches[0] ? e.touches[0].clientY : null);

                if (clientX === null || clientY === null) return;

                const targetX = clientX;
                const targetY = clientY;

                // Launch from the bottom center of the screen
                const startX = W / 2;
                const startY = H;

                fireworks.push(new Firework(startX, startY, targetX, targetY));

                // Cycle the hue slightly for the next firework
                hue = random(0, 360);
            }

            canvas.addEventListener('mousedown', launchFirework);
            canvas.addEventListener('touchstart', launchFirework);

            // --- Auto-Launch (Optional) ---
            let autoLaunchTimer;

            function autoLaunch() {
                if (fireworks.length < 5) { // Limit concurrent fireworks
                    const startX = random(W * 0.3, W * 0.7);
                    const targetX = random(W * 0.2, W * 0.8);
                    const targetY = random(H * 0.1, H * 0.4);

                    fireworks.push(new Firework(startX, H, targetX, targetY));
                    hue = random(0, 360);
                }
                autoLaunchTimer = setTimeout(autoLaunch, random(500, 2000));
            }

            // Start the animation loop and optional auto-launch
            window.onload = function() {
                animate();
                autoLaunch();
            }
        })(); // End of IIFE

    </script>
</body>
</html>
photos and some codes
![IMG-20250516-WA0065](https://github.com/user-attachments/assets/2174b4df-72a2-4c2f-8a94-4588f4ea0193)
![IMG-20250516-WA0070](https://github.com/user-attachments/assets/80777b9c-fb3b-4e56-95b4-7b1ba72c083d)
![IMG-20250516-WA0075](https://github.com/user-attachments/assets/1b066ff4-4fb9-46f1-b025-560a30a92b60)
![IMG-20250616-WA0043](https://github.com/user-attachments/assets/5cd3c7e6-1e57-4dc2-be41-f2721e915604)
![IMG-20250616-WA0044](https://github.com/user-attachments/assets/15ced4b7-2dee-4bf0-bb22-398103dfcd60)
![IMG-20250616-WA0050](https://github.com/user-attachments/assets/608d7810-88df-480f-8c98-7455fbfe8d91)
![IMG-20250616-WA0051](https://github.com/user-attachments/assets/f1dcebf4-78c0-4430-aa57-2a7b60e1bf8c)
![IMG-20250616-WA0075](https://github.com/user-attachments/assets/904a72c6-cfc7-49df-835e-0fbca8ef776f)
![IMG-20250616-WA0076](https://github.com/user-attachments/assets/a3001a29-474d-47a4-abcc-f6c2987f7c93)
![IMG-20250616-WA0078](https://github.com/user-attachments/assets/6b88a787-f26d-4440-ac14-3045e07c9cd3)
![IMG-20250616-WA0079](https://github.com/user-attachments/assets/8b2913a1-f293-4d58-b943-cf39debf1e08)
![IMG-20250616-WA0080](https://github.com/user-attachments/assets/fccb5221-cabd-4e8d-a6f8-8606439f8fed)
![IMG-20250616-WA0089](https://github.com/user-attachments/assets/60804985-dcd6-47e9-8cbe-58aae510cdad)
![IMG-20250616-WA0094](https://github.com/user-attachments/assets/4b92eb0d-3c25-4cf9-b98d-b1abf64992f9)
![IMG-20250616-WA0095](https://github.com/user-attachments/assets/55db2140-715a-45e7-a2c9-869dd2477507)
![IMG-20250617-WA0062](https://github.com/user-attachments/assets/5fe34745-e683-4809-b4e5-37ba080bb7ba)
![IMG-20250720-WA0043](https://github.com/user-attachments/assets/8c2b919e-08ff-43e0-838f-42ce6f571616)
![IMG-20250728-WA0016](https://github.com/user-attachments/assets/58bd4863-277c-4c9a-98cf-5ee758f0dd36)
![IMG-20250728-WA0042](https://github.com/user-attachments/assets/1f9d2d82-5647-4db4-b6cc-bcc983dc4cc9)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fireworks Generator</title>
    <style>
        /*
        Custom CSS for the game/simulation environment.
        Aesthetics are crucial here for the dark sky and glow effects.
        */
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden; /* Prevent scrollbars */
            background-color: #000;
            font-family: "Inter", sans-serif;
        }

        /* The canvas should fill the entire viewport */
        #fireworksCanvas {
            display: block;
            background-color: #000;
        }

        .controls {
            position: fixed;
            top: 10px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            padding: 10px 20px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
            backdrop-filter: blur(5px);
            text-align: center;
        }
        .controls p {
            margin: 0;
            font-size: 0.9rem;
        }

    </style>
</head>
<body>

    <canvas id="fireworksCanvas"></canvas>

    <div class="controls">
        <p>✨ Click anywhere to launch a firework! ✨</p>
    </div>

    <script>
        // Wrap the entire script in an IIFE to create a local scope and prevent
        // 'Identifier already declared' errors in the global scope.
        (function() {
            // --- Setup Canvas and Constants ---
            const canvas = document.getElementById('fireworksCanvas');
            const ctx = canvas.getContext('2d');
            let fireworks = []; // Array to hold all active firework rockets and explosions
            let hue = 0; // Base color hue for cycling colors
            let W = 0, H = 0; // Canvas dimensions

            // Physics constants
            const GRAVITY = 0.05;
            const PARTICLE_COUNT = 75; // Number of particles per explosion

            // Function to resize canvas to full screen
            function resizeCanvas() {
                W = canvas.width = window.innerWidth;
                H = canvas.height = window.innerHeight;
            }
            window.addEventListener('resize', resizeCanvas);
            resizeCanvas(); // Initial call

            // Helper function for random numbers
            const random = (min, max) => Math.random() * (max - min) + min;

            // --- Particle Class (The Sparks) ---
            class Particle {
                constructor(x, y, hue) {
                    this.x = x;
                    this.y = y;
                    // Random velocity in a circle
                    const angle = random(0, Math.PI * 2);
                    const speed = random(1, 10);
                    this.vx = Math.cos(angle) * speed;
                    this.vy = Math.sin(angle) * speed;

                    this.friction = 0.95; // Slows particle movement slightly
                    this.gravity = GRAVITY;
                    this.alpha = 1;
                    this.decay = random(0.015, 0.03); // Rate of fade
                    this.color = `hsl(${hue}, 100%, 50%)`;
                    this.size = random(1, 3);
                }

                update() {
                    // Apply friction (drag)
                    this.vx *= this.friction;
                    this.vy *= this.friction;
                    // Apply gravity
                    this.vy += this.gravity;

                    this.x += this.vx;
                    this.y += this.vy;

                    // Fade out
                    this.alpha -= this.decay;
                }

                draw() {
                    ctx.beginPath();
                    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2, false);
                    ctx.fillStyle = `hsla(${hue}, 100%, 50%, ${this.alpha})`;
                    ctx.fill();
                }
            }

            // --- Firework Class (The Rocket) ---
            class Firework {
                constructor(startX, startY, targetX, targetY) {
                    this.x = startX;
                    this.y = startY;
                    this.sx = startX; // Starting X
                    this.sy = startY; // Starting Y
                    this.targetX = targetX;
                    this.targetY = targetY;
                    this.distanceToTarget = this.calculateDistance(startX, startY, targetX, targetY);
                    this.distanceTraveled = 0;

                    this.hue = hue;
                    this.lineWidth = 2;

                    // Calculate the velocity needed to travel
                    const angle = Math.atan2(targetY - startY, targetX - startX);
                    // Adjust speed based on distance for a more uniform rise time
                    const speed = 10;
                    this.vx = Math.cos(angle) * speed;
                    this.vy = Math.sin(angle) * speed;

                    this.exploded = false;
                    this.particles = [];
                }

                calculateDistance(p1x, p1y, p2x, p2y) {
                    return Math.sqrt(Math.pow(p2x - p1x, 2) + Math.pow(p2y - p1y, 2));
                }

                update() {
                    if (this.exploded) {
                        // Update all particles in the explosion
                        for (let i = this.particles.length - 1; i >= 0; i--) {
                            this.particles[i].update();
                            // Remove dead particles
                            if (this.particles[i].alpha <= this.particles[i].decay) {
                                this.particles.splice(i, 1);
                            }
                        }
                        // Firework is considered done once all particles are gone
                        return this.particles.length === 0;

                    } else {
                        // Check if target is reached
                        const currentDistance = this.calculateDistance(this.sx, this.sy, this.x, this.y);
                        if (currentDistance >= this.distanceToTarget) {
                            this.exploded = true;
                            this.explode();
                            return false; // Still active because particles are flying
                        } else {
                            // Move the rocket
                            this.x += this.vx;
                            this.y += this.vy;
                            // Apply gravity slightly (makes the arc feel more natural)
                            this.vy += GRAVITY * 0.5;
                            return false; // Not done yet
                        }
                    }
                }

                explode() {
                    // Create many particles shooting out in all directions
                    for (let i = 0; i < PARTICLE_COUNT; i++) {
                        this.particles.push(new Particle(this.x, this.y, this.hue));
                    }
                }

                draw() {
                    if (this.exploded) {
                        // Draw all particles
                        this.particles.forEach(p => p.draw());
                    } else {
                        // Draw the ascending rocket (a small, bright circle)
                        ctx.beginPath();
                        ctx.arc(this.x, this.y, this.lineWidth, 0, Math.PI * 2, false);
                        ctx.fillStyle = `hsl(${this.hue}, 100%, 75%)`;
                        ctx.fill();

                        // Optional: Draw a subtle trail
                        const trailLength = 5;
                        const trailX = this.x - this.vx * trailLength;
                        const trailY = this.y - this.vy * trailLength;
                        ctx.beginPath();
                        ctx.moveTo(this.x, this.y);
                        ctx.lineTo(trailX, trailY);
                        ctx.strokeStyle = `hsla(${this.hue}, 100%, 75%, 0.8)`;
                        ctx.lineWidth = 1;
                        ctx.stroke();
                    }
                }
            }

            // --- Animation Loop ---
            function animate() {
                // Request the next frame
                requestAnimationFrame(animate);

                // 1. Draw a semi-transparent black rectangle over the whole canvas
                // This creates the trail/fade effect. Lower alpha for longer trails.
                ctx.fillStyle = 'rgba(0, 0, 0, 0.1)';
                ctx.fillRect(0, 0, W, H);

                // 2. Set blend mode for brighter colors and glow effect
                ctx.globalCompositeOperation = 'lighter';

                // 3. Update and draw fireworks
                for (let i = fireworks.length - 1; i >= 0; i--) {
                    const firework = fireworks[i];
                    firework.draw();
                    const isFinished = firework.update();

                    // If the firework is finished (rocket hit target and all particles are gone)
                    if (isFinished) {
                        fireworks.splice(i, 1);
                    }
                }

                // 4. Reset blend mode
                ctx.globalCompositeOperation = 'source-over';

                // 5. Cycle the base color hue slowly
                hue += 0.5;
                if (hue > 360) hue = 0;
            }

            // --- Interaction ---

            function launchFirework(e) {
                // Handle both mouse and touch events
                const clientX = e.clientX || (e.touches && e.touches[0] ? e.touches[0].clientX : null);
                const clientY = e.clientY || (e.touches && e.touches[0] ? e.touches[0].clientY : null);

                if (clientX === null || clientY === null) return;

                const targetX = clientX;
                const targetY = clientY;

                // Launch from the bottom center of the screen
                const startX = W / 2;
                const startY = H;

                fireworks.push(new Firework(startX, startY, targetX, targetY));

                // Cycle the hue slightly for the next firework
                hue = random(0, 360);
            }

            canvas.addEventListener('mousedown', launchFirework);
            canvas.addEventListener('touchstart', launchFirework);

            // --- Auto-Launch (Optional) ---
            let autoLaunchTimer;

            function autoLaunch() {
                if (fireworks.length < 5) { // Limit concurrent fireworks
                    const startX = random(W * 0.3, W * 0.7);
                    const targetX = random(W * 0.2, W * 0.8);
                    const targetY = random(H * 0.1, H * 0.4);

                    fireworks.push(new Firework(startX, H, targetX, targetY));
                    hue = random(0, 360);
                }
                autoLaunchTimer = setTimeout(autoLaunch, random(500, 2000));
            }

            // Start the animation loop and optional auto-launch
            window.onload = function() {
                animate();
                autoLaunch();
            }
        })(); // End of IIFE

    </script>
</body>
</html>
