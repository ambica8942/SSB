# SSB
//Smart Sign Board
  
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>  
#include <ArduinoJson.h>        
#include <Wire.h>              
#include <Adafruit_GFX.h>      
#include <Adafruit_SSD1306.h>  
 
#define OLED_RESET   5    
Adafruit_SSD1306 display(OLED_RESET);
#include <PubSubClient.h>
#define SSD1306_LCDHEIGHT 64
#define trig1 D5
#define echo1 D6
#define trig2 D7
#define echo2 D8
void PublishData(int command);
void callback(char* subtopic, byte* payload, unsigned int payloadLength);
 
// set Wi-Fi SSID and password
const char *ssid     = "vivo 1713";
const char *password = "99999999";
//---------DEVICE CRED----------
#define ORG "ks6fk6"
#define DEVICE_TYPE "node123"
#define DEVICE_ID "nodes4321"
#define TOKEN "0987654321"
String command;
int x;
int h;
char server[] = ORG ".messaging.internetofthings.ibmcloud.com";
char subtopic[] = "iot-2/cmd/SSB/fmt/String";
char pubtopic[] = "iot-2/evt/SSB/fmt/json";
char authMethod[] = "use-token-auth";
char token[] = TOKEN;
char clientId[] = "d:" ORG ":" DEVICE_TYPE ":" DEVICE_ID;
//Serial.println(clientID);


 
// set location and API key
String Location = "Tokyo, JP";
String API_Key  = "61de582dc03d7d7da65b115c26156c39";


 
WiFiClient wifiClient;
PubSubClient client(server, 1883, callback, wifiClient);

void wifiConnect();
void mqttConnect();
void initManagedDevice();
void setup(void)
{
  pinMode(trig1,OUTPUT);
  pinMode(echo1,INPUT);
  pinMode(trig2,OUTPUT);
  pinMode(echo2,INPUT);
  Serial.begin(9600);
  delay(1000);
 
  Wire.begin(4, 0);
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C); 
  Wire.setClock(400000L);   
  display.clearDisplay();  // clear the display buffer
  display.setTextColor(WHITE, BLACK);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("  The Originals  ");
  display.display();
 
  WiFi.begin(ssid, password);
 
  Serial.print("Connecting.");
  display.setCursor(0, 24);
  display.println("Connecting...");
  display.display();
  while ( WiFi.status() != WL_CONNECTED )
  {
    delay(500);
    Serial.print(".");
  }
  Serial.println("connected");
  display.print("connected");
  display.display();
   mqttConnect(); 
  delay(1000);
 
}
 
void loop()
{
  if (WiFi.status() == WL_CONNECTED)  
  {
    HTTPClient http;  
    http.begin("http://api.openweathermap.org/data/2.5/weather?q=" + Location + "&APPID=" + API_Key);  
    int httpCode = http.GET(); 
    if (httpCode > 0)  
    {
      String payload = http.getString();   //Get the request response payload
 
      DynamicJsonBuffer jsonBuffer(512);
 
      // Parse JSON object
      JsonObject& root = jsonBuffer.parseObject(payload);
      if (!root.success()) {
        Serial.println(F("Parsing failed!"));
        return;
      }
 
      float temp = (float)(root["main"]["temp"]) - 273.15;        
       h= root["main"]["humidity"];                  
      float pressure = (float)(root["main"]["pressure"]) / 1000;  
      float wind_speed = root["wind"]["speed"];                   
      int  wind_degree = root["wind"]["deg"];                     
    }
 
    http.end();   //Close connection
 
  }
  digitalWrite(trig1, LOW);
  delay(100);
  digitalWrite(trig1, HIGH);
  delay(100);
  digitalWrite(trig1, LOW);
  int du1=pulseIn(echo1,HIGH);
  int dis1=(du1*0.034)/2;
  digitalWrite(trig2, LOW);
  delay(100);
  digitalWrite(trig2, HIGH);
  delay(100);
  digitalWrite(trig2, LOW);
  int du2=pulseIn(echo2,HIGH);
  int dis2=(du2*0.034)/2;
  Serial.print("Obstacle is at distance1:");
  Serial.println(dis1);  
  Serial.print("Obstacle is at distance2:");
  Serial.println(dis2);
  display.clearDisplay();
  display.setCursor(0,0);
  if(dis1<100 && dis2<100 && h<80)
  {
    delay(2000);
    display.print("Speed Limit: 60");
    display.setCursor(0,10);
    if(dis1<100 && dis2<100)
    {
        Serial.print(command);
        if(command == "Road under construction")  
        {
          x=1;
          display.print("Road under construction");
        }
        else if(command == "Accident Occured"){
          x=2;
          display.print("Accident Occured");
        }
      else{
        x=3;
        display.print("Traffic ahead ");

      }
    display.setCursor(0,20);
    display.println("Take diversion");    
    display.display();
    }
  }
  else if(dis1>100 && dis2>100 && h<80)
  {
    display.print("Speed Limit: 60");
    display.setCursor(0,10);
    display.print("Go Ahead");
    display.display();
    command="";
    display.clearDisplay();
    x=101;
  }
   else if(dis1<100 && dis2<100 && h>80)
  {
    display.print("Speed Limit: 40");
    display.setCursor(0,10);
    display.print("Go Slow");
    display.setCursor(0,20);
    display.print("Traffic Ahead");
    display.display();
    command="";
    display.clearDisplay();
    x=101;
  }
  else if (h>80)
  {
    display.print("Speed Limit: 40");
    display.setCursor(0,10);
    display.print("Go Slow");
    display.display();
    command="";
  }
  PublishData(x);  
  if (!client.loop()) {
    mqttConnect();
  }   // wait 1 minute
 
}
void wifiConnect() {
  Serial.print("Connecting to "); Serial.print(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.print("WiFi connected, IP address: "); 
  Serial.println(WiFi.localIP());
}
void mqttConnect() {
  if (!client.connected()) {
    Serial.print("Reconnecting MQTT client to "); 
    Serial.println(server);
    while (!client.connect(clientId, authMethod, token)) {
      Serial.print(".");
      delay(500);
    }
    initManagedDevice();
    Serial.println();
  }
}

void initManagedDevice() {
  if (client.subscribe(subtopic)) {
    Serial.println("subscribe to cmd OK");
  } else {
    Serial.println("subscribe to cmd FAILED");
  }
}
void callback(char* subtopic, byte* payload, unsigned int payloadLength) {
Serial.print("callback invoked for sub topic: "); 
Serial.println(subtopic);
command="";
for (int i = 0; i < payloadLength; i++) {
  command += (char)payload[i];
}
  Serial.println(command);
}

void PublishData(int command){ 
 if (!!!client.connected()) {
 Serial.print("Reconnecting client to ");
 Serial.println(server);
 while (!!!client.connect(clientId, authMethod, token)) {
 Serial.print(".");
 delay(500);
 }
 Serial.println();
 }
  String payload = "{\"d\":{\"command\":";
  payload += command;
  payload += "}}";
 Serial.print("Sending payload: ");
 Serial.println(payload);
  
 if (client.publish(pubtopic, (char*) payload.c_str())) {
 Serial.println("Publish ok");
 } else {
 Serial.println("Publish failed");
 }

}
