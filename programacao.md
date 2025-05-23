#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <AccelStepper.h>


// ===================== Configuração Wi-Fi ===================== 
const char* ssid = "";       // Nome da rede Wi-Fi
const char* password = "";   // Senha da rede Wi-Fi

WebServer server(80);

// ===================== Configurações Gerais =====================
#define PIN_TEMPERATURA 4
#define PIN_PH A0
#define PIN_VENTILADOR 35
#define PIN_RESISTENCIA 17
#define PIN_DOSADORA_1 18
#define PIN_DOSADORA_2 19
#define PIN_BOMBA_AGUA 21
#define PIN_ILUMINACAO 22
#define PIN_MOTOR_1 25
#define PIN_MOTOR_2 26
#define PIN_MOTOR_3 27
#define PIN_MOTOR_4 14
#define BOTAO_ILUMINACAO 32
#define PIN_BOIA 13


OneWire oneWire(PIN_TEMPERATURA);
DallasTemperature sensors(&oneWire);
AccelStepper motor(AccelStepper::FULL4WIRE, PIN_MOTOR_1, PIN_MOTOR_3, PIN_MOTOR_2, PIN_MOTOR_4);

// ===================== Variáveis de Controle =====================
float temperaturaMin = 20.0;
float temperaturaMax = 30.0;
float phCentral = 7.0;
float faixaPH = 0.2;
bool iluminacaoLigada = false;
String logMensagens = "";

// ===================== Página HTML =====================
const char* htmlPage = R"rawliteral()
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Aquário Automatizado</title>
  <style>
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background: linear-gradient(120deg, #a1c4fd, #c2e9fb);
      padding: 40px;
      color: #333;
    }
    h1, h2 {
      text-align: center;
      color: #2a2a72;
    }
    .container {
      max-width: 600px;
      margin: auto;
      background: white;
      border-radius: 10px;
      box-shadow: 0 5px 15px rgba(0,0,0,0.1);
      padding: 20px;
    }
    .status, form, .controls, .log {
      margin-bottom: 20px;
    }
    label {
      display: block;
      margin: 10px 0 5px;
      font-weight: bold;
    }
    input[type="number"] {
      width: 100%;
      padding: 8px;
      border: 1px solid #ccc;
      border-radius: 5px;
    }
    button {
      display: inline-block;
      margin-top: 10px;
      padding: 10px 15px;
      background-color: #2a2a72;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      transition: background-color 0.3s;
    }
    button:hover {
      background-color: #1a1a60;
    }
    .status span {
      font-weight: bold;
    }
    .log-container {
      background: #f9f9f9;
      border: 1px solid #ccc;
      border-radius: 5px;
      padding: 10px;
      max-height: 200px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 0.9em;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🐠 Controle do Aquário 🐠</h1>

    <div class="status">
      <h2>Status</h2>
      <p><strong>Iluminação:</strong> <span id="iluminacao"></span></p>
      <p><strong>Sensor de Temperatura:</strong> <span id="sensorTemp"></span></p>
      <p><strong>Sensor de pH:</strong> <span id="sensorPH"></span></p>
    </div>

    <form id="configForm">
      <h2>Parâmetros</h2>

      <label for="tempMin">Temperatura Mínima (°C)</label>
      <input type="number" step="0.1" id="tempMin" name="tempMin">

      <label for="tempMax">Temperatura Máxima (°C)</label>
      <input type="number" step="0.1" id="tempMax" name="tempMax">

     <label for="phCentral">pH Desejado ao Áquario (valor ideal)</label>
     <input type="number" step="0.1" id="phCentral" name="phCentral">

      <label for="faixaPH">Tolerância do pH (±)</label>
      <input type="number" step="0.1" id="faixaPH" name="faixaPH">

      <button type="submit">💾 Salvar Configurações</button>
    </form>

    <div class="controls">
      <h2>Controles</h2>
      <button onclick="toggleIluminacao()">💡 Alternar Iluminação</button>
      <button onclick="alimentar()">🍽️ Alimentar Peixes</button>
    </div>

    <div class="log">
      <h2>📋 Log do Sistema</h2>
      <div class="log-container" id="log"></div>
    </div>
  </div>

  <script>
    async function carregarStatus() {
      const res = await fetch('/status');
      const data = await res.json();

      document.getElementById('tempMin').value = data.tempMin;
      document.getElementById('tempMax').value = data.tempMax;
      document.getElementById('phCentral').value = data.phCentral;
      document.getElementById('faixaPH').value = data.faixaPH;

      document.getElementById('iluminacao').textContent = data.iluminacao === "true" ? 'Ligada ✅' : 'Desligada ❌';
      document.getElementById('sensorTemp').textContent = data.ds18b20 === true ? 'Conectado ✅' : 'Desconectado ❌';
      document.getElementById('sensorPH').textContent = data.phSensor === true ? 'Conectado ✅' : 'Desconectado ❌';
    }

    async function carregarLog() {
      const res = await fetch('/log');
      const log = await res.text();
      document.getElementById('log').innerHTML = log;
    }

    document.getElementById('configForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const form = new FormData(e.target);
      const params = new URLSearchParams(form);

      const res = await fetch('/salvar?' + params.toString());
      alert(await res.text());
      carregarStatus();
    });

    function toggleIluminacao() {
      fetch('/iluminacao').then(() => carregarStatus());
    }

    function alimentar() {
      fetch('/alimentar').then(() => alert('Alimentação concluída!'));
    }

    // Atualiza status e log ao carregar a página
    carregarStatus();
    carregarLog();

    // Atualiza o log a cada 5 segundos
    setInterval(carregarLog, 5000);
  </script>

