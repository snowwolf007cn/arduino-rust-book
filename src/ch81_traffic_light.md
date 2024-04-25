# 交通信号灯
本示例展示了如何设置一个行人过马路的路灯

开始时，汽车灯为绿灯，行人灯为红灯，代表车行人停。一旦行人，也就是你，按下按钮，请求过马路，那么汽车灯开始由绿变黄，等待3秒，然后，行人灯由红变绿，汽车灯变红。在行人通行的过程中，设置了一个过马路的时间 cross_time，一旦到点，行人绿灯开始闪烁3秒，提醒行人快速过马路。 闪烁完毕，最终，又回到了开始的状态，汽车灯为绿灯，行人灯为红灯。

## 硬件要求
- Arduino板卡
- 2个红色LED，2个绿色LED和1个黄色LED
- 5个220欧姆电阻
- 1个按钮
- 1个10k欧姆电阻
- 连接线
- 面包板

## 电路
用1个绿色LED和1个红色LED表示行人交通灯，通过220欧电阻将正极连接到数字信号引脚7和8上。用1个绿色LED、一个黄色LED和1个红色LED表示汽车交通灯，通过220欧姆电阻将正极连接到数字信号引脚10、11和12上。

将按钮同侧的两个引脚分别连接到5V电压和通过10k欧姆电阻接地，将接地引脚同时连接到数字引脚上接收按钮发出的信号。

### 电路图
![交通信号灯](images/traffic_light.png "交通信号灯" =400x)

## 代码

编译并运行示例
```shell
cargo build
cargo run
```
之后打开VS Code串行监视器，我们会看到终端上显示字母a，而如果我们输入任意字符串后，终端上会显示类似于"0,518,324"这样的信息。

完整代码如下：

src/main.rs
```rust
/*!
 * Traffic light for vehicles and passagers
 *
 * At the beginning, the car light is green and the pedestrian light is red, indicating that cars and pedestrians
 * have stopped.
 * Once the pedestrian, that is, you, presses the button and requests to cross the road, the car lights start to
 * change from green to yellow, wait for 3 seconds, then the pedestrian lights change from red to green, and the
 * car lights turn red. During the pedestrian passage, a cross_time is set for crossing the road. Once the cross_time
 * is reached, the pedestrian green light starts to flash for 3 seconds to remind pedestrians to cross the road quickly.
 * The flashing is completed, and finally, it returns to the starting state, with the car light being green and the
 * pedestrian light being red.
 */

#![no_std]
#![no_main]

use arduino_hal::{delay_ms, entry, pins, Peripherals};
use arduino_uno_example::utils::millis::{millis, millis_init};
use panic_halt as _;

#[entry]
fn main() -> ! {
    let dp = Peripherals::take().unwrap();
    let pins = pins!(dp);

    const PREPARE_TIME: u32 = 3000;
    const CROSS_TIME: u32 = 5000;
    const WHOLE_TIME: u32 = 2 * PREPARE_TIME + CROSS_TIME;
    const CROSS_ALARM_TIME: u32 = PREPARE_TIME + CROSS_TIME;
    const BLINK_INTERVAL: u32 = 200;

    let passenger_btn = pins.d9.into_floating_input();
    let mut last_btn_is_high = false;
    let mut padestrian_red = pins.d8.into_output();
    let mut pedestrian_green = pins.d7.into_output();

    padestrian_red.set_high();
    pedestrian_green.set_high();

    let mut vehicle_red = pins.d12.into_output();
    let mut vehicle_yellow = pins.d11.into_output();
    let mut vehicle_green = pins.d10.into_output();

    vehicle_red.set_high();
    vehicle_yellow.set_high();
    vehicle_green.set_high();

    delay_ms(1000);

    pedestrian_green.set_low();
    vehicle_red.set_low();
    vehicle_yellow.set_low();

    let mut pedestrian_start_time = 0u32;
    let mut prev_time = 0u32;

    millis_init(dp.TC0);
    unsafe {
        avr_device::interrupt::enable();
    }

    loop {
        let elapsed_time = millis() - pedestrian_start_time;
        let btn_is_high = passenger_btn.is_high();

        if pedestrian_start_time == 0 || elapsed_time > CROSS_TIME + 2 * PREPARE_TIME {
            if btn_is_high != last_btn_is_high {
                if btn_is_high {
                } else {
                    pedestrian_start_time = millis();
                    vehicle_green.set_low();
                    vehicle_yellow.set_high();
                }

                last_btn_is_high = btn_is_high;
            }
        } else {
            if elapsed_time < PREPARE_TIME {
                let current_time = millis() - pedestrian_start_time;
                if current_time - prev_time > BLINK_INTERVAL {
                    vehicle_yellow.toggle();
                    prev_time = current_time;
                }
            } else if elapsed_time < CROSS_ALARM_TIME && elapsed_time >= PREPARE_TIME {
                prev_time = 0;
                vehicle_red.set_high();
                pedestrian_green.set_high();
                padestrian_red.set_low();
                vehicle_yellow.set_low();
            } else if elapsed_time >= CROSS_ALARM_TIME && elapsed_time < WHOLE_TIME {
                let current_time = elapsed_time - CROSS_ALARM_TIME;
                if current_time - prev_time > BLINK_INTERVAL {
                    pedestrian_green.toggle();
                    prev_time = current_time;
                }
            } else if elapsed_time == WHOLE_TIME {
                vehicle_green.set_high();
                vehicle_red.set_low();
                pedestrian_green.set_low();
                padestrian_red.set_high();
                prev_time = 0;
            }
        }
    }
}
```