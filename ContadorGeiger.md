// Definiciones de versión y configuraciones condicionales
#define Version "V1"
#define CONS
#define WIFI
#define IFTT
#ifdef CONS
#define PRINT_DEBUG_MESSAGES
#endif

// Inclusión de bibliotecas y definición de constantes
#ifdef WIFI
#include <WiFi.h>
#include <WiFiClientSecure.h>
#ifdef IFTT
#include "IFTTTWebhook.h"
#endif
#include <ThingSpeak.h>
#endif
#include <SSD1306.h>
#include <esp_task_wdt.h>

// Definición de parámetros y credenciales
#define WIFI_TIMEOUT_DEF 30
#define PERIOD_LOG 15                // Periodo de registro
#define PERIOD_THINKSPEAK 3600       // En segundos, debería ser mayor a 60
#define WDT_TIMEOUT 10

#ifndef CREDENTIALS
// Configuración de la red WLAN
#ifdef WIFI
#define mySSID "House Stark"
#define myPASSWORD "WinterIsComing"

// Configuración de IFTTT
#ifdef IFTT
#define IFTTT_KEY "ppvsCJbpqGrPCsgT_sBIW1198_U4qg1_nfSLLioG8zP"
#endif

// Configuración de ThingSpeak
#define SECRET_CH_ID 2170273      // Número del canal
#define SECRET_WRITE_APIKEY "G4KSAVPD96EV7BZP"   // Clave API de escritura

#endif

// Configuración de IFTTT
#ifdef IFTT
#define EVENT_NAME "Radioactividad" // Nombre del applet en IFTTT
#endif
#endif

IPAddress ip;

#ifdef WIFI
WiFiClient client;
WiFiClientSecure secure_client;
#endif

// Configuración del display OLED
SSD1306  display(0x3c, 5, 4);

const int inputPin = 26;

int counts = 0;  // Contador de eventos del tubo
int counts2 = 0;
int cpm = 0;                                             // CPM (conteo por minuto)
unsigned long lastCountTime;                            // Tiempo de la última medición
unsigned long lastEntryThingspeak;
unsigned long startCountTime;                            // Tiempo de inicio de la medición
unsigned long startEntryThingspeak;

#ifdef WIFI
unsigned long myChannelNumber = SECRET_CH_ID;
const char * myWriteAPIKey = SECRET_WRITE_APIKEY;
#endif

// Función de interrupción para capturar el recuento de eventos del tubo Geiger
void IRAM_ATTR ISR_impulse() {
  counts++;
  counts2++;
}

// Inicialización del display
void displayInit() {
  display.init();
  display.flipScreenVertically();
  display.setFont(ArialMT_Plain_24);
}

// Función para mostrar un entero en el display
void displayInt(int dispInt, int x, int y) {
  display.setColor(WHITE);
  display.setTextAlignment(TEXT_ALIGN_CENTER);
  display.drawString(x, y, String(dispInt));
  display.setFont(ArialMT_Plain_24);
  display.display();
}

// Función para mostrar una cadena en el display
void displayString(String dispString, int x, int y) {
  display.setColor(WHITE);
  display.setTextAlignment(TEXT_ALIGN_CENTER);
  display.drawString(x, y, dispString);
  display.setFont(ArialMT_Plain_24);
  display.display();
}

// Función de reinicio de software
void software_Reset() {
#ifdef CONS
  Serial.println("Reset por software");
#endif
  displayString("Myreset", 64, 15);
  delay(1000);
  esp_restart();
}

// Función para enviar datos a IFTTT
void IFTTT(int postValue) {
#ifdef WIFI
#ifdef IFTT
  IFTTTWebhook webhook(IFTTT_KEY, EVENT_NAME);
  if (!webhook.trigger(String(postValue).c_str())) {
#ifdef CONS
    Serial.println("Enviado exitosamente a IFTTT");
  } else {
    Serial.println("¡IFTTT falló!");
#endif
  }
#endif
#endif
}

// Función para enviar datos a ThingSpeak
void postThingspeak( int value) {
#ifdef WIFI
  int x = ThingSpeak.writeField(myChannelNumber, 1, value, myWriteAPIKey);
#ifdef CONS
  if (x == 200) {
    Serial.println("Actualización del canal exitosa");
  } else {
    Serial.println("Problema al actualizar el canal. Código de error HTTP: " + String(x));
  }
#endif
#endif
}

// Función para imprimir información sobre la pila de memoria
void printStack() {
#ifdef CONS
  char *SpStart = NULL;
  char *StackPtrAtStart = (char *)&SpStart;
  UBaseType_t watermarkStart = uxTaskGetStackHighWaterMark(NULL);
  char *StackPtrEnd = StackPtrAtStart - watermarkStart;
  Serial.printf("=== Información de la pila === ");
  Serial.printf("La pila libre es:  %d \r\n", (uint32_t)StackPtrAtStart - (uint32_t)StackPtrEnd);
#endif
}

// Configuración inicial
void setup() {
  esp_task_wdt_init(WDT_TIMEOUT, true); // Activar el pánico para reiniciar ESP32
  esp_task_wdt_add(NULL); // Añadir el hilo actual a la vigilancia WDT
  
#ifdef CONS
  Serial.begin(115200);
  Serial.print("Esta es la versión ") ; Serial.println(Version) ;
#endif 

  if (PERIOD_LOG > PERIOD_THINKSPEAK) {
#ifdef CONS
    Serial.println("PERIOD_THINKSPEAK debe ser mayor que PERIODE_LOG");
#endif
    while (1);
  }
  displayInit();
#ifdef WIFI
  ThingSpeak.begin(client);  // Inicializar ThingSpeak
#endif
  displayString("Bienvenido", 64, 0);
  displayString(Version, 64, 30);
  printStack();
#ifdef WIFI
#ifdef CONS
  Serial.println("Conectando a Wi-Fi");
#endif 

  WiFi.begin(mySSID, myPASSWORD);

  int wifi_loops = 0;
  int wifi_timeout = WIFI_TIMEOUT_DEF;
  while (WiFi.status() != WL_CONNECTED) {
    wifi_loops++;
#ifdef CONS
    Serial.print(".");
#endif
    delay(500);
    if (wifi_loops > wifi_timeout) software_Reset();
  }
#ifdef CONS
  Serial.println();
  Serial.println("Wi-Fi conectado");
#endif
#endif
  display.clear();
  displayString("Midien", 64, 15);
  pinMode(inputPin, INPUT);                            // Clavija para capturar eventos del tubo
#ifdef CONS
  Serial.println("Definido el pin de entrada");
#endif
  attachInterrupt(digitalPinToInterrupt(inputPin), ISR_impulse, FALLING);     // Definir interrupción en bajada
  Serial.println("IRQ instalada");

  startEntryThingspeak = lastEntryThingspeak = millis();
  startCountTime = lastCountTime = millis();
#ifdef CONS
  Serial.println("Inicial
