# 震动探测
使用震动传感器触发中断，并处理中端信号

震动传感器，我们从名字中应该就可以判断，传感器能够检测震动中的物体。我们用什么来做震动传感器呢？那就是滚珠开关。滚珠开关，其内部含有导电珠子，器件一旦震动，珠子随之滚动，就能使两端的导针导通。

通过这个原理，我们可以做一些小玩具结合起来。只要传感器检测到东西震动，就会有信号输出。这里，我们想通过滚珠开关做个简单的震动传感器，并把震动传感器和LED的结合，当传感器检测到物体震动时，LED亮起，停止震动时，LED关闭。

## 硬件要求
- Arduino板卡
- 倾斜开关
- 220欧姆电阻
- 连接线
- 面包板

## 电路
用三根线连接倾斜开关，其中红线连接到5V电源，黑色线通过220欧姆电阻连接到地线GND，且接地脚接到数字引脚3。

### 电路图
![震动检测](images/detecting_vibration.png "震动检测" =400x)

## 代码
在没有任何打扰的情况下，程序正常运行，让LED一直处于关闭。如果板子被摇晃，就触发中断。按照[LED灯带](./ch61_range_for_led.md)中的关于中断的说明，我们知道，连接在引脚3上的开关触发的中断类型为INT1。因此，我们需要对这个终端进行配置。

首先，我们配置中断信号的检测为上沿(RISING)。在arduino_hal中，需要写入外部中断控制寄存器(External Interrupt Control Register)，并设置外部中断检测寄存器的值为RISING即0x03。这一段代码可以理解为，我们要求外部中断控制寄存器为我们对外部中断检测寄存器执行一段操作，对应的引脚3，INT1的中断检测为isc1()函数返回的对象，并且设置值为0x03。这样，INT1中断请求就会在引脚电平，由低变高时对MCU发出。
```rust
dp.EXINT.eicra.write(|w| w.isc1().bits(0x03));
```
然后，我们配置中断服务路由启用INT1中断。在arduino_hal中，需要写入一段运行指令到外部中断屏蔽寄存器(External Interrupt Mask Register)，要求将int1()返回的对象对应的屏蔽位设置为1，int1()返回的是INT1中断的请求对象。
```rust
dp.EXINT.eimsk.write(|w| w.int1().set_bit());
```
之后，我们全局启用中断调度。
```rust
unsafe {
    interrupt::enable();
}
```
此时，中断服务路由会根据这个配置来接受中断请求，并且执行对应的任务。因此，我们只需震动开关，而不需要在代码里主动读取任何引脚数据，就可以自动找到对应的执行步骤了。将状态值设为true。
```rust
#[interrupt(atmega328p)]
fn INT1() {
    unsafe {
        STATE.store(true, Ordering::SeqCst);
    }
}
```
而当主程序检测到状态值为true的时候，就会将LED点亮并重新将STATE设为false。如果之后震动持续，则后续会继续产生中断，将状态值设为true，继续触发点亮LED的动作。而震动停止后，因为STATE为false，所以LED熄灭。这里，我们因为需要在`INIT`和`main`之间共享STATE变量，所以我们将STATE设置为全局变量，我们选了使用原子布尔类型来作为这个全局变量的类型。
```rust
static STATE: AtomicBool = AtomicBool::new(false);
```

编译并运行示例
```shell
cargo build
cargo run
```
当你晃动面包板的时候，LED灯会被点亮；停止晃动后，LED灯熄灭。

完整代码如下：

src/main.rs
```rust
/*!
 * Blinks a 4 leds in sequence on pins D3 - D6. When an external interrupt on D2/INT0 comes in
 * the sequence is reversed.
 * 
 * Note: The use of the either crate requires the deactivation of std to use it in core. See the Cargo.toml 
 * in this directory for details.
 */
#![no_std]
#![no_main]
#![feature(abi_avr_interrupt)]

use panic_halt as _;
use core::sync::atomic::{AtomicBool, Ordering};
use arduino_hal::port::{mode, Pin};
use either::*;

static REVERSED: AtomicBool = AtomicBool::new(false);

fn is_reversed() -> bool {
    REVERSED.load(Ordering::SeqCst)
}

#[avr_device::interrupt(atmega328p)]
fn INT0() {
    let current = REVERSED.load(Ordering::SeqCst);
    REVERSED.store(!current, Ordering::SeqCst);
}

fn blink_for_range(range: impl Iterator<Item = u16>, leds: &mut [Pin<mode::Output>]) {
    range.map(|i| i * 100).for_each(|ms| {
        let iter = if is_reversed() {
            Left(leds.iter_mut().rev())
        } else {
            Right(leds.iter_mut())
        };
        iter.for_each(|led| {
            led.toggle();
            arduino_hal::delay_ms(ms as u16);
        })
    });
}

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    // thanks to tsemczyszyn and Rahix: https://github.com/Rahix/avr-hal/issues/240
    // Configure INT0 for falling edge. 0x03 would be rising edge.
    dp.EXINT.eicra.modify(|_, w| w.isc0().bits(0x02));
    // Enable the INT0 interrupt source.
    dp.EXINT.eimsk.modify(|_, w| w.int0().set_bit());

    let mut leds: [Pin<mode::Output>; 4] = [
        pins.d3.into_output().downgrade(),
        pins.d4.into_output().downgrade(),
        pins.d5.into_output().downgrade(),
        pins.d6.into_output().downgrade(),
    ];

    unsafe { avr_device::interrupt::enable() };

    loop {
        blink_for_range(0..10, &mut leds);
        blink_for_range((0..10).rev(), &mut leds);
    }
}
```