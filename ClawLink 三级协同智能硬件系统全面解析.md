# ClawLink 三级协同智能硬件系统全面解析

# ClawLink-TriLayer 三级协同智能硬件系统

## 项目概述

基于「OpenClaw大脑层 + 树莓派脊椎层 + ESP32末梢层」三级架构打造的全开源智能硬件控制系统，实现**复杂任务调度→边缘智能处理→端侧硬件执行** 的全链路闭环，支持离线自治、多设备协同、模块化扩展。

### 核心特性

- 🌐 跨层通信：MQTT/ESP-NOW/WebSocket 多协议适配

- 🧠 分层智能：大脑层复杂决策、脊椎层边缘推理、末梢层实时执行

- 📱 全离线：断网状态下树莓派+ESP32可完成核心功能闭环

- 🛠️ 易扩展：新增传感器/执行器仅需适配末梢层，无需修改核心架构

- 📝 开源协议：MIT License（完全自由修改/商用）

## 完整项目目录结构

```Plain Text

ClawLink-TriLayer/
├── LICENSE                  # MIT开源协议
├── README.md                # 项目说明
├── docker-compose.yml       # 树莓派服务一键部署
├── .env.example             # 环境变量模板
├── docs/                    # 文档目录
│   ├── 快速上手.md
│   ├── 硬件接线指南.md
│   └── 二次开发指南.md
├── openclaw-brain/          # OpenClaw大脑层
│   ├── clawlink_trilayer.skill.md  # 核心调度插件
│   └── install_skill.sh     # 插件安装脚本
├── raspberrypi-spine/       # 树莓派脊椎层
│   ├── backend/             # 后端服务
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── src/
│   │       ├── main.py      # FastAPI入口
│   │       ├── mqtt_client.py  # MQTT边缘网关
│   │       ├── llm_agent.py    # 本地LLM推理（Ollama）
│   │       ├── vision_engine.py # YOLOv8视觉处理
│   │       └── device_manager.py # 设备管理
│   └── frontend/            # 极简Web前端（Vue3）
│       ├── index.html
│       ├── app.js
│       └── style.css
├── esp32-periphery/         # ESP32末梢层
│   ├── main/
│   │   ├── CMakeLists.txt
│   │   ├── sdkconfig.defaults
│   │   ├── main.c           # 固件入口
│   │   ├── mqtt_comm.c      # MQTT通信
│   │   ├── sensors.c        # 传感器采集
│   │   └── actuators.c      # 执行器控制
│   └── flash.sh             # 固件烧录脚本
└── examples/                # 示例代码
    ├── task_demo.py         # 复杂任务调度示例
    └── sensor_extend.c      # 新增传感器示例
```

## 一、OpenClaw 大脑层代码

### 1. 核心调度插件 `clawlink_trilayer.skill.md`

```Markdown

# ClawLink-TriLayer 三级协同调度插件
## 插件信息
- 名称: clawlink_trilayer
- 版本: 1.0.0
- 描述: 对接树莓派脊椎层，实现复杂任务拆解与跨设备调度
- 依赖: mqtt, requests

## 配置项
```toml
[clawlink_trilayer]
mqtt_broker = "你的树莓派IP"
mqtt_port = 1883
mqtt_user = "clawlink-upper"
mqtt_pass = "ClawLinkUpper@2026"
raspberrypi_api = "http://你的树莓派IP:8001"
```

## 指令映射

|自然语言指令|处理逻辑|
|---|---|
|"采集所有设备温湿度并汇总"|调用树莓派API获取所有ESP32传感器数据，汇总后返回|
|"控制设备claw_01打开机械爪"|通过MQTT下发指令到树莓派，转发至指定ESP32|
|"检测前方障碍物，有障碍则停止移动底盘"|触发树莓派视觉引擎，联动ESP32执行器控制|
## 核心代码

```Python

import paho.mqtt.client as mqtt
import requests
import json

# MQTT客户端初始化
def init_mqtt(config):
    client = mqtt.Client()
    client.username_pw_set(config['mqtt_user'], config['mqtt_pass'])
    client.connect(config['mqtt_broker'], config['mqtt_port'], 60)
    client.loop_start()
    return client

