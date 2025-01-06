 WOKWI  改寫  ESP32 MQTT – Publish DHT11/DHT22 Temperature and Humidity Readings (Arduino IDE)

![2025-01-05 16 52 03](https://hackmd.io/_uploads/rkbQVh_Lke.png)
![2025-01-05 18 55 52](https://hackmd.io/_uploads/SJZQV2dL1e.png)
![2025-01-05 19 00 01](https://hackmd.io/_uploads/r1bQV2OIJe.png)



WOKWI程式

#include <WiFi.h>
#include <MQTTPubSubClient.h>
#include <Adafruit_Sensor.h>
#include <DHT_U.h>
extern "C" {
  #include "freertos/FreeRTOS.h"
  #include "freertos/timers.h"
}


#define Relay1            15     //D15 LED-Orangs = Water-1
#define Relay2            2     //D2   LED-Blue  = Water-2
#define Relay3            19    //D19  LED-Blue   = Motor
#define Relay4            23    //D23  LED-RED    = Fan

#define WIFI_SSID "Wokwi-GUEST"
#define WIFI_PASSWORD ""

#define MQTT_HOST "test.mosquitto.org"
#define MQTT_PORT 1883

#define DHTPIN 12
// DHT parameters
#define DHTTYPE    DHT22     // DHT 11
DHT_Unified dht(DHTPIN, DHTTYPE);

WiFiClient client;
MQTTPubSubClient mqtt;
int count;
float temp, hum;
bool Send= false;
String json="";

//=============================================================================
void connect() {
  connect_to_wifi:
    Serial.print("connecting to wifi...");
    while (WiFi.status() != WL_CONNECTED) {
        Serial.print(".");
        delay(1000);
    }
    Serial.println(" connected!");

  connect_to_host:
    Serial.print("connecting to host...");
    client.stop();
    while (!client.connect(MQTT_HOST, 1883)) {//while (!client.connect("public.cloud.shiftr.io", 1883)) {
        Serial.print(".");
        delay(1000);
        if (WiFi.status() != WL_CONNECTED) {
            Serial.println("WiFi disconnected");
            goto connect_to_wifi;
        }
    }
    Serial.println(" connected!");

    Serial.print("connecting to mqtt broker...");
    mqtt.disconnect();
    while (!mqtt.connect("arduino", "", "")) {
        Serial.print(".");
        delay(1000);
        if (WiFi.status() != WL_CONNECTED) {
            Serial.println("WiFi disconnected");
            goto connect_to_wifi;
        }
        if (client.connected() != 1) {
            Serial.println("WiFiClient disconnected");
            goto connect_to_host;
        }
    }
    Serial.println(" connected!");
}
//=============================================================================
void  Publish_message() {
  if (Send){
    mqtt.publish("alex9ufo/esp32/led/status", json);

    Serial.println();  
    Serial.print("publish message to MQTT ---");
    Serial.println(json);
    
    Send= false;
    json="";
  }

}
//=============================================================================
void setup() {

    pinMode(Relay1, OUTPUT);
    pinMode(Relay2, OUTPUT);
    pinMode(Relay3, OUTPUT);
    pinMode(Relay4, OUTPUT);

    Serial.begin(115200);
    dht.begin();
    // Get temperature sensor details.
    sensor_t sensor;
    dht.temperature().getSensor(&sensor);
    dht.humidity().getSensor(&sensor);



    WiFi.begin(WIFI_SSID, WIFI_PASSWORD, 6);
    mqtt.begin(client);
    connect();

    mqtt.subscribe([](const String& topic, const String& payload, const size_t size) {
        Serial.println("mqtt received: " + topic + " - " + payload);
    });

    mqtt.subscribe("alex9ufo/esp32/led/control", [] (const String& payload, const size_t size) {
        Serial.print("alex9ufo/esp32/led/control");
        Serial.println(payload);
       
        String message=payload;
        message.trim();
        Serial.print(message);


        if ( message == "1on" ) {
            digitalWrite(Relay1, HIGH);  // Turn on the LED
            Send = true;
            json="Relay1on";

        }
        if ( message == "1off" ) {
            digitalWrite(Relay1, LOW);  // Turn off the LED
            Send = true;
            json="Relay1off"; 
        }    
        if ( message == "2on" ) {
            digitalWrite(Relay2, HIGH);  // Turn on the LED 
            Send = true;
            json="Relay2on";  
        }
        if ( message == "2off" ) {
            digitalWrite(Relay2, LOW);  // Turn off the LED 
            Send = true;
            json="Relay2off"; 
        }  
        if ( message == "3on" ) {
            digitalWrite(Relay3, HIGH);  // Turn on the LED 
            Send = true;
            json="Relay3on"; 
        }
        if ( message == "3off" ) {
            digitalWrite(Relay3, LOW);  // Turn off the LED 
            Send = true;
            json="Relay3off"; 
        }    
        if ( message == "4on" ) {
            digitalWrite(Relay4, HIGH);  // Turn on the LED 
            Send = true;
            json="Relay4on"; 
        }
        if ( message == "4off" ) {
            digitalWrite(Relay4, LOW);  // Turn off the LED 
            Send = true;
            json="Relay4off"; 
        } 
    });

    /***
    mqtt.subscribe("alex9ufo/esp32/dht/temp", [](const String& payload, const size_t size) {
        Serial.print("alex9ufo/esp32/dht/temp");
        Serial.println(payload);
    });

    mqtt.subscribe("alex9ufo/esp32/dht/humi", [](const String& payload, const size_t size) {
        Serial.print("alex9ufo/esp32/dht/humi");
        Serial.println(payload);
    });
    ***/

}

//=============================================================================
void loop() {
    mqtt.update();
    Publish_message();

    if (!mqtt.isConnected()) {
        connect();
    }

    // publish message
    static uint32_t prev_ms = millis();
    if (millis() > prev_ms + 2500) {
        prev_ms = millis();
        sensors_event_t event;
      dht.temperature().getEvent(&event);
      if (isnan(event.temperature)) {
          Serial.println(F("Error reading temperature!"));
      }
      else {
        Serial.print(F("Temperature: "));
        temp = event.temperature;
        Serial.print(temp);
        Serial.println(F("°C"));
      }
      // Get humidity event and print its value
      dht.humidity().getEvent(&event);
      if (isnan(event.relative_humidity)) {
        Serial.println(F("Error reading humidity!"));
      }
      else {
        Serial.print(F("Humidity: "));
        hum = event.relative_humidity;
        Serial.print(hum);
        Serial.println(F("%"));
      }
      String msgStr = String(temp) + "," + String(hum) + ",";

      //mqtt.publish("alex9ufo/esp32/dht/temp", String(temp));
      //delay(100);
      //mqtt.publish("alex9ufo/esp32/dht/humi", String(hum));
      delay(100);  
      mqtt.publish("alex9ufo/esp32/dht/temphumi", msgStr);
    }
}


Node-Red程式


[
    {
        "id": "5640b148882b4cf5",
        "type": "comment",
        "z": "6fc7559b07049ecb",
        "name": "資料表 :TABLE DHT22STATUS",
        "info": "CREATE TABLE DHT22STATUS (\nid INTEGER,\nTemp TEXT,\nHumi TEXT,\nDate DATE,\nTime TIME,\nPRIMARY KEY (id)\n);\n\n",
        "x": 290,
        "y": 480,
        "wires": []
    },
    {
        "id": "1ded01ae7510fc0c",
        "type": "ui_gauge",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "7aa2048fc0eab3a0",
        "order": 10,
        "width": 6,
        "height": 3,
        "gtype": "gage",
        "title": "溫度",
        "label": "°C",
        "format": "{{value}}",
        "min": "-40",
        "max": "80",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "20",
        "seg2": "40",
        "className": "",
        "x": 670,
        "y": 160,
        "wires": []
    },
    {
        "id": "f8d97407e55e141b",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "get 濕度",
        "func": "msg.payload=msg.payload[1];\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 480,
        "y": 240,
        "wires": [
            [
                "abb8dd5398d39144",
                "9b99804f9ec58f33"
            ]
        ]
    },
    {
        "id": "e02dae7010eb1b11",
        "type": "mqtt in",
        "z": "6fc7559b07049ecb",
        "name": "DHT22 Subscribe",
        "topic": "alex9ufo/esp32/dht/temphumi",
        "qos": "1",
        "datatype": "auto-detect",
        "broker": "21957383cfd8785a",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 230,
        "y": 120,
        "wires": [
            [
                "eea7936e61bcdc82",
                "a50c4a7dfd49eed1"
            ]
        ]
    },
    {
        "id": "eea7936e61bcdc82",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 338",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 450,
        "y": 100,
        "wires": []
    },
    {
        "id": "a50c4a7dfd49eed1",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "分離",
        "func": "msg.payload = msg.payload.split(\",\");\nreturn msg;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 330,
        "y": 240,
        "wires": [
            [
                "263eb42e72d328b9",
                "f8d97407e55e141b",
                "4707f2164188b011"
            ]
        ]
    },
    {
        "id": "263eb42e72d328b9",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "get 溫度",
        "func": "msg.payload=msg.payload[0];\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 480,
        "y": 160,
        "wires": [
            [
                "a6a437f0145c0c2e",
                "1ded01ae7510fc0c"
            ]
        ]
    },
    {
        "id": "a6a437f0145c0c2e",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 339",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 690,
        "y": 200,
        "wires": []
    },
    {
        "id": "abb8dd5398d39144",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 340",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 690,
        "y": 240,
        "wires": []
    },
    {
        "id": "9b99804f9ec58f33",
        "type": "ui_gauge",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "7aa2048fc0eab3a0",
        "order": 11,
        "width": 6,
        "height": 3,
        "gtype": "gage",
        "title": "濕度",
        "label": "%",
        "format": "{{value}}",
        "min": 0,
        "max": "100",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "40",
        "seg2": "60",
        "className": "",
        "x": 670,
        "y": 280,
        "wires": []
    },
    {
        "id": "461cfa80b0273d7e",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 770,
        "y": 540,
        "wires": [
            [
                "6d524339e6dff301"
            ]
        ]
    },
    {
        "id": "268ba01972a277f5",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "CREATE　DATABASE",
        "func": "//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\nmsg.topic = \"CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,Date DATE,Time TIME,PRIMARY KEY (id))\";\nreturn msg;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 460,
        "y": 540,
        "wires": [
            [
                "461cfa80b0273d7e"
            ]
        ]
    },
    {
        "id": "42b83c22313a88aa",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 6,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "建立資料庫",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "建立資料庫",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 230,
        "y": 540,
        "wires": [
            [
                "268ba01972a277f5",
                "f7e5c311d42cc72b"
            ]
        ]
    },
    {
        "id": "4707f2164188b011",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "INSERT",
        "func": "var temp1=msg.payload[0];\nvar humi1=msg.payload[1];\n\n\nvar Today = new Date();\nvar yyyy = Today.getFullYear(); //年\nvar MM = Today.getMonth()+1;    //月\nvar dd = Today.getDate();       //日\nvar h = Today.getHours();       //時\nvar m = Today.getMinutes();     //分\nvar s = Today.getSeconds();     //秒\nif(MM<10)\n{\n   MM = '0'+MM;\n}\n\nif(dd<10)\n{\n   dd = '0'+dd;\n}\n\nif(h<10)\n{\n   h = '0'+h;\n}\n\nif(m<10)\n{\n  m = '0' + m;\n}\n\nif(s<10)\n{\n  s = '0' + s;\n}\nvar var_date = yyyy+'/'+MM+'/'+dd;\nvar var_time = h+':'+m+':'+s;\n\nvar myLED = msg.payload;\n\n\nmsg.topic = \"INSERT INTO DHT22STATUS (Temp,Humi , Date , Time ) VALUES ($temp1, $humi1, $var_date ,  $var_time ) \" ;\nmsg.payload = [temp1, humi1 , var_date , var_time ]\nreturn msg;\n\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 500,
        "y": 420,
        "wires": [
            [
                "91d1cdd83e777129"
            ]
        ]
    },
    {
        "id": "91d1cdd83e777129",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 690,
        "y": 420,
        "wires": [
            [
                "d3aa708c457c7204",
                "e1ecb014ec0a045e"
            ]
        ]
    },
    {
        "id": "6d524339e6dff301",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug ",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 970,
        "y": 540,
        "wires": []
    },
    {
        "id": "d3aa708c457c7204",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug ",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 950,
        "y": 420,
        "wires": []
    },
    {
        "id": "79ed12d6d169a758",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 7,
        "width": 6,
        "height": 1,
        "passthru": false,
        "label": "檢視資料庫資料",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "檢視資料",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 240,
        "y": 660,
        "wires": [
            [
                "e1ecb014ec0a045e",
                "f7e5c311d42cc72b"
            ]
        ]
    },
    {
        "id": "e1ecb014ec0a045e",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "檢視資料",
        "func": "//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n\nmsg.topic = \"SELECT * FROM DHT22STATUS ORDER BY id DESC LIMIT 50\";\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 520,
        "y": 660,
        "wires": [
            [
                "196d0a848e9b51d8"
            ]
        ]
    },
    {
        "id": "034d60ab31a25883",
        "type": "ui_table",
        "z": "6fc7559b07049ecb",
        "group": "60bc276d0fdd5026",
        "name": "",
        "order": 1,
        "width": 10,
        "height": 14,
        "columns": [],
        "outputs": 0,
        "cts": false,
        "x": 1070,
        "y": 660,
        "wires": []
    },
    {
        "id": "196d0a848e9b51d8",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 770,
        "y": 660,
        "wires": [
            [
                "034d60ab31a25883"
            ]
        ]
    },
    {
        "id": "d183a13f88978c3a",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 3,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "刪除資料庫 ",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "刪除資料庫 ",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 230,
        "y": 920,
        "wires": [
            [
                "537d0cedc28a1115",
                "7de63fde2be756a0"
            ]
        ]
    },
    {
        "id": "7d5838e569f8aad8",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "DROP　DATABASE",
        "func": "\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n\nmsg.topic = \"DROP TABLE DHT22STATUS\";\nreturn msg;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 900,
        "y": 920,
        "wires": [
            [
                "12154f6e2c3364fe"
            ]
        ]
    },
    {
        "id": "12154f6e2c3364fe",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 1110,
        "y": 900,
        "wires": [
            [
                "63797f4590221f1d"
            ]
        ]
    },
    {
        "id": "e40f2b7932674ebb",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 2,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "刪除所有資料 ",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "刪除所有資料 ",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 240,
        "y": 860,
        "wires": [
            [
                "7de63fde2be756a0",
                "066b3cc3b01fb17b"
            ]
        ]
    },
    {
        "id": "f502278d96e3a9dd",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "DELETE ALL DATA",
        "func": "\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n\nmsg.topic = \"DELETE from DHT22STATUS\";\nreturn msg;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 890,
        "y": 860,
        "wires": [
            [
                "12154f6e2c3364fe"
            ]
        ]
    },
    {
        "id": "537d0cedc28a1115",
        "type": "ui_toast",
        "z": "6fc7559b07049ecb",
        "position": "prompt",
        "displayTime": "3",
        "highlight": "",
        "sendall": true,
        "outputs": 1,
        "ok": "OK",
        "cancel": "Cancel",
        "raw": true,
        "className": "",
        "topic": "",
        "name": "",
        "x": 490,
        "y": 920,
        "wires": [
            [
                "f27780b726f51f50"
            ]
        ]
    },
    {
        "id": "f27780b726f51f50",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "OK or Cancel",
        "func": "var topic=msg.payload;\nif (topic==\"\"){\n    return [msg,null];\n    \n}\nif (topic==\"Cancel\"){\n    return [null,msg];\n    \n}\nreturn msg;",
        "outputs": 2,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 680,
        "y": 920,
        "wires": [
            [
                "7d5838e569f8aad8"
            ],
            []
        ]
    },
    {
        "id": "7de63fde2be756a0",
        "type": "ui_audio",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "voice": "Microsoft Hanhan - Chinese (Traditional, Taiwan)",
        "always": true,
        "x": 375,
        "y": 900,
        "wires": [],
        "l": false
    },
    {
        "id": "066b3cc3b01fb17b",
        "type": "ui_toast",
        "z": "6fc7559b07049ecb",
        "position": "prompt",
        "displayTime": "3",
        "highlight": "",
        "sendall": true,
        "outputs": 1,
        "ok": "OK",
        "cancel": "Cancel",
        "raw": true,
        "className": "",
        "topic": "",
        "name": "",
        "x": 490,
        "y": 860,
        "wires": [
            [
                "474334f8e3deb630"
            ]
        ]
    },
    {
        "id": "474334f8e3deb630",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "OK or Cancel",
        "func": "var topic=msg.payload;\nif (topic==\"\"){\n    return [msg,null];\n    \n}\nif (topic==\"Cancel\"){\n    return [null,msg];\n    \n}\nreturn msg;",
        "outputs": 2,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 680,
        "y": 860,
        "wires": [
            [
                "f502278d96e3a9dd"
            ],
            []
        ]
    },
    {
        "id": "f7e5c311d42cc72b",
        "type": "ui_audio",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "voice": "Microsoft Hanhan - Chinese (Traditional, Taiwan)",
        "always": true,
        "x": 365,
        "y": 600,
        "wires": [],
        "l": false
    },
    {
        "id": "63797f4590221f1d",
        "type": "link out",
        "z": "6fc7559b07049ecb",
        "name": "link out 65",
        "mode": "link",
        "links": [
            "07427ec2928408b7"
        ],
        "x": 1235,
        "y": 860,
        "wires": []
    },
    {
        "id": "07427ec2928408b7",
        "type": "link in",
        "z": "6fc7559b07049ecb",
        "name": "link in 63",
        "links": [
            "63797f4590221f1d"
        ],
        "x": 435,
        "y": 740,
        "wires": [
            [
                "e1ecb014ec0a045e"
            ]
        ]
    },
    {
        "id": "bcceb14254aac9d8",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 1,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "查詢一筆資料",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "查詢一筆資料",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 240,
        "y": 1000,
        "wires": [
            [
                "7696fc84f97e7be5",
                "d4d5dc7503fbe57e"
            ]
        ]
    },
    {
        "id": "e077ddca23d5302e",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "查詢一筆資料",
        "func": "//\nvar id = msg.payload.id;\nvar s=global.get(\"SEL1\")\nmsg.topic=\"\";\nvar temp=\"\";\n\nif (s==1)\n{\n    temp =\"SELECT * FROM DHT22STATUS\";\n    temp=temp+\" WHERE id LIKE '\"+ id +\"'\";\n}\nmsg.topic=temp;\nglobal.set(\"SEL1\",0);\n\nreturn msg;\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 780,
        "y": 1000,
        "wires": [
            [
                "d5c8b3eca9615bc2"
            ]
        ]
    },
    {
        "id": "e6a06ec2490f8bf9",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 4,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "刪除一筆資料",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "刪除一筆資料",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 240,
        "y": 1060,
        "wires": [
            [
                "d4d5dc7503fbe57e",
                "4fb71e1248d7770c"
            ]
        ]
    },
    {
        "id": "d4d5dc7503fbe57e",
        "type": "ui_audio",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "voice": "Microsoft Hanhan - Chinese (Traditional, Taiwan)",
        "always": true,
        "x": 375,
        "y": 1100,
        "wires": [],
        "l": false
    },
    {
        "id": "39b5b457d1d984e2",
        "type": "ui_form",
        "z": "6fc7559b07049ecb",
        "name": "",
        "label": "輸入id",
        "group": "2fc188c006769769",
        "order": 8,
        "width": 0,
        "height": 0,
        "options": [
            {
                "label": "ID",
                "value": "id",
                "type": "number",
                "required": true,
                "rows": null
            }
        ],
        "formValue": {
            "id": ""
        },
        "payload": "",
        "submit": "Submit",
        "cancel": "Cancle",
        "topic": "Form",
        "topicType": "str",
        "splitLayout": false,
        "className": "",
        "x": 630,
        "y": 1060,
        "wires": [
            [
                "e077ddca23d5302e",
                "d561260c49287216",
                "bb120f4b335c6e46"
            ]
        ]
    },
    {
        "id": "d561260c49287216",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "刪除一筆資料",
        "func": "//\nvar id = msg.payload.id;\nvar s=global.get(\"SEL2\")\nmsg.topic=\"\";\nvar temp=\"\";\n\nif (s==2)\n{\n    temp =\"DELETE FROM DHT22STATUS\";\n    temp=temp+\" WHERE id LIKE '\"+ id +\"'\";\n}\n\nmsg.topic=temp;\nglobal.set(\"SEL2\",0)\nreturn msg;\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 800,
        "y": 1060,
        "wires": [
            [
                "1f60cb4d9347e867",
                "cf26aff8968959b8"
            ]
        ]
    },
    {
        "id": "1f60cb4d9347e867",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 1190,
        "y": 1060,
        "wires": [
            [
                "63797f4590221f1d"
            ]
        ]
    },
    {
        "id": "3272a5b7c20bfa19",
        "type": "ui_button",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "order": 5,
        "width": 2,
        "height": 1,
        "passthru": false,
        "label": "更正一筆資料",
        "tooltip": "",
        "color": "",
        "bgcolor": "",
        "className": "",
        "icon": "",
        "payload": "更正一筆資料",
        "payloadType": "str",
        "topic": "topic",
        "topicType": "msg",
        "x": 240,
        "y": 1140,
        "wires": [
            [
                "d4d5dc7503fbe57e",
                "a338d5a3635e077c"
            ]
        ]
    },
    {
        "id": "8a1ebe7bba14a463",
        "type": "comment",
        "z": "6fc7559b07049ecb",
        "name": "UPDATE查詢的WHERE",
        "info": "UPDATE查詢的WHERE子句的基本語法如下：\n\nUPDATE table_name\nSET column1 = value1, column2 = value2...., columnN = valueN\nWHERE [condition];",
        "x": 260,
        "y": 1180,
        "wires": []
    },
    {
        "id": "26110da0b86c2667",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "更正一筆資料",
        "func": "//\nvar id = global.get(\"ID\");\nvar temp2 = msg.payload.Temp;\nvar humi2 = msg.payload.Humi;\nvar date = msg.payload.date;\nvar time = msg.payload.time;\n\nvar s=global.get(\"SEL3\")\nmsg.topic=\"\";\nvar temp=\"\";\n\nif (s==3)\n{\n    temp =\"update DHT22STATUS set \";\n    temp = temp +\"  Temp= '\" + temp2 +\"'\";\n    temp = temp +\" , Humi= '\" + humi2 +\"'\";\n    temp=temp+\" , Date= '\" + date +\"'\";\n    temp=temp+\" , Time= '\" + time +\"'\";\n    temp=temp+\"  WHERE id=\" + id;\n   \n\n}\nmsg.topic=temp;\n\nreturn msg;\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 900,
        "y": 1180,
        "wires": [
            [
                "1f60cb4d9347e867",
                "8cd4f794290938a0"
            ]
        ]
    },
    {
        "id": "7696fc84f97e7be5",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "function flow set1",
        "func": "var s1=1;\nglobal.set(\"SEL1\",s1);\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 450,
        "y": 1000,
        "wires": [
            [
                "39b5b457d1d984e2"
            ]
        ]
    },
    {
        "id": "4fb71e1248d7770c",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "function flow set2",
        "func": "var s1=2;\nglobal.set(\"SEL2\",s1);\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 450,
        "y": 1060,
        "wires": [
            [
                "39b5b457d1d984e2"
            ]
        ]
    },
    {
        "id": "a338d5a3635e077c",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "function flow set3",
        "func": "var s1=3;\nglobal.set(\"SEL3\",s1);\nreturn msg;",
        "outputs": 1,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 450,
        "y": 1140,
        "wires": [
            [
                "39b5b457d1d984e2"
            ]
        ]
    },
    {
        "id": "9f645b00e8806dec",
        "type": "ui_form",
        "z": "6fc7559b07049ecb",
        "name": "",
        "label": "更正欄位",
        "group": "2fc188c006769769",
        "order": 9,
        "width": 0,
        "height": 0,
        "options": [
            {
                "label": "Temperature",
                "value": "Temp",
                "type": "text",
                "required": true,
                "rows": null
            },
            {
                "label": "Humidity",
                "value": "Humi",
                "type": "text",
                "required": true,
                "rows": null
            },
            {
                "label": "DATE",
                "value": "date",
                "type": "text",
                "required": true,
                "rows": null
            },
            {
                "label": "TIME",
                "value": "time",
                "type": "text",
                "required": true,
                "rows": null
            }
        ],
        "formValue": {
            "Temp": "",
            "Humi": "",
            "date": "",
            "time": ""
        },
        "payload": "",
        "submit": "Submit",
        "cancel": "Cancle",
        "topic": "Form",
        "topicType": "str",
        "splitLayout": false,
        "className": "",
        "x": 720,
        "y": 1180,
        "wires": [
            [
                "26110da0b86c2667"
            ]
        ]
    },
    {
        "id": "cf26aff8968959b8",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 341",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "topic",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 990,
        "y": 1040,
        "wires": []
    },
    {
        "id": "bb120f4b335c6e46",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "Store ID資料",
        "func": "//\nvar id = msg.payload.id;\nglobal.set(\"ID\",id)\nreturn msg;\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 730,
        "y": 1120,
        "wires": [
            [
                "9f645b00e8806dec",
                "96d76efbc888515b"
            ]
        ]
    },
    {
        "id": "96d76efbc888515b",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "查詢一筆資料",
        "func": "//\nvar id = global.get(\"ID\");\nmsg.topic=\"\";\nvar temp=\"\";\ntemp =\"SELECT * FROM DHT22STATUS\";\ntemp=temp+\" WHERE id LIKE '\"+ id +\"'\";\n\nmsg.topic=temp;\n\nreturn msg;\n\n//CREATE TABLE DHT22STATUS(\n//    id INTEGER,\n//    Temp TEXT,\n//    Humi TEXT,\n//    Date DATE,\n//    Time TIME,\n//    PRIMARY KEY(id)\n//);\n\n//CREATE TABLE DHT22STATUS (id INTEGER,Temp TEXT, Humi TEXT,  Date DATE,Time TIME,PRIMARY KEY (id));\n//SELECT * FROM DHT22STATUS ORDER BY  id DESC LIMIT 50;\n",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 920,
        "y": 1120,
        "wires": [
            [
                "d5c8b3eca9615bc2"
            ]
        ]
    },
    {
        "id": "8cd4f794290938a0",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 342",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "topic",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 1090,
        "y": 1180,
        "wires": []
    },
    {
        "id": "0ecb56d1c3d20ec5",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "function ",
        "func": "msg.payload=\" ---ESP32回來資料---\" +msg.payload;\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 400,
        "y": 1300,
        "wires": [
            [
                "fd26b6bf4d0cba30",
                "1bc91052ac871f04"
            ]
        ]
    },
    {
        "id": "fd26b6bf4d0cba30",
        "type": "function",
        "z": "6fc7559b07049ecb",
        "name": "Set Line API ",
        "func": "msg.headers = {'content-type':'application/x-www-form-urlencoded','Authorization':'Bearer A4wwPNh2WqB7dlfeQyyIAwtggn1kfZSI5LkkCdia1gB'};\nmsg.payload = {\"message\":msg.payload};\nreturn msg;\n\n//oR7KdXvK1eobRr2sRRgsl4PMq23DjDlhfUs96SyUBZu",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 590,
        "y": 1300,
        "wires": [
            [
                "09f4f949214d30ea"
            ]
        ]
    },
    {
        "id": "09f4f949214d30ea",
        "type": "http request",
        "z": "6fc7559b07049ecb",
        "name": "",
        "method": "POST",
        "ret": "txt",
        "paytoqs": false,
        "url": "https://notify-api.line.me/api/notify",
        "tls": "",
        "persist": false,
        "proxy": "",
        "authType": "",
        "x": 760,
        "y": 1300,
        "wires": [
            [
                "cf0157a631e1cabb"
            ]
        ]
    },
    {
        "id": "cf0157a631e1cabb",
        "type": "debug",
        "z": "6fc7559b07049ecb",
        "name": "debug 343",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 930,
        "y": 1300,
        "wires": []
    },
    {
        "id": "d7074c27b1464c96",
        "type": "comment",
        "z": "6fc7559b07049ecb",
        "name": "Line Notify Message ",
        "info": "",
        "x": 610,
        "y": 1260,
        "wires": []
    },
    {
        "id": "08a695bdc148df10",
        "type": "ui_audio",
        "z": "6fc7559b07049ecb",
        "name": "",
        "group": "2fc188c006769769",
        "voice": "Microsoft Hanhan - Chinese (Traditional, Taiwan)",
        "always": "",
        "x": 760,
        "y": 1340,
        "wires": []
    },
    {
        "id": "1bc91052ac871f04",
        "type": "delay",
        "z": "6fc7559b07049ecb",
        "name": "",
        "pauseType": "delay",
        "timeout": "1",
        "timeoutUnits": "seconds",
        "rate": "1",
        "nbRateUnits": "1",
        "rateUnits": "second",
        "randomFirst": "1",
        "randomLast": "5",
        "randomUnits": "seconds",
        "drop": false,
        "allowrate": false,
        "outputs": 1,
        "x": 580,
        "y": 1340,
        "wires": [
            [
                "08a695bdc148df10"
            ]
        ]
    },
    {
        "id": "d5c8b3eca9615bc2",
        "type": "sqlite",
        "z": "6fc7559b07049ecb",
        "mydb": "647ea72ac9a75c97",
        "sqlquery": "msg.topic",
        "sql": "",
        "name": "DHT22_STATUS",
        "x": 990,
        "y": 1000,
        "wires": [
            [
                "034d60ab31a25883"
            ]
        ]
    },
    {
        "id": "538a60919a2f110d",
        "type": "mqtt in",
        "z": "6fc7559b07049ecb",
        "name": "DHT22 Subscribe",
        "topic": "alex9ufo/esp32/dht/temphumi",
        "qos": "1",
        "datatype": "auto-detect",
        "broker": "21957383cfd8785a",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 210,
        "y": 1300,
        "wires": [
            [
                "0ecb56d1c3d20ec5"
            ]
        ]
    },
    {
        "id": "7aa2048fc0eab3a0",
        "type": "ui_group",
        "name": "溫度濕度",
        "tab": "fbd26925fd4f8d9b",
        "order": 3,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "21957383cfd8785a",
        "type": "mqtt-broker",
        "name": "test.mosquitto.org",
        "broker": "test.mosquitto.org",
        "port": "1883",
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "647ea72ac9a75c97",
        "type": "sqlitedb",
        "db": "2025DHT22.db",
        "mode": "RWC"
    },
    {
        "id": "2fc188c006769769",
        "type": "ui_group",
        "name": "DHT22 資料",
        "tab": "fbd26925fd4f8d9b",
        "order": 1,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "60bc276d0fdd5026",
        "type": "ui_group",
        "name": "DHT22顯示區",
        "tab": "fbd26925fd4f8d9b",
        "order": 2,
        "disp": true,
        "width": "10",
        "collapse": false,
        "className": ""
    },
    {
        "id": "fbd26925fd4f8d9b",
        "type": "ui_tab",
        "name": "DHT22 Database",
        "icon": "dashboard",
        "disabled": false,
        "hidden": false
    }
]