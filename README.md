# IoT Project: ESP32 LED with MQTT and Web Server

**Course:**  Desarrollo de Sistemas IoT (IoT systems development)  

**University:**  National University of Colombia  

---
This repo contains a basic IoT sistem where a ESP32-C6 controls a LED (Intern LED) that has color changes every 3 seconds, sends the info trough MQTT to an EC2 AWS server, and a web page shows the color changing on real time.


##  Components

- **ESP32-C6-DevKitC-1 v1.2** with LED RGB WS2812
- **AWS EC2 server** with Ubuntu
- **Broker MQTT Mosquitto** with WebSocket support
- **Apache web server** 
- **Wep page** with real time visualization
  
##  Architecture

```
ESP32-C6 ──(WiFi)──> Internet ──> AWS EC2 Server
    │                                │
    └─ LED RGB WS2812               ├─ Mosquitto MQTT (1883 port)
                                    ├─ WebSocket MQTT (9001 port)
                                    └─ Apache Web (80 port)
                                           └─ Web Page
```

---

# Application

## Part 1: AWS EC2 server

First of all was necessary creat an account on AWS Console ([AWS Console](https://aws.amazon.com/console/)). Then, it was created an instance (EC2) following the guide provide by the teacher. 

![EC2](./img/LED_(1).png)

This instance will be the web server. So, the configuration contains this parameters: 
   - **Name**: `esp-webserver`
   - **AMI**: Ubuntu Server 22.04 LTS (Free tier eligible)
   - **Instance Type**: t2.micro (Free tier eligible)
   - **SSH (22)**: Tu IP (para acceso remoto)
   - **HTTP (80)**: 0.0.0.0/0 (para la página web)
   - **MQTT (1883)**: 0.0.0.0/0 (para el ESP32)
   - **WebSocket (9001)**: 0.0.0.0/0 (para la página web)

Additionally, a `.pem` document is obtained for access via SSH.

That instance is launched and connected. We can see the IP (we use that one for the page web) and the info for the MQTT connection on the AWS page, like this:

![Instance info](./img/LED_(2).png)


# Part 2: Server connection

It's open the path where `.pem` is saved and it ejecutes the following commands

```bash
chmod 400 key-descargada.pem
ssh -i "esp-webserver.pem" ubuntu@ec2-52-15-214-71.us-east-2.compute.amazonaws.com
```

Before connect or try anything else, it's necessary make all the installations and updates:

```bash
sudo apt update
sudo apt install apache2
```

```bash
sudo mkdir /var/www/html/mi-sitio
sudo chown -R Gabriela:Gabriela /var/www/html/mi-sitio
```

### Mosquitto Server. (MQTT)

The MQTT server on Ubuntu is created with Mosquitto. For that, we make the respective installations and configurations:


```bash

sudo apt update
sudo apt install mosquitto mosquitto-clients

```

```bash

sudo systemctl enable mosquitto
sudo systemctl start mosquitto

```
After doing all the necessary configurations, we make a test of the server (before using it with the ESP32). Using two terminals:

Terminal 1:

![Terminal 1](./img/LED_(4).png)

Terminal 2:

![Terminal 2](./img/LED_(5).png)

## Part 3: Configuration of Index.html

For the correct visualization on web, is necessary the following HTML code:

```bash

sudo nano /var/www/html/index.html

```
**Contenido del archivo:**
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>ESP32 LED Monitor</title>
  <style>
    body { font-family: Arial, sans-serif; text-align: center; margin-top: 40px; }
    #circle {
      width: 120px;
      height: 120px;
      border-radius: 50%;
      background: #222;
      margin: 0 auto 20px auto;
      box-shadow: 0 0 20px #888;
      transition: background 0.5s;
    }
    #status {
      margin: 20px 0;
      font-weight: bold;
    }
    .connected { color: green; }
    .disconnected { color: red; }
    .connecting { color: orange; }
  </style>
