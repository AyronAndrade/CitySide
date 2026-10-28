<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Super Panda Jump - Exemplo Poki</title>
    <style>
        /* CSS para estilizar o canvas e a página */
        body {
            margin: 0;
            overflow: hidden;
            background-color: #333;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            color: white;
            font-family: 'Arial', sans-serif;
        }

        #gameContainer {
            position: relative;
            width: 800px;
            height: 600px;
            border: 2px solid #fff;
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
            background-color: #87ceeb; /* Cor de fundo do jogo */
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* Estilos para os menus */
        .menu {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0, 0, 0, 0.7);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            text-align: center;
            display: none; /* Oculto por padrão */
        }

        .active-menu {
            display: flex;
        }

        h1 {
            font-size: 64px;
            margin-bottom: 20px;
            text-shadow: 2px 2px 4px #000;
        }

        button {
            padding: 15px 30px;
            font-size: 32px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            transition: background-color 0.3s;
        }

        button:hover {
            background-color: #45a049;
        }

        #scoreDisplay {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 32px;
            text-shadow: 2px 2px 4px #000;
            display: none; /* Oculto no menu */
        }
    </style>
</head>
<body>

<div id="gameContainer">
    <canvas id="gameCanvas"></canvas>

    <!-- Menu Principal -->
    <div id="mainMenu" class="menu">
        <h1>Super Panda Jump</h1>
        <button onclick="iniciarJogo()">Jogar</button>
    </div>

    <!-- Menu de Game Over -->
    <div id="gameOverMenu" class="menu">
        <h1>Game Over</h1>
        <p id="finalScore" style="font-size: 32px; margin-bottom: 20px;">Pontuação: 0</p>
        <button onclick="reiniciarJogo()">Jogar Novamente</button>
    </div>

    <!-- Exibição da Pontuação durante o jogo -->
    <div id="scoreDisplay">Pontuação: <span id="scoreValue">0</span></div>

</div>