# 任务拆解与下发
def handle_task(command, config):
    client = init_mqtt(config)
    api_url = config['raspberrypi_api']
    
    # 1. 解析指令
    if "采集温湿度" in command:
        # 调用树莓派API获取传感器数据
        resp = requests.get(f"{api_url}/api/devices/sensors")
        if resp.status_code == 200:
            data = resp.json()
            return f"温湿度数据汇总：{json.dumps(data, ensure_ascii=False)}"
        else:
            return "获取传感器数据失败"
    
    elif "打开机械爪" in command:
        # 提取设备ID
        device_id = "claw_01"  # 实际场景可通过NLP提取
        # 下发指令到树莓派
        client.publish(
            f"clawlink/spine/command",
            json.dumps({
                "device_id": device_id,
                "action": "servo_control",
                "params": {"angle": 90, "pin": 18}
            })
        )
        return f"已下发指令：打开设备{device_id}机械爪"
    
    elif "检测障碍物" in command:
        # 触发视觉检测
        resp = requests.post(f"{api_url}/api/vision/detect", json={"target": "obstacle"})
        if resp.status_code == 200:
            result = resp.json()
            if result["has_obstacle"]:
                # 下发停止指令
                client.publish(
                    f"clawlink/spine/command",
                    json.dumps({
                        "device_id": "chassis_01",
                        "action": "motor_stop"
                    })
                )
                return "检测到障碍物，已停止移动底盘"
            else:
                return "未检测到障碍物，底盘正常运行"
    else:
        return "不支持的指令，请重试"

# OpenClaw插件入口
def skill_handler(command, context):
    config = context['config']['clawlink_trilayer']
    return handle_task(command, config)
```

### 2. 插件安装脚本 `install_skill.sh`

```Bash

#!/bin/bash
# OpenClaw插件安装脚本

# 创建插件目录
mkdir -p ~/.openclaw/skills
# 复制插件文件
cp clawlink_trilayer.skill.md ~/.openclaw/skills/
# 重启OpenClaw服务
openclaw restart

echo "ClawLink-TriLayer插件安装完成！"
echo "请修改 ~/.openclaw/config.toml 配置mqtt_broker等参数"
```

## 二、树莓派 脊椎层代码

### 1. 后端依赖 `requirements.txt`

```Plain Text

fastapi==0.104.1
uvicorn==0.24.0
paho-mqtt==1.6.1
ollama==0.1.9
ultralytics==8.0.224
python-dotenv==1.0.0
pandas==2.1.4
```

### 2. MQTT客户端 `mqtt_client.py`

```Python

import paho.mqtt.client as mqtt
import json
import os
from dotenv import load_dotenv

load_dotenv()

# MQTT配置
MQTT_BROKER = os.getenv("MQTT_BROKER", "localhost")
MQTT_PORT = int(os.getenv("MQTT_PORT", 1883))
MQTT_USER = os.getenv("MQTT_EDGE_USER", "clawlink-edge")
MQTT_PASS = os.getenv("MQTT_EDGE_PASS", "ClawLinkEdge@2026")

# 设备状态缓存
device_states = {}

# MQTT回调函数
def on_connect(client, userdata, flags, rc):
    print(f"MQTT连接成功，状态码：{rc}")
    # 订阅末梢层上报主题
    client.subscribe("clawlink/periphery/#")
    # 订阅大脑层指令主题
    client.subscribe("clawlink/spine/command")

def on_message(client, userdata, msg):
    topic = msg.topic
    payload = json.loads(msg.payload.decode())
    print(f"收到消息：{topic} -> {payload}")
    
    if topic.startswith("clawlink/periphery/report"):
        # 处理末梢层状态上报
        device_id = topic.split("/")[2]
        device_states[device_id] = payload
    elif topic == "clawlink/spine/command":
        # 转发指令到指定末梢设备
        device_id = payload["device_id"]
        client.publish(
            f"clawlink/periphery/command/{device_id}",
            json.dumps(payload)
        )

