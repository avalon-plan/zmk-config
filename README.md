# Knob Macropad — ZMK 配置

主控 Super52840(= XIAO nRF52840 替代品)+ 旋钮 + WS2812B 状态灯 + 3 个按键。

## 板子 & 烧录
- `board: seeeduino_xiao_ble`(Super52840 与 XIAO 引脚相同、用 XIAO Bootloader)
- 烧录:双击复位出现 U 盘,把 Actions 编出的 `.uf2` 拖进去

## 引脚(XIAO 焊盘)
| 功能 | 引脚 |
|------|------|
| I²C SDA/SCL → AS5600 | P0.04 / P0.05 (D4/D5) |
| WS2812B DIN | P1.12 (D7) |
| 切层键 / 蓝牙键(接 GND) | P0.02 / P0.03 (D0/D1) |
| 复位键 | 硬件 RST↔GND |

## 三层 + 灯珠颜色反馈
按切层键循环:第一层(绿)→第二层(蓝)→第三层(红)→…,落到哪层闪哪色。

## ZMK Studio 在线改键(已启用)
- 已加:`CONFIG_ZMK_STUDIO=y`、`studio-rpc-usb-uart` snippet、physical-layout(2 键)。
- 用法:USB 连电脑,打开 ZMK Studio(网页版 studio.zmk.dev 或桌面 App),即可在线改这 2 个键的绑定,改完保存到设备、断电不丢。
- 已设 `CONFIG_ZMK_STUDIO_LOCKING=n`,免去物理解锁键(2 个键不够分一个解锁键)。
- 注意:Studio 主要改**按键**绑定;旋钮(sensor/输入)的行为 Studio 改不了,得改配置重新编译。
- 同一个 USB CDC 不能既跑 Studio 又跑日志。调 AS5600 磁铁 AGC 时,把 build.yaml 的 snippet 临时换回 `zmk-usb-logging` 并把 `CONFIG_..._LOG_AGC=y`,调好再换回来。

## 旋钮分层功能(已整合,实验性)
旋钮按层做不同事,已经配好(用 AS5600,不用换 EC11):
- 第一层(绿):滚动
- 第二层(蓝):音量加/减
- 第三层(红):PageUp / PageDown

实现:`west.yml` 拉了第三方模块 `te9no/zmk-input-processor-keybind`;keymap 里
AS5600 的 input-listener 用了**按层切换的输入处理器**——第一层走滚动,第二/三层先把
滚轮事件映射成 Y 轴(`zip_wheel_to_y`),再用 `zip_kb_vol` / `zip_kb_pg` 把转动变成按键。

### 上设备后要调的两点
1. **灵敏度**:keymap 里两个处理器的 `tick = <40>` 是累计阈值。AS5600 分辨率高,
   如果轻轻一转音量/翻页就跳一大片,把 tick 调大;如果要转很多才动一下,调小。
2. **方向反了**就把对应 bindings 的上下两项对调(例如 `<&kp C_VOL_DN>, <&kp C_VOL_UP>`)。

### 已知风险
- ZMK 曾有 bug:input-listener 的按层处理器没能真正按层隔离(2025 年中报)。
  本配置把三层都写成 listener 的子节点、不留基础处理器来规避;若切到音量/翻页层还漏出
  滚动,就是该 bug 在你拉的 ZMK 版本里没修——可临时把不需要的层的处理器去掉验证。
- `zip_keybind_keys` 是第三方模块;若编译报缺 Kconfig,按其仓库 README 补上对应 `CONFIG_`。
- 还是想最省心:换 EC11 + `sensor-bindings`(`&inc_dec_kp C_VOL_UP C_VOL_DN` 等)更稳,
  但要动硬件、且真正的鼠标滚轮滚动在 EC11 上不开箱即用。

## 其它要点
- WS2812B 接 3V3 偏暗;要亮接 USB 5V 或电池 BAT+。
- AS5600 需正上方径向充磁磁铁 0.5–3mm。
- 蓝牙键长按 `BT_CLR_CMD`(清当前);清全部改 `BT_CLR_ALL_CMD`。