<!-- Incluir o SDK da Poki -->
<script src="https://game-cdn.poki-sdk.com/v2/poki-sdk.js"></script>
<script>
    // --- Inicialização da Poki ---
    let pokiSDK;

    PokiSDK.init()
        .then(() => {
            console.log("Poki SDK inicializado com sucesso");
            pokiSDK = PokiSDK;
            mostrarMenuPrincipal();
        })
        .catch(() => {
            console.error("Falha ao inicializar o Poki SDK");
            // Mesmo sem SDK, o jogo deve funcionar para testes locais
            mostrarMenuPrincipal();
        });

    // --- Configurações do Jogo ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const container = document.getElementById('gameContainer');

    // Ajustar o tamanho do canvas para coincidir com o container
    function resizeCanvas() {
        canvas.width = container.clientWidth;
        canvas.height = container.clientHeight;
    }
    resizeCanvas();
    window.addEventListener('resize', resizeCanvas);

    // Variáveis do Jogo
    let gameState = 'loading'; // 'loading', 'menu', 'playing', 'paused', 'gameOver'
    let score = 0;
    let gameLoopInterval;

    // Elementos do jogo
    const jogador = {
        x: 50,
        y: canvas.height - 100,
        largura: 40,
        altura: 60,
        cor: '#fff', // Panda é branco
        velocidadeX: 0,
        velocidadeY: 0,
        gravidade: 0.8,
        forcaPulo: -15,
        noChao: false,
        olhos: ['#000', '#000'], // Olhos pretos
        orelhas: ['#000', '#000'] // Orelhas pretas
    };

    const plataforma = {
        x: 100,
        y: canvas.height - 50,
        largura: canvas.width - 200,
        altura: 20,
        cor: '#555'
    };

    const moedas = [];
    const tamanhoMoeda = 20;
    let proximaMoedaTempo = 0;

    // --- Controle de Teclado ---
    const teclasPressionadas = {};

    window.addEventListener('keydown', (e) => {
        teclasPressionadas[e.code] = true;
    });

    window.addEventListener('keyup', (e) => {
        teclasPressionadas[e.code] = false;
    });

    // --- Funções do Jogo ---

    function mostrarMenuPrincipal() {
        gameState = 'menu';
        document.getElementById('mainMenu').classList.add('active-menu');
        document.getElementById('gameOverMenu').classList.remove('active-menu');
        document.getElementById('scoreDisplay').style.display = 'none';
    }

    function iniciarJogo() {
        gameState = 'playing';
        score = 0;
        document.getElementById('mainMenu').classList.remove('active-menu');
        document.getElementById('gameOverMenu').classList.remove('active-menu');
        document.getElementById('scoreDisplay').style.display = 'block';
        document.getElementById('scoreValue').innerText = score;

        // Resetar posição do jogador e elementos
        jogador.y = canvas.height - 100;
        jogador.velocidadeY = 0;
        moedas.length = 0;

        // Notificar Poki que o jogo começou
        if (pokiSDK) pokiSDK.gameStart();

        iniciarGameLoop();
    }

    function iniciarGameLoop() {
        if (gameLoopInterval) clearInterval(gameLoopInterval);
        gameLoopInterval = setInterval(atualizar, 1000 / 60); // 60 FPS
    }

    function atualizar() {
        if (gameState !== 'playing') return;

        // --- Jogador ---
        // Movimento Horizontal
        if (teclasPressionadas['ArrowLeft']) {
            jogador.velocidadeX = -5;
        } else if (teclasPressionadas['ArrowRight']) {
            jogador.velocidadeX = 5;
        } else {
            jogador.velocidadeX *= 0.9; // Atrito
        }

        jogador.x += jogador.velocidadeX;

        // Gravidade
        jogador.velocidadeY += jogador.gravidade;
        jogador.y += jogador.velocidadeY;

        // Pulo
        if (teclasPressionadas['ArrowUp'] && jogador.noChao) {
            jogador.velocidadeY = jogador.forcaPulo;
            jogador.noChao = false;
        }

        // Limites da Tela (Horizontal)
        if (jogador.x < 0) jogador.x = 0;
        if (jogador.x + jogador.largura > canvas.width) jogador.x = canvas.width - jogador.largura;

        // Limites da Tela (Vertical) - Chão
        if (jogador.y + jogador.altura > plataforma.y && jogador.y + jogador.altura < plataforma.y + plataforma.altura &&
            jogador.x + jogador.largura > plataforma.x && jogador.x < plataforma.x + plataforma.largura) {
            jogador.y = plataforma.y - jogador.altura;
            jogador.velocidadeY = 0;
            jogador.noChao = true;
        } else if (jogador.y + jogador.altura > plataforma.y) {
            // Caiu fora da plataforma
            jogador.y = plataforma.y - jogador.altura;
            jogador.velocidadeY = 0;
            jogador.noChao = true; // Considerando a plataforma como o único chão
        } else {
            jogador.noChao = false;
        }

        // Colisão com o fundo da tela (Game Over)
        if (jogador.y > canvas.height) {
            gameOver();
        }

        // --- Moedas ---
        // Gerar moedas periodicamente
        if (Date.now() > proximaMoedaTempo) {
            gerarMoeda();
            proximaMoedaTempo = Date.now() + 1500; // A cada 1.5 segundos
        }

        // Atualizar e verificar colisão com moedas
        for (let i = moedas.length - 1; i >= 0; i--) {
            const moeda = moedas[i];
            moeda.y += moeda.velocidadeY;

            // Colisão com o jogador
            if (moeda.x < jogador.x + jogador.largura &&
                moeda.x + tamanhoMoeda > jogador.x &&
                moeda.y < jogador.y + jogador.altura &&
                moeda.y + tamanhoMoeda > jogador.y) {
                // Coletou a moeda
                score += 10;
                document.getElementById('scoreValue').innerText = score;
                moedas.splice(i, 1);
                continue;
            }

            // Remover moeda se sair da tela
            if (moeda.y > canvas.height) {
                moedas.splice(i, 1);
            }
        }

        // --- Renderização ---
        limparCanvas();
        desenharPlataforma();
        desenharJogador();
        desenharMoedas();
    }

    function gerarMoeda() {
        const moedaX = Math.random() * (canvas.width - tamanhoMoeda);
        moedas.push({
            x: moedaX,
            y: -tamanhoMoeda,
            velocidadeY: 2 + Math.random() * 2,
            cor: '#FFD700' // Moedas são douradas
        });
    }

    function limparCanvas() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
    }

    function desenharPlataforma() {
        ctx.fillStyle = plataforma.cor;
        ctx.fillRect(plataforma.x, plataforma.y, plataforma.largura, plataforma.altura);
    }

    function desenharJogador() {
        // Corpo do Panda
        ctx.fillStyle = jogador.cor;
        ctx.fillRect(jogador.x, jogador.y, jogador.largura, jogador.altura);

        // Orelhas
        ctx.fillStyle = jogador.orelhas[0];
        ctx.beginPath();
        ctx.arc(jogador.x + jogador.largura * 0.15, jogador.y - jogador.altura * 0.1, jogador.largura * 0.2, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(jogador.x + jogador.largura * 0.85, jogador.y - jogador.altura * 0.1, jogador.largura * 0.2, 0, Math.PI * 2);
        ctx.fill();

        // Olhos
        ctx.fillStyle = jogador.olhos[0];
        ctx.beginPath();
        ctx.arc(jogador.x + jogador.largura * 0.3, jogador.y + jogador.altura * 0.3, jogador.largura * 0.15, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(jogador.x + jogador.largura * 0.7, jogador.y + jogador.altura * 0.3, jogador.largura * 0.15, 0, Math.PI * 2);
        ctx.fill();
    }

    function desenharMoedas() {
        for (const moeda of moedas) {
            ctx.fillStyle = moeda.cor;
            ctx.beginPath();
            ctx.arc(moeda.x + tamanhoMoeda / 2, moeda.y + tamanhoMoeda / 2, tamanhoMoeda / 2, 0, Math.PI * 2);
            ctx.fill();
        }
    }

    function gameOver() {
        if (gameLoopInterval) clearInterval(gameLoopInterval);
        gameState = 'gameOver';
        document.getElementById('gameOverMenu').classList.add('active-menu');
        document.getElementById('finalScore').innerText = "Pontuação: " + score;
        document.getElementById('scoreDisplay').style.display = 'none';

        // Notificar Poki que o jogo terminou
        if (pokiSDK) pokiSDK.gameover();
    }

    function reiniciarJogo() {
        iniciarJogo();
    }

</script>
</body>
</html>