</body>
</html>
)rawliteral";

// ===================== Funções de Alarme =====================
void emitirAlarme(const char* mensagem) {
    Serial.println(mensagem);
    logMensagens += String(mensagem) + "<br>";
    Serial.println(mensagem); 
}

// ===================== Monitoramento da Temperatura =====================
  void verificarTemperatura() {
      sensors.requestTemperatures();
      float temp = sensors.getTempCByIndex(0);

      if (temp == -127.0) { 
          Serial.println("⚠️ Erro: Sensor DS18B20 não detectado!");
        return;
      }

      Serial.print("🌡️ Temperatura: ");
      Serial.print(temp);
      Serial.println("°C");

        String tempLog = "🌡️ Temperatura: " + String(temp) + "°C<br>";
        logMensagens += tempLog;  // Adiciona ao log para exibir no site

      if (temp >= temperaturaMax) {
          digitalWrite(PIN_VENTILADOR, HIGH);
          digitalWrite(PIN_RESISTENCIA, LOW); 
          emitirAlarme("🔥 Alerta: Temperatura ALTA!");
      } else if (temp <= temperaturaMin) {
          digitalWrite(PIN_VENTILADOR, LOW);
          digitalWrite(PIN_RESISTENCIA, HIGH);
          emitirAlarme("❄️ Alerta: Temperatura BAIXA!");
      } else {
          digitalWrite(PIN_VENTILADOR, LOW);
          digitalWrite(PIN_RESISTENCIA, LOW);
      }
  }

// ===================== Monitoramento do pH =====================
void verificarPH() {
    int leituraAnalogica = analogRead(PIN_PH);
    float ph = map(leituraAnalogica, 0, 4095, 0, 14);

    Serial.print("🔬 pH: ");
    Serial.println(ph);

    if (ph >= (phCentral + faixaPH)) {
        digitalWrite(PIN_DOSADORA_1, HIGH);
        digitalWrite(PIN_DOSADORA_2, LOW);
        emitirAlarme("⚠️ Alerta: pH ALTO!");
    } else if (ph <= (phCentral - faixaPH)) {
        digitalWrite(PIN_DOSADORA_1, LOW);
        digitalWrite(PIN_DOSADORA_2, HIGH);
        emitirAlarme("⚠️ Alerta: pH BAIXO!");
    } else {
        digitalWrite(PIN_DOSADORA_1, LOW);
        digitalWrite(PIN_DOSADORA_2, LOW);
    }
}

// ===================== Verificação do Nível de Água =====================
void verificarNivelAgua() {
    

    bool nivelAlto = digitalRead(PIN_BOIA) == LOW; // LOW = boia ativada = nível alto

    if (!nivelAlto) {
        digitalWrite(PIN_BOMBA_AGUA, HIGH); // Liga a bomba
        emitirAlarme("⚠️ Alerta: Nível de água BAIXO! Bomba ligada.");
        Serial.println("🔄 Enchendo reservatório...");
    } else {
        digitalWrite(PIN_BOMBA_AGUA, LOW); // Desliga a bomba
        emitirAlarme("✅ Nível de água OK. Bomba desligada.");
    }
}


