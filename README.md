<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Central IoT - Potenciômetro e LEDs</title>
  
  <script src="https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.0.1/mqttws31.min.js"></script>

  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    }

    body {
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      background: linear-gradient(145deg, #d4eaff 0%, #f0f9ff 100%);
      padding: 1.5rem;
    }

    .glass-card {
      max-width: 850px;
      width: 100%;
      background: rgba(255, 255, 255, 0.55);
      backdrop-filter: blur(14px);
      -webkit-backdrop-filter: blur(14px);
      border-radius: 2.5rem;
      border: 1px solid rgba(255, 255, 255, 0.7);
      box-shadow: 0 20px 40px -12px rgba(0, 20, 40, 0.25);
      padding: 2rem;
    }

    .header {
      display: flex;
      flex-wrap: wrap;
      align-items: baseline;
      justify-content: space-between;
      margin-bottom: 1.5rem;
      gap: 0.5rem;
    }

    .title-main { font-size: 1.8rem; font-weight: 600; color: #0b2b44; }
    .title-sub { font-size: 0.9rem; color: #3d5a73; opacity: 0.8; }

    .status-pill {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      background: rgba(255, 255, 255, 0.6);
      padding: 0.45rem 1.1rem;
      border-radius: 60px;
      border: 1px solid rgba(255, 255, 255, 0.8);
      font-size: 0.85rem;
      font-weight: 500;
      color: #1e2f3f;
    }

    .status-dot {
      width: 12px;
      height: 12px;
      border-radius: 50%;
      background: #fbbf24;
      transition: all 0.3s ease;
    }
    .status-dot.online { background: #10b981; box-shadow: 0 0 10px #10b981; }
    .status-dot.offline { background: #ef4444; box-shadow: 0 0 10px #ef4444; }

    .topic-area { margin-bottom: 1.5rem; }
    .topic-label {
      display: block; font-size: 0.75rem; font-weight: 600;
      text-transform: uppercase; color: #2c4b66; margin-bottom: 0.4rem;
    }
    .topic-input {
      width: 100%; padding: 0.8rem 1.2rem;
      background: rgba(255, 255, 255, 0.6);
      border: 1px solid rgba(255, 255, 255, 0.8);
      border-radius: 60px; font-size: 0.95rem; color: #0b2b44; outline: none;
    }

    /* Painel do Potenciômetro */
    .pot-card {
      background: rgba(255, 255, 255, 0.6);
      border-radius: 1.5rem;
      padding: 1.5rem;
      margin-bottom: 1.5rem;
      text-align: center;
      border: 1px solid rgba(255, 255, 255, 0.8);
    }

    .pot-display {
      font-size: 3rem;
      font-weight: 700;
      color: #0b2b44;
      margin: 0.5rem 0;
    }

    .progress-bar-bg {
      width: 100%;
      height: 16px;
      background: #cbdde9;
      border-radius: 10px;
      overflow: hidden;
    }

    .progress-bar-fill {
      height: 100%;
      width: 0%;
      background: linear-gradient(90deg, #ef4444 0%, #10b981 50%, #3b82f6 100%);
      transition: width 0.1s ease-out;
    }

    /* Grid com LEDs */
    .leds-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
      gap: 1.2rem;
    }

    .led-card {
      background: rgba(255, 255, 255, 0.4);
      border: 1px solid rgba(255, 255, 255, 0.7);
      border-radius: 1.5rem;
      padding: 1.2rem;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 1rem;
    }

    .led-circle {
      width: 80px; height: 80px;
      border-radius: 50%; background: #cbdde9;
      display: flex; align-items: center; justify-content: center;
      transition: all 0.3s ease; color: #3a5a72;
      font-size: 0.8rem; font-weight: 600;
    }

    .led-circle.active-led1 { background: #ef4444; color: white; box-shadow: 0 0 20px rgba(239, 68, 68, 0.8); }
    .led-circle.active-led2 { background: #10b981; color: white; box-shadow: 0 0 20px rgba(16, 185, 129, 0.8); }
    .led-circle.active-led3 { background: #3b82f6; color: white; box-shadow: 0 0 20px rgba(59, 130, 246, 0.8); }

    .log-box {
      margin-top: 1.5rem; width: 100%;
      background-color: #1e1e1e; color: #00ff66;
      font-family: monospace; padding: 12px;
      border-radius: 12px; height: 100px;
      overflow-y: auto; font-size: 0.8rem;
    }
  </style>
</head>
<body>

<div class="glass-card">
  <div class="header">
    <div>
      <div class="title-main">🎛️ Central IoT - Leitura Potenciômetro</div>
      <div class="title-sub">Entrada Analógica A1 · Acendimento Gradativo (13, 12 e 11)</div>
    </div>
    <div class="status-pill">
      <span class="status-dot offline" id="statusDot"></span>
      <span class="status-text" id="statusText">Desconectado</span>
    </div>
  </div>

  <div class="topic-area">
    <label class="topic-label" for="topicInput">📡 Tópico MQTT</label>
    <input class="topic-input" id="topicInput" type="text" value="bruno_correa_led_exclusivo/arduino" spellcheck="false">
  </div>

  <!-- Display do Potenciômetro -->
  <div class="pot-card">
    <span class="topic-label">VALOR ATUAL DO POTENCIÔMETRO (A1)</span>
    <div class="pot-display" id="potValue">0</div>
    <div class="progress-bar-bg">
      <div class="progress-bar-fill" id="progressFill"></div>
    </div>
  </div>

  <!-- Status das Lâmpadas -->
  <div class="leds-grid">
    <div class="led-card">
      <span style="font-weight: 600; color: #0b2b44;">LED 1 (Pino 13)</span>
      <div class="led-circle" id="ledCircle1">Off</div>
    </div>

    <div class="led-card">
      <span style="font-weight: 600; color: #0b2b44;">LED 2 (Pino 12)</span>
      <div class="led-circle" id="ledCircle2">Off</div>
    </div>

    <div class="led-card">
      <span style="font-weight: 600; color: #0b2b44;">LED 3 (Pino 11)</span>
      <div class="led-circle" id="ledCircle3">Off</div>
    </div>
  </div>

  <div class="log-box" id="logBox">> Aguardando dados do hardware...<br></div>
</div>

<script>
  (function() {
    const statusDot = document.getElementById('statusDot');
    const statusText = document.getElementById('statusText');
    const topicInput = document.getElementById('topicInput');
    const potValue = document.getElementById('potValue');
    const progressFill = document.getElementById('progressFill');
    const logBox = document.getElementById('logBox');

    const led1 = document.getElementById('ledCircle1');
    const led2 = document.getElementById('ledCircle2');
    const led3 = document.getElementById('ledCircle3');

    let currentTopic = topicInput.value.trim();
    let client = null;

    function addLog(msg) {
      const hora = new Date().toLocaleTimeString();
      logBox.innerHTML += `[${hora}] ${msg}<br>`;
      logBox.scrollTop = logBox.scrollHeight;
    }

    function setStatus(mode, msg) {
      statusDot.className = 'status-dot ' + mode;
      statusText.textContent = msg;
    }

    function processarLeitura(valor) {
      potValue.innerText = valor;
      
      // Calcula porcentagem para a barra (0 a 1023 -> 0% a 100%)
      const porcentagem = Math.min(100, Math.max(0, (valor / 1023) * 100));
      progressFill.style.width = porcentagem + '%';

      // Atualiza o estado dos LEDs na interface de acordo com as faixas do Arduino
      if (valor > 50) {
        led1.className = 'led-circle active-led1'; led1.textContent = 'On';
      } else {
        led1.className = 'led-circle'; led1.textContent = 'Off';
      }

      if (valor > 350) {
        led2.className = 'led-circle active-led2'; led2.textContent = 'On';
      } else {
        led2.className = 'led-circle'; led2.textContent = 'Off';
      }

      if (valor > 700) {
        led3.className = 'led-circle active-led3'; led3.textContent = 'On';
      } else {
        led3.className = 'led-circle'; led3.textContent = 'Off';
      }
    }

    function connectMqtt() {
      if (client) { try { client.disconnect(); } catch (_) {} }

      const clientId = 'web_pot_' + Math.random().toString(16).substring(2, 10);
      client = new Paho.MQTT.Client('broker.hivemq.com', 8884, clientId);

      client.connect({
        useSSL: true,
        timeout: 10,
        onSuccess: () => {
          setStatus('online', 'Online');
          addLog("Conectado ao Broker MQTT!");
          client.subscribe(currentTopic);
        },
        onFailure: (err) => {
          setStatus('offline', 'Erro');
          addLog("Falha ao conectar: " + err.errorMessage);
          setTimeout(connectMqtt, 5000);
        }
      });

      client.onConnectionLost = () => {
        setStatus('offline', 'Desconectado');
        addLog("Conexão perdida. Reconectando...");
        setTimeout(connectMqtt, 4000);
      };

      client.onMessageArrived = (msg) => {
        const payload = msg.payloadString.trim();
        if (!isNaN(payload)) {
          processarLeitura(parseInt(payload));
        }
      };
    }

    topicInput.addEventListener('change', function() {
      const newTopic = this.value.trim();
      if (newTopic && newTopic !== currentTopic) {
        client.unsubscribe(currentTopic);
        currentTopic = newTopic;
        client.subscribe(currentTopic);
        addLog(`Inscrito no novo tópico: '${currentTopic}'`);
      }
    });

    connectMqtt();
  })();
</script>
</body>
</html>
