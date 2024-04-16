# 如何对进行按钮连线和编程
了解如何对按钮进行接线和编程来控制 LED。

当您按下按钮或开关时，它们会连接电路中的两点。当您按下按钮时，本示例将打开引脚 13 上的内置 LED。

## 硬件要求
- Arduino板卡
- 瞬时按钮或开关
- 10k欧姆电阻
- 连接线
- 面包板

## 电路
将三根电线连接到板上。前两个（红色和黑色）连接到面包板侧面的两个长垂直行，以提供对 5 伏电源和接地的访问。第三根电线从数字引脚 2 连接到按钮的一条腿。按钮的同一条腿通过下拉电阻（此处为 10K 欧姆）连接到地。按钮的另一条腿连接到 5 伏电源。

当按钮打开（未按下）时，按钮的两条腿之间没有连接，因此该引脚接地（通过下拉电阻），我们读取低电平。当按钮关闭（按下）时，它会在两个引脚之间建立连接，将引脚连接到 5 伏，以便我们读取高电平。

您还可以以相反的方式连接该电路，使用上拉电阻将输入保持为高电平，并在按下按钮时变为低电平。如果是这样，草图的行为将相反，LED 常亮，按下按钮时熄灭。

如果断开数字 I/O 引脚与所有设备的连接，LED 可能会不规律地闪烁。这是因为输入是“浮动”的——也就是说，它将随机返回高电平或低电平。这就是电路中需要上拉或下拉电阻的原因。

### 电路图
![如何对按钮进行连线和编程来控制LED](images/pushbutton_connection.png "如何对按钮进行连线和编程来控制LED" =400x)

## 代码
编译并运行示例
```shell
cargo build
cargo run
```
完整代码如下：

src/main.rs
```rust
/*!
 * Button
 *
 * Turns on and off a light emitting diode(LED) connected to digital pin 13,
  when pressing a pushbutton attached to pin 2.
 */
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut led = pins.d13.into_output();
    let d2 = pins.d2.into_floating_input().downgrade();

    loop {
        if d2.is_high() {
            led.set_high();
        } else {
            led.set_low();
        }
    }
}
```