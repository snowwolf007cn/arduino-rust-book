# 无延迟闪烁
不使用delay_ms() 函数使LED 闪烁。

有时您需要同时做两件事。例如，您可能希望在读取按钮按下情况时使 LED 闪烁。在这种情况下，您不能使用delay_ms()，因为Arduino会在delay_ms()期间暂停您的程序。如果在 Arduino 暂停等待delay_ms()通过时按下按钮，您的程序将错过按钮按下。

该草图演示了如何在不使用delay_ms()的情况下使LED闪烁。它会打开 LED，然后记下时间。然后，每次通过loop()，它都会检查是否已经过了所需的闪烁时间。如果有，它会打开或关闭 LED 并记下新时间。通过这种方式，LED 会连续闪烁，而草图执行不会滞后于单个指令。

打个比方，就像用微波炉加热披萨，同时等待一些重要的电子邮件。您将披萨放入微波炉中，加热 10 分钟。使用delay_ms() 的类比是坐在微波炉前看着计时器从10 分钟倒计时直到计时器达到零。如果重要的电子邮件在此期间到达，您将会错过它。

在现实生活中，你会做的就是打开披萨，然后检查你的电子邮件，然后可能会做其他事情（这不会花太长时间！），每隔一段时间你就会回到微波炉看看如果计时器已为零，则表明您的披萨已完成。 在本教程中，您将学习如何设置类似的计时器。

## 硬件要求
- Arduino板卡
- 220欧电阻
- LED

## 电路
要构建电路，请将电阻器的一端连接到电路板的引脚 13。将 LED 的长腿（正极腿，称为阳极）连接到电阻器的另一端。将 LED 的短脚（负极脚，称为阴极）连接到电路板 GND，

大多数 Arduino 板已在板本身的引脚 13 上连接了一个 LED。如果您在没有连接硬件的情况下运行此示例，您应该会看到 LED 闪烁。

## 代码
我们可以通过检查当前时间点和上次时间点的差值，来开启或者关闭LED，以实现闪烁效果。

编译并运行示例
```shell
cargo build
cargo run
```
完整代码如下：

src/main.rs
```rust
/*!
 * Blink an LED without using the delay_ms() function.
 */
#![no_std]
#![no_main]

use panic_halt as _;
use arduino_uno_example::utils::{millis_init,millis};

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut led = pins.d13.into_output();
    const INTERVAL: u32 = 1000;
    let mut previous_time = 0;

    millis_init(dp.TC0);

    // Enable interrupts globally
    unsafe { avr_device::interrupt::enable() };

    loop {
        let current_time = millis();

        if current_time - previous_time >= INTERVAL {
            previous_time = current_time;
            led.toggle();
        }
    }
}
```