</head>
<body>
  <h2>ESP32 LED Monitor</h2>
  <div id="circle"></div>
  <div id="status" class="connecting">Conectando a MQTT...</div>
  <div>Color actual: <span id="colorText">---</span></div>
  <div>Mensajes recibidos: <span id="messageCount">0</span></div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.0.1/mqttws31.min.js"></script>
  <script>
    window.onload = function() {
      const brokerHost = "IP_PUBLICA_SERVER"; // CAMBIAR  LA IP
      const brokerPort = 9001;
      const topic = "esp32/led";
      const username = "esp32aby";
      const password = "gaby1405";

      const colorMap = {
        pink: "#ff1493",
        purple: "#800080",
        blue: "#0000ff"
      };

      let messageCount = 0;
      let client = new Paho.MQTT.Client(brokerHost, brokerPort, "webclient_" + Math.random().toString(16).substr(2, 8));

      client.onConnectionLost = function(responseObject) {
        console.log("Conexión perdida: " + responseObject.errorMessage);
        document.getElementById('status').textContent = "Desconectado de MQTT";
        document.getElementById('status').className = "disconnected";
      };

      client.onMessageArrived = function(message) {
        console.log("Mensaje recibido: " + message.payloadString);
        const color = message.payloadString.toLowerCase();
        messageCount++;
        
        document.getElementById('colorText').textContent = color;
        document.getElementById('messageCount').textContent = messageCount;
        document.getElementById('circle').style.background = colorMap[color] || "#222";
      };

      const connectOptions = {
        userName: username,
        password: password,
        timeout: 10,
        keepAliveInterval: 30,
        onSuccess: function() {
          console.log("Conectado exitosamente a MQTT broker");
          document.getElementById('status').textContent = "Conectado a MQTT";
          document.getElementById('status').className = "connected";
          client.subscribe(topic);
          console.log("Suscrito al topic: " + topic);
        },
        onFailure: function(error) {
          console.log("Error de conexión: " + error.errorMessage);
          document.getElementById('status').textContent = "Fallo la conexión MQTT: " + error.errorMessage;
          document.getElementById('status').className = "disconnected";
        }
      };

      console.log("Intentando conectar a: " + brokerHost + ":" + brokerPort);
      client.connect(connectOptions);
    };
  </script>
</body>
</html>
```

With that we have the page- We can visualize it using ```http://52.15.214.71/index.html ```

![WEB](./img/LED_(3).png)

## Part 4: ESP32 Configuration

#### 4.1 Prerequisites

- **ESP-IDF v5.5** installed
- **ESP32-C6-DevKitC-1 v1.2**
- **USB-C Cable**

#### 4.2 Estructure

The project has the following files:

```
main/
├── CMakeLists.txt           # Dependencies configuration
├── idf_component.yml        # Dependency led_strip
├── Kconfig.projbuild        # Project configuration
└── station_example_main.c   # Main code
```


#### 4.4 Build and Flash

On Visual Studio using `ctrl + shift + P` we can compile, build, flash a set a monitor on ESP32 to run all our project:

![esp32RUN](./img/LED_(6).png)

## Results

**Serial Monitor** 

![SerialMonitor](./img/LED_(8).png)

** Physical LED**

![LED](./img/LED.jpg)

at the same time on the web page we can see: 
- **Status**: "Conectado a MQTT"
- **Circle**: Changes its color in sincronization with the physical one.
- **Counter**: Increments with each message
- **Current color**: It shows "pink", "purple", "blue"

---

## System's flow diagram

```
Start
  ↓
NVS Initialization
  ↓
Wifi Connection
  ↓
LED Initialization
  ↓
MQTT Connection
  ↓
LED Task Creation
  ↓
Infinite Loop:
  ├─ Change LED Color (pink, purple,blue)
  ├─ Publish color through MQTT
  ├─ Wait 3 seconds
  ├─ LED Turn Off for a moment
  └─ Repeat with the next color
```
---


