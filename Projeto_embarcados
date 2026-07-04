#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <time.h>
#include "BluetoothSerial.h"

BluetoothSerial SerialBT;

// --- MAPEAMENTO DE PINOS ---
const int PINO_TEMP   = 35;  
const int PINO_LDR    = 34;  
const int PINO_BOTAO  = 12;  
const int PINO_RELE   = 13;  
const int PINO_BUZZER = 19;  

const int PINO_RGB_R  = 21;  
const int PINO_RGB_G  = 22;  
const int PINO_RGB_B  = 23;  

// --- VARIÁVEIS DE CONFIGURAÇÃO E ESTADO ---
int limiteTemperatura  = 30;
int limiteLuminosidade = 100;
bool sistemaLigado       = false; 
int estadoBotaoAnterior  = HIGH;  

const char* ssid        = "nome da rede";    
const char* password    = "senha do wifi";   
 
const char* mqtt_server = "seu ip"; //em casa es ip 

const int mqtt_port     = 1885;              
const char* mqtt_topic  = "dados";           

unsigned long tempoAnteriorEnvio = 0;
const long intervaloEnvio        = 2000;    

WiFiClient espClient;
PubSubClient mqttClient(espClient);

void controlarRGB(int r, int g, int b) {
  digitalWrite(PINO_RGB_R, r);
  digitalWrite(PINO_RGB_G, g);
  digitalWrite(PINO_RGB_B, b);
}

void processarComandoBluetooth(String comando) {
  comando.trim();
  if (comando.equalsIgnoreCase("ligar")) {
    if (!sistemaLigado) {
      sistemaLigado = true;
      digitalWrite(PINO_BUZZER, HIGH);
      delay(150);
      digitalWrite(PINO_BUZZER, LOW);
      SerialBT.println(">>> Sistema LIGADO");
      Serial.println(">>> Sistema LIGADO via BT");
    }
  } 
  else if (comando.equalsIgnoreCase("desligar")) {
    sistemaLigado = false;
    SerialBT.println(">>> Sistema DESLIGADO");
    Serial.println(">>> Sistema DESLIGADO via BT");
  } 
  else if (comando.startsWith("temp=")) {
    limiteTemperatura = comando.substring(5).toInt();
    SerialBT.printf(">>> Limite Temp: %d\n", limiteTemperatura);
  } 
  else if (comando.startsWith("luz=")) {
    limiteLuminosidade = comando.substring(4).toInt();
    SerialBT.printf(">>> Limite Luz: %d\n", limiteLuminosidade);
  }
}

void connectToWiFi() {
  if (WiFi.status() == WL_CONNECTED) return;
  
  Serial.println("Tentando conectar ao WiFi...");
  WiFi.begin(ssid, password);
  
  // Tenta conectar por no máximo 10 vezes para não travar o resto do sistema
  int tentativas = 0;
  while (WiFi.status() != WL_CONNECTED && tentativas < 10) {
    delay(500);
    Serial.print(".");
    tentativas++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Conectado!");
    Serial.print("IP: "); Serial.println(WiFi.localIP());
  } else {
    Serial.println("\nFalha temporaria no WiFi. Tentando novamente mais tarde...");
  }
}

void reconnectToMQTT() {
  if (WiFi.status() != WL_CONNECTED) return;
  
  if (!mqttClient.connected()) {
    Serial.println("Conectando ao MQTT...");
    if (mqttClient.connect("ESP32_Monit_Sensores")) { 
      Serial.println("Conectado ao Broker MQTT!");
    } else {
      Serial.print("Erro MQTT, rc="); Serial.println(mqttClient.state());
    }
  }
}