// ===================== Controle do Motor de Passo =====================
void alimentar() {
    motor.setSpeed(500);
    motor.moveTo(2048);

    while (motor.distanceToGo() != 0) {
        motor.run();
    }

    delay(1000);

    motor.moveTo(0);
    while (motor.distanceToGo() != 0) {
        motor.run();
    }
}

// ===================== Controle da Iluminação =====================
void toggleIluminacao() {
    iluminacaoLigada = !iluminacaoLigada;
    digitalWrite(PIN_ILUMINACAO, iluminacaoLigada);
}

// ===================== Funções do Servidor Web =====================

void handleRoot() {
    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "text/html", htmlPage);
}

void handleSalvar() {
    if (server.hasArg("tempMin")) temperaturaMin = server.arg("tempMin").toFloat();
    if (server.hasArg("tempMax")) temperaturaMax = server.arg("tempMax").toFloat();
    if (server.hasArg("phCentral")) phCentral = server.arg("phCentral").toFloat();
    if (server.hasArg("faixaPH")) faixaPH = server.arg("faixaPH").toFloat();

    Serial.println("✅ Configurações atualizadas!");

    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "text/plain", "Configurações salvas!");
}

void handleGetLog() {
    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "text/html", logMensagens);  // Retorna o log de mensagens
}

void handleIluminacao() {
    toggleIluminacao();
    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "text/plain", "Iluminação alternada!");
}

void handleAlimentacao() {
    alimentar();
    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "text/plain", "Alimentação ativada!");
}

void handleGetStatus() {
    bool ds18b20Conectado = sensors.getTempCByIndex(0) != -127.0;
    int leituraPH = analogRead(PIN_PH);
    bool phSensorConectado = leituraPH > 100 && leituraPH < 4000; // valor razoável

    String json = "{";
    json += "\"tempMin\":" + String(temperaturaMin) + ",";
    json += "\"tempMax\":" + String(temperaturaMax) + ",";
    json += "\"phCentral\":" + String(phCentral) + ",";
    json += "\"faixaPH\":" + String(faixaPH) + ",";
    json += "\"iluminacao\":\"" + String(iluminacaoLigada ? "true" : "false") + "\",";
    json += "\"ds18b20\":" + String(ds18b20Conectado ? "true" : "false") + ",";
    json += "\"phSensor\":" + String(phSensorConectado ? "true" : "false");
    json += "}";

    server.sendHeader("Access-Control-Allow-Origin", "*");
    server.send(200, "application/json", json);
}


// ===================== Setup =====================

void setup() {
    
    Serial.begin(115200);
    Serial.println("\n🚀 Iniciando ESP32...");

    WiFi.begin(ssid, password);

    Serial.println("🔄 Conectando ao WiFi...");
    while (WiFi.status() != WL_CONNECTED) {
        delay(1000);
        Serial.print(".");
    }

    Serial.println("\n✅ WiFi Conectado!");
    Serial.print("📡 Endereço IP: ");
    Serial.println(WiFi.localIP());

    pinMode(PIN_VENTILADOR, OUTPUT);
    pinMode(PIN_RESISTENCIA, OUTPUT);
    pinMode(PIN_DOSADORA_1, OUTPUT);
    pinMode(PIN_DOSADORA_2, OUTPUT);
    pinMode(PIN_BOMBA_AGUA, OUTPUT);
    pinMode(PIN_ILUMINACAO, OUTPUT);
    pinMode(PIN_BOIA, INPUT_PULLUP);     // Boia ligada entre pino e GND
   


    server.on("/", handleRoot);
    server.on("/salvar", handleSalvar);
    server.on("/iluminacao", handleIluminacao);
    server.on("/alimentar", handleAlimentacao);
    server.on("/status", handleGetStatus);
    server.on("/log", handleGetLog);

    server.begin();
    Serial.println("✅ Servidor web iniciado!");
    sensors.begin();
}

// ===================== Loop Principal =====================
void loop() {
    server.handleClient();
    verificarTemperatura();
    verificarPH();
    verificarNivelAgua();
    delay(2000);
   
}
