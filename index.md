# guinea wheek

berkeley eecs class of 2022

washed first robotics alumni

embedded rust developer

my github: https://github.com/guineawheek

# things i am currently involved in

## [redux robotics](https://reduxrobotics.com)

we sell high-performing electronics to FRC teams at affordable prices.

my responsibilities are product firmware, [user SDKs](https://apidocs.reduxrobotics.com/current/java/), [documentation](https://docs.reduxrobotics.com/canandmag/), internal tooling and debugging, and other stuff that's hard to put into words.

* [canandgyro](redux/canandgyro.md) CAN-enabled IMU firmware **(Cortex-M4F, Rust/RTIC)**
* [canandmag](https://shop.reduxrobotics.com/helium-canandmag) CAN-enabled PWM encoder firmware (esp32, freertos/c++)

We are using and implementing embedded Rust across the stack and I have a lot of [random thoughts on the subject here.](redux/embedded_rust.md)

In the course of Rust development we've had to make/fork some packages, including:
* HALs for the devices we use in our products ([gd32c1x3-hal](https://github.com/guineawheek/gd32c1x3-hal), more TODO)
  * Forks of existing packages such as [bxcan](https://github.com/guineawheek/bxcan-ng) and [synopsys-usb-otg](https://github.com/guineawheek/gd32-synopsys-usb-otg) to add functionality or make them work at all due to inactive upstreams
* Ports of code from ARM's [CMSIS-DSP](https://github.com/ARM-software/CMSIS-DSP/blob/main/Source/ControllerFunctions/arm_sin_cos_f32.c) to provide better speed/accuracy tradeoffs than existing libraries can offer.


# things i did when i was in college

## [chess-playing robot](chessrobot/index)

<video controls loop autoplay muted>
    <source src="chessrobot/img/pick_and_place.mp4" type="video/mp4">
</video>

## [esp32-powered line following car](pidcar/index)

<video controls loop autoplay muted>
    <source src="pidcar/img/step.mp4" type="video/mp4">
</video>

## [computational photography](194-26)

![doug_wozniak](https://guineawheek.github.io/194-26/proj3/out/yt_morph.gif)

# things i did when i was in highschool (washed)

[FTC team 5484 lead programmer 2016-2018](https://www.youtube.com/watch?v=sxHxKHK9qTY)

[the first usable opencv binding for the FIRST Tech Challenge, EnderCV](https://github.com/guineawheek/endercv)