void setup() {
  // 1. INICIALIZAÇÃO IMEDIATA DO SERIAL E BLUETOOTH
  Serial.begin(115200); 
  delay(500); 
  
  Serial.println("\n--- INICIALIZANDO ESP32 ---");
  
  SerialBT.begin("ESP32_Automacao"); 
  Serial.println("Bluetooth Ativado! Nome: 'ESP32_Automacao'");

  // 2. CONFIGURAÇÃO DE PINOS
  pinMode(PINO_TEMP, INPUT);
  pinMode(PINO_LDR, INPUT);
  pinMode(PINO_BOTAO, INPUT_PULLUP);
  pinMode(PINO_RELE, OUTPUT);
  pinMode(PINO_BUZZER, OUTPUT);
  pinMode(PINO_RGB_R, OUTPUT);
  pinMode(PINO_RGB_G, OUTPUT);
  pinMode(PINO_RGB_B, OUTPUT);

  digitalWrite(PINO_RELE, LOW);
  digitalWrite(PINO_BUZZER, LOW);
  controlarRGB(HIGH, HIGH, LOW); // Começa Amarelo

  // 3. CONEXÕES SECUNDÁRIAS
  connectToWiFi();
  mqttClient.setServer(mqtt_server, mqtt_port);

  // Configura o NTP sem travar o código esperando resposta
  configTime(-3 * 3600, 0, "pool.ntp.org", "time.nist.gov"); 
  Serial.println("Setup finalizado. Pronto para comandos.");
}

void loop() {
  // Conexões não travantes
  if (WiFi.status() != WL_CONNECTED) connectToWiFi();
  if (WiFi.status() == WL_CONNECTED && !mqttClient.connected()) reconnectToMQTT();
  mqttClient.loop();

  // 1. LEITURA DE COMANDOS VIA BLUETOOTH
  if (SerialBT.available()) {
    String comandoRecebido = SerialBT.readStringUntil('\n');
    processarComandoBluetooth(comandoRecebido);
  }

  // 2. BOTÃO FÍSICO
  int leituraBotao = digitalRead(PINO_BOTAO);
  if (leituraBotao == LOW && estadoBotaoAnterior == HIGH) {
    sistemaLigado = !sistemaLigado; 
    digitalWrite(PINO_BUZZER, HIGH);
    delay(150);
    digitalWrite(PINO_BUZZER, LOW);
    Serial.printf("Sistema alterado via botao para: %s\n", sistemaLigado ? "LIGADO" : "DESLIGADO");
    delay(50); 
  }
  estadoBotaoAnterior = leituraBotao;

  // 3. AUTOMAÇÃO E LEITURAS
  if (sistemaLigado) {
    int leituraTemp = analogRead(PINO_TEMP); 
    int leituraLDR  = analogRead(PINO_LDR);  

    if (leituraTemp > limiteTemperatura && leituraLDR > limiteLuminosidade) {
      digitalWrite(PINO_RELE, HIGH); 
      controlarRGB(LOW, HIGH, LOW); // VERDE
    } else {
      digitalWrite(PINO_RELE, LOW);  
      controlarRGB(HIGH, LOW, LOW); // VERMELHO
    }

    // Envio MQTT
    unsigned long tempoAtual = millis();
    if (tempoAtual - tempoAnteriorEnvio >= intervaloEnvio) {
      tempoAnteriorEnvio = tempoAtual;

      time_t now = time(nullptr);
      struct tm* timeinfo = localtime(&now);
      char datetime[30];
      if (now > 100000) {
        strftime(datetime, sizeof(datetime), "%Y-%m-%d %H:%M:%S", timeinfo);
      } else {
        strcpy(datetime, "Aguardando NTP...");
      }

      StaticJsonDocument<256> doc; 
      doc["datetime"]     = datetime;
      doc["temperatura"]  = leituraTemp; 
      doc["luminosidade"] = leituraLDR;  
      doc["status_rele"]  = digitalRead(PINO_RELE); 

      char jsonBuffer[256]; 
      serializeJson(doc, jsonBuffer);
      
      if(mqttClient.connected()) {
        mqttClient.publish(mqtt_topic, jsonBuffer);
        Serial.print("Dados enviados: "); Serial.println(jsonBuffer);
      }
    }
  } else {
    digitalWrite(PINO_RELE, LOW);
    controlarRGB(HIGH, HIGH, LOW); // AMARELO
  }
}