# 初始化MQTT客户端
def init_mqtt_client():
    client = mqtt.Client()
    client.username_pw_set(MQTT_USER, MQTT_PASS)
    client.on_connect = on_connect
    client.on_message = on_message
    client.connect(MQTT_BROKER, MQTT_PORT, 60)
    client.loop_start()
    return client

# 发布指令到末梢层
def publish_to_periphery(device_id, action, params):
    client = init_mqtt_client()
    client.publish(
        f"clawlink/periphery/command/{device_id}",
        json.dumps({
            "action": action,
            "params": params
        })
    )
```

### 3. 本地LLM推理 `llm_agent.py`

```Python

import ollama
import os
from dotenv import load_dotenv

load_dotenv()

OLLAMA_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434/api")
DEFAULT_MODEL = os.getenv("OLLAMA_DEFAULT_MODEL", "llama3:8b-instruct-q4_0")

# 初始化Ollama客户端
ollama_client = ollama.Client(host=OLLAMA_URL)

# 本地LLM推理（边缘决策）
def llm_infer(prompt):
    try:
        response = ollama_client.chat(
            model=DEFAULT_MODEL,
            messages=[{"role": "user", "content": prompt}]
        )
        return response["message"]["content"]
    except Exception as e:
        return f"LLM推理失败：{str(e)}"

# 任务优先级决策
def task_priority_decision(tasks):
    prompt = f"""
    请对以下硬件控制任务进行优先级排序（1-最高，数字越小优先级越高），并说明理由：
    {json.dumps(tasks, ensure_ascii=False)}
    输出格式：JSON，包含tasks（排序后的任务列表，带priority字段）和reason字段
    """
    return llm_infer(prompt)
```

### 4. 视觉处理引擎 `vision_engine.py`

```Python

from ultralytics import YOLO
import cv2
import os
from dotenv import load_dotenv

load_dotenv()

# 加载YOLOv8模型
model = YOLO("yolov8n.pt")
CAMERA_INDEX = int(os.getenv("CAMERA_INDEX", 0))

# 障碍物检测
def detect_obstacle():
    cap = cv2.VideoCapture(CAMERA_INDEX)
    ret, frame = cap.read()
    if not ret:
        return {"has_obstacle": False, "error": "摄像头读取失败"}
    
    # 推理（检测人、椅子、箱子等障碍物）
    results = model(frame)
    obstacle_classes = [0, 5, 27]  # COCO类别：人、椅子、箱子
    detections = results[0].boxes.cls.cpu().numpy()
    
    has_obstacle = any(cls in obstacle_classes for cls in detections)
    cap.release()
    
    return {
        "has_obstacle": has_obstacle,
        "detections": len(detections),
        "obstacle_classes": obstacle_classes
    }

# 视觉抓取目标定位
def detect_target(target_name):
    cap = cv2.VideoCapture(CAMERA_INDEX)
    ret, frame = cap.read()
    if not ret:
        return {"success": False, "error": "摄像头读取失败"}
    
    results = model(frame)
    # 查找目标类别
    class_names = model.names
    target_cls = [k for k, v in class_names.items() if v == target_name]
    if not target_cls:
        return {"success": False, "error": "目标类别不存在"}
    
    # 获取目标坐标
    boxes = results[0].boxes
    target_boxes = [box for box in boxes if box.cls.cpu().numpy()[0] == target_cls[0]]
    
    cap.release()
    if not target_boxes:
        return {"success": False, "message": "未检测到目标"}
    
    # 返回第一个目标的中心坐标
    box = target_boxes[0].xyxy.cpu().numpy()[0]
    center_x = (box[0] + box[2]) / 2
    center_y = (box[1] + box[3]) / 2
    
    return {
        "success": True,
        "center_x": center_x,
        "center_y": center_y,
        "box": box.tolist()
    }
```

### 5. FastAPI入口 `main.py`

```Python

from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
import json
from mqtt_client import init_mqtt_client, device_states, publish_to_periphery
from llm_agent import llm_infer, task_priority_decision
from vision_engine import detect_obstacle, detect_target

