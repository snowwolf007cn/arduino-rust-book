# 读取ASCII字符串
解析以逗号分隔的整数字符串以淡出 LED。

此草图使用read_byte()函数从串口缓冲区里读取输入的字节，并使用heapless::Vec来创建buffer来存储输入的ASCII字符串的字节，输入字节以LF即'\n'换行作为终止符。之后用split()函数按逗号分割字符串，并将其转换为u8整型。字符串由非字母数字字符分隔的值。人们通常使用逗号来表示不同的信息（这种格式通常称为逗号分隔值或 CSV），但其他字符（例如空格或句点）也可以使用。这些值被解析为整数并用于确定 RGB LED 的颜色。您将使用VS Code串行监视器将“5,220,70”等字符串发送到开发板以更改灯光颜色。

## 硬件要求
- Arduino板卡
- 共阴极RGB LED
- 3个220欧姆电阻
- 连接线
- 面包板

## 电路
您需要四根电线来构成上面的电路。一根电线将电路板的地线GND（如果是共阳极则需连接到5V电压）连接到RGB LED的最长引脚。您应该转动 LED，使最长的引脚位于右侧第二个引脚上。
将RGB LED放在面包板上，最长的引脚作为从顶部数第三个引脚。检查特定LED的数据表以验证引脚，但它们应该是 G，B，V+和R。因此，地线GND的电线应连接顶部的第二个引脚。 使用剩余的电线，将红色阴极连接到引脚9，将绿色阳极连接到引脚10，将蓝色阳极连接到引脚11，与电阻串联。 具有共阴极的RGB LED共用一个公共地线引脚（共阳极的共用一个公共电源引脚）。共阴极LED你需要将PWM引脚置于高电平点亮LED，共阳极则相反，以在LED两端产生电压差。因此，通过 set_duty() 发送0（共阳极则是255）会关闭 LED，而值255（共阳极则是0）会以全亮度打开 LED。在下面的代码中，您将在草图方面使用一些数学，因此您可以发送与预期亮度相对应的值。
### 电路图
![读取ASCII字符串](images/rgb_led_connection.png "读取ASCII字符串" =400x)

