# 读取模拟电压
读取模拟输入并将电压打印到串行监视器

此示例向您展示如何读取模拟引脚 0 上的模拟输入，将值转换为电压，并将其打印到终端或者VS Code的串行监视器。

## 硬件要求
- Arduino板卡
- 10k欧姆电位器

## 电路
将电位计的三根线连接到电路板上。 第一个从电位计的一个外部引脚接地。 第二个从电位计的另一个外部引脚电压变为 5 伏。 第三个从电位计的中间引脚连接到模拟输入 0。

通过转动电位计的轴，可以改变连接到电位计中心销的游标两侧的电阻值。 这会改变中心引脚的电压。 当中心与连接5伏的一侧之间的电阻接近于零（而另一侧的电阻接近10k欧）时，中心引脚的电压接近5伏。 当电阻反向时，中心引脚的电压接近 0 伏或接地。 该电压是您作为输入读取的模拟电压。

该板的微控制器内部有一个称为模数转换器或 ADC 的电路，它读取此变化的电压并将其转换为 0 到 1023 之间的数字。当轴沿一个方向转动到底时，有 0 引脚上有 5 伏电压，输入值为 0。当轴沿相反方向旋转到底时，引脚上有 5 伏电压，输入值为 1023。在这期间，analog_read() 返回一个数字 0 到 1023 之间，与施加到引脚的电压量成正比。

### 电路实例
![读取模拟电压](images/read_analog_voltage.jpg "读取模拟电压" =400x)

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
要将值从 0-1023 更改为与引脚正在读取的电压相对应的范围，您需要创建另一个变量（浮点数），并进行一些数学运算。 要缩放 0.0 到 5.0 之间的数字，请将 5.0 除以 1023.0，然后乘以sensor_value：
```rust
let voltage = (sensor_value as f32) * 5.0 / 1023.0;
```
最后，您需要将此信息打印到串行监视器：
```rust
ufmt::uwriteln!(&mut serial, "{}",uFmt_f32::Two(voltage)).unwrap_infallible();
```
注意：此处需要在Cargo.toml中所有profile的配置中加入```overflow-check=false```。否则，编译链接avr-gcc时会报错。这个问题可以参见github上的这个issue的相关讨论：[-Zbuild-std + lto="fat" = undefined reference to core::panicking::panic](https://github.com/rust-lang/compiler-builtins/issues/347)

编译并运行示例
```shell
cargo build
cargo run
```
应用程序成功写入板卡后显示
```
    Finished `dev` profile [optimized + debuginfo] target(s) in 0.14s
     Running `ravedude uno -cb 57600 -P /dev/tty.usbmodem14101 target/avr-atmega328p/debug/digitalreadserial.elf`
       Board Arduino Uno
 Programming target/avr-atmega328p/debug/digitalreadserial.elf => /dev/tty.usbmodem14101
avrdude: AVR device initialized and ready to accept instructions
avrdude: device signature = 0x1e950f (probably m328p)
avrdude: erasing chip

avrdude: processing -U flash:w:target/avr-atmega328p/debug/digitalreadserial.elf:e
avrdude: reading input file target/avr-atmega328p/debug/digitalreadserial.elf for flash
         with 3902 bytes in 1 section within [0, 0xf3d]
         using 31 pages and 66 pad bytes
avrdude: writing 3902 bytes flash ...
Writing | ################################################## | 100% 0.65 s 
avrdude: 3902 bytes of flash written
avrdude: verifying flash memory against target/avr-atmega328p/debug/digitalreadserial.elf
Reading | ################################################## | 100% 0.51 s 
avrdude: 3902 bytes of flash verified

avrdude done.  Thank you.

  Programmed target/avr-atmega328p/debug/digitalreadserial.elf
     Console /dev/tty.usbmodem14101 at 57600 baud
             CTRL+C to exit.
```
如果您此时尚未退出或者退出后通过VS Code的串行监视器连接到板卡串口，您应该会看到一系列范围从 0.00 - 5.00 的稳定数字。 当您转动电位器时，数值会发生变化，与进入引脚 A0 的电压相对应。

完整代码如下：

src/main.rs
```rust
/*!
 * Read readouts of A0 ADC channels and convert it to voltage.
 *
 * This example shows you how to read an analog input on analog pin 0, 
 * convert the values from analogRead() into voltage, and print it out to the serial monitor.
 */
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use panic_halt as _;
use ufmt_float::uFmt_f32;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);

    let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());

    let a0 = pins.a0.into_analog_input(&mut adc);

    loop {
        let sensor_value = a0.analog_read(&mut adc);

        let voltage = (sensor_value as f32) * 5.0 / 1023.0;

        ufmt::uwriteln!(&mut serial, "{}",uFmt_f32::Two(voltage)).unwrap_infallible();

        arduino_hal::delay_ms(1000);
    }
}
```