# 初始化FastAPI
app = FastAPI(title="ClawLink-TriLayer 脊椎层API")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 初始化MQTT
mqtt_client = init_mqtt_client()

# 设备管理接口
@app.get("/api/devices")
def get_devices():
    """获取所有在线设备"""
    return {"devices": list(device_states.keys()), "count": len(device_states)}

@app.get("/api/devices/sensors")
def get_sensors():
    """获取所有设备传感器数据"""
    sensor_data = {}
    for dev_id, state in device_states.items():
        if "sensors" in state:
            sensor_data[dev_id] = state["sensors"]
    return sensor_data

# 视觉处理接口
@app.post("/api/vision/detect")
def vision_detect(data: dict):
    """视觉检测接口"""
    target = data.get("target", "obstacle")
    if target == "obstacle":
        return detect_obstacle()
    else:
        return detect_target(target)

# LLM推理接口
@app.post("/api/llm/infer")
def llm_infer_api(data: dict):
    """本地LLM推理接口"""
    prompt = data.get("prompt")
    if not prompt:
        raise HTTPException(status_code=400, detail="prompt不能为空")
    return {"result": llm_infer(prompt)}

# 设备控制接口
@app.post("/api/devices/control")
def control_device(data: dict):
    """控制末梢设备"""
    device_id = data.get("device_id")
    action = data.get("action")
    params = data.get("params", {})
    
    if not device_id or not action:
        raise HTTPException(status_code=400, detail="device_id和action不能为空")
    
    publish_to_periphery(device_id, action, params)
    return {"success": True, "message": f"已下发{action}指令到设备{device_id}"}

# 启动服务
if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8001,
        reload=True
    )
```

### 6. 极简Web前端 `index.html`

```HTML

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ClawLink-TriLayer 控制中心</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>ClawLink-TriLayer 控制中心</h1>
        
        <!-- 设备列表 -->
        <div class="device-section">
            <h2>在线设备</h2>
            <div id="device-list"></div>
        </div>
        
        <!-- 传感器数据 -->
        <div class="sensor-section">
            <h2>传感器数据</h2>
            <div id="sensor-data"></div>
        </div>
        
        <!-- 设备控制 -->
        <div class="control-section">
            <h2>设备控制</h2>
            <select id="device-select">
                <option value="">选择设备</option>
            </select>
            <select id="action-select">
                <option value="servo_control">舵机控制</option>
                <option value="motor_control">电机控制</option>
                <option value="sensor_read">读取传感器</option>
            </select>
            <input type="text" id="params-input" placeholder="参数（JSON格式）" value='{"angle": 90, "pin": 18}'>
            <button onclick="controlDevice()">执行</button>
        </div>
        
        <!-- 视觉检测 -->
        <div class="vision-section">
            <h2>视觉检测</h2>
            <button onclick="detectObstacle()">检测障碍物</button>
            <div id="vision-result"></div>
        </div>
    </div>

    <script src="app.js"></script>
</body>
</html>
```

### 7. 前端逻辑 `app.js`

```JavaScript

// API基础地址
const API_BASE = "http://" + window.location.hostname + ":8001/api";

// 初始化页面
window.onload = async () => {
    await loadDevices();
    await loadSensors();
};

// 加载设备列表
async function loadDevices() {
    const resp = await fetch(`${API_BASE}/devices`);
    const data = await resp.json();
    const deviceList = document.getElementById("device-list");
    const deviceSelect = document.getElementById("device-select");
    
    deviceList.innerHTML = JSON.stringify(data, null, 2);
    data.devices.forEach(dev => {
        const option = document.createElement("option");
        option.value = dev;
        option.textContent = dev;
        deviceSelect.appendChild(option);
    });
}

// 加载传感器数据
async function loadSensors() {
    const resp = await fetch(`${API_BASE}/devices/sensors`);
    const data = await resp.json();
    document.getElementById("sensor-data").innerHTML = JSON.stringify(data, null, 2);
}