## 代码
创建3个PWD_LED连接，需要注意的是，如果这里我们不先创建timer1，那么直接在into_pwm内部创建timer，会因为都要使用dp.TC1而报错。
```rust
let timer1 = Timer1Pwm::new(dp.TC1, Prescaler::Prescale64);
let mut red_pin = pins.d9.into_output().into_pwm(&timer1);
red_pin.enable();
let mut green_pin = pins.d10.into_output().into_pwm(&timer1);
green_pin.enable();
let timer2 = Timer2Pwm::new(dp.TC2, Prescaler::Prescale64);
let mut blue_pin = pins.d11.into_output().into_pwm(&timer2);
blue_pin.enable();
```
创建字符串字节缓冲区，缓冲区设置为512个字节，按照UNO的技术规范，RX缓冲区的大小为64个字节，但是如果字符串很长，超过这个大小，我们需要先将缓冲区内的字节及时读出，以避免缓冲区满了后丢失字节。关于丢失字节的问题，可以参见这个讨论[Arduino Uno Serial read seems to be dropping bytes #248](https://github.com/Rahix/avr-hal/issues/248)
```rust
let mut buffer: Vec<u8, 512> = Vec::new();
```
我们在读取到`b'\n'`时，对应的ascii为`10u8`，开始处理，并在处理结束后，清空缓冲区，以接收新的输入内容。我们这里假设，用户的输入间隔远高于处理速度，所以，在下一次RX缓冲区被填满之前，不需要清空buffer，现实情况一般如此。但是，如果存在高速率的M2M的场景，不能做这样的假设，需要更复杂的处理。

通过用","对buffer内的字节分割分组获得一个迭代器，依序使用迭代器内的元素得到RGB的三个字节切片。
```rust
let mut iter = buffer.split(|num| *num == b',');

let red_byte = iter.next().unwrap_or_default();
let green_byte = iter.next().unwrap_or_default();
let blue_byte = iter.next().unwrap_or_default();
```
之后，解析切片内的字节得到RGB的duty值（整形）
```rust
let red = u8::from_str_radix(str::from_utf8(red_byte).unwrap_or_default(), 10)
    .unwrap_or_default();
let blue = u8::from_str_radix(str::from_utf8(blue_byte).unwrap_or_default(), 10)
    .unwrap_or_default();
let green = u8::from_str_radix(str::from_utf8(green_byte).unwrap_or_default(), 10)
    .unwrap_or_default();
```
当然，我们这里可以采用更有效和更省空间的方法，就是直接使用一个指针和一个u8切片来来遍历一次buffer，并且在每次碰到`b','`时解析对应的字节(u8类型)并赋给对应的变量。当我们需要处理很长的ASCII字符串的时候，可以考虑使用这种方式来编写一个宏，相关内容参照Rust语言的文档，可以自行尝试一下。

最后，设置LED的duty值，我们就可以点亮它了。
```rust
red_pin.set_duty(red);
blue_pin.set_duty(blue);
green_pin.set_duty(green);
```
我们还可以将处理后的结果打印到串口以进行调试。
```rust
ufmt::uwriteln!(&mut serial, "RGB({},{},{})", red, green, blue).unwrap_infallible();
```

编译并运行示例
```shell
cargo build
cargo run
```
打开VS Code的串行监视器，输入类似于127,32,64并将换行符设置成LF，你可以看到，LED灯随着输入值的不同而显示不同的颜色。

完整代码如下:

src/main.rs
```rust
/*!
 * Reading a serial ASCII-encoded string.
 *
 * This sketch demonstrates the heapless::Vec type as buffer, heapless::Vec::split(), u8::from_str_radix()function.
 *
 * It looks for an ASCII string of comma-separated values.
 *
 * It parses them into ints, and uses those to fade an RGB LED.
 */
#![no_std]
#![no_main]

use core::{str, u8};

use arduino_hal::{
    default_serial, pins,
    prelude::*,
    simple_pwm::{IntoPwmPin, Prescaler, Timer1Pwm, Timer2Pwm},
    Peripherals,
};
use heapless::Vec;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);

    let timer1 = Timer1Pwm::new(dp.TC1, Prescaler::Prescale64);
    let mut red_pin = pins.d9.into_output().into_pwm(&timer1);
    red_pin.enable();
    let mut green_pin = pins.d10.into_output().into_pwm(&timer1);
    green_pin.enable();
    let timer2 = Timer2Pwm::new(dp.TC2, Prescaler::Prescale64);
    let mut blue_pin = pins.d11.into_output().into_pwm(&timer2);
    blue_pin.enable();

    let mut buffer: Vec<u8, 512> = Vec::new();
    loop {
        let b = serial.read_byte();

        if b == b'\n' {
            let mut iter = buffer.split(|num| *num == b',');

            let red_byte = iter.next().unwrap_or_default();
            let green_byte = iter.next().unwrap_or_default();
            let blue_byte = iter.next().unwrap_or_default();

            let red = u8::from_str_radix(str::from_utf8(red_byte).unwrap_or_default(), 10)
                .unwrap_or_default();
            let blue = u8::from_str_radix(str::from_utf8(blue_byte).unwrap_or_default(), 10)
                .unwrap_or_default();
            let green = u8::from_str_radix(str::from_utf8(green_byte).unwrap_or_default(), 10)
                .unwrap_or_default();
            buffer.clear();
            red_pin.set_duty(red);
            blue_pin.set_duty(blue);
            green_pin.set_duty(green);
            ufmt::uwriteln!(&mut serial, "RGB({},{},{})", red, green, blue).unwrap_infallible();
        } else {
            buffer.push(b).unwrap();
        }
    }
}
```
