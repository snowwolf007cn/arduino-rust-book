# 按钮去抖
读取按钮，过滤噪音

由于机械和物理问题，按钮在按下时通常会产生虚假的打开/关闭转换：这些转换可能会被解读为在很短的时间内多次按下，从而欺骗了程序。此示例演示了如何对输入进行反跳操作，这意味着在短时间内检查两次以确保确实按下了按钮。在没有去抖的情况下，按下按钮一次可能会导致不可预测的结果。该草图使用 millis() 函数来跟踪自按下按钮以来经过的时间。

## 硬件要求
- Arduino板卡
- 瞬时按钮或开关
- 10k欧姆电阻
- 连接线
- 面包板

## 电路
### 电路图
![按钮去抖](images/pushbutton_connection.png "按钮去抖" =400x)

## 代码
下面的草图基于 Limor Fried 的去抖版本，但逻辑与她的示例相反。在她的示例中，开关在闭合时返回低电平，在打开时返回高电平。此处，开关在按下时返回高电平，在未按下时返回低电平。

编译并运行示例
```shell
cargo build
cargo run
```
完整代码如下：

src/main.rs
```rust
/*!
 * Debounce
 *
 * Each time the input pin goes from LOW to HIGH (e.g. because of a push-button
 * press), the output pin is toggled from LOW to HIGH or HIGH to LOW. There's a
 * minimum delay between toggles to debounce the circuit (i.e. to ignore noise).
 */
#![no_std]
#![no_main]
#![feature(abi_avr_interrupt)]

use arduino_hal::prelude::_unwrap_infallible_UnwrapInfallible;
use arduino_uno_example::utils::{millis, millis_init};
use embedded_hal::digital::OutputPin;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut led_pin = pins.d13.into_output();
    let button_pin = pins.d2.into_floating_input().downgrade();

    let mut last_button_is_high = false;
    let mut button_is_high: bool = false;
    let mut led_is_high: bool = true;

    let mut last_debounce_time: u32 = 0;

    const DEBOUNCE_DELAY: u32 = 50;

    millis_init(dp.TC0);

    // Enable interrupts globally
    unsafe { avr_device::interrupt::enable() };

    loop {
        let current_pin_is_high = button_pin.is_high();

        if current_pin_is_high != last_button_is_high {
            last_debounce_time = millis();
        }

        if millis() - last_debounce_time > DEBOUNCE_DELAY {
            if current_pin_is_high != button_is_high {
                button_is_high = current_pin_is_high;
            }

            if button_is_high {
                led_is_high = !led_is_high;
            }
        }

        led_pin.set_state(led_is_high.into()).unwrap_infallible();

        last_button_is_high = current_pin_is_high;
    }
}
```