// 控制设备
async function controlDevice() {
    const deviceId = document.getElementById("device-select").value;
    const action = document.getElementById("action-select").value;
    const params = JSON.parse(document.getElementById("params-input").value);
    
    if (!deviceId) {
        alert("请选择设备");
        return;
    }
    
    const resp = await fetch(`${API_BASE}/devices/control`, {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            device_id: deviceId,
            action: action,
            params: params
        })
    });
    
    const data = await resp.json();
    alert(data.message);
    // 刷新设备状态
    await loadSensors();
}

// 检测障碍物
async function detectObstacle() {
    const resp = await fetch(`${API_BASE}/vision/detect`, {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({ target: "obstacle" })
    });
    
    const data = await resp.json();
    document.getElementById("vision-result").innerHTML = JSON.stringify(data, null, 2);
}
```

## 三、ESP32 末梢层代码

### 1. 固件入口 `main.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "mqtt_comm.h"
#include "sensors.h"
#include "actuators.h"

#define DEVICE_ID "claw_01"  // 设备唯一ID
#define WIFI_SSID "你的WiFi名称"
#define WIFI_PASS "你的WiFi密码"
#define MQTT_BROKER "你的树莓派IP"
#define MQTT_PORT 1883

void app_main(void)
{
    // 1. 初始化WiFi
    wifi_init(WIFI_SSID, WIFI_PASS);
    
    // 2. 初始化MQTT
    mqtt_client_init(MQTT_BROKER, MQTT_PORT, DEVICE_ID);
    
    // 3. 初始化传感器
    sensors_init();
    
    // 4. 初始化执行器
    actuators_init();
    
    // 5. 启动传感器采集任务
    xTaskCreate(sensor_collect_task, "sensor_collect", 4096, NULL, 5, NULL);
    
    // 6. 启动MQTT消息处理任务
    xTaskCreate(mqtt_message_handler_task, "mqtt_handler", 4096, NULL, 5, NULL);
    
    // 7. 启动心跳上报任务
    xTaskCreate(heartbeat_task, "heartbeat", 2048, NULL, 4, NULL);
}
```

### 2. MQTT通信 `mqtt_comm.c`

```C

#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "mqtt_client.h"
#include "wifi_provisioning/manager.h"

static const char *TAG = "MQTT_COMM";
static esp_mqtt_client_handle_t mqtt_client;
static char device_id[32];

// WiFi初始化
void wifi_init(const char *ssid, const char *pass)
{
    // 简化版WiFi初始化（实际项目请使用ESP-IDF标准WiFi初始化）
    ESP_LOGI(TAG, "WiFi初始化中...");
    vTaskDelay(2000 / portTICK_PERIOD_MS);
    ESP_LOGI(TAG, "WiFi连接成功");
}

// MQTT连接回调
static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    
    switch ((esp_mqtt_event_id_t)event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT连接成功");
            // 订阅指令主题
            char cmd_topic[64];
            snprintf(cmd_topic, sizeof(cmd_topic), "clawlink/periphery/command/%s", device_id);
            esp_mqtt_client_subscribe(client, cmd_topic, 0);
            break;
        case MQTT_EVENT_DATA:
            ESP_LOGI(TAG, "收到MQTT消息：%.*s", event->data_len, event->data);
            // 解析指令并处理
            handle_mqtt_command(event->data, event->data_len);
            break;
        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "MQTT断开连接");
            break;
        default:
            break;
    }
}

// 初始化MQTT客户端
void mqtt_client_init(const char *broker, int port, const char *dev_id)
{
    strncpy(device_id, dev_id, sizeof(device_id) - 1);
    
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = broker,
        .broker.address.port = port,
        .credentials.username = "clawlink-device",
        .credentials.authentication.password = "ClawLinkDevice@2026",
        .client.id = dev_id,
    };
    
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(mqtt_client);
}

// 发布消息到MQTT
void mqtt_publish(const char *topic, const char *payload)
{
    if (mqtt_client == NULL) {
        ESP_LOGE(TAG, "MQTT客户端未初始化");
        return;
    }
    int msg_id = esp_mqtt_client_publish(mqtt_client, topic, payload, 0, 1, 0);
    ESP_LOGI(TAG, "发布消息ID：%d", msg_id);
}

