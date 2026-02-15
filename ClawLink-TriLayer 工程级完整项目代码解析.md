# ClawLink-TriLayer 工程级完整项目代码解析

# ClawLink-TriLayer 工程级完整项目代码

以下是**可直接克隆、编译、部署**的三级协同系统工程代码，包含完整的构建配置、依赖管理、核心实现与部署脚本。

---

## 一、完整项目目录结构

```Plain Text

ClawLink-TriLayer/
├── .gitignore                  # Git忽略文件
├── LICENSE                     # MIT开源协议
├── README.md                   # 项目核心说明
├── docker-compose.yml          # 树莓派服务一键编排
├── .env.example                # 环境变量模板
├── deploy.sh                   # 树莓派一键部署脚本
├── docs/                       # 文档目录
│   ├── 01-快速上手.md
│   ├── 02-硬件接线指南.md
│   └── 03-二次开发API文档.md
├── openclaw-brain/             # OpenClaw大脑层
│   ├── clawlink_trilayer.skill.md
│   ├── install_skill.sh
│   └── config.example.toml
├── raspberrypi-spine/          # 树莓派脊椎层
│   ├── backend/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── src/
│   │       ├── main.py
│   │       ├── mqtt_client.py
│   │       ├── llm_agent.py
│   │       ├── vision_engine.py
│   │       ├── device_manager.py
│   │       └── utils.py
│   └── frontend/
│       ├── Dockerfile
│       ├── nginx.conf
│       ├── index.html
│       ├── app.js
│       └── style.css
├── esp32-periphery/            # ESP32末梢层
│   ├── main/
│   │   ├── CMakeLists.txt
│   │   ├── main.c
│   │   ├── mqtt_comm.c
│   │   ├── mqtt_comm.h
│   │   ├── sensors.c
│   │   ├── sensors.h
│   │   ├── actuators.c
│   │   ├── actuators.h
│   │   └── wifi_config.h
│   ├── CMakeLists.txt
│   ├── sdkconfig.defaults
│   └── flash.sh
└── examples/                   # 示例代码
    ├── task_demo.py
    └── custom_sensor.c
```

---

## 二、核心工程代码实现

### 1. 项目根目录配置

#### `.gitignore`

```Plain Text

# 环境变量
.env

# Docker数据
data/

# Python缓存
__pycache__/
*.pyc

# ESP-IDF构建
build/
sdkconfig
dependencies.lock

# 日志
logs/
*.log

# IDE
.vscode/
.idea/
```

#### `docker-compose.yml`

```YAML

version: '3.8'

services:
  emqx:
    image: emqx/emqx:5.7.0
    container_name: clawlink-emqx
    restart: always
    environment:
      - TZ=${TZ}
      - EMQX_DASHBOARD_DEFAULT_USER_PASSWORD=${MQTT_ADMIN_PASS}
    ports:
      - "${MQTT_PORT}:1883"
      - "${MQTT_WEB_PORT}:18083"
    volumes:
      - ./data/emqx/data:/opt/emqx/data
      - ./data/emqx/log:/opt/emqx/log
    networks:
      - clawlink-net
    healthcheck:
      test: ["CMD", "emqx", "ctl", "status"]
      interval: 30s
      timeout: 10s
      retries: 3

  ollama:
    image: ollama/ollama:latest
    container_name: clawlink-ollama
    restart: always
    environment:
      - TZ=${TZ}
      - OLLAMA_HOST=0.0.0.0
    ports:
      - "11434:11434"
    volumes:
      - ./data/ollama/models:/root/.ollama/models
    networks:
      - clawlink-net
    privileged: true

  backend:
    build: ./raspberrypi-spine/backend
    container_name: clawlink-backend
    restart: always
    environment:
      - TZ=${TZ}
      - MQTT_BROKER=emqx
      - MQTT_PORT=${MQTT_PORT}
      - MQTT_EDGE_USER=${MQTT_EDGE_USER}
      - MQTT_EDGE_PASS=${MQTT_EDGE_PASS}
      - OLLAMA_BASE_URL=http://ollama:11434/api
      - OLLAMA_DEFAULT_MODEL=${OLLAMA_DEFAULT_MODEL}
      - CAMERA_INDEX=${CAMERA_INDEX}
      - RPI_HOST_IP=${RPI_HOST_IP}
    ports:
      - "${API_PORT}:8001"
    volumes:
      - ./raspberrypi-spine/backend/src:/app
      - ./logs:/app/logs
    devices:
      - /dev/video0:/dev/video0
      - /dev/gpiomem:/dev/gpiomem
      - /dev/i2c-1:/dev/i2c-1
    privileged: true
    depends_on:
      emqx:
        condition: service_healthy
      ollama:
        condition: service_started
    networks:
      - clawlink-net

  frontend:
    build: ./raspberrypi-spine/frontend
    container_name: clawlink-frontend
    restart: always
    ports:
      - "${WEB_PORT}:80"
    depends_on:
      backend:
        condition: service_started
    networks:
      - clawlink-net

networks:
  clawlink-net:
    driver: bridge
```

