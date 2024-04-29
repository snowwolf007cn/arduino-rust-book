# 控制舵机
控制舵机旋转

舵机是一种电机，它使用一个反馈系统来控制电机的位置。可以很好掌握电机角度。大多数舵机是可以最大旋转180°的。也有一些能转更大角度，甚至360°。舵机比较多的用于对角度有要求的场合，比如摄像头，智能小车前置探测器，需要在某个范围内进行监测的移动平台。又或者把舵机放到玩具，让玩具动起来。还可以用多个舵机，做个小型机器人，舵机就可以作为机器人的关节部分。所以，舵机的用处很多。

由于avr-hal没有像C那样提供9G舵机的类库，所以，在本示例中，我们需要手动编写一个控制它。本例使用的9G舵机可以在旋转180度。根据舵机数据表，该舵机使用PWM控制，位置“0”（1.5 ms 脉冲）位于中间，“90”（~2ms 脉冲）位于一直向右的位置，“-90”（~1ms 脉冲）一直向左，信号频率为50Hz。

而因为，UNO缺乏数模转换(DAC)输出模块，所以，我们需要自定义的PWM，通过使用定时器Timer1调用中断来自定义波形生成模式WGM来生成控制舵机的波形进行实现。

## 硬件要求
- Arduino板卡
- Micro Servo 9G舵机
- 连接线

## 电路
按照舵机的数据表，舵机有棕红橙三根不同颜色的线，分别对应GND、+5V和PWM控制信号，控制信号我们选引脚9。

按照舵机数据表的说明，我们需要在引脚9上输出一个50Hz的变动占空比，即PWM信号来控制不同时间点舵机的旋转角度。

