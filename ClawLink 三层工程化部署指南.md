# ClawLink 三层工程化部署指南

# ClawLink-TriLayer 工程化部署指南

以下是将代码组织成**可直接编译、部署、运行**的完整工程的详细步骤，涵盖环境准备、代码组织、配置修改、服务启动与验证全流程。

---

## 一、工程整体结构与文件说明

首先确保你的项目目录与以下结构完全一致，每个文件/文件夹的作用已标注：

```Plain Text

ClawLink-TriLayer/
├── .gitignore                  # Git忽略文件（排除环境变量、构建缓存等）
├── LICENSE                     # MIT开源协议
├── README.md                   # 项目说明（快速上手指南）
├── docker-compose.yml          # 树莓派服务一键编排（核心部署文件）
├── .env.example                # 环境变量模板（需复制为.env并修改）
├── deploy.sh                   # 树莓派一键部署脚本（自动化执行部署）
├── docs/                       # 文档目录
│   ├── 01-快速上手.md
│   ├── 02-硬件接线指南.md
│   └── 03-二次开发API文档.md
├── openclaw-brain/             # OpenClaw大脑层（插件+安装脚本）
│   ├── clawlink_trilayer.skill.md  # 核心调度插件
│   ├── install_skill.sh        # 插件一键安装脚本
│   └── config.example.toml     # 插件配置模板
├── raspberrypi-spine/          # 树莓派脊椎层（后端+前端）
│   ├── backend/
│   │   ├── Dockerfile          # 后端镜像构建文件
│   │   ├── requirements.txt    # Python依赖清单
│   │   └── src/                # 后端核心代码
│   │       ├── main.py         # FastAPI入口
│   │       ├── mqtt_client.py  # MQTT边缘网关
│   │       ├── llm_agent.py    # 本地LLM推理
│   │       ├── vision_engine.py # YOLOv8视觉处理
│   │       ├── device_manager.py # 设备管理
│   │       └── utils.py        # 工具函数
│   └── frontend/
│       ├── Dockerfile          # 前端镜像构建文件
│       ├── nginx.conf          # Nginx反向代理配置
│       ├── index.html          # 前端页面
│       ├── app.js              # 前端逻辑
│       └── style.css           # 前端样式
├── esp32-periphery/            # ESP32末梢层（固件代码）
│   ├── main/
│   │   ├── CMakeLists.txt      # ESP32组件构建配置
│   │   ├── main.c              # 固件入口
│   │   ├── mqtt_comm.c/h       # MQTT通信模块
│   │   ├── sensors.c/h         # 传感器采集模块
│   │   ├── actuators.c/h       # 执行器控制模块
│   │   └── wifi_config.h       # WiFi/MQTT配置（需修改）
│   ├── CMakeLists.txt          # ESP32项目构建配置
│   ├── sdkconfig.defaults      # ESP32固件默认配置
│   └── flash.sh                # 固件一键烧录脚本
└── examples/                   # 示例代码
    ├── task_demo.py            # 复杂任务调度示例
    └── custom_sensor.c         # 新增传感器示例
```

---

## 二、第一步：准备硬件与软件环境

### 1. 硬件准备

|层级|硬件型号|数量|备注|
|---|---|---|---|
|大脑层|任意PC/服务器（Windows/Linux）|1|运行OpenClaw|
|脊椎层|树莓派5/4B（8GB）|1|推荐树莓派5，需配128GB A2级TF卡、5V 5A电源|
|末梢层|ESP32-S3-WROOM-1-N16R8|1+|可扩展多个ESP32节点|
|可选配件|树莓派官方Camera Module 3、USB免驱麦克风、MG90S舵机、AHT20温湿度传感器|-|根据需求选配|
### 2. 软件环境准备

#### 树莓派环境

- 安装 **Raspberry Pi OS Bookworm 64位** 系统

- 开启SSH、摄像头、I2C接口（`raspi-config` → Interface Options）

- 确保树莓派与电脑、ESP32在**同一局域网**

#### ESP32开发环境

- 安装 **ESP-IDF v5.1+**（参考[ESP-IDF官方安装指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32s3/get-started/index.html)）

- 配置ESP-IDF环境变量（运行`export.sh`/`export.bat`）

#### OpenClaw环境

- 安装Node.js v18+

- 安装OpenClaw：`npm install -g openclaw@latest`

---

## 三、第二步：获取并组织项目代码

### 1. 创建项目目录

在树莓派、ESP32开发机、OpenClaw运行机上分别创建项目目录（或在树莓派上统一管理，通过SFTP传输ESP32代码）：

```Bash

# 树莓派/开发机上执行
mkdir -p ~/Projects/ClawLink-TriLayer
cd ~/Projects/ClawLink-TriLayer
```

### 2. 复制代码文件

将本文档“完整项目目录结构”中列出的所有代码文件，**按目录结构完整复制**到项目目录中，确保：

- 树莓派上保留完整项目（重点是`docker-compose.yml`、`raspberrypi-spine/`、`deploy.sh`）

- ESP32开发机上保留`esp32-periphery/`目录

- OpenClaw运行机上保留`openclaw-brain/`目录

---

## 四、第三步：配置树莓派脊椎层

### 1. 修改环境变量

在树莓派项目目录下，复制环境变量模板并修改：

```Bash

# 复制模板
cp .env.example .env

# 编辑.env文件（关键修改RPI_HOST_IP为树莓派实际局域网IP）
nano .env
```

`.env`文件核心配置说明：

