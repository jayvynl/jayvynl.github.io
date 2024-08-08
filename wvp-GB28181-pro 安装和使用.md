以下步骤均使用操作系统 Ubuntu 22 执行，其他操作系统仅供参考。

[wvp-GB28181-pro](https://github.com/648540858/wvp-GB28181-pro) 是视频管理平台，作为 GB28181 服务器管理下级视频设备。[ZLMediaKit](https://github.com/ZLMediaKit/ZLMediaKit/tree/master) 作为视频流服务器，接收并转发视频流。

特点：

- 功能丰富，支持直播、点播、语音对讲等功能。
- 开箱即用，提供了完整的前后端管理功能。
- 支持定制，提供了丰富的 API 接口进行对接，定制功能。

ZLMediaKit
---

### 安装

```shell
sudo apt update
sudo apt install -y build-essential cmake libssl-dev ffmpeg
git clone --depth 1 https://gitee.com/xia-chu/ZLMediaKit
cd ZLMediaKit
git submodule update --init
mkdir build
cd build
cmake ..
make -j$(nproc)
```

### 配置

`ZLMediaKit/release/linux/Debug/config.ini`

```toml
[api]
apiDebug=1
defaultSnap=./www/logo.png
downloadRoot=./www
# wvp 配置项 media.secret 需配制成一样
secret=TWSYFgYJOQWB4ftgeYut8DW4wbs7pQnj
snapRoot=./www/snap/
```

### 运行

```shell
cd ZLMediaKit/release/linux/Debug
# 通过 -h 可以了解启动参数
./MediaServer -h
# 以守护进程模式启动
./MediaServer -d &
```

WVP
---

```shell
sudo apt-get install -y openjdk-11-jre git maven nodejs npm redis-server mysql-server
git clone https://gitee.com/pan648540858/wvp-GB28181-pro.git
cd wvp-GB28181-pro/web_src/
npm --registry=https://registry.npmmirror.com install
npm run build
cd ..
mvn package
```

### 数据库配置

```shell
sudo mysql
mysql> create database if not exists wvp default character set utf8 collate utf8_unicode_ci;
mysql> CREATE USER 'wvp'@'%' IDENTIFIED BY 'wvp';
mysql> grant all privileges on wvp.* to 'wvp'@'%';
mysql> flush privileges;
mysql> use wvp;
mysql> set names utf8;
mysql> source 数据库/2.7.3/初始化-mysql-2.7.3.sql
```

### wvp 配置

编辑 `target/application.yml` 修改 `spring.profiles.active` 为 `dev`
编辑 `target/application-dev.yml` 修改以下项

```yaml
spring:
  redis:
    # [必须修改] Redis服务器IP, REDIS安装在本机的,使用127.0.0.1
    host: 127.0.0.1
    # [必须修改] 端口号
    port: 6379
    # [可选] 数据库 DB
    database: 7
    # [可选] 访问密码,若你的redis服务器没有设置密码，就不需要用密码去连接
    password:
    # [可选] 超时时间
    timeout: 10000
    # mysql数据源
  datasource:
    dynamic:
      primary: master
      datasource:
        master:
          type: com.zaxxer.hikari.HikariDataSource
          driver-class-name: com.mysql.cj.jdbc.Driver
          url: jdbc:mysql://127.0.0.1:3306/wvp?allowPublicKeyRetrieval=true&useUnicode=true&characterEncoding=UTF8&rewriteBatchedStatements=true&serverTimezone=PRC&useSSL=false&allowMultiQueries=true
          username: wvp
          password: wvp
# 作为28181服务器的配置
sip:
  # [可选] 28181服务监听的端口
  port: 8116
  # 根据国标6.1.2中规定，domain宜采用ID统一编码的前十位编码。国标附录D中定义前8位为中心编码（由省级、市级、区级、基层编号组成，参照GB/T 2260-2007）
  # 后两位为行业编码，定义参照附录D.3
  # 3701020049标识山东济南历下区 信息行业接入
  # [可选]
  domain: 4101050000
  # [可选]
  id: 41010500002000000001
  # [可选] 默认设备认证密码，后续扩展使用设备单独密码, 移除密码将不进行校验
  password: dee3060
  # 是否存储alarm信息
  alarm: true
#zlm 默认服务器配置
media:
  # 必须修改为 zlm general.mediaServerId，默认是 your_server_id
  id: your_server_id
  # [必须修改] zlm服务器的内网IP
  ip: 192.168.0.30
  # [可选] 有公网IP就配置公网IP, 不可用域名
  wan_ip:
  # [必须修改] zlm服务器的http.port
  http-port: 80
  # [可选] zlm服务器访问WVP所使用的IP, 默认使用127.0.0.1，zlm和wvp没有部署在同一台服务器时必须配置
  hook-ip: 127.0.0.1
  # [必选选] zlm服务器的api.secret=secret
  secret: TWSYFgYJOQWB4ftgeYut8DW4wbs7pQnj
  # 启用多端口模式, 多端口模式使用端口区分每路流，兼容性更好。 单端口使用流的ssrc区分， 点播超时建议使用多端口测试
  rtp:
    # [可选] 是否启用多端口模式, 开启后会在portRange范围内选择端口用于媒体流传输
    enable: false
    # [可选] 在此范围内选择端口用于媒体流传输, 必须提前在zlm上配置该属性，不然自动配置此属性可能不成功
    port-range: 50000,50300 # 端口范围
    # [可选] 国标级联在此范围内选择端口发送媒体流,
    send-port-range: 50000,50300 # 端口范围
```

### 启动

后端

```shell
cd target
java -jar wvp-pro-*.jar
```

访问 `http://ip:8080` 使用平台自带 UI，`http://ip:8080/doc.html` 查看 API 文档。

systemd 服务
---

1. 首先把编译好的程序移动到固定位置， 例如`/opt`:

```shell
mv ~/ZLMediaKit/release/linux/Debug /opt/ZLM
mv ~/wvp-GB28181-pro/target /opt/wvp
```

2. systemd unit 文件

`/etc/systemd/system/zlm.service`:

```toml
[Unit]
Description = ZLMediaKit service
Documentation = https://docs.zlmediakit.com/
After = network.target
StartLimitIntervalSec = 0

[Service]
Type = simple
WorkingDirectory=/opt/ZLM
ExecStart = /opt/ZLM/MediaServer
Restart = always
RestartSec = 60

[Install]
WantedBy=default.target
```

`/etc/systemd/system/wvp.service`:

```toml
[Unit]
Description = wvp-GB28181-pro service
Documentation = https://doc.wvp-pro.cn/
After = zlm.service
StartLimitIntervalSec = 0

[Service]
Type = simple
WorkingDirectory=/opt/wvp
ExecStart = java -jar /opt/wvp/wvp-pro.jar
Restart = always
RestartSec = 60

[Install]
WantedBy=default.target
```

3. 启动服务

```shell
systemctl daemon-reload
systemctl enable --now zlm wvp
```

设备接入
---

参考 https://doc.wvp-pro.cn/#/_content/ability/device
