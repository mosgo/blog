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

<img src= '../src/pages/images/image.png' class="center">

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

Now we've successfully collected the data and send it over to Node-RED, it's time to open Node-RED, set it up and see what data it is reading. I've already created a flow which takes all fo the data and formats it to be displayed in a live dashboard (and sent over to the database).

<img src='../src/pages/images/nodes.png' class="center"/>

A post request is first sent to the server containing all of the data collected from the sensors. Once it has been received, it's read and sent to an SQLite server to store the data, a Websocket to communicate to Unreal Engine 5 and a dashboard to visualise the data.

We need to prepare a JSON buffer first to send over to our Node-RED server, which is shown below:
```js
  HTTPClient http;

  Serial.print("Connecting to website: ");
  http.begin("http://192.168.8.172:1880/dht");  //HTTP connection, this could be any IP
  
  sprintf(buffer, "{\"temp\":%5.2f,\"humidity\":%5.2f}", t, h);
  int httpCode = http.POST(buffer);
  if (httpCode > 0) {
    Serial.printf("[HTTP] POST... code: %d\n", httpCode);
    if (httpCode == HTTP_CODE_OK) {
      String payload = http.getString();
      Serial.println(payload);
    }
  } else {
    Serial.printf("[HTTP] POST... failed, error: %s\n", http.errorToString(httpCode).c_str());
  }
  http.end();
}
```

<img src='../src/pages/images/request.png' class="center"/>

This data is formatted using JSON and is sent over as an object containing the data stored in various properties. This data is later used to differentiate the data so it can be visualised in various columns.

<img src='../src/pages/images/objectImage.png' class="center02"/>

Once this is sent over to Node-RED, it's basically stored in memory until the server shuts down or overwrites that data with the next POST request that is sent over by the ESP32. This data is then visualised in a dashboard which is constantly updated every time it detects a change in values from the server. It's split into different columns and dynamically changes based on the resolution that it is being displayed on.

<img src='../src/pages/images/dashboard.png' class="center"/>

This is the dashboard after running for just a few minutes. As you can see, the graph dynamically changes in size based on the history of data collected, and a few gauges represent the data now and how acceptable it is. The 'advice' section is merely an if/else statement that changes based on the value it reads. 

For anyone wishing to view the data outside of the dashboard, the information is accessible from outside of the dashboard from an MQTT server. MQTT as a standardised publish/subscribe push protocol that was released by IBM in 1999 and was initially designed to send data accurately under large network delays or low bandwidth connections.  Its base design defines the following structure:

<img src='../src/pages/images/mqtt.png' class="center"/>

This can even be accessed from a mobile device on the same network, which vastly expands the applications the data can be used in.

Finally, the data is represented in a virtual space using Unreal Engine 5. This is achieved by using Websockets and the game reading and presenting this data within a string from a C++ class. C++ was used because this kind of object cannot be created from regular blueprints, and thus required the further access that writing in raw C++ provides. First, we need to find a way to send the data over to the Unreal Engine game. There's many ways of achieving this, but in the interest of simplicity and time I chose Websockets, mostly because my deadline was approaching!

```cpp
void AWebSocketActor::BeginPlay() {
    Super::BeginPlay();

    if (!FModuleManager::Get().IsModuleLoaded("WebSockets")) {
        FModuleManager::Get().LoadModule("WebSockets");
    }

    //WebSocket = FWebSocketsModule::Get().CreateWebSocket(TEXT("ws://192.168.1.109:8080")); // match local machine
    WebSocket = FWebSocketsModule::Get().CreateWebSocket(TEXT("ws://192.168.1.109:1880/ue5"));

    // Server connection handers
    WebSocket->OnConnected().AddUObject(this, &AWebSocketActor::OnConnected);
    WebSocket->OnMessage().AddUObject(this, &AWebSocketActor::OnMessageReceived);
    WebSocket->OnConnectionError().AddUObject(this, &AWebSocketActor::OnConnectionError);
    WebSocket->OnClosed().AddUObject(this, &AWebSocketActor::OnClosed);

    // Connect UE5 proj to Node websocket server
    WebSocket->Connect();
}
```

```cpp
void AWebSocketActor::OnConnected() {
    UE_LOG(LogTemp, Warning, TEXT("Connected to websocket"));
    if (TextRenderComponent) {
    TextRenderComponent->SetText(FText::FromString("Connected to websocket"));
    }
}

void AWebSocketActor::OnMessageReceived(const FString& Message) {
    UE_LOG(LogTemp, Warning, TEXT("New message: %s"), *Message);
    if (TextRenderComponent) {
    TextRenderComponent->SetText(FText::FromString(Message));
    }
}

void AWebSocketActor::OnConnectionError(const FString& Error) {
    UE_LOG(LogTemp, Error, TEXT("WebSocket Error: %s"), *Error);
    if (TextRenderComponent) {
    TextRenderComponent->SetText(FText::FromString("WebSocket Error!"));
    }
}

void AWebSocketActor::OnClosed(int32 StatusCode, const FString& Reason, bool bWasClean) {
    UE_LOG(LogTemp, Warning, TEXT("WebSocket Closed!"));
    if (TextRenderComponent) {
        TextRenderComponent->SetText(FText::FromString("WebSocket Disconnected"));
    }
}
```

This code informs the user when the game has had a Websocket connection, and once it has it prints the incoming string into the game, presenting it for the player in the game. This is presented using a TextObject in the game that changes based on what is reads from the incoming Websocket data.

<img src='../src/pages/images/ingameweb.png' class="center"/>


<h1>What didn't work?</h1>

There were a few things I was wanting this project to achieve, but I really just didn't have the time to focus on them. For example, the Unreal level would dynamically change based on the air quality in the real world representing rooms. I also wanted to build a system that would advise the user what to do to improve the quality of the room; but I was only able to build a basic concept instead of developing something that could really be used in industry.

A few things did work fine though. SQL was a very functional and easy way to store data that would've otherwise been lost once the server was shutdown (because the data would've been deleted from memory). There weren't really many issues with it by the end, but I was left with the impression that a lot more could be done.

<br/>
<h1>So what did I learn?</h1>

Through this project, I learned that it's indeed possible to setup and create a system that monitors air quality and humidity at a very affordable price (under £10!). This project as able to collect constructive data and displayed the relevant benefits and flaws of using such a system. It provides good quality data, but nothing accurate enough to be used in industry. Networking capabilities and power use is generally low from this project, meaning that because of its low bandwidth it can run in various places, such as a campus or a business building at scale without hindering the existing network or compromising security.  Using the UI, it’s entirely possible for managers and users to accurately understand and benefit from reading real-time temperature, gas concentration and humidity data from a building.

If I were to do this again, I'd focus on what it could do. For example, this system can only monitor one space, but it could quite easily monitor multiple spaces at once and split them into different categories based on the room it is in. Using modern AI tools, it could also be possible to suggest to managers what could be improved or changed in a room to improve the room air quality. It has quite a lot of potential, but because of the time I had to work on this project, half of my ambitions were not met. Despite all of this, I'm really happy with how it all turned out and I'll probably return to this project once I have the time!