```TOML

# 必须修改：树莓派实际局域网IP（通过ifconfig/ip a查看）
RPI_HOST_IP=192.168.3.100

# 可选修改：MQTT管理员密码（建议修改）
MQTT_ADMIN_PASS=ClawLink@2026

# 其他配置保持默认即可
TZ=Asia/Shanghai
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

### 2. （可选）修改后端/前端代码

如需自定义功能，修改`raspberrypi-spine/backend/src/`或`raspberrypi-spine/frontend/`下的代码，Docker部署时会自动重新构建镜像。

---

## 五、第四步：一键部署树莓派服务

在树莓派项目目录下执行：

```Bash

# 给部署脚本加执行权限
chmod +x deploy.sh

# 执行一键部署（首次运行需5-10分钟，会自动下载Docker镜像、构建服务、预下载LLM模型）
./deploy.sh
```

### 部署成功验证

看到以下输出表示部署完成：

```Plain Text

=========================================
部署完成！
Web前端：http://192.168.3.100:8000
MQTT后台：http://192.168.3.100:18083
=========================================
```

此时可访问：

- **Web控制中心**：`http://树莓派IP:8000`（查看设备、控制硬件）

- **EMQX MQTT后台**：`http://树莓派IP:18083`（账号`admin`，密码为`.env`中的`MQTT_ADMIN_PASS`，查看MQTT消息）

---

## 六、第五步：配置并烧录ESP32末梢层

### 1. 修改ESP32配置

在ESP32开发机的`esp32-periphery/main/`目录下，编辑`wifi_config.h`：

```C

#ifndef WIFI_CONFIG_H
#define WIFI_CONFIG_H

// 修改为你的WiFi名称和密码
#define WIFI_SSID "你的WiFi名称"
#define WIFI_PASS "你的WiFi密码"

// 修改为树莓派的实际局域网IP（格式：mqtt://IP:端口）
#define MQTT_BROKER "mqtt://192.168.3.100"
#define MQTT_PORT 1883

// 修改为设备唯一ID（如claw_01、chassis_01，多个ESP32需不同ID）
#define DEVICE_ID "claw_01"

#endif
```

### 2. 编译并烧录固件

在ESP32开发机的`esp32-periphery/`目录下执行：

```Bash

# 1. 加载ESP-IDF环境变量（根据你的ESP-IDF安装路径调整）
. ~/esp/esp-idf/export.sh

# 2. 设置目标芯片为ESP32-S3
idf.py set-target esp32s3

# 3. 编译固件
idf.py build

# 4. 烧录固件（替换/dev/ttyUSB0为你的ESP32串口）
idf.py -p /dev/ttyUSB0 flash monitor
```

### ESP32运行验证

烧录成功后，ESP32会自动重启，在串口监视器中看到以下日志表示运行正常：

```Plain Text

I (xxxx) MQTT_COMM: WiFi连接成功
I (xxxx) MQTT_COMM: MQTT连接成功
I (xxxx) SENSORS: 温度：25.50℃，湿度：60.20%，距离：50.00cm
```

此时刷新树莓派Web前端，设备列表中会出现`claw_01`。

---

## 七、第六步：安装OpenClaw大脑层插件

在OpenClaw运行机的`openclaw-brain/`目录下执行：

```Bash

# 1. 给安装脚本加执行权限
chmod +x install_skill.sh

# 2. 执行安装脚本
./install_skill.sh
```

### 配置OpenClaw插件

编辑插件配置文件：

```Bash

nano ~/.openclaw/skills/clawlink_trilayer_config.toml
```

修改以下内容为树莓派实际信息：

```TOML

[clawlink_trilayer]
mqtt_broker = "192.168.3.100"  # 树莓派IP
mqtt_port = 1883
mqtt_user = "clawlink-upper"
mqtt_pass = "ClawLinkUpper@2026"
raspberrypi_api = "http://192.168.3.100:8001"  # 树莓派API地址
```

### 重启OpenClaw服务

```Bash

openclaw restart
```

---

## 八、第七步：全系统验证与测试

### 1. 测试Web前端控制

- 访问`http://树莓派IP:8000`

- 在“设备控制”中选择`claw_01`，动作选“舵机控制”，参数保持默认，点击“执行”

- 查看ESP32串口日志，确认舵机控制指令执行成功

### 2. 测试OpenClaw指令

在OpenClaw支持的IM平台（如飞书、Telegram）或OpenClaw命令行中发送指令：

- 「采集温湿度」：OpenClaw会返回所有ESP32的温湿度数据

- 「打开机械爪」：OpenClaw会下发指令到ESP32控制舵机

- 「检测障碍物」：OpenClaw会触发树莓派视觉检测，若有障碍则停止底盘

### 3. 测试离线自治

- 断开树莓派与外网的连接

- 在Web前端手动控制ESP32，确认仍可正常执行指令

- （可选）在树莓派本地LLM中测试推理：`curl -X POST http://树莓派IP:8001/api/llm/infer -H "Content-Type: application/json" -d '{"prompt":"你好"}'`

---

## 九、常见问题排查

### 1. 树莓派Docker服务未启动

```Bash

# 检查Docker状态
sudo systemctl status docker

# 若未启动，手动启动
sudo systemctl start docker
sudo systemctl enable docker
```

### 2. ESP32连不上WiFi/MQTT

- 确认`wifi_config.h`中的WiFi名称、密码、树莓派IP正确

- 确认ESP32与树莓派在同一局域网

- 查看EMQX MQTT后台，确认MQTT服务正常

### 3. Web前端无法访问

- 确认树莓派防火墙未拦截8000/8001端口：`sudo ufw allow 8000,8001/tcp`

- 查看Docker服务日志：`docker compose logs -f`

---

完成以上步骤后，你就拥有了一个**可直接运行、可扩展**的三级协同智能硬件系统！如需新增功能，参考“二次开发指南”修改对应模块代码即可。
> （注：文档部分内容可能由 AI 生成）