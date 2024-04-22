# 具有ASCII编码输出的串行调用和响应(握手)
使用调用和响应（握手）方法发送多个变量，并在发送前对值进行 ASCII 编码。

此示例演示了使用调用和响应（握手）方法从 Arduino 板到计算机的基于字符串的通信。
该草图在启动时发送一个 ASCII 字符串并重复该操作，直到收到计算机的串行响应。然后，它以 ASCII 编码的数字形式发送三个传感器值，以逗号分隔并以换行符和回车符终止，并等待计算机的另一个响应。
您可以使用VS Code串行监视器查看发送的数据。下面的示例将传入字符串上的逗号并再次将字符串转换为数字。 
你也可以修改此示例使用二进制值发送。虽然作为 ASCII 编码字符串发送需要更多字节，但这意味着您可以轻松地为每个传感器读数发送大于 255 的值。在串行终端程序中读取也更容易。

## 硬件要求
- Arduino板卡
- 2个模拟传感器（电位器、光电管、FSR等）
- 按钮
- 3个10k欧姆电阻
- 连接线
- 面包板

## 电路
使用用作分压器的 10K 欧姆电阻将模拟传感器连接到模拟输入引脚0和1。将按钮或开关连接到数字I/O引脚 2，并使用10K欧姆电阻作为接地参考。

### 电路图
![具有ASCII编码输出的串行调用和响应（握手）](images/handshaking.png "具有ASCII编码输出的串行调用和响应（握手）" =400x)

## 代码
前面我们使用过`serail.read_byte()`或者`nb::block!(serial.read())`来阻塞主进程运行，等待串口信息输入。而在本例中，因为我们需要在未接受到信息的时候向串口发送数据或者做其它处理，因此，不能采用阻塞的方式来运行程序。我们需要用非阻塞方式接受信息，非阻塞方式在读取信息时，会返回一个`nb::Result`，如果读取到字节，他返回该字节，如果未读取到字节，他返回一个Err。这与C/C++ API通过available()方法检查RX的缓冲区未读取的字节数。因此，我们可以用match来判断Result，如果是Ok，则运行响应的处理过程，如果是Err，则向串口发出一个字节`"a"`。

编译并运行示例
```shell
cargo build
cargo run
```
之后打开VS Code串行监视器，我们会看到终端上显示字母a，而如果我们输入任意字符串后，终端上会显示类似于"0,518,324"这样的信息。

完整代码如下：

src/main.rs
```rust
/*
 * Serial Call and Response in ASCII
 *
 * This program sends an ASCII A (byte of value 65) on startup and repeats that
 * until it gets some data in. Then it waits for ASCII string in the serial port, and
 * sends three ASCII-encoded, comma-separated sensor values, truncated by a
 * linefeed and carriage return, whenever it gets bytes in.
 */
#![no_std]
#![no_main]

use arduino_hal::{default_serial, delay_ms, pins, prelude::*, Adc, Peripherals};
use heapless::Vec;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);
    let mut serial = default_serial!(dp, pins, 57600);
    let mut adc = Adc::new(dp.ADC, Default::default());

    let push_btn = pins.d2.into_floating_input();
    let a0 = pins.a0.into_analog_input(&mut adc);
    let a1 = pins.a1.into_analog_input(&mut adc);
    let mut buffer: Vec<u8, 512> = Vec::new();
    loop {
        let b = serial.read();
        match b {
            Ok(b) => {
                if b != b'\n' {
                    buffer.push(b).unwrap_or_default();
                } else {
                    let mut state = 0u8;
                    if push_btn.is_high() {
                        state = 1u8;
                    }
                    ufmt::uwriteln!(
                        &mut serial,
                        "{},{},{}",
                        state,
                        a0.analog_read(&mut adc),
                        a1.analog_read(&mut adc)
                    )
                    .unwrap_infallible();

                    delay_ms(1000);
                }
            }

            Err(_) => {
                ufmt::uwriteln!(&mut serial, "a").unwrap_infallible();
            }
        }
    }
}
```