# 红外发射器
(由于Infrared库需要依赖旧版本的embedded-hal，而avr-hal的pwm并没有为实现PwmPin Trait，因为他已经在embedded-hal 1.0版本中被移除了，所以，目前没有找到更好的办法实现红外发射)