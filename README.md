<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>ESP e Speed Control</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            touch-action: none;
            user-select: none;
            -webkit-user-select: none;
        }
        
        body {
            font-family: Arial, sans-serif;
            background-color: #222;
            color: white;
            overflow: hidden;
            width: 100vw;
            height: 100vh;
            position: fixed;
        }
        
        #interface {
            position: absolute;
            width: 280px;
            background: rgba(30, 30, 40, 0.85);
            border: 2px solid #5a5a8a;
            border-radius: 12px;
            box-shadow: 0 0 15px rgba(0, 0, 0, 0.6);
            overflow: hidden;
            z-index: 1000;
        }
        
        #header {
            padding: 12px;
            background: rgba(45, 45, 65, 0.95);
            cursor: move;
            text-align: center;
            font-weight: bold;
            border-bottom: 1px solid #6666aa;
        }
        
        .content {
            padding: 15px;
        }
        
        .section {
            margin-bottom: 15px;
            padding: 10px;
            background: rgba(40, 40, 55, 0.7);
            border-radius: 8px;
        }
        
        h3 {
            margin-bottom: 10px;
            color: #aaccff;
            text-align: center;
        }
        
        .slider-container {
            margin: 15px 0;
        }
        
        .slider {
            -webkit-appearance: none;
            width: 100%;
            height: 12px;
            border-radius: 6px;
            background: #44446a;
            outline: none;
        }
        
        .slider::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 24px;
            height: 24px;
            border-radius: 50%;
            background: #44aaff;
            cursor: pointer;
        }
        
        .value-display {
            text-align: center;
            margin-top: 8px;
            font-weight: bold;
            color: #88ccff;
        }
        
        .button {
            display: block;
            width: 100%;
            padding: 12px;
            margin: 8px 0;
            background: #4466cc;
            border: none;
            border-radius: 8px;
            color: white;
            font-weight: bold;
            text-align: center;
            cursor: pointer;
            transition: background 0.2s;
        }
        
        .button:active {
            background: #5588ee;
        }
        
        .toggle {
            background: #44aa66;
        }
        
        .toggle.off {
            background: #aa4466;
        }
        
        .close-btn {
            background: #cc4444;
            margin-top: 15px;
        }
        
        .entity {
            position: absolute;
            border: 2px solid #00ff00;
            border-radius: 4px;
            background: rgba(0, 255, 0, 0.1);
            pointer-events: none;
        }
        
        .entity-label {
            position: absolute;
            color: #00ff00;
            font-size: 12px;
            background: rgba(0, 0, 0, 0.6);
            padding: 2px 6px;
            border-radius: 3px;
            pointer-events: none;
            white-space: nowrap;
        }
    </style>
