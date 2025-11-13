# uber-moto
uber moto
![unnamed](https://github.com/user-attachments/assets/2f6ff65d-14a6-45c7-99f3-381d76ec556e)


<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Uber Moto Clone - Rastreamento em Tempo Real</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Roboto', sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f4f7f6;
            color: #333;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
        }

        header {
            background-color: #000;
            color: #fff;
            padding: 20px;
            width: 100%;
            text-align: center;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }

        h1 {
            margin: 0;
            font-size: 2em;
        }

        main {
            flex-grow: 1;
            width: 90%;
            max-width: 1200px;
            padding: 20px 0;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .dashboard-controls {
            background-color: #fff;
            padding: 25px;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.05);
            margin-bottom: 30px;
            width: 100%;
            max-width: 600px;
            text-align: left;
        }

        .dashboard-controls p {
            margin: 10px 0;
            font-size: 1.1em;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .status-indicator {
            font-weight: bold;
            padding: 4px 8px;
            border-radius: 4px;
            color: #fff;
            background-color: #ccc;
        }
        .status-indicator.connected {
            background-color: #28a745; /* Verde */
        }
        .status-indicator.disconnected {
            background-color: #dc3545; /* Vermelho */
        }

        button {
            background-color: #007bff;
            color: #fff;
            border: none;
            padding: 12px 25px;
            border-radius: 5px;
            cursor: pointer;
            font-size: 1em;
            transition: background-color 0.3s ease;
            width: 100%;
            margin-top: 20px;
        }

        button:hover {
            background-color: #0056b3;
        }

        #map-container {
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 4px 8px rgba(0,0,0,0.05);
            width: 100%;
            height: 600px; /* Altura fixa para o mapa */
            overflow: hidden; /* Garante que o mapa se encaixe */
        }

        #map {
            width: 100%;
            height: 100%;
        }
    </style>
</head>
<body>
    <header>
        <h1>Painel de Rastreamento Uber Moto (Tempo Real)</h1>
    </header>

    <main>
        <section class="dashboard-controls">
            <h2>Status da Frota</h2>
            <p>
                <span>Motorista Conectado:</span>
                <span id="driver-status" class="status-indicator disconnected">Desconectado</span>
            </p>
            <p>
                <span>Última Posição:</span>
                <span id="last-location">Aguardando...</span>
            </p>
            <button onclick="simulateMovement()">Simular Movimento do Motorista</button>
        </section>
        
        <section id="map-container">
            <div id="map"></div>
        </section>
    </main>

    <script src="/socket.io/socket.io.js"></script>

    <script>
        // =========================================================================
        // ATENÇÃO: SUBSTITUA PELA SUA CHAVE DA API DO GOOGLE MAPS
        // Acesse o Google Cloud Console para obter/criar sua chave:
        // https://console.cloud.google.com/
        // Lembre-se de ATIVAR as APIs "Maps JavaScript API" e "Geolocation API"
        // e de configurar as restrições da chave (HTTP referrers para localhost:3000)
        // =========================================================================
        const GOOGLE_MAPS_API_KEY = 'SUA_API_KEY_GOOGLE_MAPS'; 
        
        let map;
        let driverMarker;
        const initialPosition = { lat: -15.7797, lng: -47.9297 }; // Exemplo: Brasília, Brasil
        
        // Conexão com o servidor Socket.io
        const socket = io('http://localhost:3000'); // Conecta ao seu servidor Node.js
        
        // --- Funções de Inicialização e Mapa ---

        function initMap() {
            map = new google.maps.Map(document.getElementById('map'), {
                zoom: 13,
                center: initialPosition,
                mapTypeControl: false,
                streetViewControl: false,
                fullscreenControl: false
            });

            // Ícone da moto para o marcador
            const motoIcon = {
                url: 'https://cdn-icons-png.flaticon.com/512/32/32338.png', // Exemplo de ícone de moto
                scaledSize: new google.maps.Size(40, 40), // Tamanho do ícone
                anchor: new google.maps.Point(20, 20) // Ponto de ancoragem para centralizar
            };

            driverMarker = new google.maps.Marker({
                position: initialPosition,
                map: map,
                title: 'Motorista Uber Moto',
                icon: motoIcon,
                animation: google.maps.Animation.DROP // Animação de "drop" ao aparecer
            });
        }
        
        // --- Lógica do Socket.io e Rastreamento ---
        
        socket.on('connect', () => {
            const statusElement = document.getElementById('driver-status');
            statusElement.innerText = 'Conectado';
            statusElement.classList.remove('disconnected');
            statusElement.classList.add('connected');
            console.log('Conectado ao servidor Socket.io.');
        });
        
        socket.on('disconnect', () => {
            const statusElement = document.getElementById('driver-status');
            statusElement.innerText = 'Desconectado';
            statusElement.classList.remove('connected');
            statusElement.classList.add('disconnected');
            console.log('Desconectado do servidor Socket.io.');
        });
        
        // Evento que recebe a nova localização do motorista
        socket.on('location_update', (data) => {
            const newPos = new google.maps.LatLng(data.lat, data.lng);
            
            // 1. Atualiza a posição do marcador no mapa
            driverMarker.setPosition(newPos);
            
            // 2. Centraliza o mapa no motorista (opcional, pode ser irritante se o mapa estiver em zoom alto)
            // map.panTo(newPos); 
            
            // 3. Atualiza o status no painel
            document.getElementById('last-location').innerText = `Lat: ${data.lat.toFixed(5)}, Lng: ${data.lng.toFixed(5)}`;
        });
        
        // --- Funções de Simulação (Simula o App do Motorista enviando dados) ---

        let simulationInterval;
        let currentLat = initialPosition.lat;
        let currentLng = initialPosition.lng;
        let isSimulating = false;

        function simulateMovement() {
            const button = document.querySelector('button');
            if (isSimulating) {
                clearInterval(simulationInterval);
                simulationInterval = null;
                button.innerText = 'Simular Movimento do Motorista';
                isSimulating = false;
                console.log('Simulação de movimento parada.');
            } else {
                button.innerText = 'Parar Simulação de Movimento';
                isSimulating = true;
                console.log('Simulação de movimento iniciada...');

                simulationInterval = setInterval(() => {
                    // Move ligeiramente a latitude e longitude
                    currentLat += (Math.random() - 0.5) * 0.0005; // Movimento menor para ser mais suave
                    currentLng += (Math.random() - 0.5) * 0.0005;

                    // Envia a nova localização para o servidor via Socket.io
                    socket.emit('send_location', { 
                        lat: currentLat, 
                        lng: currentLng 
                    });

                }, 1000); // Envia localização a cada 1 segundo para um movimento mais fluido
            }
        }
    </script>

    <script async defer src="https://maps.googleapis.com/maps/api/js?key=${GOOGLE_MAPS_API_KEY}&callback=initMap"></script>
</body>
</html>
