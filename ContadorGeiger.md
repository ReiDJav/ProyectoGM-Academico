## Codigo fuente del contador Geiger-Müller
por cuestiones de seguridad, se van omitir las credenciales de Wi-Fi, Thingspeak y IFTTT
```cpp
// Definimos los condicionales
#define Version "V1"
#define CONS
#define WIFI
#define IFTT
#ifdef CONS
#define PRINT_DEBUG_MESSAGES
#endif

// Incluimos las bibliotecas a utilizar y definimos las constantes
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
#define PERIOD_THINKSPEAK 3600       // En segundos, > 60
#define WDT_TIMEOUT 10

#ifndef CREDENTIALS
#ifdef WIFI // Credencial de la red WLAN
#define mySSID "XYZ"
#define myPASSWORD "XYZ"

// Credencial para IFTTT
#ifdef IFTT
#define IFTTT_KEY "XYZ"
#endif

// Credencial para ThingSpeak
#define SECRET_CH_ID "XYZ"      // Número del canal
#define SECRET_WRITE_APIKEY "XYZ"   // Clave API de escritura

#endif

// Configuración del canal de IFTTT
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

const int inputPin = 26; // Definimos el número del pin

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
  Serial.println("Resetting by software");
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
    Serial.println("Successfully sento to IFTTT");
  } else {
    Serial.println("IFTTT failed");
#endif
  }
#endif
#endif
}

// Función para enviar datos a ThingSpeak, cuenta con 8 campos en un canal, permitiendo almacenar datos en diferentes canales
void postThingspeak( int value) {
#ifdef WIFI
  int x = ThingSpeak.writeField(myChannelNumber, 1, value, myWriteAPIKey); // Mandamos los datos al campo 1
#ifdef CONS
  if (x == 200) {
    Serial.println("Channel update successful");
  } else {
    Serial.println("Problem updating channel. HTTP error code" + String(x));
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
  Serial.printf("=== Stack info === ");
  Serial.printf("Free stack is:  %d \r\n", (uint32_t)StackPtrAtStart - (uint32_t)StackPtrEnd);
#endif
}

// Configuración inicial
void setup() {
  esp_task_wdt_init(WDT_TIMEOUT, true); // Activar el "pánico" para reiniciar ESP32
  esp_task_wdt_add(NULL); // Añadir el hilo actual a la vigilancia WDT
  
#ifdef CONS
  Serial.begin(115200);
  Serial.print("This is ") ; Serial.println(Version) ;
#endif 

  if (PERIOD_LOG > PERIOD_THINKSPEAK) {
#ifdef CONS
    Serial.println("PERIOD_THINKSPEAK has to be bigger than PERIODE_LOG");
#endif
    while (1);
  }
  displayInit();
#ifdef WIFI
  ThingSpeak.begin(client);  // Inicializar ThingSpeak
#endif
  displayString("Welcome", 64, 0);
  displayString(Version, 64, 30);
  printStack();
#ifdef WIFI
#ifdef CONS
  Serial.println("Connecting to Wi-Fi");
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
  Serial.println("Wi-Fi connected");
#endif
#endif
  display.clear();
  displayString("Measurring", 64, 15);
  pinMode(inputPin, INPUT);                            // Definimos el pin para capturar los eventos del tubo
#ifdef CONS
  Serial.println("Defined Input Pin");
#endif
  attachInterrupt(digitalPinToInterrupt(inputPin), ISR_impulse, FALLING);    
  Serial.println("IRQ installed");

  startEntryThingspeak = lastEntryThingspeak = millis();
  startCountTime = lastCountTime = millis();
#ifdef CONS
  Serial.println("Initialized");
#endif  
}

int active = 0 ;

// Función principal en bucle
void loop() {
  esp_task_wdt_reset();
#ifdef WIFI
  if (WiFi.status() != WL_CONNECTED) software_Reset();
#endif

  if (millis() - lastCountTime > (PERIOD_LOG * 1000)) {
#ifdef CONS
    Serial.print("Counts: "); Serial.println(counts);
#endif
    cpm = (60000 * counts) / (millis() - startCountTime) ;
    counts = 0 ;
    startCountTime = millis() ;
    lastCountTime += PERIOD_LOG * 1000;
    display.clear();
    displayString("Radioactivity", 64, 0);
    displayInt(cpm, 64, 30);
    if (cpm >= 200) {
      if (active) {
        active = 0 ;
        display.clear();
        displayString("ALARM!!", 64, 0);
        displayInt(cpm, 64, 30);
#ifdef IFTT
        IFTTT(cpm);
#endif
      } ;
    }
    else if (cpm < 100)
    {
      active = 1 ;
    } ;
#ifdef CONS
    Serial.print("cpm: "); Serial.println(cpm);
#endif
    printStack();
  }

  if (millis() - lastEntryThingspeak > (PERIOD_THINKSPEAK * 1000)) {
#ifdef CONS
    Serial.print("Counts2: "); Serial.println(counts2);
#endif
    int averageCPH = (int)(((float)3600000 * (float)counts2) / (float)(millis() - startEntryThingspeak));
#ifdef CONS
    Serial.print("Promedio de cph: "); Serial.println(averageCPH);
#endif
    postThingspeak(averageCPH);
    lastEntryThingspeak += PERIOD_THINKSPEAK * 1000;
    startEntryThingspeak = millis();
    counts2 = 0;
  } ;
  delay(50);
}

```
