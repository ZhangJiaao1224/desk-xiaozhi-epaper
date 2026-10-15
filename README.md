# 桌面小智 + 墨水屏日历

一块 ESP32-S3 桌面小玩具：板载 OLED 跑实时对话界面，4.2 寸墨水屏当日历 / 喝水看板。
定位是玩具，不追求高可用，但代码要能隔两周还看得懂。详细设计见
`docs/specs/2026-10-15-desktop-xiaozhi-epaper-design.md`。

## 现状

Phase A（不依赖硬件的准备）已经做完大半：

- [x] 装好 ESP-IDF 6.1
- [x] 拉好上游 `xiaozhi-esp32` v2.5.2 源码并 vendored 进本仓库
- [x] 用上游自带的 `maza-ai/esp32-s3` 板型（唯一带墨水屏的官方板）跑通完整编译，
      产出 `merged-binary.bin`，8.54MB。整条工具链验证通过。
- [ ] 写 4.2 寸 SSD1683 驱动（上游只有 2.9 寸的实现）
- [x] 填 `hardware/引脚记录表.md`（用厂家原理图 + 文档读出来的，实物复核留到货当天）
- [x] 核实板型、codec、OLED、墨水屏驱动 IC 与刷新耗时（结论见设计文档第 2 节）
- [ ] 写实现计划

## 目录结构

```
desk-xiaozhi-epaper/
├── AGENTS.md                          项目约定，改代码前先读
├── docs/
│   ├── specs/2026-10-15-...-design.md 设计文档
│   └── 到货当天验证清单.md            硬件到手后照着做
├── hardware/
│   └── 引脚记录表.md                  等卖家原理图，填满才能写引脚定义
└── firmware/
    └── xiaozhi-esp32/                 上游源码（vendored，基于 v2.5.2）
```

## 环境

| 项 | 位置 |
|---|---|
| ESP-IDF 6.1 | `C:\Espressif\esp-idf` |
| 工具链与 Python 环境 | `C:\Users\lenovo\.espressif` |
| 上游 git 目录（为日后和上游比对保留） | `C:\Espressif\xiaozhi-upstream-v2.5.2.git` |

### 怎么编译

```powershell
Set-Location 'C:\Espressif\esp-idf'
. .\export.ps1
$env:PYTHONUTF8='1'
Set-Location 'C:\Users\lenovo\Desktop\desk-xiaozhi-epaper\firmware\xiaozhi-esp32'
python scripts\build.py maza-ai/esp32-s3
```

产物：`firmware\xiaozhi-esp32\build\merged-binary.bin`，烧录到地址 `0x0`。

## 踩过的坑

这几条都是实测撞出来的，别重蹈覆辙。完整版记录在 `AGENTS.md`。

1. **中文路径直接把编译打挂。** 同一份 hello_world 放含中文的路径下，
   cmake 崩在退出码 `0xC0000409`；放纯 ASCII 路径下正常。所以本项目目录名是英文的，
   别改回中文。
2. **GitHub 连接时好时坏。** 大流量下载后经常连不上。子模块走 `ghfast.top` 代理可行，
   工具链则设 `IDF_GITHUB_ASSETS=dl.espressif.com/github_assets` 走乐鑫国内 CDN，
   实测比代理快很多。工具下载卡在 0 字节临时文件上超过一分钟，基本就是僵了，直接重跑。
3. **上游的墨水屏驱动不能直接用于 4.2 寸。** `main/display/epd_display/` 里那套是给
   2.9 寸 SSD1680 系屏写的：初始化只发 `0x01/0x0C/0x2C/0x3A/0x3B/0x11/0x3C/0x44/0x45/0x4E/0x4F`、
   波形全靠 `0x32` 自写 LUT、出图只用 `0x24` 从不写 `0x26`。4.2 寸面板这几处都不一样，
   照抄必然花屏，得另写驱动。
4. **厂家资料里混了一份无关的驱动 IC 手册。** 同一目录下的 `UC8176.pdf` 是旧款 4.2 寸屏的
   驱动 IC 手册，命令集与本板完全对不上。本板面板（微雪 4.2inch e-Paper V2）驱动 IC 是
   **SSD1683**，依据是面板手册命令表 `0x34` 行明写 "refer to SSD1683 application note"。
   验收时序也一律以面板手册标称值为准：全刷 3s、快刷 1.5s/1s、局刷 0.42s。
5. **上游 openthread 的嵌套子模块是坏的。** `openthread/third_party/mbedtls/repo/framework`
   这个孙模块检不出来，会让整棵子模块更新失败。本项目只走 WiFi 不用 Thread，
   已在 esp-idf 仓库里把 openthread 及其嵌套子模块设成 `update = none`。

## 还没清理的东西

`.pathtest/` 是当初测中文路径编译留下的残留，已被 git 忽略，可以直接删掉。
