# StopWatch

-----------
## hardware

* MCU:ESP32-S3
    * Flash: 16MiB
    * PSRAM: 8MiB
    * SYS_I2C/I2C1
        * SCL -- G48
        * SCA -- G47
* PMIC: M5PM1@0x6E
    * Interface: SYS_I2C
* IO Expander: M5IOE1@0x4F
    * Interface: SYS_I2C
* Power
    * Charge Enable -- M5PM1_CHG_EN_PP
    * Charge State -- M5PM1_G2
    * Charge Current Prog -- M5PM1_G3
    * Motor -- M5IOE1_G9
    * Ext output -- M5PM1_BOOST5V_EN_PP
    * Ext output voltage -- M5PM1_5VOUT_ADC_IN
* Display
    * Power -- M5IO1_G8
    * Interface: QSPI
    * Driver: CO5300
    * Resolution: 466x466
    * 圆形显示屏 1.75 Inch Amoled
    * Pin Map
        * RST      -- M5IOE1_G5
        * TE       -- G38
        * QSPI_CS  -- G39
        * QSPI_SCK -- G40
        * QSPI_D0  -- G41
        * QSPI_D1  -- G42
        * QSPI_D2  -- G46
        * QSPI_D3  -- G45
* CTP: CST820B@0x15
    * RST -- M5IOE1_G4
    * INT -- G13
    * Interface: SYS_I2C
* Audio
    * Power -- M5IOE1_G3
    * CODEC: ES8311@0x18
        * Contral Interface: SYS_I2C
        * Data Interface:
            * I2S_MCLK -- G18
            * I2S_BCLK -- G17
            * I2S_LRCK -- G15
            * I2S_DIN  -- G21
            * I2S_DOUT -- G16
    * PA -- M5IOE1_G10
* Button
    * Btuuon1 - G1
    * Button2 - G2
* RTC: RX8130CE@0x32
    * Interface: SYS_I2C
* Grove
    * GND
    * 5V
    * G10
    * G11

-----------
## 私有 OTA（路径 B）

本板型的默认 OTA URL 为：

```text
http://47.97.244.51:16688/xiaozhi/ota/
```

`main/Kconfig.projbuild` 在选择 `BOARD_TYPE_M5STACK_STOPWATCH` 时使用上述默认值，
最终生成 `CONFIG_OTA_URL`。其它板型保留上游默认值。
固件仍通过 OTA 响应取得连接配置，不在源码中固定 WebSocket 地址或 token。
`main/ota.cc` 将响应的 `websocket` 对象写入 NVS，WebSocket 协议读取其中的
`url`、`token`、`version`。当前上游同时收到 `mqtt` 和 `websocket` 对象时优先选 MQTT；
若部署目标使用 WS，应确认 OTA 实际返回 WS 配置，且没有优先选中的 MQTT 对象。

注意两层覆盖关系：

- 编译时：已保存的 `sdkconfig` 中的 `CONFIG_OTA_URL` 优先于 Kconfig 默认值。
- 运行时：NVS 的 `wifi/ota_url` 非空时优先于 `CONFIG_OTA_URL`。

仅重新烧录应用、重新配网或改源码默认值，不保证清除旧 NVS 覆盖项。
首次迁移建议在接受清空 Wi-Fi、设备 UUID 和其它保存设置后，执行下文整片擦除并烧录完整镜像。
需要保留设置时，应在现有配网页支持的 OTA 设置项中把地址改成上述 URL，再检查实际请求。

## 环境与编译

当前上游使用原生 ESP-IDF，推荐 `release/v6.1` 分支，最低 v6.0.1；不支持 IDF 5.x。
本仓库没有对应的 PlatformIO 构建配置，不使用 `pio run`。
以下命令在 Linux/macOS 的 Bash 中、已应用补丁的仓库根目录执行。
板型目录是 `m5stack/stopwatch`，OTA 上报的板型及构建变体名仍为 `m5stack-stopwatch`。

若尚未安装 SDK，可按 ESP-IDF 官方安装要求准备系统依赖后运行一次：

