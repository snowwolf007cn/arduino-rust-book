# 感光灯
使用光敏电阻控制LED明灭

这个示例中将介绍光敏电阻。在黑暗的环境中，光敏电阻具有非常高阻值的电阻。光线越强，电阻值反而越低。通过读取这个电阻值，就可以检查光线的亮暗了。我们这里选用的是光敏二极管，光敏二极管其实就是光敏电阻中的一种，只是它还具有正负极性。

当环境黑暗的时候，光敏电阻的阻值提高，LED变亮；当环境明亮的时候，LED变暗。

## 硬件要求
- Arduino板卡
- 光敏电阻
- 10k电阻

可选
- 220欧电阻
- LED灯
- 手电筒

## 电路
用三根线连接倾斜开关，其中红线通过10k欧姆电阻连接到5V电源，黑色连接到地线GND，且接电源的引脚也接到模拟引脚3。这里我们使用内置LED。如果需要使用外接LED，可以用220欧姆电阻和LED串联接通。注意，光敏二极管一般是反向使用的。长脚为正极，短脚为负极。

当光照变强的时候，光照电流变大，电压降低，输出的数值变小，当低于一个阈值的时候，灯就熄灭了。反之，光电流变小，电压提升，输出的数值变大，当高于一个阈值时，灯被点亮。

### 电路图
![感光灯](images/photoresistor_connection.png "感光灯" =400x)

## 代码
编译并运行示例
```shell
cargo build
cargo run
```
使用VS Code的串行监视器，你会在终端看到一个读数，当你用手电筒等强光光源照射光敏原件时，读数会变低。当读数高于阈值，本例是1000时，LED灯会被点亮；而当读数低于1000时，LED灯熄灭。

完整代码如下：

src/main.rs
```rust
/*!
 * Photosensive Light
 *
 * When the environment goes dark, the resistance of the photoresistor increases and the LED becomes brighter;
 * when the environment goes bright, the LED becomes darker.
 */
#![no_std]
#![no_main]

use arduino_hal::{
    default_serial, delay_ms, entry, pins, prelude::_unwrap_infallible_UnwrapInfallible, Adc,
    Peripherals,
};
use panic_halt as _;

#[entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);
    let mut adc = Adc::new(dp.ADC, Default::default());

    let mut led = pins.d13.into_output();
    let sensor = pins.a0.into_analog_input(&mut adc);

    loop {
        let sensor_value = sensor.analog_read(&mut adc);
        ufmt::uwriteln!(&mut serial, "sensor value:{}", sensor_value).unwrap_infallible();

        if sensor_value >1000 {
            led.set_high();
        } else {
            led.set_low();
        }
        delay_ms(500);
    }
}
```