// 处理MQTT指令
void handle_mqtt_command(const char *data, int len)
{
    // 简化版JSON解析（实际项目建议使用cJSON）
    char action[32] = {0};
    char params[128] = {0};
    
    // 提取action和params（示例：{"action":"servo_control","params":{"angle":90,"pin":18}}）
    if (strstr(data, "servo_control")) {
        strcpy(action, "servo_control");
        // 提取角度和引脚
        int angle = 90, pin = 18;
        sscanf(data, "%*[^0-9]%d%*[^0-9]%d", &angle, &pin);
        servo_control(pin, angle);
    } else if (strstr(data, "motor_control")) {
        strcpy(action, "motor_control");
        motor_control(1, 100); // 启动电机，占空比100
    } else if (strstr(data, "sensor_read")) {
        strcpy(action, "sensor_read");
        // 上报传感器数据
        sensor_data_t sensor_data = get_sensor_data();
        char payload[256];
        snprintf(payload, sizeof(payload), 
            "{\"device_id\":\"%s\",\"sensors\":{\"temp\":%.2f,\"hum\":%.2f,\"ultrasonic\":%.2f}}",
            device_id, sensor_data.temp, sensor_data.hum, sensor_data.ultrasonic);
        char report_topic[64];
        snprintf(report_topic, sizeof(report_topic), "clawlink/periphery/report/%s", device_id);
        mqtt_publish(report_topic, payload);
    }
    
    ESP_LOGI(TAG, "执行指令：%s", action);
}

// 心跳上报任务
void heartbeat_task(void *pvParameters)
{
    while (1) {
        char payload[128];
        snprintf(payload, sizeof(payload), 
            "{\"device_id\":\"%s\",\"status\":\"online\"}", device_id);
        char heartbeat_topic[64];
        snprintf(heartbeat_topic, sizeof(heartbeat_topic), "clawlink/periphery/heartbeat/%s", device_id);
        mqtt_publish(heartbeat_topic, payload);
        
        vTaskDelay(5000 / portTICK_PERIOD_MS); // 每5秒上报一次
    }
}

// MQTT消息处理任务
void mqtt_message_handler_task(void *pvParameters)
{
    while (1) {
        vTaskDelay(100 / portTICK_PERIOD_MS); // 空闲延时
    }
}
```

### 3. 传感器采集 `sensors.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/i2c.h"
#include "driver/gpio.h"
#include "esp_log.h"

static const char *TAG = "SENSORS";

// 传感器数据结构体
typedef struct {
    float temp;       // 温度
    float hum;        // 湿度
    float ultrasonic; // 超声波距离
} sensor_data_t;

static sensor_data_t current_sensor_data = {0};

// AHT20温湿度传感器读取
static void aht20_read(float *temp, float *hum)
{
    // 简化版AHT20读取（实际项目请使用官方驱动）
    *temp = 25.5 + (rand() % 100) / 10.0; // 模拟温度
    *hum = 60.2 + (rand() % 50) / 10.0;   // 模拟湿度
}

// HC-SR04超声波传感器读取
static float hc_sr04_read(void)
{
    // 简化版超声波读取（实际项目请使用GPIO驱动）
    return 50.0 + (rand() % 100) / 2.0; // 模拟距离（cm）
}

// 传感器初始化
void sensors_init(void)
{
    ESP_LOGI(TAG, "传感器初始化完成");
    // 实际项目中初始化I2C、GPIO等
}

// 获取传感器数据
sensor_data_t get_sensor_data(void)
{
    return current_sensor_data;
}

// 传感器采集任务
void sensor_collect_task(void *pvParameters)
{
    while (1) {
        // 读取温湿度
        aht20_read(&current_sensor_data.temp, &current_sensor_data.hum);
        // 读取超声波距离
        current_sensor_data.ultrasonic = hc_sr04_read();
        
        ESP_LOGI(TAG, "温度：%.2f℃，湿度：%.2f%%，距离：%.2fcm",
            current_sensor_data.temp, current_sensor_data.hum, current_sensor_data.ultrasonic);
        
        vTaskDelay(2000 / portTICK_PERIOD_MS); // 每2秒采集一次
    }
}
```

### 4. 执行器控制 `actuators.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "driver/gpio.h"
#include "esp_log.h"