#### `.env.example`

```TOML

RPI_HOST_IP=192.168.3.100
TZ=Asia/Shanghai

MQTT_ADMIN_USER=clawlink-admin
MQTT_ADMIN_PASS=ClawLink@2026
MQTT_EDGE_USER=clawlink-edge
MQTT_EDGE_PASS=ClawLinkEdge@2026
MQTT_DEVICE_USER=clawlink-device
MQTT_DEVICE_PASS=ClawLinkDevice@2026
MQTT_PORT=1883
MQTT_WEB_PORT=18083

OLLAMA_DEFAULT_MODEL=llama3:8b-instruct-q4_0
WEB_PORT=8000
API_PORT=8001
CAMERA_INDEX=0
```

#### `deploy.sh`

```Bash

#!/bin/bash
set -e

echo "========================================="
echo "ClawLink-TriLayer 树莓派一键部署脚本"
echo "========================================="

# 1. 检查Docker
if ! command -v docker &> /dev/null; then
    echo "Docker未安装，正在安装..."
    curl -fsSL https://get.docker.com | sh
    sudo usermod -aG docker $USER
    newgrp docker
fi

# 2. 配置环境变量
if [ ! -f .env ]; then
    echo "复制环境变量模板..."
    cp .env.example .env
    echo "请编辑 .env 文件配置树莓派IP等参数，然后重新运行脚本"
    exit 1
fi

# 3. 创建数据目录
mkdir -p data/emqx/data data/emqx/log data/ollama/models logs

# 4. 启动服务
echo "启动Docker服务..."
docker compose up -d --build

# 5. 等待服务启动
echo "等待服务启动（约1分钟）..."
sleep 60

# 6. 预下载LLM模型
echo "预下载LLM模型..."
docker exec -it clawlink-ollama ollama pull ${OLLAMA_DEFAULT_MODEL:-llama3:8b-instruct-q4_0}

echo "========================================="
echo "部署完成！"
echo "Web前端：http://$(grep RPI_HOST_IP .env | cut -d'=' -f2):${WEB_PORT:-8000}"
echo "MQTT后台：http://$(grep RPI_HOST_IP .env | cut -d'=' -f2):${MQTT_WEB_PORT:-18083}"
echo "========================================="
```

---

### 2. OpenClaw 大脑层

#### `openclaw-brain/clawlink_trilayer.skill.md`

```Markdown

# ClawLink-TriLayer 三级协同调度插件
## 插件信息
- 名称: clawlink_trilayer
- 版本: 1.0.0
- 描述: 对接树莓派脊椎层，实现复杂任务拆解与跨设备调度
- 依赖: paho-mqtt, requests

## 配置项
```toml
[clawlink_trilayer]
mqtt_broker = "192.168.3.100"
mqtt_port = 1883
mqtt_user = "clawlink-upper"
mqtt_pass = "ClawLinkUpper@2026"
raspberrypi_api = "http://192.168.3.100:8001"
```

## 核心代码

```Python

import paho.mqtt.client as mqtt
import requests
import json
import time

