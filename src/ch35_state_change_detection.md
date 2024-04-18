# 按钮的状态变化检测（边缘检测）
计算按下按钮的次数。

一旦按钮开始工作，您通常希望根据按钮被按下的次数来执行一些操作。为此，您需要知道按钮何时将状态从关闭更改为打开，并计算这种状态更改发生的次数。这称为状态变化检测或边缘检测。在本教程中，我们学习如何检查状态更改，向串行监视器发送包含相关信息的消息，并计算四个状态更改以打开和关闭 LED。

## 硬件要求
- Arduino板卡
- 瞬时按钮或开关
- 10k欧姆电阻
- 连接线
- 面包板

## 电路
将三根电线连接到板上。第一个从按钮的一条腿通过下拉电阻（此处为 10k 欧姆）接地。第二个从按钮的相应支路连接到 5 伏电源。第三个连接到数字 I/O 引脚（此处为引脚 2），用于读取按钮的状态。

当按钮打开（未按下）时，按钮的两条腿之间没有连接，因此该引脚接地（通过下拉电阻），我们读取低电平。当按钮关闭（按下）时，它会在两个引脚之间建立连接，将引脚连接到电压，以便我们读取高电平。 （该引脚仍然接地，但电阻器阻止电流流动，因此电阻最小的路径是+5V。）

如果断开数字 I/O 引脚与所有设备的连接，LED 可能会不规律地闪烁。这是因为输入是“浮动”的，即未连接到电压或接地。它或多或少会随机返回高电平或低电平。这就是电路中需要下拉电阻的原因。

### 电路图
![按钮状态变化检测（边缘检测）](images/pushbutton_connection.png "按钮状态变化检测（边缘检测）" =400x)

## 代码
下面的草图不断读取按钮的状态。然后，它将按钮的状态与上次通过主循环的状态进行比较。如果当前按钮状态与上一个按钮状态不同并且当前按钮状态为高，则按钮从关闭变为打开。然后，该草图会递增按钮按下计数器。

该草图还检查按钮按下计数器的值，如果它是四的整数倍，它将打开引脚 13 上的 LED。否则，它会将其关闭。

编译并运行示例
```shell
cargo build
cargo run
```

完整代码如下：

src/main.rs
```rust
/*!
 * State change detection (edge detection)

 * Often, you don't need to know the state of a digital input all the time, but
 * you just need to know when the input changes from one state to another.
 * For example, you want to know when a button goes from OFF to ON. This is called
 * state change detection, or edge detection.

 * This example shows how to detect when a button or button changes from off to on
 * and on to off.
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

    let mut led_pin = pins.d13.into_output();
    let button_pin = pins.d2.into_floating_input();

    let mut button_push_counter: u32 = 0;
    let mut last_button_is_high = false;

    loop {
        let button_is_high = button_pin.is_high();

        if button_is_high != last_button_is_high {
            if button_is_high {
                button_push_counter += 1;
                ufmt::uwriteln!(&mut serial, "on").unwrap_infallible();
                ufmt::uwriteln!(
                    &mut serial,
                    "number of button pushes: {}",
                    button_push_counter
                )
                .unwrap_infallible();
            } else {
                ufmt::uwriteln!(&mut serial, "off").unwrap_infallible();
            }
        }
		
		last_button_is_high = button_is_high;

		if button_push_counter % 4 == 0 {
			led_pin.set_high();
		} else {
			led_pin.set_low();
		}
    }
}
```