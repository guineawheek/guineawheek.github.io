# guinea wheek

embedded rust/c++ developer

berkeley eecs class of 2022

washed first robotics alumni and volunteer

my github: https://github.com/guineawheek

# about myself

I like working on things that move, or things that get used in things that move.

I believe that embedded Rust definitely has potential for a strong future and I want to help it cook.

# things i am currently involved in

## [redux robotics](https://reduxrobotics.com)

As a part-time side project, we sell high-performing electronics to FIRST Robotics Competition teams at affordable prices.

My responsibilities are product firmware, [user SDKs](https://apidocs.reduxrobotics.com/current/java/), [documentation](https://docs.reduxrobotics.com/canandmag/), internal tooling and debugging, and other stuff that's hard to put into words.

Firmware I have worked on includes so far:

* [Canandgyro](https://shop.reduxrobotics.com/boron-canandgyro) CAN-enabled high-performance IMU firmware shipping a Rust firmware **(Cortex-M4F, RTIC)**
* [Canandmag](https://shop.reduxrobotics.com/helium-canandmag) CAN-enabled PWM encoder firmware (esp32, freertos/c++)

We are using and implementing Rust across the stack. 
I am in the process of writing a series of posts here describing some of the challenges we faced and ways we overcame them:
 * [Footguns with Embedded Rust and Memory-Mapped I/O](redux/rust_mmio.md)


In the course of Rust development we've had to make/fork some packages, including:
* HALs for the devices we use in our products ([gd32c1x3-hal](https://github.com/guineawheek/gd32c1x3-hal), more TODO)
  * Forks of existing packages such as [bxcan](https://github.com/guineawheek/bxcan-ng) and [synopsys-usb-otg](https://github.com/guineawheek/gd32-synopsys-usb-otg) to add functionality or make them work at all due to inactive upstreams
* Ports of code from ARM's [CMSIS-DSP](https://github.com/ARM-software/CMSIS-DSP/blob/main/Source/ControllerFunctions/arm_sin_cos_f32.c) to provide better speed/accuracy tradeoffs than existing libraries can offer.


# things i did when i was in college

## [chess-playing robot](chessrobot/index)

<video controls loop autoplay muted>
    <source src="chessrobot/img/pick_and_place.mp4" type="video/mp4">
</video>

I basically played project manager for this particular project as it was done smack-dab in the middle of the pandemic and required significant coordination to make sure all the parts came together to build a physical product. 

## [esp32-powered line following car](pidcar/index)

I was the firmware lead on a line-following RC racecar project class. This project included various challenges such as:
* 50Hz PID to control steering and velocity
* a line sensor whose communication protocol can be best described as "SSI but the CIPO line is an analog reading and the exposure depends on the clock frequency"
* responding to input over TCP over the esp32's wifi

<video controls loop autoplay muted>
    <source src="pidcar/img/step.mp4" type="video/mp4">
</video>

[Here's a link to the final oral presentation PDF](EECS192_Oral_Presentation.pdf)

[Here's it following an S-curve](https://www.youtube.com/watch?v=RaqXHxoh_rU)

[And here's it doing a step response](https://www.youtube.com/shorts/3Fo1jC7Y9VY)

## [computational photography](194-26)

I took a class that had you write entire computational photography projects basically from scratch every 2 weeks. 
These projects covered a wide variety of topics including:

* [recursive image pyramids](194-26/proj1)
* [digital signal processing as applied to blurring, blending, and scaling images](194-26/proj2)
* [morphing faces from keypoints](194-26/proj3)
* [merging images via homographies](194-26/proj4)
* [facial detection with convnets](194-26/proj5)
* [basic 3d rendering](194-26/projfin)
* [projecting 3d models of a room from a back wall and a vanishing point](194-26/projfin)

![doug_wozniak](https://guineawheek.github.io/194-26/proj3/out/yt_morph.gif)

# things i did when i was in highschool (washed)

[FTC team 5484 lead programmer 2016-2018](https://www.youtube.com/watch?v=sxHxKHK9qTY)

[the first usable opencv binding for the FIRST Tech Challenge, EnderCV](https://github.com/guineawheek/endercv)