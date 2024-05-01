# 编写millis()函数

## 代码
在C++代码中，因为Arduino上没有任何类型的时钟，所以，我们只能通过millis()来获得程序开始后运行的时间。而std::time::SystemTime::now()位于标准库中，我们在嵌入式环境下无法使用，所以，我们需要自己编写一个简单函数来实现此功能。关于此函数实现的详细分析，可以参看[Write your own Arduino millis() in Rust](https://blog.rahix.de/005-avr-hal-millis/)。

因为ATMega328P的时钟频率是16MHz，如果我们需要得到一个精度在1ms的计时器，因此，计时器的工作频率在1/0.001=1kHz。

这里，我们就需要用预分频从CPU时钟得到一个频率更低的时钟。预分频器有Direct，8，64，256，1024几种。预分频后的频率为`CPU频率/预分频值`，假设预分频值选择64，即得到的时钟频率为`16M/64=250k`。因此，我们经过250次时钟tick，就是1ms。
```rust
const PRESCALER: u32 = 64;
const TIMER_COUNTS: u32 = 250;

const MILLIS_INCREMENT: u32 = PRESCALER * TIMER_COUNTS / 16000;
```
我们需要开辟一个内存空间(Cell)，记录下这个毫秒值。因为这个时钟值是全局共享的，考虑到并发访问时，对这个变量的访问必须是线程安全的，因此，我们使用一个Mutex类型来保证每次只有一个线程会读写这个值。
```rust
static MILLIS_COUNTER: avr_device::interrupt::Mutex<cell::Cell<u32>> =
    avr_device::interrupt::Mutex::new(cell::Cell::new(0));
```
接下来，我们就可以使用中断请求来触发，每经过TIMER_COUNTER次定时器的Tick，就对MILLIS_COUNTER触发一次+1操作，这样就在计时器上增加了1ms的计数。也就是说，定时器每数250个数，就恢复到0，那么我们这里就需要使用定时器的CTC模式。CTC模式可以设定一个TOP值，当计数器达到TOP值时，就会触发一次中断。因为我们的TOP值<255，即他可以是一个u8类型，所以，我们可以使用TC0，也是一个8位定时器来作为计时器。ATMega328P一共有三个不同类型的定时器，Timer0、Timer1、Timer2。Timer0和Timer2都是8位定时器，即他们tick可以count的最大数值为255，Timer2比Timer0多了异步请求的特性；而Timer1是16位定时器。因此，我们这里只需要使用Timer0作为millis()函数的定时器即可。为了实现这一点，我们需要设置TC0的控制寄存器CR（Control Register），设置使用分频值为64使用CTC模式的波形生成器，并且设置输出寄存器ocr0a的值为249（减1原则），并且在屏蔽寄存器上设置ocie0a位为1来启用输出比较中断，这样就可以得到我们需要的计时器锯齿波。
```rust
pub fn millis_init(tc0: arduino_hal::pac::TC0) {
    // Configure the timer for the above interval (in CTC mode)
    // and enable its interrupt.
    tc0.tccr0a.write(|w| w.wgm0().ctc());
    tc0.ocr0a.write(|w| w.bits(TIMER_COUNTS as u8));
    tc0.tccr0b.write(|w| w.cs0().variant(PRESCALER));
    tc0.timsk0.write(|w| w.ocie0a().set_bit());
}
```
我们可以在系统标记为暂停接受中断请求后，进行中断处理操作。即用到avr_device::interrupt::free()方法。这个方法可以接受一个执行一次(FnOnce)的函数，即每次接收到中断请求时执行一次中断处理操作。这个函数接受一个类型为CriticalSection的参数，Critical Section是一个保护内存区域，当我们需要对某个共享内存值进行修改的时候，我们可以将当前值的指针Borrow到区域上，这样外部程序就无法访问变量指针，因为这个变量所指向的内存地址已经没有了。当对保护区的数据完成操作后，随着中断处理流程完毕，保护区域的指针被释放，原始变量重新获得了数据的内存地址，那么，修改后的数据就可以访问了。
```rust
#[avr_device::interrupt(atmega328p)]
fn TIMER0_COMPA() {
    avr_device::interrupt::free(|cs| {
        let counter_cell = MILLIS_COUNTER.borrow(cs);
        let counter = counter_cell.get();
        counter_cell.set(counter.wrapping_add(1));
    })
}
```
之后，我们如果需要读取当前的毫秒数，我们只需要在中断处于free上下文的时候，读取MILLIS_COUNTER值即可。
```rust
fn millis() -> u32 {
    avr_device::interrupt::free(|cs| MILLIS_COUNTER.borrow(cs).get())
}
```
我们可以将其放在::util::millis模块里，以供后续项目使用
完整代码如下：

src/utils/millis.rs
```rust
use arduino_hal::pac::tc0::tccr0b::CS0_A;
use avr_device::interrupt::Mutex;
use core::cell::Cell;
use panic_halt as _;

/*
 * Possible Values:
 *
 * ╔═══════════╦══════════════╦═══════════════════╗
 * ║ PRESCALER ║ TIMER_COUNTS ║ Overflow Interval ║
 * ╠═══════════╬══════════════╬═══════════════════╣
 * ║        64 ║          250 ║              1 ms ║
 * ║       256 ║          125 ║              2 ms ║
 * ║       256 ║          250 ║              4 ms ║
 * ║      1024 ║          125 ║              8 ms ║
 * ║      1024 ║          250 ║             16 ms ║
 * ╚═══════════╩══════════════╩═══════════════════╝
 */
const PRESCALER: CS0_A = CS0_A::PRESCALE_64;
const TIMER_COUNTS: u32 = 249;

static MILLIS_COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

pub fn millis_init(tc0: arduino_hal::pac::TC0) {
    // Configure the timer for the above interval (in CTC mode)
    // and enable its interrupt.
    tc0.tccr0a.write(|w| w.wgm0().ctc());
    tc0.ocr0a.write(|w| w.bits(TIMER_COUNTS as u8));
    tc0.tccr0b.write(|w| w.cs0().variant(PRESCALER));
    tc0.timsk0.write(|w| w.ocie0a().set_bit());
}

#[avr_device::interrupt(atmega328p)]
fn TIMER0_COMPA() {
    avr_device::interrupt::free(|cs| {
        let counter_cell = MILLIS_COUNTER.borrow(cs);
        let counter = counter_cell.get();
        counter_cell.set(counter.wrapping_add(1));
    })
}

pub fn millis() -> u32 {
    avr_device::interrupt::free(|cs| MILLIS_COUNTER.borrow(cs).get())
}
```
之后，我们可以编写一个主程序里面测试这个库。

编译并运行示例
```shell
cargo build
cargo run
```
打开Vs Code的串行监视器，连接到板卡，开启监控终端的时间戳，我们可以看到输出的毫秒数到板卡启动的时间差与时间戳的时间差基本上是一致的，可能会有几个毫秒的差距，这是因为ufmt也需要占用串行输出，而他的损耗又比较高，在时间输出上有一定的延迟。

完整代码如下：

src/main.rs
```rust
/*!
 * Test millis function
 */
#![no_std]
#![no_main]

use arduino_hal::{default_serial, delay_ms, pins, Peripherals};
use arduino_uno_example::utils::millis::{millis, millis_init};
use avr_device::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);

    millis_init(dp.TC0);

    unsafe { avr_device::interrupt::enable() };

    loop {
        let now = millis();
        ufmt::uwriteln!(&mut serial, "now:{}", now).unwrap();
        delay_ms(1000);
    }
}
```
