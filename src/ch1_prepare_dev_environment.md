# 搭建开发环境
在Arduino上使用Rust需要一些额外的步骤，因为Arduino通常使用C/C++编程语言。在阅读本书之前，需要一些预备知识：
- 熟悉Rust语言。本书假设你已完整阅读过[The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html)，能够熟练使用Rust语言开发应用程序。
- 需要了解Rust Embedded开发的基本概念。本书假设你已完整阅读过[The Embedded Rust Book](https://docs.rust-embedded.org/book/intro/index.html)的Introduction部分，了解Rust进行嵌入式开发的基本架构。
本部分内容参考了

## 工具
搭建完整的开发环境，你需要预先准备一些软件和硬件

### 软件
- 一台可以编写、编译和写入程序到板卡的开发电脑
- 安装了Cargo，并用Cargo安装了cargo-generate
- 安装了nightly版本的Rust编译器
### 硬件
- Arduino Uno
本书编写时，使用的VS Code作为IDE，本书的示例演示一般使用终端或者串行监视器。

## 安装和设置

您需要一个nightly版本的Rust编译器来为AVR编译Rust代码。Rust版本可以在稍后查看模板生成的项目脚手架中的rust-toolchain.toml文件，编译时将将自动安装正确的版本。
安装依赖: 
- Ubuntu
	```shell
	sudo apt install avr-libc gcc-avr pkg-config avrdude libudev-dev build-essential
	```
- Macos
	```shell
	xcode-select --install # if you haven't already done so
	brew tap osx-cross/avr
	brew install avr-gcc avrdude
	```
- Windows

	在Windows 10和11上使用[Winget](https://learn.microsoft.com/en-us/windows/package-manager/winget/)：

	```powershell
	winget install AVRDudes.AVRDUDE ZakKemble.avr-gcc
	```

	在更早的系统上，你需要先使用Powershell安装[Scoop](https://scoop.sh/)：
	```powershell
	Set-ExecutionPolicy RemoteSigned -Scope CurrentUser # Needed to run a remote script the first time
	irm get.scoop.sh | iex
	```
	之后安装avr-gcc和avrdude
	```
	scoop install avr-gcc
	scoop install avrdude
	```
然后，安装[ravedude](https://github.com/Rahix/avr-hal/blob/main/ravedude)，一个用来集成到cargo工作流中无缝刷写您的板卡的工具：
```shell
cargo +stable install ravedude
```
## 创建项目
您可以使用github模板来创建您的项目：
```shell
cargo generate --git https://github.com/Rahix/avr-hal-template.git
```
## 运行
将您的Arduino连接到开发电脑上，修改项目配置文件，以在您的板和计算机之间以每秒 57600 位数据的速度进行串行通信，修改.cargo/cargo.toml中[targe.'cfg(target_arch = "avr")']下的runner配置
```toml
[target.'cfg(target_arch = "avr")']
runner = "ravedude uno -cb 57600 -P /dev/tty.usbmodem14101"
```
打开生成的项目中的src/main.rs文件，输入下列代码：
```rust
/*!
 * Toggle an LED on and off every second.
 *
 * This example shows you how toggle LED on and off every second.
 */
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut led = pins.d13.into_output();

    loop {
        led.toggle();
        arduino_hal::delay_ms(1000);
    }
}
```
在项目根目录的终端中执行
```shell
cargo build
cargo run
```
如果您看到板卡上的绿色LED闪烁，证明您的开发环境已经搭建成功。修改delay_ms中的值，可以使LED闪烁的频率发生变化。
## 参考内容
- [A complete guide to running Rust on Arduino](https://blog.logrocket.com/complete-guide-running-rust-arduino/)
- [avr-hal](https://github.com/Rahix/avr-hal?tab=readme-ov-file#readme)

