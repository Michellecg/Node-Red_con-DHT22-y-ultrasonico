# Practica Node-RED con DHT22 y ultrasónico
En este repositorio se realizó una práctica juntando los proyectos con sensor de **temperatura y humedad** y **ultrasónico**.

## Introducción

### Descripción

En esta práctica se ocupa nuevamente el software Node-red para la comunicación de datos en línea mediante un servidor público, para la muestra de datos se utilizó un sensor **ultrasónico de distancia** y **DHT22** para su demostración.


## Material Necesario

Para realizar esta práctica se ocuparon las siguientes herramientas y componentes:

- [Wokwi](https://wokwi.com/)
- [Node-red](https://nodejs.org/en)
- Tarjeta ESP32
- Sensor **ultrasónico de distancia**
- Sensor **DHT22**


## Instrucciones

### Arranque de Node-red 

1. Dentro de Node-red, colocar un bloque ```mqtt in```, este hará la conexión mediante una IP.

2. Ingresar a la página de [emqx](https://www.emqx.com/en/mqtt/public-mqtt5-broker), el cual es un servidor público para la generación de IP, que se ocupa para el bloque del paso anterior.

3. Abrir una nueva terminal (no es necesario ejecutar como administrador) para la generación de IP. A continuación, escribir ```nslookup broker.emqx.io```

4. Copiar la IP ```44.195.202.69```, porteriormente dirigirse al ```localhost:1880```, hacer *doble click* en **mqtt in**, en el apartado *Server*, hacer click en el ícono de lápiz y pegar en el *Server*. Cambiar el nombre del bloque en *Topic*, en este caso se utilizó el nombre de "MichelleCG".

5. Añadir un bloque json, hacer *doble click* y en el apartado *Action* y seleccionar *Always convert to JavaScript Object*

6. Añadir tres bloques *Function*, hacer *doble click* para editarlo. Cambiar en los tres el nombre de la etiqueta, de **Distancia**, **Temperatura** y **Humedad** y en la pestaña **On Mesagge* escribir el siguiente código en cada uno de ellos, respectivamente:
```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
```
```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```
```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```

7. Una vez hecho, ingresar a la página de [WOKWI](https://wokwi.com/) se selecciona la tarjeta ESP32, un sensor **ultrasónico de distancia** y un sensor **DHT22**.

8. Una vez seleccionado la tarjeta ```Esp32``` junto a los componentes, en la parte izquierda se encuentra la pestaña de código donde se agrega lo siguiente:
```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
DHTesp dhtSensor;

const int Trigger=13; //Pin digital 13 para el Trigger del sensor
const int Echo=12; //Pin digital 3 para el Echo del sensor
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "44.195.202.69";
String username_mqtt="MichelleCG";
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
  pinMode(Trigger,OUTPUT); //Pin como salida
  pinMode(Echo,INPUT); //Pin como entrada
  digitalWrite(Trigger, LOW);
}

void loop() {

delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
long t; //Tiempo de demora en llegar el eco
long d; //Distancia en centimetros
digitalWrite(Trigger, HIGH);
delayMicroseconds(10);
digitalWrite(Trigger, LOW);

t=pulseIn(Echo,HIGH);
d=t/59;

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
    doc["DISTANCIA"] = String(d);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("MichelleCG", output.c_str());
  }
}

``` 
9. En la pestaña de *Library Manager*, instalar las librerías de **ArduinoJson**, **PubSubClient** y **DHT sensor library for ESPx** como se muestra en la siguente imagen.

![](https://github.com/Michellecg/Node-Red_con-DHT22-y-ultrasonico/blob/main/Lib_Ult_DHT22.PNG)

10. Hacer la conexión del **sensor ultrasónico de distancia**, **DHT22** y **LCD de 16x2** a la tarjeta **ESP32** como se muestra en la siguente imagen.

![](https://github.com/Michellecg/DHT22_y_sensor_ultrasonico/blob/main/Conex_sens.PNG)

11. Regresando a la página de Node-RED, añadir tres bloques **gauge**, se modifica el nombre a **Distancia**, **Temperatura y **Humedad**, además, se agrega un bloque **chart** para la gráfica.

12. En la esquina superior derecha (debajo del ícono de propiedades), se encuentra un ícono de triángulo, hacer click en el apartado _dashboard_, seleccionar _tab_>_group_>_spacer_. Crear un segundo grupo.
Al primer grupo creado, modificar el nombre a _Gráficos_ y el segundo _Indicadores_.

13. Hacer _doble click_ en el bloque de **Distancia**, en las propiedades de **Group**, seleccionar el grupo de **Indicadores**. En el apartado **Label**, cambiar el nombre a Distancia, las unidades a **cm** y el rango de datos. Repetir para el paso 12 y 13 para **Temperatura** y **Humedad**.

14. Modificar el bloque **chart** haciendo _doble click_. En las propiedades cambiar el apartado **Group** y elegir el grupo de _Gráficos_, cambiar el nombre a **Temp-Hum-Dist** y el rango del eje Y.

15. En Node-red, se debe observar de la siguiente manera el diagrama:

![](https://github.com/Michellecg/Node-Red_con-DHT22-y-ultrasonico/blob/main/Layout_Flow.PNG)

## Resultados

El dashboard muestra la gráfica los datos de desplazamiento junto con la temperatura y humedad contra el tiempo. El indicador que nivel de distancia se encuentra, así como la temperatura y humedad.

![](https://github.com/Michellecg/Node-Red_con-DHT22-y-ultrasonico/blob/main/Grafica_Resultados.PNG)

# Créditos

Desarrollado por Michelle Cuatlapantzi González

- [GitHub](https://github.com/Michellecg/)