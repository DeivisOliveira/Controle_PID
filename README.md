#include <ESP8266WiFi.h>
#include <ESPAsyncWebServer.h>
#include <EEPROM.h>
#include <PID_v1.h>

// Configuração do Wi-Fi
const char* ssid = "Seu_SSID";
const char* password = "Sua_Senha";

// Pinos da Ponte H L298N
#define IN1 D1
#define IN2 D2
#define ENA D3  // PWM

// Pino do potenciômetro de posição real
#define POT_POSICAO A0   

// Variáveis PID
double setpoint, posicaoReal, output;
double Kp = 2.0, Ki = 5.0, Kd = 1.0;

// Controle de EEPROM
double ultimoSetpointSalvo = 0;

// Instância do PID
PID myPID(&posicaoReal, &output, &setpoint, Kp, Ki, Kd, DIRECT);

// Servidor Web
AsyncWebServer server(80);

void setup() {
  Serial.begin(115200);

  // Inicializar EEPROM (512 bytes disponíveis no ESP8266)
  EEPROM.begin(512);

  // Ler o setpoint salvo da EEPROM
  EEPROM.get(0, setpoint);
  ultimoSetpointSalvo = setpoint;
  Serial.print("Setpoint carregado da EEPROM: ");
  Serial.println(setpoint);

  // Conectar ao Wi-Fi
  WiFi.begin(ssid, password);
  Serial.print("Conectando ao Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConectado!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // Configuração da Ponte H
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);

  // Inicializar PID
  myPID.SetMode(AUTOMATIC);
  myPID.SetOutputLimits(-255, 255); // PWM com sinal positivo e negativo

  // Página Web - Ajuste de Setpoint
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request) {
    String pagina = "<html><head><title>Controle PID</title></head><body>";
    pagina += "<h1>Controle PID do Motor</h1>";
    pagina += "<form action='/set' method='GET'>";
    pagina += "Setpoint: <input type='number' name='sp' step='1' value='" + String(setpoint) + "'>";
    pagina += "<input type='submit' value='Atualizar'>";
    pagina += "</form>";
    pagina += "<p>Posição Atual: " + String(posicaoReal) + "</p>";
    pagina += "<script>setInterval(function(){ location.reload(); }, 5000);</script>";
    pagina += "</body></html>";
    request->send(200, "text/html", pagina);
  });

  // Rota para atualizar o setpoint
  server.on("/set", HTTP_GET, [](AsyncWebServerRequest *request) {
    if (request->hasParam("sp")) {
      double novoSetpoint = request->getParam("sp")->value().toDouble();

      // Grava apenas se mudou significativamente
      if (abs(novoSetpoint - ultimoSetpointSalvo) > 0.1) {
        setpoint = novoSetpoint;
        EEPROM.put(0, setpoint);
        EEPROM.commit();
        ultimoSetpointSalvo = setpoint;
        Serial.print("Novo Setpoint salvo na EEPROM: ");
        Serial.println(setpoint);
      } else {
        Serial.println("Setpoint não alterado. Nenhuma gravação feita.");
      }
    }
    request->redirect("/");
  });

  server.begin();
}

void loop() {
  // Leitura da posição real
  posicaoReal = analogRead(POT_POSICAO) * (180.0 / 1023.0);

  // Atualização do PID
  myPID.Compute();

  // Controle do motor via ponte H
  if (output > 0) {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, output);
  } else if (output < 0) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    analogWrite(ENA, -output);
  } else {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, 0);
  }

  // Debug Serial
  Serial.print("Setpoint: ");
  Serial.print(setpoint);
  Serial.print(" | Posição: ");
  Serial.print(posicaoReal);
  Serial.print(" | PID Output: ");
  Serial.println(output);

  delay(100);
}
# Controle_PID
