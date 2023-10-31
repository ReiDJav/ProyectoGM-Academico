## Codigo fuente del contador Geiger-Müller
Por cuestiones de seguridad, se van omitir las credenciales de Wi-Fi, Thingspeak y IFTTT
```cpp
// Definición de macros
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

const int inputPin = 26; // Definición del número de pin

int counts = 0;  // Contador de eventos del tubo
int counts2 = 0;
int cpm = 0;                     // CPM (conteo por minuto)
unsigned long lastCountTime;     // Tiempo de última medición
unsigned long lastEntryThingspeak;
unsigned long startCountTime;    // Tiempo de inicio de medición
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

// Función para enviar datos a ThingSpeak, al contar con 8 campos en un canal, permite almacenar datos en diferentes canales
void postThingspeak( int value) {
#ifdef WIFI
  int x = ThingSpeak.writeField(myChannelNumber, 1, value, myWriteAPIKey); // Envio de datos al campo 1
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
  esp_task_wdt_init(WDT_TIMEOUT, true); // Inicializa el watchdog del ESP32 con un tiempo de espera y reinicia el reinicio de "pánico"
  esp_task_wdt_add(NULL); // Añade el hilo actual a la vigilancia del watchdog
  
#ifdef CONS
  Serial.begin(115200); // Inicializa la comunicación serie a 115,200 baudios si la macro CONS esta habilitada
  Serial.print("This is ") ; Serial.println(Version) ; // Imprime la version del programa en el monitor serie
#endif 

  if (PERIOD_LOG > PERIOD_THINKSPEAK) {
#ifdef CONS
    Serial.println("PERIOD_THINKSPEAK has to be bigger than PERIODE_LOG"); //Comprueba si el periodo de envio a Thingspeak es mayor que el periodo de regiswtro y muetra un mensaje de error de no ser así
#endif
    while (1); // Entra en un bucle de no cumplirse la condicción anterior 
  }
  displayInit(); // Inicializa el display
#ifdef WIFI
  ThingSpeak.begin(client);  // Inicializa ThingSpeak si se habilita Wi-Fi
#endif
  displayString("Welcome", 64, 0);
  displayString(Version, 64, 30);
  printStack();
#ifdef WIFI
#ifdef CONS
  Serial.println("Connecting to Wi-Fi");
#endif 

  WiFi.begin(mySSID, myPASSWORD); // Inicia la conexión a una red WiFi utilizando las credenciales definidas

  int wifi_loops = 0;
  int wifi_timeout = WIFI_TIMEOUT_DEF;
  while (WiFi.status() != WL_CONNECTED) {  // Espera a que el dispositivo se conecte a la red WiFi
    wifi_loops++;
#ifdef CONS
    Serial.print(".");
#endif
    delay(500);
    if (wifi_loops > wifi_timeout) software_Reset(); // Reinicia el programa si la conexión WiFi no se establece en un tiempo límite
  }
#ifdef CONS
  Serial.println();
  Serial.println("Wi-Fi connected");
#endif
#endif
  display.clear();
  displayString("Measurring", 64, 15); // Muestra un mensaje indicando que el programa está midiendo en la pantalla OLED.
  pinMode(inputPin, INPUT);    // Configuración del pin como entrada para capturar eventos del tubo
#ifdef CONS
  Serial.println("Defined Input Pin");
#endif
  attachInterrupt(digitalPinToInterrupt(inputPin), ISR_impulse, FALLING);    
  Serial.println("IRQ installed");

  startEntryThingspeak = lastEntryThingspeak = millis(); // Inicializa las variables relacionadas con el tiempo para el registro y envío de datos.

  startCountTime = lastCountTime = millis(); // Inicializa las variables relacionadas con el conteo de eventos.
#ifdef CONS
  Serial.println("Initialized");
#endif  
}

int active = 0 ;

// Función principal en bucle
void loop() {
  esp_task_wdt_reset(); // Reinicia el watchdog para evitar bloqueos del ESP32
#ifdef WIFI
  if (WiFi.status() != WL_CONNECTED) software_Reset(); // Comprueba si el ESP32 no está conectado a WiFi y, en ese caso, reinicia el programa
#endif

  if (millis() - lastCountTime > (PERIOD_LOG * 1000)) {  // Comprueba si ha pasado el tiempo para registrar eventos
#ifdef CONS
    Serial.print("Counts: "); Serial.println(counts);
#endif
    cpm = (60000 * counts) / (millis() - startCountTime) ;  // Calcula la tasa de cuentas por minuto (CPM) en función de los eventos registrados
    counts = 0 ; // Restablece el contador de eventos
    startCountTime = millis() ;
    lastCountTime += PERIOD_LOG * 1000;
    display.clear();
    displayString("Radioactivity", 64, 0); // Limpia el display mostrar información sobre la radioactividad
    displayInt(cpm, 64, 30);

    if (cpm >= 200) {  // Comprueba si la CPM supera el umbral de alarma (200)
      if (active) {
        active = 0 ; // Desactiva la alarma si ya estaba activa
        display.clear();
        displayString("ALARM!!", 64, 0);
        displayInt(cpm, 64, 30);
#ifdef IFTT  // Envia una notificación a través de IFTTT si está habilitado
        IFTTT(cpm);
#endif
      } ;
    }
    else if (cpm < 100) 
    {
      active = 1 ; // Activa la alarma si la CPM cae por debajo de 100
    } ;
#ifdef CONS
    Serial.print("cpm: "); Serial.println(cpm); // Imprime el valor de CPM en modo de depuración
#endif
    printStack();
  }
  // Comprueba si ha pasado el tiempo para enviar datos a ThingSpeak
  if (millis() - lastEntryThingspeak > (PERIOD_THINKSPEAK * 1000)) {
#ifdef CONS
    Serial.print("Counts2: "); Serial.println(counts2);  // Imprime el número de eventos registrados en modo de depuración
#endif
 // Calcula el promedio de cuentas por hora (CPH) y enviarlo a ThingSpeak
    int averageCPH = (int)(((float)3600000 * (float)counts2) / (float)(millis() - startEntryThingspeak));
#ifdef CONS
    Serial.print("Average cph: "); Serial.println(averageCPH);  // Imprime el promedio de CPH en modo de depuración
#endif
    postThingspeak(averageCPH); // Envia el valor promedio a ThingSpeak
    lastEntryThingspeak += PERIOD_THINKSPEAK * 1000;
    startEntryThingspeak = millis();
    counts2 = 0; // Restablece el contador de eventos
  } ;
  delay(50); // Espera un breve período para evitar ciclos de loop muy rápidos
}

```