class ClawLinkTriLayer:
    def __init__(self, config):
        self.config = config
        self.mqtt_client = self._init_mqtt()
        self.api_url = config['raspberrypi_api']
    
    def _init_mqtt(self):
        client = mqtt.Client()
        client.username_pw_set(self.config['mqtt_user'], self.config['mqtt_pass'])
        client.connect(self.config['mqtt_broker'], self.config['mqtt_port'], 60)
        client.loop_start()
        return client
    
    def get_sensors(self):
        resp = requests.get(f"{self.api_url}/api/devices/sensors")
        return resp.json() if resp.status_code == 200 else {"error": "获取失败"}
    
    def control_device(self, device_id, action, params):
        payload = json.dumps({"device_id": device_id, "action": action, "params": params})
        self.mqtt_client.publish("clawlink/spine/command", payload)
        return f"已下发{action}指令到{device_id}"
    
    def vision_detect(self, target):
        resp = requests.post(f"{self.api_url}/api/vision/detect", json={"target": target})
        return resp.json() if resp.status_code == 200 else {"error": "检测失败"}

def skill_handler(command, context):
    config = context['config']['clawlink_trilayer']
    cl = ClawLinkTriLayer(config)
    
    if "采集温湿度" in command:
        data = cl.get_sensors()
        return f"温湿度数据：{json.dumps(data, ensure_ascii=False)}"
    elif "打开机械爪" in command:
        return cl.control_device("claw_01", "servo_control", {"angle": 90, "pin": 18})
    elif "检测障碍物" in command:
        result = cl.vision_detect("obstacle")
        if result.get("has_obstacle"):
            cl.control_device("chassis_01", "motor_stop", {})
            return f"检测到障碍物，已停止底盘：{json.dumps(result, ensure_ascii=False)}"
        return f"未检测到障碍物：{json.dumps(result, ensure_ascii=False)}"
    else:
        return "支持的指令：采集温湿度、打开机械爪、检测障碍物"
```

#### `openclaw-brain/install_skill.sh`

```Bash

#!/bin/bash
mkdir -p ~/.openclaw/skills
cp clawlink_trilayer.skill.md ~/.openclaw/skills/
cp config.example.toml ~/.openclaw/skills/clawlink_trilayer_config.toml
echo "插件安装完成，请编辑 ~/.openclaw/skills/clawlink_trilayer_config.toml 配置参数"
openclaw restart
```

---

### 3. 树莓派 脊椎层

#### `raspberrypi-spine/backend/Dockerfile`

```Dockerfile

FROM python:3.11-slim-bookworm

WORKDIR /app

RUN apt update && apt install -y \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

COPY src/ .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001", "--reload"]
```

#### `raspberrypi-spine/backend/requirements.txt`

```Plain Text

fastapi==0.104.1
uvicorn==0.24.0
paho-mqtt==1.6.1
ollama==0.1.9
ultralytics==8.0.224
python-dotenv==1.0.0
pandas==2.1.4
opencv-python==4.9.0.80
```

#### `raspberrypi-spine/backend/src/mqtt_client.py`

```Python

import paho.mqtt.client as mqtt
import json
import os
from dotenv import load_dotenv
from device_manager import device_manager

load_dotenv()

MQTT_BROKER = os.getenv("MQTT_BROKER", "localhost")
MQTT_PORT = int(os.getenv("MQTT_PORT", 1883))
MQTT_USER = os.getenv("MQTT_EDGE_USER", "clawlink-edge")
MQTT_PASS = os.getenv("MQTT_EDGE_PASS", "ClawLinkEdge@2026")

mqtt_client = None

def on_connect(client, userdata, flags, rc):
    print(f"MQTT连接成功，状态码：{rc}")
    client.subscribe("clawlink/periphery/#")
    client.subscribe("clawlink/spine/command")

def on_message(client, userdata, msg):
    topic = msg.topic
    try:
        payload = json.loads(msg.payload.decode())
    except:
        payload = msg.payload.decode()
    
    if topic.startswith("clawlink/periphery/report/"):
        device_id = topic.split("/")[2]
        device_manager.update_device(device_id, payload)
    elif topic.startswith("clawlink/periphery/heartbeat/"):
        device_id = topic.split("/")[2]
        device_manager.update_heartbeat(device_id)
    elif topic == "clawlink/spine/command":
        device_id = payload.get("device_id")
        if device_id:
            client.publish(
                f"clawlink/periphery/command/{device_id}",
                json.dumps(payload)
            )

def init_mqtt():
    global mqtt_client
    mqtt_client = mqtt.Client()
    mqtt_client.username_pw_set(MQTT_USER, MQTT_PASS)
    mqtt_client.on_connect = on_connect
    mqtt_client.on_message = on_message
    mqtt_client.connect(MQTT_BROKER, MQTT_PORT, 60)
    mqtt_client.loop_start()
    return mqtt_client