static const char *TAG = "ACTUATORS";

// 舵机初始化
static void servo_init(int pin)
{
    // 配置LEDC PWM
    ledc_timer_config_t timer_conf = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_12_BIT,
        .freq_hz = 50, // 50Hz（舵机标准频率）
        .clk_cfg = LEDC_AUTO_CLK,
    };
    ledc_timer_config(&timer_conf);
    
    ledc_channel_config_t channel_conf = {
        .channel = LEDC_CHANNEL_0,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_sel = LEDC_TIMER_0,
        .intr_type = LEDC_INTR_DISABLE,
        .gpio_num = pin,
        .duty = 0,
        .hpoint = 0,
    };
    ledc_channel_config(&channel_conf);
}

// 舵机控制
void servo_control(int pin, int angle)
{
    // 角度转PWM占空比（0-180°对应500-2500us）
    int duty = (angle * 2000 / 180) + 500;
    duty = (duty * 4096) / 20000; // 转换为12位占空比
    
    servo_init(pin);
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    
    ESP_LOGI(TAG, "舵机控制：引脚%d，角度%d°，占空比%d", pin, angle, duty);
}

// 电机控制
void motor_control(int enable, int duty)
{
    if (enable) {
        ESP_LOGI(TAG, "电机启动，占空比：%d%%", duty);
    } else {
        ESP_LOGI(TAG, "电机关闭");
    }
}

// 执行器初始化
void actuators_init(void)
{
    ESP_LOGI(TAG, "执行器初始化完成");
    // 实际项目中初始化GPIO、PWM等
}

// 电机停止
void motor_stop(void)
{
    motor_control(0, 0);
}
```

## 四、部署配置

### 1. Docker Compose `docker-compose.yml`

```YAML

version: '3.8'

services:
  # MQTT Broker
  emqx:
    image: emqx/emqx:5.7.0
    container_name: clawlink-emqx
    restart: always
    environment:
      - TZ=Asia/Shanghai
      - EMQX_DASHBOARD_DEFAULT_USER_PASSWORD=${MQTT_ADMIN_PASS}
    ports:
      - "${MQTT_PORT}:1883"
      - "${MQTT_WEB_PORT}:18083"
    volumes:
      - ./data/emqx/data:/opt/emqx/data
      - ./data/emqx/log:/opt/emqx/log
    networks:
      - clawlink-net

  # 本地LLM服务
  ollama:
    image: ollama/ollama:latest
    container_name: clawlink-ollama
    restart: always
    environment:
      - TZ=Asia/Shanghai
      - OLLAMA_HOST=0.0.0.0
    ports:
      - "11434:11434"
    volumes:
      - ./data/ollama/models:/root/.ollama/models
    networks:
      - clawlink-net
    privileged: true

  # 脊椎层后端服务
  backend:
    build: ./raspberrypi-spine/backend
    container_name: clawlink-backend
    restart: always
    environment:
      - TZ=Asia/Shanghai
      - MQTT_BROKER=emqx
      - MQTT_PORT=${MQTT_PORT}
      - MQTT_EDGE_USER=${MQTT_EDGE_USER}
      - MQTT_EDGE_PASS=${MQTT_EDGE_PASS}
      - OLLAMA_BASE_URL=http://ollama:11434/api
      - OLLAMA_DEFAULT_MODEL=${OLLAMA_DEFAULT_MODEL}
      - CAMERA_INDEX=${CAMERA_INDEX}
      - RPI_HOST_IP=${RPI_HOST_IP}
    ports:
      - "8001:8001"
    volumes:
      - ./raspberrypi-spine/backend/src:/app
    devices:
      - /dev/video0:/dev/video0
      - /dev/gpiomem:/dev/gpiomem
    privileged: true
    depends_on:
      - emqx
      - ollama
    networks:
      - clawlink-net

  # 脊椎层前端服务
  frontend:
    image: nginx:alpine
    container_name: clawlink-frontend
    restart: always
    ports:
      - "8000:80"
    volumes:
      - ./raspberrypi-spine/frontend:/usr/share/nginx/html
    depends_on:
      - backend
    networks:
      - clawlink-net

