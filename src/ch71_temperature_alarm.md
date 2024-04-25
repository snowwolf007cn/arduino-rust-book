# 温度报警器
当温度超过阈值的时候蜂鸣器会报警

当温度到达我们设定的限定值时，报警器就会响。我们可以用于厨房温度检测报警等等，各种需要检测温度的场合。这个项目中，除了要用到蜂鸣器外，还需要一个LM35温度传感器。

## 硬件要求
- Arduino板卡
- LM35温度传感器
- 蜂鸣器
- 连接线
- 面包板

## 电路
将蜂鸣器的一端接在数字引脚8上，另一端直接接地。

在接LM35温度传感器时，注意三个引脚的位置，有LM35字样的一面面向自己，从左至右依次接5V、Analog 0、GND。注意不要将正负极接反。

### 电路图
![温度报警器](images/temperature-alarm.png "温度报警器" =400x)

## 代码
设置温度报警阈值
```rust
const TEMP_THRESHOLD: f32 = 30.0;
```
依公式将传感器数值转换为温度
```rust
let temp = sensor_value * 5.0 / 10.24;
```
编译并运行示例
```shell
cargo build
cargo run
```
之后打开VS Code串行监视器，我们会在终端上看到当前测量到的温度。

完整代码如下：

src/main.rs
```rust
/*!
 * Temperature Alarm
 *
 * This example shows how to read value from a LM35 temperature sensor.
 * When the temperature is above the TMEP_THRESHOLD, the buzzer will ring.
 */
#![no_std]
#![no_main]

use arduino_hal::{
    default_serial, entry, pins, prelude::_unwrap_infallible_UnwrapInfallible, Adc, Peripherals,
};
use panic_halt as _;
use ufmt_float::uFmt_f32;

#[entry]
fn main() -> ! {
    const TEMP_THRESHOLD: f32 = 30.0;

    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);
    let mut adc = Adc::new(dp.ADC, Default::default());

    let temp_input = pins.a0.into_analog_input(&mut adc);
    let mut buzzer_output = pins.d8.into_output();
    loop {
        let sensor_value = temp_input.analog_read(&mut adc) as f32;
        let temp = sensor_value * 5.0 / 10.24;
        ufmt::uwriteln!(&mut serial, "temp:{}", uFmt_f32::Two(temp)).unwrap_infallible();
        if temp > TEMP_THRESHOLD {
            buzzer_output.toggle();
        }
    }
}
```