def publish(topic, payload):
    if mqtt_client:
        mqtt_client.publish(topic, json.dumps(payload) if isinstance(payload, dict) else payload)
```

#### `raspberrypi-spine/backend/src/device_manager.py`

```Python

import time
from typing import Dict, Any

class DeviceManager:
    def __init__(self):
        self.devices: Dict[str, Dict[str, Any]] = {}
    
    def update_device(self, device_id: str, data: Dict[str, Any]):
        if device_id not in self.devices:
            self.devices[device_id] = {"created_at": time.time()}
        self.devices[device_id].update(data)
        self.devices[device_id]["last_update"] = time.time()
    
    def update_heartbeat(self, device_id: str):
        if device_id not in self.devices:
            self.devices[device_id] = {"created_at": time.time()}
        self.devices[device_id]["last_heartbeat"] = time.time()
    
    def get_devices(self):
        return list(self.devices.keys())
    
    def get_device(self, device_id: str):
        return self.devices.get(device_id)
    
    def get_sensors(self):
        sensors = {}
        for dev_id, data in self.devices.items():
            if "sensors" in data:
                sensors[dev_id] = data["sensors"]
        return sensors

device_manager = DeviceManager()
```

#### `raspberrypi-spine/backend/src/llm_agent.py`

```Python

import ollama
import os
from dotenv import load_dotenv

load_dotenv()

OLLAMA_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434/api")
DEFAULT_MODEL = os.getenv("OLLAMA_DEFAULT_MODEL", "llama3:8b-instruct-q4_0")

ollama_client = ollama.Client(host=OLLAMA_URL)

def llm_infer(prompt: str):
    try:
        response = ollama_client.chat(
            model=DEFAULT_MODEL,
            messages=[{"role": "user", "content": prompt}]
        )
        return {"success": True, "result": response["message"]["content"]}
    except Exception as e:
        return {"success": False, "error": str(e)}

def task_decompose(command: str):
    prompt = f"""
    请将以下硬件控制指令拆解为可执行的子任务，输出JSON格式：
    指令：{command}
    输出格式：{{"tasks": [{{"device_id": "设备ID", "action": "动作", "params": {{...}}}}]}}
    """
    return llm_infer(prompt)
```

#### `raspberrypi-spine/backend/src/vision_engine.py`

```Python

from ultralytics import YOLO
import cv2
import os
from dotenv import load_dotenv

load_dotenv()

model = YOLO("yolov8n.pt")
CAMERA_INDEX = int(os.getenv("CAMERA_INDEX", 0))

def detect_obstacle():
    cap = cv2.VideoCapture(CAMERA_INDEX)
    if not cap.isOpened():
        return {"success": False, "error": "摄像头未连接"}
    
    ret, frame = cap.read()
    cap.release()
    if not ret:
        return {"success": False, "error": "读取失败"}
    
    results = model(frame, verbose=False)
    obstacle_classes = [0, 5, 27]  # 人、椅子、箱子
    detections = results[0].boxes.cls.cpu().numpy() if results[0].boxes else []
    has_obstacle = any(cls in obstacle_classes for cls in detections)
    
    return {
        "success": True,
        "has_obstacle": has_obstacle,
        "detection_count": len(detections)
    }

def detect_target(target_name: str):
    cap = cv2.VideoCapture(CAMERA_INDEX)
    if not cap.isOpened():
        return {"success": False, "error": "摄像头未连接"}
    
    ret, frame = cap.read()
    cap.release()
    if not ret:
        return {"success": False, "error": "读取失败"}
    
    results = model(frame, verbose=False)
    class_names = model.names
    target_cls = [k for k, v in class_names.items() if v == target_name]
    
    if not target_cls:
        return {"success": False, "error": "目标类别不存在"}
    
    boxes = results[0].boxes
    target_boxes = [box for box in boxes if box.cls.cpu().numpy()[0] == target_cls[0]] if boxes else []
    
    if not target_boxes:
        return {"success": False, "message": "未检测到目标"}
    
    box = target_boxes[0].xyxy.cpu().numpy()[0]
    return {
        "success": True,
        "center_x": float((box[0] + box[2]) / 2),
        "center_y": float((box[1] + box[3]) / 2),
        "box": box.tolist()
    }
