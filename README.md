# Calidad-del-Aire

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <MHZ19.h>
#include "DHT.h"
#include "ThingSpeak.h"
#include "WiFi.h"

// Dimensiones de la pantalla OLED
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

// Crear objeto de la pantalla OLED
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Datos de conexión WiFi
// Ingresar los datos de la red wifi a la que se va a conectar, poner los datos entre las comillas.  
const char* ssid = "";
const char* password = "";

// Configuración de ThingSpeak
unsigned long channelID = 2715215;
const char* WriteAPIKey = "4QX80G3VXMKWWPU5";
WiFiClient Client;

// Pines de los sensores
const int pinLED_GP2Y = 2;
const int pinVO_GP2Y = 32;
const int pinMQ135 = 35;
const int pinDHT = 4;
const int measurePin = 34;
const int ledPower = 2;

// Configuración del sensor DHT
#define DHTTYPE DHT11
DHT dht(pinDHT, DHTTYPE);

// Configuración del sensor MH-Z19B
MHZ19 mhz19;
#define RX2 16
#define TX2 17

// Variables para los sensores
float voMeasured = 0;
float calcVoltage = 0;
float dustDensity = 0;
float pm05 = 0;
float voltajeMQ135 = 0;
float temperatura = NAN;
float humedad = NAN;
int co2 = 0;

void setup() {
  Serial.begin(115200);

  // Conectar a WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Conectado");

  // Iniciar ThingSpeak
  ThingSpeak.begin(Client);

  // Configurar pines
  pinMode(pinLED_GP2Y, OUTPUT);
  digitalWrite(pinLED_GP2Y, HIGH);
  pinMode(ledPower, OUTPUT);

  // Iniciar sensores
  dht.begin();
  Serial2.begin(9600, SERIAL_8N1, RX2, TX2);
  mhz19.begin(Serial2);
  mhz19.autoCalibration(false);

  // Iniciar pantalla OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("Error al inicializar la pantalla OLED"));
    for (;;);
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
}

void loop() {
  // Leer sensor GP2Y1010
  digitalWrite(ledPower, LOW);
  delayMicroseconds(280);
  voMeasured = analogRead(measurePin);
  delayMicroseconds(40);
  digitalWrite(ledPower, HIGH);
  delayMicroseconds(9680);
  calcVoltage = (voMeasured / 4095.0) * 3.3;
  dustDensity = max(0.0, 0.17 * calcVoltage - 0.1);
  pm05 = (calcVoltage - 0.0356) * 120000;

  // Leer sensor MQ135
  int valorMQ135 = analogRead(pinMQ135);
  voltajeMQ135 = (valorMQ135 / 4095.0) * 3.3;

  // Leer sensor DHT11
  temperatura = dht.readTemperature();
  humedad = dht.readHumidity();

  // Leer sensor MH-Z19B
  co2 = mhz19.getCO2();

  // Mostrar en Serial Monitor y OLED
  display.clearDisplay();
  display.setCursor(0, 0);
  
  Serial.println("----- Lectura de Sensores -----");

  Serial.print("Particulas (Voltaje): ");
  Serial.print(calcVoltage, 2);
  Serial.println(" V");
  display.print("Particulas: ");
  display.print(calcVoltage, 2);
  display.println(" V");
  
  Serial.print("MQ135 (Voltaje): ");
  Serial.print(voltajeMQ135, 2);
  Serial.println(" V");
  display.print("MQ135: ");
  display.print(voltajeMQ135, 2);
  display.println(" V");

  Serial.print("Temperatura: ");
  Serial.print(temperatura);
  Serial.println(" °C");
  display.print("Temp: ");
  display.print(temperatura);
  display.println(" °C");

  Serial.print("Humedad: ");
  Serial.print(humedad);
  Serial.println(" %");
  display.print("Humedad: ");
  display.print(humedad);
  display.println(" %");

  Serial.print("CO2: ");
  Serial.print(co2);
  Serial.println(" ppm");
  display.print("CO2: ");
  display.print(co2);
  display.println(" ppm");

  display.display(); // Actualizar OLED

  Serial.println("-----------------------------");

  // Enviar datos a ThingSpeak
  Serial.println("Enviando datos a ThingSpeak...");
  ThingSpeak.setField(1, calcVoltage);      // Voltaje de partículas
  ThingSpeak.setField(2, voltajeMQ135);     // Voltaje del sensor MQ135
  ThingSpeak.setField(3, temperatura);      // Temperatura del sensor DHT11
  ThingSpeak.setField(4, humedad);          // Humedad del sensor DHT11
  ThingSpeak.setField(5, co2);              // CO2 del sensor MH-Z19B

  int x = ThingSpeak.writeFields(channelID, WriteAPIKey);
  if (x == 200) {
    Serial.println("Datos enviados a ThingSpeak con éxito.");
  } else {
    Serial.print("Error al enviar datos a ThingSpeak. Código de error: ");
    Serial.println(x);
  }

  delay(15000); // Espera de 15 segundos antes de la próxima lectura
}