关于Arduino Uno的PWM的详细教程，可以参见[Secrets of Arduino PWM](https://docs.arduino.cc/tutorials/generic/secrets-of-arduino-pwm/)

关于计时器说明可以参考[ATMega328P数据表](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf)

定时器是一种[中断](https://circuitdigest.com/microcontroller-projects/arduino-interrupt-tutorial-with-examples)。 它就像一个简单的时钟，可以测量事件的时间间隔。 每个微控制器都有一个时钟（振荡器），比如在 Arduino Uno 中它是 16Mhz。 这对速度负责。 时钟频率越高，处理速度越高。 定时器使用计数器，该计数器根据时钟频率以一定的速度进行计数。 Arduino UNO时钟以16MHz运行。计数器的一个刻度值表示1 / 16,000,000秒（~63ns），跑完1s需要计数值16,000,000。Arduino UNO有3种功能不同的定时器：
1. Timer0：8位定时器。用来执行delay(), millis()这些函数。
2. Timer1：16位定时器。用来运行本例的舵机控制。
3. Timer2：8位定时器。比如用来运行C类库中的tone()函数。
### 定时器寄存器
定时器寄存器用来修改寄存器的配置。
1. 定时器/计数器控制寄存器(TCCRnA/B)：

该寄存器保存定时器的主要控制位，用于控制定时器的预分频器。它还允许使用 WGM 位控制定时器的模式。

帧格式：
|TCCR1A|7|6|5|4|3|2|1|0|
|------|-|-|-|-|-|-|-|-|
||COM1A1|COM1A0|COM1B1|COM1B0|COM1C1|COM1C0|WGM11|WGM10|
|TCCR1B|7|6|5|4|3|2|1|0|
|------|-|-|-|-|-|-|-|-|
||ICNC1|ICES1|-|WGM13|WGM12|CS12|CS11|CS10|

预分频器：

TCCR1B 中的 CS12、CS11、CS10 位设置预分频器值。预分频器用于设置定时器的时钟速度。 Arduino Uno 的预分频器为 1、8、64、256、1024。
|CS12|CS11|CS10|描述
|---|---|---|---|
|0|0|0|没有时钟源（定时器/计数器停止）|
|0|0|1|未分频|
|0|1|0|8预分频|
|0|1|1|64预分频|
|1|0|0|256预分频|
|1|0|1|1024预分频|
|1|1|0|T1 引脚上的外部时钟源。时钟在下降沿。|
|1|1|1|T1 引脚上的外部时钟源。时钟在上升沿。|

2. 定时器/计数器寄存器（TCNTn）:

该寄存器用于控制计数器值并设置预加载器值。

所需时间（以秒为单位）的预加载器值的公式： \\[ TCNTn = 65535 –（16 \times 10^{10} \times 秒数/预分频器值）\\] 
要计算 2 秒时间内定时器 1 的预加载器值： \\[TCNT1 = 65535 – (16 \times 10^{10} \times 2/1024) = 34285 \\]

### 定时器中断
#### 输出比较寄存器（OCRnA/B）：
当输出比较匹配中断发生时，中断服务ISR (TIMERx_COMPy_vect)被调用，并且TIFRx寄存器中的OCFxy标志位将被设置。 该ISR通过设置TIMSKx寄存器中 OCIExy中的启用位来启用。 其中TIMSKx是定时器中断屏蔽寄存器。

当定时器达到比较寄存器值时，相应的输出被切换。
#### 定时器中断捕获：
接下来，当定时器输入捕捉中断发生时，将调用中断服务 ISR (TIMERx_CAPT_vect)，并且 TIFRx（定时器中断标志寄存器）中的 ICFx 标志位将被置位。 通过设置 TIMSKx 寄存器中 ICIEx 中的使能位来启用此 ISR。

定时器可以在溢出和/或与任一输出比较寄存器匹配时生成中断。
### 电路图
![控制舵机](images/servo-connection.png "控制舵机" =400x)

## 代码
这里我们需要利用定时器来触发定时中断以向舵机发送转角数据。

按照舵机数据表，我们首先需要得到一个50Hz的中断脉冲，通过改变每个中断期间的占空比，方波的宽度来控制舵机的角度。我们可以在每次方波发出后等待1/50s，即20ms来接受下一次的脉冲信号。我们可以通过定时器中断寄存器来配置64预分频，波形生成模式为0b11，表示此时使用Fast PWM模式，生成的是锯齿波，每个duty cycle可以有0-255种值，即占用255个tick，因此，波形频率为16M/64/256=976.56Hz。分频后的时钟频率为250kHz，每个tick为4µs。那么，按照舵机的数据表，我们需要设置输出的tick值在100-600之间，即0.4-2.4ms之间。0-180度的对应的具体的测试值，最终为120-600。如果我们选择的分频数高于这个值，比如256分频，那么每个tick就会上升到16µs，输出的tick值就在50-150之间了。而我们需要180个角度值，缺乏足够的分辨率。
```rust
tc1.tccr1b
    .write(|w| w.wgm1().bits(0b11).cs1().prescale_64());
```
而因为simple pwm的set_duty只能接受u8类型的值，即0-255，而我们需要输入的值为u16类型，按照手册，我们只能选用可以输出16位值的寄存器，也就是Timer1（Timer0是8位寄存器，Timer2也是8位寄存器但是带有异步模式）。而因为Timer1的输出比较匹配A输出到数字引脚9上，而Timer1对应的另一个引脚是引脚10是外部源的输入捕获输入引脚。因此，我们只能将舵机连接到引脚9上。
```rust
pins.d9.into_output();
```
上述代码中，`wgm1().bits(0b11)`代表，设置了tccr1b中的WGM的模式配置位3,2，为1，WGM在AVR中有15种工作模式，因此用4个位表示，另外两个位在tccr1a中，为模式配置位1和0，在下面这段代码中，我们设置这两位的值为1和0，那么，WGM工作模式就设置为了1110，即14。14表示Fast PWM，输出的是锯齿波。而wgm和com1a的配置，共同决定了波形生成器的工作模式，所以我们还需要设置com1a的值。match_clear()设置了com1a的值为2，这个时候在匹配前波形为低电平，匹配后，波形为高电平，正好符合我们驱动舵机的需要。
```rust
tc1.tccr1a
    .write(|w| w.wgm1().bits(0b10).com1a().match_clear());
```
我们需要把触发中断输出的频率变为50Hz，即需要976.56/50=19.53，差不多每20个波触发一次比较输出中断。因此，根据"减一原则"我们需要设置输入捕获寄存器(Input Capture Register)的值为20*256-1=5119。这个值设置了在波形生成器在14的工作模式下设置了定时器计数器的TOP值，这是OC1A的一个特性，这个值决定了波形生成的周期，这样我们就得到了一个50Hz的Fast PWM锯齿波。
```rust
tc1.icr1.write(|w| w.bits(5119));
```

按照上面的说明，我们需要设置一个宽度为在1ms-2ms之间并且被等分为180份的duty序列，来控制舵机的运动角度，序列的每个值对应1度。而输出序列值，我们要通过设置定时器输出比较寄存器OCR1A来写入当前输出的duty值。
```rust
for degree in (0..=180).chain((0..179).rev()) {
    let duty = degree as f32 * ((600.0 - 120.0) / 180.0) + 120.0;
    ufmt::uwriteln!(
        &mut serial,
        "degree:{},duty:{}",
        degree,
        ufmt_float::uFmt_f32::Two(duty.clone())
    )
    .unwrap_infallible();
    tc1.ocr1a.write(|w| w.bits(duty as u16));
    delay_ms(20);
}
```

编译并运行示例
```shell
cargo build
cargo run
```
此时，我们可以看到，舵机开始移动到0度，之后每次移动∼1度
完整代码如下：

src/main.rs
```rust
/*!
 * Servo Control
 * 
 * Sweep a standard SG90 compatible servo from its left limit all the way to its right limit and back.
 *
 * Because avr-hal does not have a dedicated servo driver library yet, we do this manually using
 * timer TC1.  The servo should be connected to D9 (AND D9 ONLY!  THIS DOES NOT WORK ON OTHER PINS
 * AS IT IS).
 *
 * As the limits are not precisely defined, we undershoot the datasheets 1ms left limit and
 * overshoot the 2ms right limit by a bit - you can figure out where exactly the limits are for
 * your model by experimentation.
 *
 */
#![no_std]
#![no_main]

use arduino_hal::{
    default_serial, delay_ms, pins, prelude::_unwrap_infallible_UnwrapInfallible, Peripherals,
};
use avr_device::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);

    // Important because this sets the bit in the DDR register!
    pins.d9.into_output();

    // - TC1 runs off a 250kHz clock, with 5000 counts per overflow => 50 Hz signal.
    // - Each count increases the duty-cycle by 4us.
    // - Use OC1A which is connected to D9 of the Arduino Uno.
    let tc1 = dp.TC1;
    tc1.icr1.write(|w| w.bits(5119));
    tc1.tccr1a
        .write(|w| w.wgm1().bits(0b10).com1a().match_clear());
    tc1.tccr1b
        .write(|w| w.wgm1().bits(0b11).cs1().prescale_64());

    loop {
        for degree in (0..=180).chain((0..179).rev()) {
            let duty = degree as f32 * ((600.0 - 120.0) / 180.0) + 120.0;
            ufmt::uwriteln!(
                &mut serial,
                "degree:{},duty:{}",
                degree,
                ufmt_float::uFmt_f32::Two(duty.clone())
            )
            .unwrap_infallible();
            tc1.ocr1a.write(|w| w.bits(duty as u16));
            delay_ms(20);
        }
    }
}
```