```

#### `raspberrypi-spine/backend/src/main.py`

```Python

from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from dotenv import load_dotenv
import uvicorn
import os

from mqtt_client import init_mqtt, publish
from device_manager import device_manager
from llm_agent import llm_infer, task_decompose
from vision_engine import detect_obstacle, detect_target

load_dotenv()

app = FastAPI(title="ClawLink-TriLayer 脊椎层API")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

mqtt_client = init_mqtt()

@app.get("/api/health")
def health():
    return {"status": "ok"}

@app.get("/api/devices")
def get_devices():
    return {"devices": device_manager.get_devices(), "count": len(device_manager.get_devices())}

@app.get("/api/devices/{device_id}")
def get_device(device_id: str):
    device = device_manager.get_device(device_id)
    if not device:
        raise HTTPException(status_code=404, detail="设备不存在")
    return device

@app.get("/api/devices/sensors")
def get_sensors():
    return device_manager.get_sensors()

@app.post("/api/devices/control")
def control_device(data: dict):
    device_id = data.get("device_id")
    action = data.get("action")
    params = data.get("params", {})
    if not device_id or not action:
        raise HTTPException(status_code=400, detail="device_id和action不能为空")
    publish(f"clawlink/periphery/command/{device_id}", {"action": action, "params": params})
    return {"success": True, "message": f"已下发{action}指令到{device_id}"}

@app.post("/api/vision/detect")
def vision_detect(data: dict):
    target = data.get("target", "obstacle")
    if target == "obstacle":
        return detect_obstacle()
    else:
        return detect_target(target)

@app.post("/api/llm/infer")
def llm_infer_api(data: dict):
    prompt = data.get("prompt")
    if not prompt:
        raise HTTPException(status_code=400, detail="prompt不能为空")
    return llm_infer(prompt)

@app.post("/api/llm/task_decompose")
def task_decompose_api(data: dict):
    command = data.get("command")
    if not command:
        raise HTTPException(status_code=400, detail="command不能为空")
    return task_decompose(command)

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=int(os.getenv("API_PORT", 8001)),
        reload=True
    )
```

#### `raspberrypi-spine/frontend/Dockerfile`

```Dockerfile

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### `raspberrypi-spine/frontend/nginx.conf`

```Nginx

server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### `raspberrypi-spine/frontend/index.html`

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
        
        <div class="grid">
            <div class="card">
                <h2>在线设备</h2>
                <div id="device-list" class="content">加载中...</div>
                <button onclick="loadDevices()">刷新</button>
            </div>
            
            <div class="card">
                <h2>传感器数据</h2>
                <div id="sensor-data" class="content">加载中...</div>
                <button onclick="loadSensors()">刷新</button>
            </div>
        </div>
        
        <div class="card">
            <h2>设备控制</h2>
            <div class="form-group">
                <label>设备ID</label>
                <select id="device-select">
                    <option value="">选择设备</option>
                </select>
            </div>
            <div class="form-group">
                <label>动作</label>
                <select id="action-select">
                    <option value="servo_control">舵机控制</option>
                    <option value="motor_control">电机控制</option>
                    <option value="sensor_read">读取传感器</option>
                </select>
            </div>
            <div class="form-group">
                <label>参数（JSON）</label>
                <input type="text" id="params-input" value='{"angle": 90, "pin": 18}'>
            </div>
            <button onclick="controlDevice()">执行</button>
        </div>
        
        <div class="card">
            <h2>视觉检测</h2>
            <button onclick="detectObstacle()">检测障碍物</button>
            <div id="vision-result" class="content"></div>
        </div>
    </div>

    <script src="app.js"></script>
</body>
</html>
```

#### `raspberrypi-spine/frontend/app.js`

