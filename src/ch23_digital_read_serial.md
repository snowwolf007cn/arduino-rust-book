# 读取串口数字信号
读取开关，将状态打印到串行监视器。

此示例向您展示如何通过 USB 在 Arduino 和计算机之间建立串行通信来监控开关的状态。

## 硬件要求
- Arduino板卡
- 瞬时开关、按钮或拨动开关
- 10k欧电阻
- 连接线
- 面包板

## 电路
将三根电线连接到板上。 前两个（红色和黑色）连接到面包板侧面的两个长垂直行，以提供对 5 伏电源和接地的访问。 第三根电线从数字引脚 2 连接到按钮的一条腿。 按钮的同一条腿通过下拉电阻（此处为 10k 欧姆）连接到地。 按钮的另一条腿连接到 5 伏电源。

当您按下按钮或开关时，它们会连接电路中的两点。当按钮打开（未按下）时，按钮的两个引脚之间没有连接，因此该引脚接地（通过下拉电阻）并读取为低电平或 0。当按钮关闭（按下）时），它在两个引脚之间建立连接，将引脚连接到 5 伏，以便引脚读取为高电平或 1。

如果您断开数字 I/O 引脚与所有设备的连接，其读数可能会发生不稳定变化。这是因为输入是“浮动”的 - 也就是说，它没有与电压或接地牢固连接，并且它会随机返回高电平或低电平。这就是电路中需要下拉电阻的原因。

### 电路实例
![读取串口数字信号](images/digital_read_serial.jpg "读取串口数字信号" =400x)

## 代码
创建串口连接
```rust
let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
```
获取d2引脚并设置为数据输入
```rust
let pin = pins.d2.into_floating_input().downgrade();
```
判断引脚输入是否为高电平，并转换为十进制数
```rust
let sensor_value = pin.is_high() as u8;
```
最后，您需要将此信息打印到串行监视器：
```rust
ufmt::uwriteln!(&mut serial, "{}",sensor_value).unwrap_infallible();
```
编译并运行示例
```shell
cargo build
cargo run
```
应用程序成功写入板卡后显示
```
   Compiling digitalreadserial v0.1.0 (/Users/zhangzhi/workspace/rust/arduino/tutorial/digitalreadserial)
    Finished `dev` profile [optimized + debuginfo] target(s) in 0.68s
     Running `ravedude uno -cb 57600 -P /dev/tty.usbmodem14101 target/avr-atmega328p/debug/digitalreadserial.elf`
       Board Arduino Uno
 Programming target/avr-atmega328p/debug/digitalreadserial.elf => /dev/tty.usbmodem14101
avrdude: AVR device initialized and ready to accept instructions
avrdude: device signature = 0x1e950f (probably m328p)
avrdude: erasing chip

avrdude: processing -U flash:w:target/avr-atmega328p/debug/digitalreadserial.elf:e
avrdude: reading input file target/avr-atmega328p/debug/digitalreadserial.elf for flash
         with 404 bytes in 1 section within [0, 0x193]
         using 4 pages and 108 pad bytes
avrdude: writing 404 bytes flash ...
Writing | ################################################## | 100% 0.10 s 
avrdude: 404 bytes of flash written
avrdude: verifying flash memory against target/avr-atmega328p/debug/digitalreadserial.elf
Reading | ################################################## | 100% 0.07 s 
avrdude: 404 bytes of flash verified

avrdude done.  Thank you.

  Programmed target/avr-atmega328p/debug/digitalreadserial.elf
     Console /dev/tty.usbmodem14101 at 57600 baud
             CTRL+C to exit.
```
如果您此时尚未退出或者退出后通过VS Code的串行监视器连接到板卡串口，如果开关打开，您将看到一串“0”；如果开关关闭，您将看到“1”。

完整代码如下：
src/main.rs
```rust
/*!
 * Read a switch, print the state out to the Serial Monitor.
 *
 * This example shows you how to read a switch state input on digital pin 2.
 */
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);

    let pin = pins.d2.into_floating_input().downgrade();

    loop {
        let sensor_value = pin.is_high() as u8;

        ufmt::uwriteln!(&mut serial, "{}", sensor_value).unwrap_infallible();

        arduino_hal::delay_ms(1000);
    }
}
```