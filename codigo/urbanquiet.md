#include <Adafruit_NeoPixel.h>
#include <PubSubClient.h>
#include <WiFi.h>

// Bibliotecas nativas do ESP32 para controle de registradores
#include "soc/rtc_cntl_reg.h"
#include "soc/soc.h"

// --- Configurações de Hardware ---
#define PIN_LED 32
#define NUM_LEDS 16
#define PIN_SENSOR 34

// --- Credenciais da Rede ---
const char *ssid = "A&V Escritório 2GHZ";
const char *password = "1122112222v";

// --- Broker MQTT Oficial e Público (Nuvem) ---
const char *mqtt_server = "broker.hivemq.com";

Adafruit_NeoPixel strip =
    Adafruit_NeoPixel(NUM_LEDS, PIN_LED, NEO_GRB + NEO_KHZ800);

// Janela de 50ms para capturar a amplitude da onda sonora
const int janelaAmostragem = 50;

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long ultimoEnvio = 0;
unsigned long inicioAlertaAtuador = 0;

void callback(char *topic, byte *payload, unsigned int length) {
  String msg = "";
  for (int i = 0; i < length; i++) {
    msg += (char)payload[i];
  }

  if (String(topic) == "urbanquiet/ping") {
    // Responde o ping imediatamente para cálculo de RTT/Offset
    String pongPayload = msg + "," + String(millis());
    client.publish("urbanquiet/pong", pongPayload.c_str());
  } else if (String(topic) == "urbanquiet/comando") {
    // Registra o tempo exato em que a ação do atuador foi engatada
    unsigned long t_acao = millis();
    inicioAlertaAtuador = millis(); // Aciona efeito visual por 1 segundo
    
    // Responde com o timestamp original do comando e o millis do ESP32 no instante da ação
    String confirmPayload = msg + "," + String(t_acao);
    client.publish("urbanquiet/confirmacao", confirmPayload.c_str());
  }
}

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando na rede: ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nWi-Fi Conectado com Sucesso!");
  Serial.print("IP do ESP32: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Tentando conexão com o Broker Oficial (HiveMQ)...");

    String clientId = "ESP32_UrbanQuiet-";
    clientId += String(random(0, 10000));

    if (client.connect(clientId.c_str())) {
      Serial.println("Conectado com sucesso na Nuvem!");
      // Inscreve-se nos tópicos de calibração e atuador
      client.subscribe("urbanquiet/ping");
      client.subscribe("urbanquiet/comando");
    } else {
      Serial.print("Falhou, erro: ");
      Serial.print(client.state());
      Serial.println(" Retentando em 5 segundos...");
      delay(5000);
    }
  }
}

void setup() {
  // CORREÇÃO AQUI: Adicionado os underlines corretos para a versão atual do SDK
  // do ESP32
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);

  Serial.begin(115200);

  randomSeed(analogRead(35));

  strip.begin();
  strip.setBrightness(128); // Limita o brilho global a 50% para conforto visual e economia
  strip.show();

  pinMode(PIN_SENSOR, INPUT);

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);

  Serial.println(
      "\n--- UrbanQuiet: Modo MQTT Nuvem + Proteção de Energia Ativa ---");
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // --- Lógica de Amostragem do Som ---
  unsigned long inicioAmostra = millis();
  unsigned int somMax = 0;
  unsigned int somMin = 4095;

  while (millis() - inicioAmostra < janelaAmostragem) {
    unsigned int leitura = analogRead(PIN_SENSOR);
    if (leitura < 4095) {
      if (leitura > somMax)
        somMax = leitura;
      if (leitura < somMin)
        somMin = leitura;
    }
  }

  unsigned int amplitude = somMax - somMin;

  int nivelRuido = map(amplitude, 0, 2000, 0, 100);
  if (nivelRuido > 100)
    nivelRuido = 100;
  if (nivelRuido < 0)
    nivelRuido = 0;

  // --- Atualização Visual Física (Matriz de LED ou Comando) ---
  if (inicioAlertaAtuador > 0 && millis() - inicioAlertaAtuador < 1000) {
    // Força cor azul por 1 segundo ao receber comando
    definirCorMatriz(strip.Color(0, 0, 255));
  } else {
    inicioAlertaAtuador = 0;
    if (nivelRuido < 25) {
      definirCorMatriz(strip.Color(0, 255, 0));
    } else if (nivelRuido < 80) {
      definirCorMatriz(strip.Color(255, 110, 0));
    } else {
      definirCorMatriz(strip.Color(255, 0, 0));
    }
  }

  // --- Processamento e Envio do Payload Formatado em JSON (A cada 1 segundo) ---
  if (millis() - ultimoEnvio > 1000) {
    ultimoEnvio = millis();

    String statusTexto = "";

    if (nivelRuido < 25) {
      statusTexto = "Som Ambiente";
    } else if (nivelRuido < 80) {
      statusTexto = "Som Limite";
    } else {
      statusTexto = "Ruido Alto";
    }

    // Criamos o JSON incluindo a leitura, status e timestamp interno do ESP32 (millis)
    String payload = "{\"ruido\":" + String(nivelRuido) + 
                     ",\"status\":\"" + statusTexto + 
                     "\",\"t_esp\":" + String(millis()) + "}";

    Serial.print("Enviando para a Nuvem -> ");
    Serial.println(payload);

    client.publish("urbanquiet/ruido", payload.c_str());
  }
}

void definirCorMatriz(uint32_t cor) {
  for (int i = 0; i < NUM_LEDS; i++) {
    strip.setPixelColor(i, cor);
  }
  strip.show();
}