```bash
git clone --recursive --branch release/v6.1 https://github.com/espressif/esp-idf.git "$HOME/esp/esp-idf"
"$HOME/esp/esp-idf/install.sh" esp32s3
```

加载环境并按上游入口编译：

```bash
. "$HOME/esp/esp-idf/export.sh"
idf.py --version
python3 scripts/build.py --list-boards
python3 scripts/build.py m5stack/stopwatch --name m5stack-stopwatch --language zh-CN --zip
```

脚本会选择 ESP32-S3、StopWatch、8 MB Octal PSRAM / 80 MHz，并使用上游的
16 MB Flash 与 `partitions/v2/16m.csv`。它会重新生成本地 `sdkconfig`，
依次调用 IDF 配置、编译及 `merge-bin`。更换 SDK 版本时先执行 `idf.py fullclean`。

构建后检查实际配置（必须有下列两个值）：

```bash
grep -Fx 'CONFIG_BOARD_TYPE_M5STACK_STOPWATCH=y' sdkconfig
grep -Fx 'CONFIG_OTA_URL="http://47.97.244.51:16688/xiaozhi/ota/"' sdkconfig
```

完整合并固件为 `build/merged-binary.bin`，压缩包为
`releases/v<PROJECT_VER>_m5stack-stopwatch.zip`（版本取根目录 `CMakeLists.txt`）。
不要将仅包含应用的 `build/xiaozhi.bin` 当作完整镜像烧到 `0x0`。

需要进一步调整时，在上述构建生成的配置基础上执行：

```bash
idf.py menuconfig
# Xiaozhi Assistant -> Default OTA URL：核对本页指定地址（保留末尾 /）
idf.py build
idf.py merge-bin
```

后续直接用 `idf.py` 构建会保留配置；再次运行 `scripts/build.py` 会重新生成配置。
合并镜像应由 IDF 按实际烧录清单生成，不手写 bootloader、应用或 assets 分区偏移。

## USB 下载模式

用 USB-C 数据线连接电脑，长按电源/复位键约 2 秒，直到绿色 LED 出现下载模式指示后松开。
若串口重新枚举，重新确认端口名。下面以 `/dev/ttyACM0` 为例；按实际情况改成
`/dev/ttyUSB0`、macOS 的 `/dev/cu.*` 或 Windows 的 `COMx`。
烧录结束后若设备仍停留在下载模式，短按电源键复位启动。

## 用 esptool 或 IDF 烧录

使用 ESP-IDF 环境自带的 esptool（以下为当前版本的连字符命令）：

```bash
PORT=/dev/ttyACM0

# 仅首次迁移、且接受丢失原有设备设置时执行；会清空整个 Flash。
python3 -m esptool --chip esp32s3 --port "$PORT" erase-flash

# 将刚构建的完整镜像写入 0x0。
python3 -m esptool --chip esp32s3 --port "$PORT" --baud 460800 \
    write-flash 0x0 build/merged-binary.bin

# 复位并观察日志；Ctrl+] 退出。
idf.py -p "$PORT" monitor
```

不使用合并镜像时，也可由 IDF 按构建清单烧录全部必要分区：

```bash
PORT=/dev/ttyACM0
idf.py -p "$PORT" flash monitor
```

`idf.py flash` 本身不保证清除旧 NVS；需要首次迁移清空时，可先使用上面的 `erase-flash`。

## 用 M5Burner 烧录

M5Burner 的官方 `STOPWATCH` 分类可烧录官方固件，但它不等同于本补丁构建的私有 OTA 固件。
本次构建的 `.zip` 是上游发布包，不能假设它就是某版 M5Burner 可直接导入的自定义包。

本补丁不包含 M5Burner 发布或共享操作。若工程人员另外提供了**由本补丁构建的**
M5Burner 自定义固件及 Share Code，可按下面步骤烧录：

1. 安装 M5Burner，按设备说明进入下载模式。
2. 在 `USER CUSTOM` 中使用 `Share Burn` 加载工程人员提供的 Share Code，
   核对固件说明、版本及 StopWatch 板型。