</head>
<body>
    <div id="interface">
        <div id="header">ESP & Controle de Velocidade</div>
        <div class="content">
            <div class="section">
                <h3>Controle de Velocidade</h3>
                <div class="slider-container">
                    <input type="range" min="1" max="5" value="1" step="0.1" class="slider" id="speedSlider">
                    <div class="value-display">Velocidade: <span id="speedValue">1.0</span>x</div>
                </div>
            </div>
            
            <div class="section">
                <h3>Controles ESP</h3>
                <div class="button toggle" id="toggleESP">ESP: LIGADO</div>
                <div class="button" id="refreshEntities">Atualizar Entidades</div>
            </div>
            
            <div class="button close-btn" id="closeInterface">Fechar Interface</div>
        </div>
    </div>

    <script>
        // Variáveis globais
        let interfaceVisible = true;
        let espEnabled = true;
        let speedMultiplier = 1.0;
        let entities = [];
        let drag = false;
        let dragOffsetX, dragOffsetY;
        
        // Elementos da interface
        const interface = document.getElementById('interface');
        const header = document.getElementById('header');
        const speedSlider = document.getElementById('speedSlider');
        const speedValue = document.getElementById('speedValue');
        const toggleESP = document.getElementById('toggleESP');
        const refreshEntities = document.getElementById('refreshEntities');
        const closeInterface = document.getElementById('closeInterface');
        
        // Posição inicial da interface
        interface.style.left = '20px';
        interface.style.top = '20px';
        
        // Eventos de arrastar a interface
        header.addEventListener('touchstart', (e) => {
            drag = true;
            const touch = e.touches[0];
            const interfaceRect = interface.getBoundingClientRect();
            dragOffsetX = touch.clientX - interfaceRect.left;
            dragOffsetY = touch.clientY - interfaceRect.top;
            e.preventDefault();
        });
        
        document.addEventListener('touchmove', (e) => {
            if (!drag) return;
            
            const touch = e.touches[0];
            const x = touch.clientX - dragOffsetX;
            const y = touch.clientY - dragOffsetY;
            
            // Limitar a interface à área visível da tela
            const maxX = window.innerWidth - interface.offsetWidth;
            const maxY = window.innerHeight - interface.offsetHeight;
            
            interface.style.left = Math.max(0, Math.min(x, maxX)) + 'px';
            interface.style.top = Math.max(0, Math.min(y, maxY)) + 'px';
            
            e.preventDefault();
        });
        
        document.addEventListener('touchend', () => {
            drag = false;
        });
        
        // Controle de velocidade
        speedSlider.addEventListener('input', () => {
            speedMultiplier = parseFloat(speedSlider.value);
            speedValue.textContent = speedMultiplier.toFixed(1);
            adjustSpeed(speedMultiplier);
        });
        
        // Controle do ESP
        toggleESP.addEventListener('click', () => {
            espEnabled = !espEnabled;
            toggleESP.textContent = espEnabled ? 'ESP: LIGADO' : 'ESP: DESLIGADO';
            toggleESP.classList.toggle('off', !espEnabled);
            
            if (espEnabled) {
                updateESP();
            } else {
                clearESP();
            }
        });
        
        refreshEntities.addEventListener('click', () => {
            scanEntities();
            if (espEnabled) updateESP();
        });
        
        closeInterface.addEventListener('click', () => {
            interfaceVisible = false;
            interface.style.display = 'none';
            clearESP();
        });
        
        // Funções do sistema
        function adjustSpeed(multiplier) {
            console.log(`Velocidade ajustada para: ${multiplier}x`);
            // Aqui você implementaria a lógica real para ajustar a velocidade do personagem
        }
        
        function scanEntities() {
            // Simulação de entidades - em um ambiente real, você buscaria entidades reais do jogo
            entities = [];
            const entityCount = Math.floor(Math.random() * 5) + 3; // 3-7 entidades
            
            for (let i = 0; i < entityCount; i++) {
                entities.push({
                    id: i,
                    name: `Jogador${i+1}`,
                    x: Math.random() * window.innerWidth,
                    y: Math.random() * window.innerHeight,
                    width: 40 + Math.random() * 40,
                    height: 80 + Math.random() * 40,
                    distance: Math.floor(Math.random() * 100) + 10
                });
            }
            
            console.log(`${entities.length} entidades detectadas`);
        }
        
        function updateESP() {
            clearESP();
            
            if (!espEnabled) return;
            
            entities.forEach(entity => {
                // Criar caixa do ESP
                const box = document.createElement('div');
                box.className = 'entity';
                box.style.left = `${entity.x}px`;
                box.style.top = `${entity.y}px`;
                box.style.width = `${entity.width}px`;
                box.style.height = `${entity.height}px`;
                document.body.appendChild(box);
                
                // Criar label do ESP
                const label = document.createElement('div');
                label.className = 'entity-label';
                label.textContent = `${entity.name} [${entity.distance}m]`;
                label.style.left = `${entity.x}px`;
                label.style.top = `${entity.y - 20}px`;
                document.body.appendChild(label);
            });
        }
        
        function clearESP() {
            document.querySelectorAll('.entity, .entity-label').forEach(el => {
                el.remove();
            });
        }
        
        // Inicialização
        scanEntities();
        updateESP();
        
        // Simular movimento de entidades (apenas para demonstração)
        setInterval(() => {
            if (espEnabled && entities.length > 0) {
                entities.forEach(entity => {
                    entity.x += (Math.random() - 0.5) * 10;
                    entity.y += (Math.random() - 0.5) * 5;
                    
                    // Manter dentro dos limites da tela
                    entity.x = Math.max(0, Math.min(entity.x, window.innerWidth - entity.width));
                    entity.y = Math.max(0, Math.min(entity.y, window.innerHeight - entity.height));
                });
                updateESP();
            }
        }, 1000);
    </script>
</body>
</html>
