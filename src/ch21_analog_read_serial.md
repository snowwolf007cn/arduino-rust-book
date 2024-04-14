# 模拟读取串口
读取电位计，将其状态打印到终端或者VS Code的串行监视器。

此示例向您展示如何使用电位计从物理世界读取模拟输入。 **电位计**是一种简单的机械装置，当其轴转动时，它会提供不同大小的阻力。 通过将电压通过电位计传递到板上的模拟输入，可以将电位计（或简称电位器）产生的电阻值测量为模拟值。 在此示例中，您将在 Arduino 和VS Code的计算机之间建立串行通信后监视电位计的状态。

## 硬件要求
- Arduino Board
- 10k ohm potentiometer

## 电路
将电位计的三根线连接到电路板上。 第一个从电位计的一个外部引脚接地。 第二个从电位计的另一个外部引脚施加到 5 伏电压。 第三个从电位计的中间引脚到模拟引脚 A0。 

通过转动电位计的轴，可以改变连接到电位计中心销的游标两侧的电阻值。 这会改变中心引脚的电压。 当中心与连接5伏的一侧之间的电阻接近于零（而另一侧的电阻接近10k欧姆）时，中心引脚的电压接近5伏。 当电阻反向时，中心引脚的电压接近 0 伏或接地。 该电压是您作为输入读取的模拟电压。 

Arduino 板内部有一个称为模数转换器或 ADC 的电路，它读取此变化的电压并将其转换为 0 到 1023 之间的数字。当轴沿一个方向转动到底时，会有 0 伏电压 到引脚，输入值为 0。当轴沿相反方向转动到底时，有 5 伏电压流向引脚，输入值为 1023。在这期间，analogRead() 返回 0 之间的数字 1023 与施加到引脚的电压量成正比。

## 电路实例
![读取串口模拟信号](images/analog_read_serial.jpg "读取串口模拟信号" =400x)

## 代码
在下面的程序中，您要做的第一件事是在配置文件中，开始在您的板和计算机之间以每秒 57600 位数据的速度进行串行通信，修改.cargo/cargo.toml中[targe.'cfg(target_arch = "avr")']下的runner配置
```toml
[target.'cfg(target_arch = "avr")']
runner = "ravedude uno -cb 57600 -P /dev/tty.usbmodem14101"
```
创建串口连接
```rust
let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
```
创建ADC连接
```rust
let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());
```
获取A0引脚并设置为数据输入
```rust
let a0 = pins.a0.into_analog_input(&mut adc);
```
接下来，在代码的主循环中，您需要建立一个变量来存储来自电位计的电阻值（介于 0 到 1023 之间，该变量为u16类型）：
```rust
let sensor_value = a0.analog_read(&mut adc);
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
    Finished `dev` profile [optimized + debuginfo] target(s) in 0.10s
     Running `ravedude uno -cb 57600 -P /dev/tty.usbmodem14101 target/avr-atmega328p/debug/analogreadserial.elf`
       Board Arduino Uno
 Programming target/avr-atmega328p/debug/analogreadserial.elf => /dev/tty.usbmodem14101
avrdude: AVR device initialized and ready to accept instructions
avrdude: device signature = 0x1e950f (probably m328p)
avrdude: erasing chip

avrdude: processing -U flash:w:target/avr-atmega328p/debug/analogreadserial.elf:e
avrdude: reading input file target/avr-atmega328p/debug/analogreadserial.elf for flash
         with 732 bytes in 1 section within [0, 0x2db]
         using 6 pages and 36 pad bytes
avrdude: writing 732 bytes flash ...
Writing | ################################################## | 100% 0.14 s 
avrdude: 732 bytes of flash written
avrdude: verifying flash memory against target/avr-atmega328p/debug/analogreadserial.elf
Reading | ################################################## | 100% 0.10 s 
avrdude: 732 bytes of flash verified

avrdude done.  Thank you.

  Programmed target/avr-atmega328p/debug/analogreadserial.elf
     Console /dev/tty.usbmodem14101 at 57600 baud
             CTRL+C to exit.
```
如果您此时尚未退出或者退出后通过VS Code的串行监视器连接到板卡串口，您应该会看到稳定的 范围从 0 到 1023 的数字流，与旋钮的位置相关。 当您转动电位器时，这些数字几乎会立即响应。

完整代码如下：
src/main.js
```rust
/*!
 * Show readouts of analog input on analog pin 0.
 *
 * This example shows you how to read an analog input on analog pin 0,
 * convert the values from analogRead() into voltage, and print it out to the serial monitor.
 */
#![no_std]
#![no_main]


use arduino_hal::prelude::*;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp,pins,57600);
    let mut adc = arduino_hal::Adc::new(dp.ADC,Default::default());
    let a0 = pins.a0.into_analog_input(&mut adc);

    loop {
        let sensor_value = a0.analog_read(&mut adc);

        ufmt::uwriteln!(&mut serial, "{}", sensor_value).unwrap_infallible();

        arduino_hal::delay_ms(1000);
    }
}
```