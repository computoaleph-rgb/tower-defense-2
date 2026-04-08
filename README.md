<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Neon Grid Tower Defense</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.0/p5.js"></script>
    <style>
        :root {
            --bg-color: #0b0f19;
            --accent: #00f2ff;
            --danger: #ff0055;
            --money: #ffd700;
            --panel: rgba(20, 26, 42, 0.9);
        }

        body {
            margin: 0;
            background-color: var(--bg-color);
            color: white;
            font-family: 'Segoe UI', Roboto, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
        }

        #game-wrapper {
            position: relative;
            border: 2px solid #1e293b;
            border-radius: 12px;
            box-shadow: 0 0 40px rgba(0, 0, 0, 0.7);
        }

        .hud {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            padding: 15px 25px;
            display: flex;
            justify-content: space-between;
            background: linear-gradient(to bottom, var(--panel), transparent);
            pointer-events: none;
            box-sizing: border-box;
            z-index: 10;
        }

        .stat-box {
            display: flex;
            flex-direction: column;
        }

        .label { font-size: 0.7rem; text-transform: uppercase; letter-spacing: 1px; opacity: 0.7; }
        .value { font-size: 1.4rem; font-weight: 800; }
        .money-val { color: var(--money); }
        .lives-val { color: var(--danger); }

        .instructions {
            position: absolute;
            bottom: 15px;
            right: 15px;
            background: var(--panel);
            padding: 8px 15px;
            border-radius: 6px;
            font-size: 0.8rem;
            border: 1px solid #334155;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="game-wrapper">
        <div class="hud">
            <div class="stat-box">
                <span class="label">Créditos</span>
                <span id="money" class="value money-val">200</span>
            </div>
            <div class="stat-box" style="text-align: right;">
                <span class="label">Integridad Base</span>
                <span id="lives" class="value lives-val">10</span>
            </div>
        </div>
        <div class="instructions">Click en casilla vacía: 50 créditos</div>
    </div>

    <script>
        const GRID_SIZE = 50;
        let enemies = [];
        let towers = [];
        let projectiles = [];
        let waypoints = [];
        let pathCells = [];
        let money = 200;
        let lives = 10;

        function setup() {
            let cnv = createCanvas(800, 600);
            cnv.parent('game-wrapper');

            // Definición de la ruta (Waypoints)
            waypoints = [
                {x: -50, y: 325},
                {x: 225, y: 325},
                {x: 225, y: 125},
                {x: 575, y: 125},
                {x: 575, y: 475},
                {x: 850, y: 475}
            ];

            mapOutPath();
        }

        function draw() {
            background(11, 15, 25);
            
            drawGrid();
            drawPath();

            // Spawn de enemigos
            if (frameCount % 100 === 0) enemies.push(new Enemy());

            // Actualización de Torres
            for (let t of towers) {
                t.update();
                t.display();
            }

            // Actualización de Proyectiles
            for (let i = projectiles.length - 1; i >= 0; i--) {
                projectiles[i].update();
                projectiles[i].display();
                if (projectiles[i].dead) projectiles.splice(i, 1);
            }

            // Actualización de Enemigos
            for (let i = enemies.length - 1; i >= 0; i--) {
                enemies[i].update();
                enemies[i].display();

                if (enemies[i].x > width + 20) {
                    lives--;
                    updateUI();
                    enemies.splice(i, 1);
                } else if (enemies[i].hp <= 0) {
                    money += 20;
                    updateUI();
                    enemies.splice(i, 1);
                }
            }

            drawGhostTower();
            if (lives <= 0) gameOver();
        }

        // --- LÓGICA DE ESCENARIO ---

        function drawGrid() {
            stroke(30, 41, 59, 100);
            strokeWeight(1);
            for (let x = 0; x <= width; x += GRID_SIZE) line(x, 0, x, height);
            for (let y = 0; y <= height; y += GRID_SIZE) line(0, y, width, y);
        }

        function drawPath() {
            stroke(30, 41, 59);
            strokeWeight(GRID_SIZE);
            noFill();
            beginShape();
            for (let p of waypoints) vertex(p.x, p.y);
            endShape();
            
            // Línea central neón del camino
            stroke(0, 242, 255, 30);
            strokeWeight(2);
            beginShape();
            for (let p of waypoints) vertex(p.x, p.y);
            endShape();
        }

        function mapOutPath() {
            // Identificar qué casillas del grid son camino
            for (let i = 0; i < waypoints.length - 1; i++) {
                let start = waypoints[i];
                let end = waypoints[i+1];
                for (let x = 0; x < width; x += GRID_SIZE) {
                    for (let y = 0; y < height; y += GRID_SIZE) {
                        if (distToSegment(x + 25, y + 25, start.x, start.y, end.x, end.y) < 25) {
                            pathCells.push(`${x},${y}`);
                        }
                    }
                }
            }
        }

        function drawGhostTower() {
            let gx = floor(mouseX / GRID_SIZE) * GRID_SIZE;
            let gy = floor(mouseY / GRID_SIZE) * GRID_SIZE;
            
            if (isValidPlacement(gx, gy)) fill(0, 242, 255, 40);
            else fill(255, 0, 85, 40);
            
            noStroke();
            rect(gx, gy, GRID_SIZE, GRID_SIZE, 4);
        }

        function isValidPlacement(gx, gy) {
            if (pathCells.includes(`${gx},${gy}`)) return false;
            for (let t of towers) {
                if (t.gx === gx && t.gy === gy) return false;
            }
            return true;
        }

        // --- CLASES ---

        class Enemy {
            constructor() {
                this.x = waypoints[0].x;
                this.y = waypoints[0].y;
                this.hp = 100;
                this.maxHp = 100;
                this.speed = 1.5;
                this.node = 1;
            }
            update() {
                let target = waypoints[this.node];
                let angle = atan2(target.y - this.y, target.x - this.x);
                this.x += cos(angle) * this.speed;
                this.y += sin(angle) * this.speed;
                if (dist(this.x, this.y, target.x, target.y) < 2) this.node++;
            }
            display() {
                fill(255, 0, 85);
                noStroke();
                ellipse(this.x, this.y, 18, 18);
                // Salud
                fill(15);
                rect(this.x - 10, this.y - 15, 20, 3);
                fill(0, 255, 150);
                rect(this.x - 10, this.y - 15, (this.hp/this.maxHp)*20, 3);
            }
        }

        class Tower {
            constructor(gx, gy) {
                this.gx = gx; this.gy = gy;
                this.x = gx + GRID_SIZE/2;
                this.y = gy + GRID_SIZE/2;
                this.range = 150;
                this.timer = 0;
            }
            update() {
                if (this.timer > 0) this.timer--;
                else {
                    for (let e of enemies) {
                        if (dist(this.x, this.y, e.x, e.y) < this.range) {
                            projectiles.push(new Projectile(this.x, this.y, e));
                            this.timer = 35;
                            break;
                        }
                    }
                }
            }
            display() {
                // Glow
                noStroke();
                fill(0, 242, 255, 20);
                ellipse(this.x, this.y, this.range * 2);
                // Body
                fill(20, 30, 50);
                stroke(0, 242, 255);
                strokeWeight(2);
                rectMode(CENTER);
                rect(this.x, this.y, 30, 30, 4);
                fill(0, 242, 255);
                ellipse(this.x, this.y, 10, 10);
            }
        }

        class Projectile {
            constructor(x, y, target) {
                this.x = x; this.y = y; this.target = target;
                this.dead = false;
            }
            update() {
                let angle = atan2(this.target.y - this.y, this.target.x - this.x);
                this.x += cos(angle) * 7;
                this.y += sin(angle) * 7;
                if (dist(this.x, this.y, this.target.x, this.target.y) < 10) {
                    this.target.hp -= 25;
                    this.dead = true;
                }
            }
            display() {
                stroke(255, 255, 100);
                strokeWeight(3);
                line(this.x, this.y, this.x - cos(0)*5, this.y - sin(0)*5);
            }
        }

        // --- UTILIDADES ---

        function mousePressed() {
            let gx = floor(mouseX / GRID_