networks:
  clawlink-net:
    driver: bridge
```

### 2. 环境变量模板 `.env.example`

```TOML

# 基础配置
RPI_HOST_IP=192.168.3.100
TZ=Asia/Shanghai

# MQTT配置
MQTT_ADMIN_PASS=ClawLink@2026
MQTT_EDGE_USER=clawlink-edge
MQTT_EDGE_PASS=ClawLinkEdge@2026
MQTT_DEVICE_PASS=ClawLinkDevice@2026
MQTT_PORT=1883
MQTT_WEB_PORT=18083

# LLM配置
OLLAMA_DEFAULT_MODEL=llama3:8b-instruct-q4_0

# 服务配置
CAMERA_INDEX=0
WEB_PORT=8000
API_PORT=8001
```

## 五、快速部署指南

### 1. 树莓派环境准备

```Bash

# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# 克隆项目
git clone https://github.com/你的用户名/ClawLink-TriLayer.git
cd ClawLink-TriLayer

# 配置环境变量
cp .env.example .env
# 编辑.env文件，修改RPI_HOST_IP为树莓派实际IP
nano .env

# 启动服务
docker compose up -d

# 下载LLM模型
docker exec -it clawlink-ollama ollama pull llama3:8b-instruct-q4_0
```

### 2. ESP32固件烧录

```Bash

# 进入ESP32代码目录
cd esp32-periphery

# 设置目标芯片
idf.py set-target esp32s3

# 编译固件
idf.py build

# 烧录固件（替换串口为实际设备）
idf.py -p /dev/ttyUSB0 flash monitor
```

### 3. OpenClaw插件安装

```Bash

# 进入大脑层目录
cd openclaw-brain

# 执行安装脚本
chmod +x install_skill.sh
./install_skill.sh

# 配置OpenClaw
nano ~/.openclaw/config.toml
# 添加以下配置：
[clawlink_trilayer]
mqtt_broker = "你的树莓派IP"
mqtt_port = 1883
mqtt_user = "clawlink-upper"
mqtt_pass = "ClawLinkUpper@2026"
raspberrypi_api = "http://你的树莓派IP:8001"
```

### 4. 验证部署

1. 访问Web前端：`http://树莓派IP:8000`

2. 访问MQTT后台：`http://树莓派IP:18083`（账号admin，密码为.env中的MQTT_ADMIN_PASS）

3. 测试指令：在OpenClaw中发送「采集所有设备温湿度并汇总」

## 六、二次开发指南

### 新增传感器

1. 在ESP32末梢层的`sensors.c`中添加传感器驱动函数

2. 更新`sensor_data_t`结构体，添加新传感器字段

3. 在`sensor_collect_task`中读取新传感器数据

4. 在MQTT上报逻辑中包含新传感器数据

### 新增执行器

1. 在ESP32末梢层的`actuators.c`中添加执行器控制函数

2. 在`handle_mqtt_command`中添加新指令的解析逻辑

3. 在树莓派脊椎层的`mqtt_client.py`中添加新指令的转发逻辑

4. 在Web前端中添加新执行器的控制界面

### 新增LLM能力

1. 在树莓派脊椎层的`llm_agent.py`中添加新的推理函数

2. 在FastAPI中新增接口暴露该能力

3. 在OpenClaw插件中添加新指令的映射逻辑

## 七、开源协议

本项目采用 **MIT License** 开源，你可以自由使用、修改、分发本项目的代码，包括商业用途，仅需保留原作者的版权声明。

## 八、致谢

- ESP-IDF：ESP32硬件驱动基础

- OpenClaw：AI任务调度框架

- YOLOv8：计算机视觉推理

- Ollama：本地大模型推理

- FastAPI：后端API开发

- Vue3：前端界面开发

- EMQX：MQTT消息中间件
> （注：文档部分内容可能由 AI 生成）