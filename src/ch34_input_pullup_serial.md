# 输入串行上拉

此示例演示了 INPUT_PULLUP 与 pinMode() 的使用。它通过 USB 在 Arduino 和计算机之间建立串行通信来监视开关的状态。

此外，当输入为高电平时，连接到引脚 13 的板载 LED 将点亮；当为低电平时，LED 将关闭。

## 硬件要求
- Arduino板卡
- 瞬时按钮或开关
- 连接线
- 面包板

## 电路
将两根电线连接到 Arduino 板。黑线将地线连接至按钮的一根腿。第二根电线从数字引脚 2 连接到按钮的另一条腿。

当您按下按钮或开关时，它们会连接电路中的两点。当按钮打开（未按下）时，按钮的两条腿之间没有连接。由于引脚 2 上的内部上拉处于活动状态并连接到 5V，因此当按钮打开时我们读取为高电平。当按钮关闭时，Arduino 读数为低电平，因为接地连接已完成。

### 电路图
![输入串行上拉](images/pullup_button.png "输入串行上拉" =400x)

## 代码
创建串口连接
```rust
let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
```
接下来，将数字引脚2初始化为输入并启用内部上拉电阻：
```rust
let button_pin = pins.d2.into_pull_up_input();
```
以下行使引脚 13（带有板载 LED）成为输出：
```rust
let mut led_pin= pins.d13.into_output();
```
现在您的设置已完成，请进入代码的主循环。当您的按钮未被按下时，内部上拉电阻连接至 5 伏。这会导致 Arduino 报告高电平。当按下按钮时，Arduino 引脚被拉至地，导致 Arduino 报告低电平。

在程序的主循环中需要做的第一件事是建立一个变量来保存来自开关的信息。由于来自开关的信息要么是高，要么是低，因此您可以使用 bool 数据类型。将此变量命名为is_high，并将其设置为等于数字引脚2上读取的值。您只需一行代码即可完成所有这些：
```rust
let is_high = button_pin.is_high();
```
Arduino 读取输入后，将其以布尔值的形式打印回计算机。您可以使用最后一行代码中的宏ufmt::uwrintln!()来执行此操作：
```rust
ufmt::uwriteln!(&mut serial, "{}", is_high).unwrap_infallible();
```
编译并运行示例
```shell
cargo build
cargo run
```
如果您此时尚未退出或者退出后通过VS Code的串行监视器连接到板卡串口，如果按钮按下，您将看到一串false，如果按钮松开，您将看到true流。 当开关处于高电平时，引脚 13 上的 LED 将亮起，而当开关处于低电平时，引脚 13 上的 LED 将熄灭。

完整代码如下：

src/main.rs
```rust
/*!
 * Input Pull-up Serial
 *
 * This example demonstrates the use of into_pull_up_input. It reads a digital
 * input on pin 2 and prints the results to the Serial Monitor.
 * 
 * Unlike into_floating_input(), there is no pull-down resistor necessary. An internal
 * 20K-ohm resistor is pulled to 5V. This configuration causes the input to read
 * HIGH when the switch is open, and LOW when it is closed.
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

	let button_pin = pins.d2.into_pull_up_input();
	let mut led_pin= pins.d13.into_output();
	
	loop {
		let is_high = button_pin.is_high();
		ufmt::uwriteln!(&mut serial, "{}", is_high).unwrap_infallible();

		if is_high {
			led_pin.set_low();
		} else {
			led_pin.set_high();
		}
	}
}
```