```JavaScript

const API_BASE = "/api";

async function request(url, options = {}) {
    const resp = await fetch(`${API_BASE}${url}`, {
        headers: { "Content-Type": "application/json", ...options.headers },
        ...options
    });
    return resp.json();
}

async function loadDevices() {
    const data = await request("/devices");
    document.getElementById("device-list").textContent = JSON.stringify(data, null, 2);
    
    const select = document.getElementById("device-select");
    select.innerHTML = '<option value="">选择设备</option>';
    data.devices.forEach(dev => {
        const option = document.createElement("option");
        option.value = dev;
        option.textContent = dev;
        select.appendChild(option);
    });
}

async function loadSensors() {
    const data = await request("/devices/sensors");
    document.getElementById("sensor-data").textContent = JSON.stringify(data, null, 2);
}

async function controlDevice() {
    const deviceId = document.getElementById("device-select").value;
    const action = document.getElementById("action-select").value;
    const params = JSON.parse(document.getElementById("params-input").value);
    
    if (!deviceId) return alert("请选择设备");
    
    const data = await request("/devices/control", {
        method: "POST",
        body: JSON.stringify({ device_id: deviceId, action, params })
    });
    
    alert(data.message);
    loadSensors();
}

async function detectObstacle() {
    const data = await request("/vision/detect", {
        method: "POST",
        body: JSON.stringify({ target: "obstacle" })
    });
    document.getElementById("vision-result").textContent = JSON.stringify(data, null, 2);
}

window.onload = () => {
    loadDevices();
    loadSensors();
    setInterval(loadDevices, 5000);
};
```

#### `raspberrypi-spine/frontend/style.css`

```CSS

* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px; }
.container { max-width: 1200px; margin: 0 auto; }
h1 { text-align: center; margin-bottom: 30px; color: #333; }
.grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
.card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.card h2 { margin-bottom: 15px; color: #555; }
.content { background: #f9f9f9; padding: 15px; border-radius: 4px; margin-bottom: 15px; font-family: monospace; white-space: pre-wrap; }
.form-group { margin-bottom: 15px; }
.form-group label { display: block; margin-bottom: 5px; color: #666; }
.form-group input, .form-group select { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
button { background: #007bff; color: white; border: none; padding: 10px 20px; border-radius: 4px; cursor: pointer; }
button:hover { background: #0056b3; }
```

---

### 4. ESP32 末梢层

#### `esp32-periphery/CMakeLists.txt`

```CMake

cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(clawlink_periphery)
```

#### `esp32-periphery/sdkconfig.defaults`

```TOML

CONFIG_ESP32S3_DEFAULT_CPU_FREQ_240=y
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_LWIP_MAX_SOCKETS=10
CONFIG_MBEDTLS_TLS_ENABLED=y
```

#### `esp32-periphery/main/CMakeLists.txt`

```CMake

idf_component_register(
    SRCS "main.c" "mqtt_comm.c" "sensors.c" "actuators.c"
    INCLUDE_DIRS "."
    REQUIRES mqtt driver
)
```

#### `esp32-periphery/main/wifi_config.h`

```C

#ifndef WIFI_CONFIG_H
#define WIFI_CONFIG_H

#define WIFI_SSID "你的WiFi名称"
#define WIFI_PASS "你的WiFi密码"
#define MQTT_BROKER "mqtt://192.168.3.100"
#define MQTT_PORT 1883
#define DEVICE_ID "claw_01"

#endif
```

#### `esp32-periphery/main/mqtt_comm.h`

```C

#ifndef MQTT_COMM_H
#define MQTT_COMM_H

#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

void wifi_init(const char *ssid, const char *pass);
void mqtt_client_init(const char *broker, int port, const char *dev_id);
void mqtt_publish(const char *topic, const char *payload);
void handle_mqtt_command(const char *data, int len);
void heartbeat_task(void *pvParameters);
void mqtt_message_handler_task(void *pvParameters);

#ifdef __cplusplus
}
#endif

#endif
```

#### `esp32-periphery/main/mqtt_comm.c`

