# NODE-RED-J-CON-LOS-DOS-SENSORES
## Descripción
### En este repositorio se muestra como  podemos programar en Node Red dos sensores diferentes y mostrar sus gráficas, para esto utilizamos el simulador de ESP32 de Wokwi y lo conectamos por medio de un servidor a nuestro Node Red, con esto logramos que al mover nuestra distancia, temperatura y humedad en el wokwi se actualice en tiempo real en el Node-RED.
## Material Necesario
- wokwi
- ESP32
- Ultrasonic Distance Sensor
- DHT22
- Node-RED
## Proceso para Iniciar NodeRed
1. Debemos tenerlo instalada pero lo abrimos desde el CMD de Windows 
![image](https://github.com/user-attachments/assets/15a7580d-2b93-4b1d-ae2f-fb33e8ed81da)
2. Debemos copiar y pegar esto en el CMD para acceder "npm install -g --unsafe-perm node-red" ya de de alli siempre tenemos que simplemente escribir "node-red" en el CMD y luego en el explorador escribir  "localhost:1880" en la barra de nuestro explorador deseado. 
3. También necesitamos instalar "Node Red- Dashboard para visualizar las gráficas"
![image](https://github.com/user-attachments/assets/b6b1c081-1084-4151-b46a-90b76c9e71a9)
4. Necesitamos obtener el servidor de un broker
![image](https://github.com/user-attachments/assets/322f1f09-8251-498d-abc0-e3ee32fba9bf)
5. Copiamos la dirección en nuestro cmd pero antes ponemos "NSLOOKUP"
![image](https://github.com/user-attachments/assets/b1c283c7-f199-4bae-8dfd-008c0d132e4a)
6. Después de eso copiamos y pegamos la dirección en nuestro codigo y añadimos el server tambien a Node-RED
![image](https://github.com/user-attachments/assets/b2991452-7867-46f0-93ea-86e81aa5cacb)
![image](https://github.com/user-attachments/assets/fae4523e-84e3-4a9e-85d0-564bd0091d3f)
7.- En Node-RED tenemos que crear un diagrama conectando todo, un mqtti a un json, a un debug, a tres funciones, y estas tres funciones van conectadas cada una a tres gráficas, de modo que es una función de humedad, una de temperatura y otra de distancia. 




## Programación
```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
DHTesp dhtSensor;
// Update these with values suitable for your network.
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 0;   
const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "35.172.255.228";
String username_mqtt="Guillermo";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
}

void loop() {

 long t; //timepo que demora en llegar el eco
  long d; //distancia en centimetros

  digitalWrite(Trigger, HIGH);
  delayMicroseconds(10);          //Enviamos un pulso de 10us
  digitalWrite(Trigger, LOW);
  
  t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
  d = t/59;             //escalamos el tiempo a una distancia en cm
  
  Serial.print("Distancia: ");
  Serial.print(d);      //Enviamos serialmente el valor de la distancia
  Serial.print("cm");
  Serial.println();
  delay(1000);          //Hacemos una pausa de 100ms
delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    //doc["Empresa"] = "Educatronicos";
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
    doc["DISTANCIA"]= String(d);
     doc["nombre"] = "Guillermo Hernandez";
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("NOMBRERANDOM", output.c_str());
  }
}
 ```
## Librerías

1. Arduino Json
2. DHT sensor library for ESPx
3. PubSubClient

## Conexión

![image](https://github.com/user-attachments/assets/0092530a-1ebb-461f-8d87-6b5a00509294)
![image](https://github.com/user-attachments/assets/df9c7a2f-9855-4051-bd34-3a1d9a427013)



##Instrucciones de operación 

1. Iniciar Simulador
2. Visualizar los datos en el apartado de Dashboard del node-RED
3. Subir y Bajar la distancia dando doble click al sensor ultrasonico o al dht 22 

## Resultados

Cuando funcione y Corra los valores serán mostrados en la pantalla de DashBoard, cada 2 segundos se actualizará, se muestran 3 gráficas tipo gauge (indicadores) y 3 de tipo chart (lineales)
![image](https://github.com/user-attachments/assets/66faa15f-8e79-4c88-8899-0711a55a86ff)




## Desarrollado por

Guillermo de Jesús Hernández Yáñez
[GitHub](https://github.com/inward182)
