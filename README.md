# ESP32 micro-ROS Car 🤖🪶
Wi-Fi controlled ESP32 robot using **micro-ROS**, **ROS 2 Humble**, and **teleop_twist_keyboard**

![Image](https://github.com/user-attachments/assets/0a0e2b0c-90f1-459f-8495-9b509c9c90b8)

---

## 🚀 Project Overview
- A small ESP32 robot car project that connects to ROS 2 using micro-ROS.  
- The car can be controlled via keyboard teleoperation and streams live video using a phone IP camera app connected over the same Wi-Fi network.

---

## ✨ Features
- Keyboard teleoperation  
- Live camera feed (viewable in rqt or browser)  
- ROS 2 integration with micro-ROS agent  
- Easily extendable for sensors or autonomous features  

---

## 🧩 Hardware Components
- ESP32
- L298N motor driver  
- 2-wheel drive chassis  
- Power supply / battery  

---

## 🧠 Software Requirements
- ROS 2 Humble (or newer)  
- micro-ROS Agent package  
- teleop_twist_keyboard  
- Arduino IDE or PlatformIO  
- rqt / RViz (for visualization)
- IPcam(for visuals) installed in phone

---

## 🧠 Arduino Code Working️

- The ESP32 runs a simple **micro-ROS client** that subscribes to velocity commands (`cmd_vel`) and from the ROS 2 system and controls the car’s motors accordingly.
- The micro-Ros client also subscribe's to (`geometry_msgs/msg/twist`) used to represent velocity commands for a robot — both linear (forward/backward motion) and angular (turning).



### 🔍 Overview
- Subscribes to `/cmd_vel` topic  
- Receives velocity commands from `teleop_twist_keyboard`  
- Controls motor driver pins using PWM  
- Communicates with ROS 2 via micro-ROS Agent over Wi-Fi  

### ⚙️ Arduino Code
```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <micro_ros_arduino.h>
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <geometry_msgs/msg/twist.h>

// ==== MOTOR PINS ====
#define IN1 26
#define IN2 25
#define IN3 14
#define IN4 27
#define ENA 32
#define ENB 33

// ==== Wi-Fi Credentials ====
const char* ssid = "Arakkis";
const char* password = "Narasimha1@god";
const char* agent_ip = "192.168.31.88";  // micro-ROS Agent
const int agent_port = 8888;

// ==== micro-ROS Setup ====
rcl_subscription_t subscriber;
rcl_node_t node;
rclc_executor_t executor;
rcl_allocator_t allocator;
rclc_support_t support;
geometry_msgs__msg__Twist msg;

// ==== Speed ====
int baseSpeed = 255;

// ==== Web Server ====
WebServer server(80);

// ==== Phone IP Camera URL ====
String cam_url = "http://192.168.31.185:8080/video";  // <— your phone IP

// ==== Motor Functions ====
void moveForward() {
  analogWrite(ENA, baseSpeed);
  analogWrite(ENB, baseSpeed);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);//'http://192.168.31.127:8080/video'
  digitalWrite(IN4, HIGH);
}

void moveBackward() {
  analogWrite(ENA, baseSpeed);
  analogWrite(ENB, baseSpeed);
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void turnLeft() {
  analogWrite(ENA, baseSpeed / 2);
  analogWrite(ENB, baseSpeed);
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void turnRight() {
  analogWrite(ENA, baseSpeed);
  analogWrite(ENB, baseSpeed / 2);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void stopCar() {
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

// ==== ROS Callback ====
void twist_callback(const void* msgin) {
  const geometry_msgs__msg__Twist* twist = (const geometry_msgs__msg__Twist*)msgin;
  float linear = twist->linear.x;
  float angular = twist->angular.z;

  if (linear > 0.1) moveForward();
  else if (linear < -0.1) moveBackward();
  else if (angular > 0.2) turnLeft();
  else if (angular < -0.2) turnRight();
  else stopCar();
}

// ==== Web Page ====
const char htmlPage[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <title>ESP32 2WD Car + IP Cam</title>
  <style>
    body {
      background-color: #f2f2f2;
      text-align: center;
      font-family: Arial;
    }
    img {
      border-radius: 10px;
      width: 320px;
      height: 240px;
      transform: rotate(0deg);
    }
    button {
      width: 100px;
      height: 40px;
      margin: 5px;
      background-color: #007bff;
      color: white;
      font-size: 16px;
      border: none;
      border-radius: 8px;
    }
    button:hover { background-color: #0056b3; }
  </style>
</head>
<body>
  <h2>ESP32 2WD Car Control</h2>
  <img src="%CAM_URL%" alt="Camera Feed"><br>
  <div>
    <button onclick="fetch('/forward')">Forward</button>
    <button onclick="fetch('/backward')">Backward</button><br>
    <button onclick="fetch('/left')">Left</button>
    <button onclick="fetch('/right')">Right</button><br>
    <button onclick="fetch('/stop')">Stop</button>
  </div>
</body>
</html>
)rawliteral";

// ==== Web Handlers ====
void handleRoot() {
  String page = htmlPage;
  page.replace("%CAM_URL%", cam_url);
  server.send(200, "text/html", page);
}
void handleForward() { moveForward(); server.send(200, "text/plain", "Forward"); }
void handleBackward() { moveBackward(); server.send(200, "text/plain", "Backward"); }
void handleLeft() { turnLeft(); server.send(200, "text/plain", "Left"); }
void handleRight() { turnRight(); server.send(200, "text/plain", "Right"); }
void handleStop() { stopCar(); server.send(200, "text/plain", "Stop"); }

// ==== Setup ====
void setup() {
  Serial.begin(115200);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);

  WiFi.begin(ssid, password);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected!");
  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  // Web server routes
  server.on("/", handleRoot);
  server.on("/forward", handleForward);
  server.on("/backward", handleBackward);
  server.on("/left", handleLeft);
  server.on("/right", handleRight);
  server.on("/stop", handleStop);
  server.begin();
  Serial.println("Web server started.");

  // micro-ROS setup
  set_microros_wifi_transports("Arakkis", "Narasimha1@god", "192.168.31.88", 8888);

Serial.println("micro-ROS transport initialized");

  delay(2000);
  allocator = rcl_get_default_allocator();
  rclc_support_init(&support, 0, NULL, &allocator);
  rclc_node_init_default(&node, "esp32_car_node", "", &support);
  rclc_subscription_init_default(
      &subscriber,
      &node,
      ROSIDL_GET_MSG_TYPE_SUPPORT(geometry_msgs, msg, Twist),
      "/cmd_vel");
  rclc_executor_init(&executor, &support.context, 1, &allocator);
  rclc_executor_add_subscription(&executor, &subscriber, &msg, &twist_callback, ON_NEW_DATA);
  Serial.println("micro-ROS ready!");
}

// ==== Loop ====
void loop() {
  server.handleClient();
  rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100));
}

```
## 🖥️ ROS 2 Terminal Preview

Here’s a preview of the ROS 2 nodes, teleop, and camera feed running on the ESP32 robot:

![Image](https://github.com/user-attachments/assets/dfd9d6f1-84a2-470f-9e95-18b915ee5649)







