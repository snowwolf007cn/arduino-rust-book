# 模拟输入，串行输出
读取模拟输入引脚，映射结果，然后使用该数据来调暗或调亮 LED。

此示例向您展示如何读取模拟输入引脚，将结果映射到 0 到 255 的范围，使用该结果设置输出引脚的脉宽调制 (PWM) 以调暗或调亮 LED，并将值打印在VS Code的串行监视器或者终端。

## 硬件要求
- Arduino 板卡
- 电位器
- 红色LED
- 220欧姆电阻

## 电路
将电位计的一个引脚连接到 5V，中心引脚连接到模拟引脚 0，其余引脚接地。接下来，将 220 欧姆限流电阻连接到数字引脚 9，并串联一个 LED。 LED 的长正引脚（阳极）应连接到电阻器的输出，而较短的负引脚（阴极）应接地。

### 电路图
![模拟输入，串行输出](images/analog_in_out_serial.png "模拟输入，串行输出" =400x)

## 代码
在下面的草图中，声明两个引脚分配（我们的电位器的模拟 0 和 LED 的数字 9）和两个变量（sensor_value 和 output_value）后，创建一个串口连接。

接下来，在主循环中，sensor_value 被分配来存储从电位计读取的原始模拟值。 Arduino 的  analog_read范围为0到1023，而 set_duty_cycle_percent 范围仅为 0 到 100，因此，在使用电位计调暗 LED 之前，需要将电位计的数据转换为更小的范围。

转换这个值：
```rust
let output_value = (sensor_value * 100/1023) as u8;
```
输出值被指定为等于电位计的缩放值。 map() 接受五个参数：要映射的值、输入数据的低范围和高值以及要重新映射到的数据的低值和高值。在这种情况下，传感器数据从其原始范围 0 到 1023 向下映射到 0 到 255。

然后，新映射的传感器数据会输出到analogOutPin，随着电位计的转动，LED 会变暗或变亮。最后，原始传感器值和缩放传感器值均以稳定的数据流发送到VS Code串行监视器窗口。

编译并运行示例
```shell
cargo build
cargo run
```

完整代码如下：

src/main.rs
```rust
/*!
 * Analog input, analog output, serial output
 *
 * Reads an analog input pin, maps the result to a range from 0 to 100 and uses
 * the result to set the pulse width modulation (PWM) of an output pin.
 * 
 * Also prints the results to the Serial Monitor.
 *
 * We are using an embedded_hal compatible version.
 */
#![no_std]
#![no_main]

use arduino_hal::prelude::_unwrap_infallible_UnwrapInfallible;
use arduino_hal::simple_pwm::*;
use embedded_hal::delay::DelayNs;
use embedded_hal::pwm::SetDutyCycle;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
    let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());

    let a0 = pins.a0.into_analog_input(&mut adc);
    let mut pwm_led = pins
        .d9
        .into_output()
        .into_pwm(&Timer1Pwm::new(dp.TC1, Prescaler::Prescale64));
    pwm_led.enable();

    let mut delay = arduino_hal::Delay::new();

    loop {
        let sensor_value = a0.analog_read(&mut adc) as u32;

        let output_value = (sensor_value * 100 / 1023) as u8;

        pwm_led.set_duty_cycle_percent(output_value).unwrap();

        ufmt::uwriteln!(
            &mut serial,
            "sensor: {}, output: {}",
            sensor_value,
            output_value
        )
        .unwrap_infallible();
        delay.delay_ms(2);
    }
}
```