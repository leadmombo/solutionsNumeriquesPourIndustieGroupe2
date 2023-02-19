# solutionsNumeriquesPourIndustieGroupe2


Ce projet  a été réalisé par : 

- Takia BENMEHIDI
- Quentin ALVES 
- Sarah-Angelique TENAILLEAU
- MOMBO MOMBO Daël

## Description du projet

  le projet que nous avons réalisé dans le cadre de notre cours de 5 eme année solutions Numeriques Pour Industie 3.0 a pour but ne nous donner des armes afin de comprendre les enjeux liés aux solutions du numeriques pour l'appliquer au domaine de l'industrie.
  Pour ce faire nous avons realisé pas à pas en TP une installation domotique(un controleur l'humidité et temperature) en s'appuyant sur des solutions numeriques.

  À cette fin, la partie 1 détaille brièvement les composants principaux utilisé dans la realisation du tp ainsi que l'envoi de la température et l'humidité de façon périodique avec MQTT.

  Partie 2 Affichage graphique de la température avec Node RED et Spark plug , l'affichage la température et l'humiditer sur ignition.

  ## Partie 1: Présentation des composants et lecture de la température et l'humidité de façon périodique avec MQTT

  Afin de réaliser ce projet, nous avions besoin du materiel suivant


| Composants              				| Nombre|
| :----------------------------------------------------:|:---------------:|
| un microcontrôleur ESP8266 disposant d’une interface capable de communiquer en Wi-Fi			| 1         |
| capteur de température et d'humidité DHT22      			| 1     	  | 
| Fils male-male  | 3-5            |
| cable micro USB-USBA				| 1	  |
| breadboard					| 1	  |
| Ordinateur portable					| 1	  |


 l’ESP8266 est une puce électronique capable de se connecter à un réseau sans fil Wi-Fi. Cette puce comporte une série de broches qui peuvent être des entrées et/ou des sorties digitales ou analogiques. Elles permettent donc une communication avec leur environnement.

 DHT22 est un capteur numérique qui communique ses informations par une suite de signaux binaires (1 et 0).Il est dédié à la mesure de l’air et non de liquide.


Les logiciels utilisés dans ce  TP. Citons :

   * Arduino IDE : pour la programmation des puces ESP8266.
   * Ignition : pour la visualisation de la temperature.
   * Node Red : pour **

Realisation :

Le principe du programme repose sur la lecture de la température ambiante et la consigne de température à une fréquence d’environ une lecture toutes les minutes.Nous avons donc dans un premier temps gerer la conection wifi avec l'architecture ESP8266 qui offre la possibilté de se connecter à un réseau Wi-Fi.
Sur arduino IDE nous avons donc gerer cette connection.
![affichage Capture connection wifi](./temp.PNG)
```
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include "DHT.h"          // Librairie des capteurs DHT
#include <Ticker.h>
#include <AsyncMqttClient.h>

const char* ssid = "Invite-ESIEA"; 
const char* mot_de_passe = "hQV86deaazEZQPu9a"; 


#define MQTT_HOST IPAddress(10, 8, 128, 250)
#define MQTT_PORT 1883

// Temperature MQTT Topics
#define MQTT_PUB_TEMP "esp/dht/temperature"
#define MQTT_PUB_HUM "esp/dht/humidity"

const int broche_mesure=12; 
DHT dht(broche_mesure, DHT22);  

AsyncMqttClient mqttClient;
Ticker mqttReconnectTimer;

void setup() {  
//  Serial.begin(9600);  
//  connexion_reseau(); 
  dht.begin(); 
  Serial.begin(9600); 
} 
 
void connexion_reseau(){ 
  delay(10);   
  Serial.print("Tentative de connexion: ");  
  Serial.println(ssid);  
  WiFi.begin(ssid, mot_de_passe);   
  while (WiFi.status() != WL_CONNECTED) {  
    delay(500);  
    Serial.print(".");  
  }  
  Serial.println("Connexion au réseau Wi-Fi réalisée");  
  Serial.println("L'adresse IP est: ");  
  Serial.println(WiFi.localIP()); 
} 

void loop(){ 
 delay(2000);  
 float t = dht.readTemperature();  
 float rh = dht.readHumidity();  
 
 if (isnan(t) || isnan(rh)) { // Affichage de l'alerte 
   Serial.println("Erreur de communication!");  
   return;  
 }else{                       // Affichage des mesures  
   Serial.print("Température: ");  
   Serial.print(t);  
   Serial.print(" *C  - ");  
   Serial.print("Humidité: ");  
   Serial.println(rh);  
 } 
}
```

l'envoi sur MQTT.

```
#include "DHT.h"
#include <ESP8266WiFi.h>
#include <Ticker.h>
#include <AsyncMqttClient.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define WIFI_SSID "Invite-ESIEA"
#define WIFI_PASSWORD "hQV86deaazEZQPu9a"

// Raspberri Pi Mosquitto MQTT Broker
#define MQTT_HOST IPAddress(10, 8, 128, 250)
#define MQTT_PORT 1883

// Temperature MQTT Topics
#define MQTT_PUB_DATA "dael/esp/dht"


// Digital pin connected to the DHT sensor
#define DHTPIN 12  

// Uncomment whatever DHT sensor type you're using
//#define DHTTYPE DHT11   // DHT 11
#define DHTTYPE DHT22   // DHT 22  (AM2302), AM2321
//#define DHTTYPE DHT21   // DHT 21 (AM2301)   

// Initialize DHT sensor
DHT dht(DHTPIN, DHTTYPE);

// Variables to hold sensor readings
float temp;
float hum;
char* json_fic;

AsyncMqttClient mqttClient;
Ticker mqttReconnectTimer;

WiFiEventHandler wifiConnectHandler;
WiFiEventHandler wifiDisconnectHandler;
Ticker wifiReconnectTimer;

unsigned long previousMillis = 0;   // Stores last time temperature was published
const long interval = 10000;        // Interval at which to publish sensor readings

void connectToWifi() {
  Serial.println("Connecting to Wi-Fi...");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void onWifiConnect(const WiFiEventStationModeGotIP& event) {
  Serial.println("Connected to Wi-Fi.");
  connectToMqtt();
}

void onWifiDisconnect(const WiFiEventStationModeDisconnected& event) {
  Serial.println("Disconnected from Wi-Fi.");
  mqttReconnectTimer.detach(); // ensure we don't reconnect to MQTT while reconnecting to Wi-Fi
  wifiReconnectTimer.once(2, connectToWifi);
}

void connectToMqtt() {
  Serial.println("Connecting to MQTT...");
  mqttClient.connect();
}

void onMqttConnect(bool sessionPresent) {
  Serial.println("Connected to MQTT.");
  Serial.print("Session present: ");
  Serial.println(sessionPresent);
}

void onMqttDisconnect(AsyncMqttClientDisconnectReason reason) {
  Serial.println("Disconnected from MQTT.");

  if (WiFi.isConnected()) {
    mqttReconnectTimer.once(2, connectToMqtt);
  }
}

char* to_json(const float data1, const float data2) {
    char* json_str;
    int json_size = snprintf(NULL, 0, "{\"temperature\":%f,\"humidity\":%f}", data1, data2);
    json_str = (char*)malloc(json_size + 1);    
    snprintf(json_str, json_size + 1, "{\"temperature\":%f,\"humidity\":%f}", data1, data2);
    return json_str;
}

/*void onMqttSubscribe(uint16_t packetId, uint8_t qos) {
  Serial.println("Subscribe acknowledged.");
  Serial.print("  packetId: ");
  Serial.println(packetId);
  Serial.print("  qos: ");
  Serial.println(qos);
}

void onMqttUnsubscribe(uint16_t packetId) {
  Serial.println("Unsubscribe acknowledged.");
  Serial.print("  packetId: ");
  Serial.println(packetId);
}*/

void onMqttPublish(uint16_t packetId) {
  Serial.print("Publish acknowledged.");
  Serial.print("  packetId: ");
  Serial.println(packetId);
}
void setup() {
  Serial.begin(115200);
  Serial.println();

  dht.begin();
  
  wifiConnectHandler = WiFi.onStationModeGotIP(onWifiConnect);
  wifiDisconnectHandler = WiFi.onStationModeDisconnected(onWifiDisconnect);

  mqttClient.onConnect(onMqttConnect);
  mqttClient.onDisconnect(onMqttDisconnect);
  //mqttClient.onSubscribe(onMqttSubscribe);
  //mqttClient.onUnsubscribe(onMqttUnsubscribe);
  mqttClient.onPublish(onMqttPublish);
  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  // If your broker requires authentication (username and password), set them below
  //mqttClient.setCredentials("REPlACE_WITH_YOUR_USER", "REPLACE_WITH_YOUR_PASSWORD");
  
  connectToWifi();
}

void loop() {
  unsigned long currentMillis = millis();
  // Every X number of seconds (interval = 10 seconds) 
  // it publishes a new MQTT message
  if (currentMillis - previousMillis >= interval) {
    // Save the last time a new reading was published
    previousMillis = currentMillis;
    // New DHT sensor readings
    hum = dht.readHumidity();
    Serial.print("Humidité: ");  
    Serial.print(hum); 
    // Read temperature as Celsius (the default)
    temp = dht.readTemperature();
    Serial.print("Température: ");  
    Serial.print(temp); 
    // Read temperature as Fahrenheit (isFahrenheit = true)
    //temp = dht.readTemperature(true);
    char* json_str = to_json(temp, hum);
  

    
    // Publish an MQTT message on topic ESIEA-P/grp2
    uint16_t packetIdPub1 = mqttClient.publish(MQTT_PUB_DATA, 1, true, json_str);                            


  }
}
```

## Partie 2: Partie 2 Affichage graphique de la température avec Node RED et Spark plug , l'affichage la température et l'humiditer sur ignition.

Voici notre worflow.
![Workflow](./nodeRedCap.PNG)

Le resultat .
![display](./jauge.png)

## Annexes et autres captures :
![connectionBroker](./connectToBroker.PNG)

![IngnitionConfig](./ignitionLauncher.PNG)

![Ingnition2Config](./configServer.PNG)
