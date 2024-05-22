# Dumitru

#include <ArduinoMqttClient.h>
#include <WiFi.h>

///////please enter your sensitive data in the Secret tab/arduino_secrets.h
char ssid[] = "MikroLAB";        // your network SSID (name)
char pass[] = "@2wsxdr%5";    // your network password (use for WPA, or use as key for WEP)

WiFiClient wifiClient;
MqttClient mqttClient(wifiClient);

const char broker[] = "91.121.93.94";
int        port     = 1883;
const char topic[]  = "Temperature_pol";

// Configurați pinul la care este conectat termistorul
const int termistorPin = 39;  // Pinul analogic la care este conectat termistorul

const int relayPin = 23;

// Definiți constantele pentru calculul temperaturii
const float nominalResistance = 10000;    // Rezistența nominală a termistorului la 25°C (în ohmi)
const float nominalTemperature = 25;      // Temperatura nominală pentru termistor (în grade Celsius)
const float bCoefficient = 3950;          // Coeficientul B pentru termistor (în K)
const float seriesResistor = 10000;       // Rezistența seriei (în ohmi)

void setup() {
  pinMode(relayPin, OUTPUT);
  Serial.begin(9600);

  // attempt to connect to Wifi network:
  Serial.print("Attempting to connect to WPA SSID: ");
  Serial.println(ssid);
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    // failed, retry
    Serial.print(".");
    delay(5000);
  }

  Serial.println("You're connected to the network");
  Serial.println();

  Serial.print("Attempting to connect to the MQTT broker: ");
  Serial.println(broker);

  if (!mqttClient.connect(broker, port)) {
    Serial.print("MQTT connection failed! Error code = ");
    Serial.println(mqttClient.connectError());

    while (1);
  }

  Serial.println("You're connected to the MQTT broker!");
  Serial.println();
}


void loop() {
  // Citește valoarea analogică de la termistor
  int analogValue = analogRead(termistorPin);

  // Calculează rezistența termistorului pe baza valorii citite
  float resistance = seriesResistor / ((4095.0 / analogValue) - 1);

  // Calculează temperatura în grade Celsius folosind ecuația Steinhart-Hart
  float steinhart;
  steinhart = resistance / nominalResistance;           // (R/Ro)
  steinhart = log(steinhart);                           // ln(R/Ro)
  steinhart /= bCoefficient;                            // 1/B * ln(R/Ro)
  steinhart += 1.0 / (nominalTemperature + 273.15);     // + (1/To)
  steinhart = 1.0 / steinhart;                          // Inversează
  steinhart -= 273.15;                                  // Convertește la Celsius

  // Afișează valoarea temperaturii
  Serial.print("Temperature: ");
  Serial.print(steinhart);
  Serial.println(" *C");

    Serial.print("Sending message to topic: ");
    Serial.println(topic);
    Serial.println(steinhart);

    mqttClient.beginMessage(topic);
    mqttClient.print(steinhart);
    mqttClient.endMessage();
 

  if(steinhart <= 20 ){
    digitalWrite(relayPin, HIGH);

  }
  else{
     digitalWrite(relayPin, LOW);
  }

   
  delay(3000); // Așteaptă 1 secundă înainte de a face următoarea citire
}
