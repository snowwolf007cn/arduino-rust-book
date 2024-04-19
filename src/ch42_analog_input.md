# 模拟输入
使用电位器来控制 LED 的闪烁。

在此示例中，我们使用可变电阻器（电位计或光敏电阻器），使用 Arduino 板的一个模拟输入读取其值，并相应地更改内置 LED 的闪烁率。电阻器的模拟值被读取为电压，因为这就是模拟输入的工作原理。

## 硬件要求
- Arduino板卡
- 电位器或10k欧姆光明电阻和10k欧姆电阻
- 引脚13上的内置LED或220欧电阻和红色LED

## 电路
将三根线连接到 Arduino 板。第一个从电位计的一个外部引脚接地。第二个电压从 5 伏到电位计的另一个外部引脚。第三个从模拟输入 0 到电位器的中间引脚。

对于此示例，可以使用连接到引脚 13 的电路板内置 LED。要使用附加 LED，请将其较长的引脚（正极引脚或阳极）连接到与 220 欧姆电阻串联的数字引脚 13，它是连接引脚 13 旁边的接地 (GND) 引脚的较短引脚（负引脚或阴极）。

基于光敏电阻的电路使用电阻分压器来允许高阻抗模拟输入测量电压。这些输入几乎不消耗任何电流，因此根据欧姆定律，无论电阻器的值如何，在连接到 5V 的电阻器另一端测得的电压始终为 5V。为了获得与光敏电阻值成比例的电压，需要一个电阻分压器。该电路使用一个可变电阻、一个固定电阻，测量点位于电阻中间。测量的电压 (Vout) 遵循以下公式：

__Vout=Vin*(R2/(R1+R2))__

其中 Vin 为 5V，R2 为 10k 欧姆，R1 为光敏电阻值，范围从黑暗中的 1M 欧姆到日光下的 10k 欧姆（10 流明），在亮光或阳光下小于 1k 欧姆（>100 流明）。

### 电路图
电位器

![模拟输入(电位器)](images/analog_in_out_serial.png "模拟输入(电位器)" =400x)

光敏电阻器


![模拟输入(光敏电阻)](images/photoresistor_connection.png "模拟输入(光敏电阻)" =400x)

## 代码
在此草图的开头，变量sensor_pin设置为连接电位计的模拟引脚0，led_pin设置为数字引脚9。您还将创建另一个变量sensor_value来存储从传感器读取的值。

analog_read() 将输入​​电压范围（0 至 5 伏）转换为 0 至 1023 之间的数字值。这是由微控制器内部称为模数转换器或 ADC 的电路完成的。

通过转动电位计的轴，可以改变电位计中心销（或游标）两侧的电阻值。这会改变中心引脚和两个外部引脚之间的相对电阻，从而在模拟输入处提供不同的电压。当轴沿一个方向转动到底时，中心销和接地销之间没有电阻。此时中心引脚的电压为 0 伏，analog_read() 返回 0。当轴沿另一个方向旋转到底时，中心引脚和连接到 +5 伏的引脚之间没有电阻。中心引脚的电压为 5 伏，analog_read() 返回 1023。在这之间，analog_read() 返回 0 到 1023 之间的数字，该数字与施加到引脚的电压量成正比。

该值存储在sensor_value中，用于为眨眼周期设置delay_ms()。值越大，周期越长，值越小，周期越短。该值在周期开始时读取，因此开/关时间始终相等。

编译并运行示例
```shell
cargo build
cargo run
```

完整代码如下：

src/main.rs
```rust
/*!
 * Analog Input
 * 
 * Demonstrates analog input by reading an analog sensor on analog pin 0 and
 * turning on and off a light emitting diode(LED) connected to digital pin 9.
 * The amount of time the LED will be on and off depends on the value obtained
 * by analogRead().

 */
#![no_std]
#![no_main]

use embedded_hal::delay::DelayNs;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut adc = arduino_hal::Adc::new(dp.ADC, Default::default());

    let a0 = pins.a0.into_analog_input(&mut adc);
    let mut led_pin = pins
        .d9
        .into_output();

    let mut delay = arduino_hal::Delay::new();
	loop{
        let sensor_value = a0.analog_read(&mut adc) as u32;
		led_pin.set_high();
		delay.delay_ms(sensor_value);
		led_pin.set_low();
		delay.delay_ms(sensor_value)
	}
}
```