---
layout: ../../styles/MarkdownPostLayout.astro
title: 'The Process and Creation of a DIY Smart System for my Dissertation'
pubDate: 2025-07-23
description: 'The trials and tribulations of designing a DIY ESP32-based breadboard temperature and humidity sensor for my University major project.'
author: 'Astro Learner'
image:
    url: 'https://docs.astro.build/assets/rose.webp'
    alt: 'The Astro logo on a dark background with a pink glow.'
tags: ["astro", "blogging", "learning in public"]
---
<hr/>

Back in University, I had to create a dissertation to officially complete my degree. This, if you are not aware, is a formal document that is allowed to be based on any research question or area you have, so long as it's not copying existing research directly and is approved by your supervisor. This is the single most important document and assignment you are given in a University course. My choice was to focus on the effects of room conditions on students, employees and in home life and how 'Smart Systems' can be used to monitor and control spaces people live or work in dynamically.

Because you're expected to work on this project for a very long time (and it's very time consuming!) it's incredibly important to put a lot of thought into your research topic and ensure it's something you're actually passionate about. If you're writing and creating something following a subject that genuinely interests you, it makes it much less of a drag for you to complete.

## Why this topic?
Low level programming, microcontroller-based computing and edge computing was a keen interest of mine through University, with the modules including such subjects being the most interesting for me by the time I had to decide on a dissertation topic through the summer. This is essentially about finding ways to build systems, both physically and in software to react to live data and settings in a convenient, time efficient and usually a 'set and forget' style.

Moreover, this is a fresh part of the industry which is consistently evolving; especially in the advent of AI, the increasing need to collect data and dynamically needing to adapt to our surroundings.

My strongest area for this kind of development was with ESP32 devices. These are small microcontrollers which are incredibly powerful for their size and price, consisting of but not limited to:

- GPIO Pins
- Onboard WiFi and bluetooth
- A dual core CPU
- Wide software and library support

My decision was to use a similar design to my second year project and improve it with more features and interactability. This would be by not just including raw strings of text, but showing this data within Unreal Engine and Node-RED, dynamically changing and modifying graphs and visualisations based on the values that were being sent over the network.

It sounds complicated, but it was easily achieved by using real world protocols that you likely use on a regular basis!

## So, what did I use?
The choices of hardware and software is rather vast in the scope of ESP32-based devices. I personally chose to use the DHT11 and MQ-135/MQ-2 for this project as they can adequately read the values of a room's state to the level and scope of this project, and were devices I was previously familiar with before - removing the initial struggle of learning the intricacies of learning about these devices and how they work.

The ESP32 microcontroller acts as the hub for all communication between the sensors and the server it is connected to. Arduino IDE was used to program and include all the necessary libraries into my code, mostly because of its convenience with programming and interacting microcontroller devices. C++ code was used to create the programs which were used to collect and send data to Node-RED and Unreal Engine 5 (through JSON to various ports or websockets).

<img src= '../../src/images/image.png' class="center">

This is the full hardware setup (not including the server) that was used to collect data and send it over the network. The components are all stationed on a breadboard so all the sensors can be used at the same time with the power rails distributing power to both sensors at once.

With everything connected, it's time to write some code to begin collecting data from our sensors!

```cpp
void sendSensorData() {
    float temperature = dht.readTemperature();
    float humidity = dht.readHumidity();
    int sensorValue = analogRead(MQ135_PIN);
    float voltage = (sensorValue * voltage_ref) / adc_val;
    float gasConcentration = (voltage / voltage_ref) * 1000;

    if (isnan(temperature) || isnan(humidity)) {
        Serial.println("[ERROR] Cannot read DHT sensor...");
        return;
    }
```

The following code utilises the DHT11 and MQ135 libraries respectively to read the data from the sensors and store them as variables within the program. Once this is read and stored, the data can be sent over to our Node-RED instance in JSON format where it can be visualised and sent over to our Websockets server to be read in Unreal Engine 5.

```cpp
    // Convert JSON object to string
    String jsonString;
    serializeJsonPretty(jsonDoc, jsonString);

    // Send JSON data over WebSocket
    webSocket.sendTXT(jsonString);
    Serial.println("Sent JSON: " + jsonString);
}

void setup() {
    Serial.begin(115200);
    dht.begin();

    // Connect to WiFi
    WiFi.begin(ssid, password);
    Serial.print("Connecting to WiFi");
    while (WiFi.status() != WL_CONNECTED) {
        Serial.print(".");
        delay(500);
    }
    Serial.println("\nWiFi Connected!");
    
    // Connect to WebSocket Server
    // webSocket.begin(ws_server, 8080, "/");
    webSocket.begin(ws_server, 8080, "/sensor");
    webSocket.onEvent(webSocketEvent);
    webSocket.setReconnectInterval(5000);
}

void loop() {
    webSocket.loop();

    // Send real sensor data every 5 seconds
    static unsigned long lastSendTime = 0;
    if (millis() - lastSendTime > 5000) {
        lastSendTime = millis();
        sendSensorData();
    }
```

Now we've successfully collected the data and send it over to Node-RED, it's time to open Node-RED, set it up and see what data it is reading.