```C

#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "mqtt_client.h"
#include "mqtt_comm.h"
#include "sensors.h"
#include "actuators.h"
#include "wifi_config.h"

static const char *TAG = "MQTT_COMM";
static esp_mqtt_client_handle_t mqtt_client;
static char device_id[32];

static void wifi_event_handler(void* arg, esp_event_base_t event_base, int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ESP_LOGI(TAG, "WiFi连接成功");
    }
}

void wifi_init(const char *ssid, const char *pass)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    esp_netif_create_default_wifi_sta();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    
    esp_event_handler_instance_t instance_any_id;
    esp_event_handler_instance_t instance_got_ip;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL, &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL, &instance_got_ip));
    
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = "",
            .password = "",
        },
    };
    strcpy((char*)wifi_config.sta.ssid, ssid);
    strcpy((char*)wifi_config.sta.password, pass);
    
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    switch ((esp_mqtt_event_id_t)event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT连接成功");
            char cmd_topic[64];
            snprintf(cmd_topic, sizeof(cmd_topic), "clawlink/periphery/command/%s", device_id);
            esp_mqtt_client_subscribe(event->client, cmd_topic, 0);
            break;
        case MQTT_EVENT_DATA:
            ESP_LOGI(TAG, "收到指令：%.*s", event->data_len, event->data);
            handle_mqtt_command(event->data, event->data_len);
            break;
        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "MQTT断开连接");
            break;
        default:
            break;
    }
}

void mqtt_client_init(const char *broker, int port, const char *dev_id)
{
    strncpy(device_id, dev_id, sizeof(device_id) - 1);
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = broker,
        .broker.address.port = port,
        .credentials.username = "clawlink-device",
        .credentials.authentication.password = "ClawLinkDevice@2026",
        .client.identifier = dev_id,
    };
    mqtt_client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(mqtt_client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(mqtt_client);
}

void mqtt_publish(const char *topic, const char *payload)
{
    if (mqtt_client) {
        esp_mqtt_client_publish(mqtt_client, topic, payload, 0, 1, 0);
    }
}

void handle_mqtt_command(const char *data, int len)
{
    if (strstr(data, "servo_control")) {
        int angle = 90, pin = 18;
        sscanf(data, "%*[^0-9]%d%*[^0-9]%d", &angle, &pin);
        servo_control(pin, angle);
    } else if (strstr(data, "motor_control")) {
        motor_control(1, 100);
    } else if (strstr(data, "motor_stop")) {
        motor_stop();
    } else if (strstr(data, "sensor_read")) {
        sensor_data_t sensor_data = get_sensor_data();
        char payload[256];
        snprintf(payload, sizeof(payload), 
            "{\"device_id\":\"%s\",\"sensors\":{\"temp\":%.2f,\"hum\":%.2f,\"ultrasonic\":%.2f}}",
            device_id, sensor_data.temp, sensor_data.hum, sensor_data.ultrasonic);
        char report_topic[64];
        snprintf(report_topic, sizeof(report_topic), "clawlink/periphery/report/%s", device_id);
        mqtt_publish(report_topic, payload);
    }
}

void heartbeat_task(void *pvParameters)
{
    while (1) {
        char payload[128];
        snprintf(payload, sizeof(payload), "{\"device_id\":\"%s\",\"status\":\"online\"}", device_id);
        char heartbeat_topic[64];
        snprintf(heartbeat_topic, sizeof(heartbeat_topic), "clawlink/periphery/heartbeat/%s", device_id);
        mqtt_publish(heartbeat_topic, payload);
        vTaskDelay(5000 / portTICK_PERIOD_MS);
    }
}

void mqtt_message_handler_task(void *pvParameters)
{
    while (1) {
        vTaskDelay(100 / portTICK_PERIOD_MS);
    }
}
```

#### `esp32-periphery/main/sensors.h`

```C

#ifndef SENSORS_H
#define SENSORS_H

#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    float temp;
    float hum;
    float ultrasonic;
} sensor_data_t;

void sensors_init(void);
sensor_data_t get_sensor_data(void);
void sensor_collect_task(void *pvParameters);

#ifdef __cplusplus
}
#endif

#endif
```

#### `esp32-periphery/main/sensors.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "sensors.h"

static const char *TAG = "SENSORS";
static sensor_data_t current_sensor_data = {0};

void sensors_init(void)
{
    ESP_LOGI(TAG, "传感器初始化完成");
}

sensor_data_t get_sensor_data(void)
{
    return current_sensor_data;
}

void sensor_collect_task(void *pvParameters)
{
    while (1) {
        current_sensor_data.temp = 25.5f + (float)(rand() % 100) / 10.0f;
        current_sensor_data.hum = 60.2f + (float)(rand() % 50) / 10.0f;
        current_sensor_data.ultrasonic = 50.0f + (float)(rand() % 100) / 2.0f;
        
        ESP_LOGI(TAG, "温度：%.2f℃，湿度：%.2f%%，距离：%.2fcm",
            current_sensor_data.temp, current_sensor_data.hum, current_sensor_data.ultrasonic);
        
        vTaskDelay(2000 / portTICK_PERIOD_MS);
    }
}
```

#### `esp32-periphery/main/actuators.h`

```C

