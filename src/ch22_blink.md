# 闪烁
每秒开关LED一次

此示例展示了使用 Arduino 可以执行的最简单的操作来查看物理输出：它使板载 LED 闪烁。

## 硬件要求
- Arduino板卡

可选
- LED
- 220欧姆电阻

## 电路
此示例使用大多数 Arduino 板都具有的内置 LED。 该 LED 连接到数字引脚，其数量可能因板类型而异。 以下是板卡和数字引脚之间的对应关系。
- D13 - 101
- D13 - Due
- D1 - Gemma
- D13 - Intel Edison
- D13 - Intel Galileo Gen2
- D13 - Leonardo and Micro
- D13 - LilyPad
- D13 - LilyPad USB
- D13 - MEGA2560
- D13 - Mini
- D6 - MKR1000
- D13 - Nano
- D13 - Pro
- D13 - Pro Mini
- D13 - UNO
- D13 - Yún
- D13 - Zero
如果您想用此草图点亮外部 LED，则需要构建此电路，将电阻器的一端连接到非内置LED对应的数字引脚。 将 LED 的长腿（正极腿，称为阳极）连接到电阻器的另一端。 将 LED 的短脚（负脚，称为阴极）连接到 GND。 与 LED 串联的电阻值可以是与 220 欧姆不同的值； 当电阻值高达 1K 欧姆时，LED 也会亮起。

## 代码
获取led引脚并设置为数据输出
```rust
let mut led = pins.d13.into_output();
```
切换led状态：
```rust
led.toggle();
```
编译并运行示例
```shell
cargo build
cargo run
```
您希望有足够的时间让人们看到更改，因此，delay_ms() 命令告诉主板在 1000 毫秒（即一秒）内不执行任何操作。 当您使用delay_ms()命令时，在这段时间内不会发生任何其他事情。 了解基本示例后，请查看 [无延迟闪烁](./ch31_blink_without_delay.md) 示例以了解如何在执行其他操作时创建延迟。

了解此示例后，请查看 [读取串口数字信号](./ch23_digital_read_serial.md)示例以了解如何读取连接到电路板的开关。

完整代码如下：
src/main.js
```rust
/*!
 * Toggle an LED on and off every second.
 *
 * This example shows you how toggle LED on and off every second.
 */
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut led = pins.d13.into_output();

    loop {
        led.toggle();
        arduino_hal::delay_ms(1000);
    }
}
```