3. 点击 `Burn`，选择实际串口，点击 `Start`，等待烧录完成后复位。
4. 首次迁移仍需清除旧 NVS，可预先用上面的 esptool `erase-flash`，再执行 M5Burner 烧录。
5. 没有该自定义固件时，直接用上面的 esptool 命令烧录 `build/merged-binary.bin`。

M5Burner 官方的自定义固件分发流程是 `USER CUSTOM -> Publish` 上传、
`Share` 获取共享码、`Share Burn` 烧录；此流程由工程人员自行决定是否另行采用。
不要将已配网设备的整片 Flash 导出文件作为分发镜像，其中可能含 Wi-Fi 或服务凭据；
应使用干净构建的固件。

## 配网及 OTA 验收

1. 首次启动后，按屏幕提示连接设备的 `Xiaozhi-...` 热点，访问 `http://192.168.4.1/`，
   配置可访问目标 OTA 的 2.4 GHz Wi-Fi。若页面提供 OTA URL 字段，填入本页完整地址。
   完成后重启。需要进入配网时，当前板级实现支持在启动且未联网时短按 Button1（GPIO1）。
2. 用 `idf.py -p "$PORT" monitor` 查看启动日志。单凭“已连上 Wi-Fi”或编译配置，
   不能证明实际 OTA 请求目的地；当前 `Ota::CheckVersion()` 也没有固定输出完整 OTA URL 的日志。
3. 在可看到该设备流量的路由器/AP/网关上抓取启动阶段请求，例如：

   ```bash
   sudo tcpdump -i any -nn -s 0 -A 'host 47.97.244.51 and tcp port 16688'
   ```

   普通同网段电脑通常看不到其它设备的单播流量，应在网关/AP 抓包，
   或由有权限的运维只读查看已有 OTA 访问日志，无需修改 hub 服务。
   确认该设备的请求目的地是 `47.97.244.51:16688`，请求路径是 `/xiaozhi/ota/`，
   `Host` 对应这个 IP/端口，并按设备 IP、时间及 `Device-Id` 核对设备。
   当前板型通常使用 `POST` 并发送设备信息；浏览器直接访问或电脑 `curl` 成功不能代替设备验收。
4. 确认 OTA 响应 HTTP 200、可解析 JSON，并包含有效的 `websocket.url` 及服务要求的其它字段。
   对于 WS 部署，响应不应带有会让当前固件优先选择 MQTT 的 `mqtt` 对象。
   若响应要求激活，按私有服务提供的激活流程处理。
5. 触发一次对话，在串口查找 `Connecting to websocket server:`，
   核对后面的 URL 与刚才 OTA 响应里的 `websocket.url` 一致；实际 WS 地址可以与 OTA 地址不同。
   再确认连接成功并能进行对话。若仅有 HTTP 请求、没有有效响应或 WS 连接，验收尚未通过。

排障重点：请求仍去旧地址时检查 NVS 覆盖与是否刷入正确镜像；默认值仍旧时检查生成的
`sdkconfig`；HTTP 200 但没有连接配置时检查响应内容。不要在固件里补写固定 WS 地址。
抓包与日志可能包含 token，只在本地核验；不要将原始响应、抓包、Wi-Fi 密码或 API key 提交到仓库。

参考：

- [上游 StopWatch 支持 PR #2057](https://github.com/78/xiaozhi-esp32/pull/2057)
- [上游 SDK 与构建说明](../../../../README.md)
- [M5Stack StopWatch 硬件与下载模式](https://docs.m5stack.com/en/core/StopWatch)
- [M5Stack StopWatch 小智烧录与配网](https://docs.m5stack.com/en/guide/stopwatch/xiaozhi_voice_assistant)
- [M5Burner 自定义固件分发说明](https://docs.m5stack.com/en/uiflow/m5burner/publish)
- [esptool ESP32-S3 烧录命令](https://docs.espressif.com/projects/esptool/en/latest/esp32s3/esptool/basic-commands.html)