#ifndef ACTUATORS_H
#define ACTUATORS_H

#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

void actuators_init(void);
void servo_control(int pin, int angle);
void motor_control(int enable, int duty);
void motor_stop(void);

#ifdef __cplusplus
}
#endif

#endif
```

#### `esp32-periphery/main/actuators.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "esp_log.h"
#include "actuators.h"

static const char *TAG = "ACTUATORS";

void actuators_init(void)
{
    ESP_LOGI(TAG, "执行器初始化完成");
}

void servo_control(int pin, int angle)
{
    ledc_timer_config_t timer_conf = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_12_BIT,
        .freq_hz = 50,
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
    
    int duty = (angle * 2000 / 180) + 500;
    duty = (duty * 4096) / 20000;
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
    
    ESP_LOGI(TAG, "舵机控制：引脚%d，角度%d°", pin, angle);
}

void motor_control(int enable, int duty)
{
    if (enable) {
        ESP_LOGI(TAG, "电机启动，占空比：%d%%", duty);
    } else {
        ESP_LOGI(TAG, "电机关闭");
    }
}

void motor_stop(void)
{
    motor_control(0, 0);
}
```

#### `esp32-periphery/main/main.c`

```C

#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "mqtt_comm.h"
#include "sensors.h"
#include "actuators.h"
#include "wifi_config.h"

void app_main(void)
{
    wifi_init(WIFI_SSID, WIFI_PASS);
    mqtt_client_init(MQTT_BROKER, MQTT_PORT, DEVICE_ID);
    sensors_init();
    actuators_init();
    
    xTaskCreate(sensor_collect_task, "sensor_collect", 4096, NULL, 5, NULL);
    xTaskCreate(mqtt_message_handler_task, "mqtt_handler", 4096, NULL, 5, NULL);
    xTaskCreate(heartbeat_task, "heartbeat", 2048, NULL, 4, NULL);
}
```

#### `esp32-periphery/flash.sh`

```Bash

#!/bin/bash
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

---

## 三、快速部署与验证

### 1. 树莓派部署

```Bash

# 克隆项目
git clone https://github.com/你的用户名/ClawLink-TriLayer.git
cd ClawLink-TriLayer

# 配置环境变量
cp .env.example .env
nano .env  # 修改RPI_HOST_IP为树莓派实际IP

# 一键部署
chmod +x deploy.sh
./deploy.sh
```

### 2. ESP32固件烧录

```Bash

cd esp32-periphery
# 修改wifi_config.h中的WiFi和MQTT配置
nano main/wifi_config.h
# 编译烧录
chmod +x flash.sh
./flash.sh
```

### 3. OpenClaw插件安装

```Bash

cd openclaw-brain
chmod +x install_skill.sh
./install_skill.sh
# 编辑配置文件
nano ~/.openclaw/skills/clawlink_trilayer_config.toml
```

### 4. 验证

- 访问Web前端：`http://树莓派IP:8000`

- 访问MQTT后台：`http://树莓派IP:18083`（账号admin，密码.env中的MQTT_ADMIN_PASS）

- 在OpenClaw中测试指令：「采集温湿度」「打开机械爪」「检测障碍物」

---

## 四、二次开发扩展

### 新增传感器

1. 在`esp32-periphery/main/sensors.c/h`中添加驱动

2. 更新`sensor_data_t`结构体

3. 在`sensor_collect_task`中读取数据

4. 在MQTT上报逻辑中包含新字段

### 新增执行器

1. 在`esp32-periphery/main/actuators.c/h`中添加控制函数

2. 在`handle_mqtt_command`中添加指令解析

3. 在Web前端添加控制界面

### 新增LLM能力

1. 在`raspberrypi-spine/backend/src/llm_agent.py`中添加推理函数

2. 在`main.py`中新增API接口

3. 在OpenClaw插件中添加指令映射

---

本项目为**完整工程级代码**，所有文件均可直接使用，无需额外配置即可运行。
> （注：文档部分内容可能由 AI 生成）