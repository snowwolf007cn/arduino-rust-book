# 红外接收管
使用[Crate::Infrared](https://crates.io/crates/infrared)库来接受红外控制器信号

此示例向您展示了，如何使用红外接收管接受红外控制器发出的信号，并将接受到的信号代码输出到串行监视器的终端上。

红外接收管有多种不同的规格，一般有三个引脚，VCC、GND和Y，分别连接电源、地线和数字/模拟信号引脚。具体的连接，请参考您购买的器件。

红外传输有多种不同的协议，其编解码方式因此而有所不同。本示例使用了常见的NEC协议。如果不知道控制器的协议，可以使用Arduino的IRremote库或者IRMP库编写一个接受程序，来获取协议信息。

## 硬件要求
- Arduino板卡
- 红外接收管
- 红外控制器
- 红色和绿色LED
- 2个220欧姆电阻
- 连接线
- 面包板

## 电路
使用3根线连接红外接收器，其中红线连接5v电源，黑线连接GND，以给红外接收管供电；另外，将绿线连接到数字引脚2上作为信号输入。

此外分别将连接红色和绿色LED连接到板卡的数字引脚6和7上。

### 电路图
![红外接收管](images/ir-receiver.png "红外接收管" =400x)

## 代码
Pin Change Interrupt（PCI）会在对应的引脚电平发生变化的时候产生中断请求，ATMega328P的0-23引脚都可以产生PCI请求，而PCI请求的配置分为BCD三个组。我们需要设置启用PCI引脚所在的组和屏蔽码来从引脚2接受红外信号。相应的内容可以查询ATmega328P的数据表。这里我们需要设置PCI的控制寄存器PCICR和屏蔽寄存器PCMSK。

PCICR是一个8位寄存器，具体描述如下：
- __位7..3 - Res：保留位__
	
	这些位是Atmel® ATmega328P 中未使用的位，并且始终读为零。 
- __位2 - PCIE2：引脚电平变化中断启用2__

	当PCIE2 位被置位（1）并且状态寄存器（SREG）中的I 位被置位（1）时，引脚电平变化中断2 被启用。 任何已启用的 PCINT23..16 引脚上的任何更改都会导致中断。 引脚改变中断请求对应的中断从PCI2中断向量执行。 PCINT23..16 引脚由 PCMSK2 寄存器单独启用。
- __位1 - PCIE1：引脚电平变化中断启用1__ 

	当PCIE1 位被置位（1）并且状态寄存器（SREG）中的I 位被置位（1）时，引脚电平变化中断1 被使能。 任何已启用的 PCINT14..8 引脚上的任何更改都会导致中断。 引脚改变中断请求对应的中断从PCI1中断向量执行。 PCINT14..8 引脚由 PCMSK1 寄存器单独启用。
- __位0 - PCIE0：引脚电平变化中断启用0__ 
	
	当PCIE0 位被置位（1）并且状态寄存器（SREG）中的I 位被置位（1）时，引脚电平变化中断0 被使能。 任何已启用的 PCINT7..0 引脚上的任何更改都会导致中断。 引脚改变中断请求对应的中断从PCI0中断向量执行。 PCINT7..0 引脚由 PCMSK0 寄存器单独启用。

查询数据表，我们可知，数字引脚2对应的是控制器的PD2，PCINT18。因此我们需要设置PCICR中的PCIE2位为1。
```rust
dp.EXINT.pcicr.write(|w| unsafe { w.bits(0b100) });
```
为了启用对应的引脚，我们还需要设置PCMSK2中对应PCINT18位
```rust
dp.EXINT.pcmsk2.write(|w| w.bits(0b100));
```
我们可以使用类库的Receiver类型的with_pin()方法，创建一个接收端实例，设置Receiver的接收引脚以及工作频率。
```rust
let ir = Receiver::with_pin(Clock::FREQ, pins.d2);
```
PortD上的PCI会产生一个PCINT2的ISR，因此，我们需要编写对应的中断处理函数，函数名为`PCINT2`。如果在中断请求处理时，我们得到解析到一个正确的命令，那么，我们会在中断free的阶段，将命令数据写入一个数据共享区，这样主循环就可以读取到解析后的命令对象，并打印到终端上。
```rust
#[avr_device::interrupt(atmega328p)]
fn PCINT2() {
    let recv = unsafe { RECEIVER.as_mut().unwrap() };
    let signal_led = unsafe { SIGNAL_LED.as_mut().unwrap() };
    let error_led = unsafe { ERROR_LED.as_mut().unwrap() };

    let now = CLOCK.now();

    match recv.event_instant(now) {
        Ok(Some(cmd)) => {
            avr_device::interrupt::free(|cs| {
                let cell = CMD.borrow(cs);
                cell.set(Some(cmd));
            });
            error_led.set_low();
        }
        Ok(None) => (),
        Err(_) => error_led.set_high(),
    }

    signal_led.toggle();
```
此外，我们还需要为了接收红外信号创建一个工作时钟，这个时钟定义了红外接收管的信号解析，大部分红外发射器的发射频率在38kHz，而红外接收器的工作频率应该保持可以完整解析的接收发射器的信号，可以理解为一连串的PWM信号。具体的设置可以参考对应接收管的手册。

我们这里将接收器的时钟频率设置在20kHz，使用8分频器，这样，每个时钟周期的宽度为`16000/8/20=100`，那么，每个周期可以分辨0.5µs的电平，NEC协议在Repeat标志中，会使用562.5µs的脉冲，因此信号分辨率足够了。关于工作时钟的编码，我们可以参见[编写millis函数](./ch26_write_millis_function.md)这个部分。我们设计一个`Clock`的结构体，他包含一个计数器属性用来存储当前的时钟周期数
```rust
struct Clock {
    cntr: Mutex<Cell<u32>>,
}
```
因为我们后续需要在中断处理函数中传递当前的时钟周期，以找到发射过来的信号电平并进行解析
```rust
let now = CLOCK.now();

match recv.event_instant(now) {
	...
}
```
同时，我们需要通过start()方法来设置和启用工作时钟及其中断
```rust
impl Clock {
    const FREQ: u32 = 20_000;
    const PRESCALER: CS0_A = CS0_A::PRESCALE_8;
    const TOP: u8 = 99;

	...

    pub fn start(&self, tc0: arduino_hal::pac::TC0) {
        // Configure the timer for the above interval (in PWM Phase mode)
        tc0.tccr0a.write(|w| w.wgm0().ctc());
        tc0.ocr0a.write(|w| w.bits(Self::TOP));
        tc0.tccr0b.write(|w| w.cs0().variant(Self::PRESCALER));

        // Enable interrupt
        tc0.timsk0.write(|w| w.ocie0a().set_bit());
    }

	...
}
```
并使用tick()方法更新工作时钟计数
```rust
#[avr_device::interrupt(atmega328p)]
fn TIMER0_COMPA() {
    CLOCK.tick();
}

impl Clock {
	...

    pub fn tick(&self) {
        avr_device::interrupt::free(|cs| {
            let c = self.cntr.borrow(cs);

            let v = c.get();
            c.set(v.wrapping_add(1));
        });
    }
}
```
以及now()方法获取当前的时钟周期
```rust
impl Clock {

    ...

    pub fn now(&self) -> u32 {
        avr_device::interrupt::free(|cs| self.cntr.borrow(cs).get())
    }

    ...
}
```
至此，我们就可以编写出完整的红外接收信号的处理代码了。

编译并运行示例
```shell
cargo build
cargo run
```
之后打开VS Code串行监视器，当终端上显示`Hello from Arduino and Rust!`字样的时候，说明程序已经准备好接收并解析红外信号。

我们按红外遥控器的任意键，终端上会显示类似于"Cmd: Address: 0, Command: 12, repeat: true"这样的信息。这表示，信号已经成功接收，绿色LED应该在接收信号时闪烁。其中Address表示当前设备的设备地址，Command表示按键的数字代码，repeat表示键是否被连续按下。

完整代码如下：

src/main.rs
```rust
/*!
 * Infrared
 *
 * This example shows how to use crate infrared to receive infrared signal from IR Remote controller
 * with Nec protocol by Pin Change Interrupt.
 *
 * The code for this example refers to the crate example from
 * https://github.com/jkristell/infrared/blob/master/examples/arduino_uno/src/bin/external-interrupt.rs
 *
 * Pins and functions:
 * d7: Signal led
 * d6: Error led
 *
 * d2: Infrared rx
 */
#![no_std]
#![no_main]
#![feature(abi_avr_interrupt)]

use core::cell::Cell;

use arduino_hal::{
    default_serial,
    hal::port::{PD2, PD6, PD7},
    pac::tc0::tccr0b::CS0_A,
    pins,
    port::{
        mode::{Floating, Input, Output},
        Pin,
    },
    prelude::*,
    Peripherals,
};
use avr_device::interrupt::Mutex;
use infrared::{
    protocol::{nec::NecCommand, *},
    Receiver,
};
use panic_halt as _;

type IrPin = Pin<Input<Floating>, PD2>;
type IrProto = Nec;
type IrCmd = NecCommand;

static CLOCK: Clock = Clock::new();
static mut RECEIVER: Option<Receiver<IrProto, IrPin>> = None;
static mut SIGNAL_LED: Option<Pin<Output, PD7>> = None;
static mut ERROR_LED: Option<Pin<Output, PD6>> = None;

static CMD: Mutex<Cell<Option<IrCmd>>> = Mutex::new(Cell::new(None));

#[arduino_hal::entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);

    // Monotonic clock to keep track of the time
    CLOCK.start(dp.TC0);

    let mut uno_led = pins.d13.into_output();
    let mut signal_led = pins.d7.into_output();
    let mut error_led = pins.d6.into_output();

    uno_led.set_low();
    signal_led.set_low();
    error_led.set_low();

    // Enable group 2 (PORTD)
    dp.EXINT.pcicr.write(|w| unsafe { w.bits(0b100) });

    // Enable pin change interrupts on PCINT18 which is pin PD2 (= d2)
    dp.EXINT.pcmsk2.write(|w| w.bits(0b100));

    let ir = Receiver::with_pin(Clock::FREQ, pins.d2);

    unsafe {
        RECEIVER.replace(ir);
        SIGNAL_LED.replace(signal_led);
        ERROR_LED.replace(error_led);
    }

    // Enable interrupts globally
    unsafe { avr_device::interrupt::enable() };

    ufmt::uwriteln!(&mut serial, "Hello from Arduino and Rust!\r").unwrap_infallible();

    loop {
        if let Some(cmd) = avr_device::interrupt::free(|cs| CMD.borrow(cs).take()) {
            ufmt::uwriteln!(
                &mut serial,
                "Cmd: Address: {}, Command: {}, repeat: {}\r",
                cmd.addr,
                cmd.cmd,
                cmd.repeat
            )
            .unwrap_infallible();
        }
    }
}

#[avr_device::interrupt(atmega328p)]
fn PCINT2() {
    let recv = unsafe { RECEIVER.as_mut().unwrap() };
    let signal_led = unsafe { SIGNAL_LED.as_mut().unwrap() };
    let error_led = unsafe { ERROR_LED.as_mut().unwrap() };

    let now = CLOCK.now();

    match recv.event_instant(now) {
        Ok(Some(cmd)) => {
            avr_device::interrupt::free(|cs| {
                let cell = CMD.borrow(cs);
                cell.set(Some(cmd));
            });
            error_led.set_low();
        }
        Ok(None) => (),
        Err(_) => error_led.set_high(),
    }

    signal_led.toggle();
}

#[avr_device::interrupt(atmega328p)]
fn TIMER0_COMPA() {
    CLOCK.tick();
}

struct Clock {
    cntr: Mutex<Cell<u32>>,
}

impl Clock {
    const FREQ: u32 = 20_000;
    const PRESCALER: CS0_A = CS0_A::PRESCALE_8;
    const TOP: u8 = 99;

    pub const fn new() -> Clock {
        Clock {
            cntr: Mutex::new(Cell::new(0)),
        }
    }

    pub fn start(&self, tc0: arduino_hal::pac::TC0) {
        // Configure the timer for the above interval (in PWM Phase mode)
        tc0.tccr0a.write(|w| w.wgm0().ctc());
        tc0.ocr0a.write(|w| w.bits(Self::TOP));
        tc0.tccr0b.write(|w| w.cs0().variant(Self::PRESCALER));

        // Enable interrupt
        tc0.timsk0.write(|w| w.ocie0a().set_bit());
    }

    pub fn now(&self) -> u32 {
        avr_device::interrupt::free(|cs| self.cntr.borrow(cs).get())
    }

    pub fn tick(&self) {
        avr_device::interrupt::free(|cs| {
            let c = self.cntr.borrow(cs);

            let v = c.get();
            c.set(v.wrapping_add(1));
        });
    }
}
```
