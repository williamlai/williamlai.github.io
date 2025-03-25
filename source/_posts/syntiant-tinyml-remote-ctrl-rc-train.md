---
title: Syntiant TinyML Remote Ctrl RC Train
date: 2022-11-29 04:31:14
categories: tech
tags:
- Syntiant
- NDP101
- AI
- Car
- Robot
- DRV8871
- Seed Studio
- Xiao BLE Sense
- BLE
---

It's an improved project for ["Syntiant TinyML Robot Car"](https://williamlai.github.io/2022/11/01/syntiant-tinyml-robot-car/). I'll reuse the current "go" and "stop" voice commands model, and apply it to an RC train.

<!-- more -->

# Ideas

A train can have at least two moves, go and stop. It makes a train a perfect example to the [Syntiant RC Go and Stop model](https://studio.edgeimpulse.com/public/42868/latest) because it recognizes two words, "go" and "stop".

On my first try, I put everything on the train, including the Syntiant TinyML board, motor controller, and batteries. But it failed for some reasons.

On my second try, which succeeded, I made two groups. The first group is for controlling, including the TinyML board and another board with wireless functions for sending commands. The second group is with the train, including a motor control board and a board with wireless functions for receiving requests.

# Select a train

My kids already have LEGO trains and tracks, so I can reuse them. When I disassembled the LEGO trains, I found I might need modify and weld the  train's control board. But my kids might be unhappy if they found I break their toys. So I decided to buy another LEGO train and modify it to my needs.

The LEGO train I bought is [LEGO Power Functions Train Motor 88002](https://www.amazon.com/LEGO-Power-Functions-Train-Motor/dp/B004RBKZ2Q). It's supposed to work with other LEGO Power Functions modules. At least it requires a battery module and a control module. However, I already decided to disassemble and rework it. So I can provide power and control signals by myself.

Inside the train, there is one brushless motor that drives two gears.

{% asset_img lego_88002_motor.jpg %}

I cut the lines and reworked them as Dupont connectors. There are four lines for almost every LEGO Power Function module. The middle two lines are control and data lines, and the left and right lines are power and ground. HOWEVER, as for motor control, the left and right lines are not connected to the motor. We use only the middle two lines for powering and controlling the motor.

{% asset_img lego_88002_lines.jpg %}

The middle two lines represent motor control 1 and 2. Each line is capable of accepting certain voltage signals. Since LEGO Power Functions batter usually needs six AAA batteries, the power source is slightly larger than 9V. So the motor should be at least able to accept a 10V signal.

| line 1 | line 2 |                  Action |
|-------:|-------:|------------------------:|
|     0V |     0V |                    Stop |
|  0~10V |     0V |        Rotate Clockwise |
|     0V |  0~10V | Rotate Counterclockwise |

It spins fater when we provide higher voltage to it.

# Select a motor drive

In the ["Syntiant TinyML Robot Car"](https://williamlai.github.io/2022/11/01/syntiant-tinyml-robot-car/) project, I used a TB6612FNG motor driver to drive two motors. Since I only needed to drive one motor this time, I chose another motor driver.

[DRV8871](https://www.adafruit.com/product/3190) is TI's solution for driving one motor. It's relatively smaller than L298N. 

{% asset_img drv8871.jpg %}

At the bottom, there are input pins.

* VM: It means VMotor, the power source.
* GND
* IN1, IN2: These are PWM control pins.

On the top, there are output pins.
* POWER +/-: DRV8871 can provide power to another board. The voltage is the same as VM at the '+' pin. And the '-' pin represents the GND.
* MOTOR 1/2: These are motor lines 1 and 2.

The PWM signals at IN1 and IN2 decide the voltage at motor line 1 and line 2. It's like it converts digital signals into analog signals.

# First Try

In the first try, I put everything together on the train. The following diagram is the connection.

{% asset_img tinyml_only.jpg %}

In the diagram, there are two batteries: one 9V battery for driving the motor and one 3.7V lithium battery for the TinyML board. The reason for the extra battery is TinyML can't use 9V as the power source, and 3.7V is not enough for driving the motor because the train is not light enough.

The TinyML control the DRV8871 with 2 PWM lines as follows.

| Voice command | IN1                                 | IN2                             |
|--------------:|:------------------------------------|:--------------------------------|
|            Go | analogWrite(MOTOR_PWM_1_PIN, speed) | analogWrite(MOTOR_PWM_2_PIN, 0) |
|          Stop | analogWrite(MOTOR_PWM_1_PIN, 0)     | analogWrite(MOTOR_PWM_2_PIN, 0) |

The `speed` here depends on the battery level. I set the `speed` to 100 when 9V battery is full.

The following picture is the assemble result.

{% asset_img tinyml_only_train.jpg %}

Then I put the train on the track. I said, "Go," and it went expectedly. However, it won't stop no matter how many times I said "Stop". I examed it without the motor, and it worked on the "Go" and "Stop" voice commands. It means the issue is related to the noise caused by the motor.

There are two approaches to deal with the noise issue.

* Suppress the motor noise: I referred to the article ["Dealing with motor Noise"](https://www.pololu.com/docs/0J15/9), and tried to suppress the noise by adding capacitors to the control lines of the motor. But it did not work because both the motor and gears produced noise. To suppress the gear noise, I might need to add some lubricating oil or move the TinyML board away from the gears. I don't want to make a mess or make the train super huge, so I give up suppressing the noise.
* Retrain the AI model with the noise: I tried to retrain the model by adding some datasets with the motor noise. But this won't work because there is only one microphone on TinyML, and removing the noise requires at least two microphones. And not to mention that the voice might not be working in the far field.

For these reasons, I gave up on the first try.

# Second Try

Since the TinyML board has only one microphone and can only work in the near field and the train is far away, I started to look for a wireless solution. There is plenty of choice for a wireless solution. I chose BLE as the wireless solution and selected [Seeed Studio XIAO nRF52840 Sense](https://www.seeedstudio.com/Seeed-XIAO-BLE-Sense-nRF52840-p-5253.html) as the development board.

## Seeed Studio XIAO nRF52840 Sense 

Seeed Studio XIAO is a series of development boards with small sizes. And [Seeed Studio XIAO nRF52840 Sense](https://www.seeedstudio.com/Seeed-XIAO-BLE-Sense-nRF52840-p-5253.html) (Xiao nRF52840 , in short) is the one with Nordic nRF52840 and onboard IMU and PDM. The design makes it easy to validate the power consumption because there are not many parts on the board. It also has plenty of pins.

It supports [Arduino APIs](https://wiki.seeedstudio.com/XIAO_BLE/) including Arduino BLE library, so we can implement BLE applicaion easily.

## Voice Controller

The voice controller consists of one Xiao nRF52840 sense and the TinyML board.

{% asset_img voice_controller.jpg %}

The TinyML board stays with the same image that was downloaded from Edge Impulse. The inference results are printed out to UART.

The Xiao nRF52840 sense receives the inference results from UART, parse it, and sends the command over BLE. The Xiao nRF52840 sense here plays a BLE central role. So it'll scan the desired BLE peripheral and connect to it.

I put them on a breadboard because I don't have to consider the size of the voice controller.

I put my Arduino code in [ble_peripheral_rc.ino](https://gist.github.com/williamlai/26b0e1f5df0d3379aac10c235d134dfd).

## Request Receiver

The Request receiver consists of another Xiao nRF52840 sense and the DRV8871 motor driver.

{% asset_img request_receiver.jpg %}

There are two batteries based on a similar reason on the first try: Xiao nRF52840 sense can't use 9V as its power source.

The Xiao nRF52840 sense here acts as BLE peripheral role and waits for a BLE central to connect. Once the connection is established, it starts receiving commands. And it drives the DRV8871 with two PWM lines with an accordingly command.

I still use a small breadboard here because it's hard to place the XiaonRF52840 sense in a stable position.

I put my Arduino code in [ble_central_ctrl.ino](https://gist.github.com/williamlai/375a90940bb4fa78f2e3baecc0aa959b).

## Test Result

Here is the demo video. It works as expected. And it's more fun to place it with LEGO blocks.

[https://www.youtube.com/watch?v=37cVn3LxgBY&t=2m59s](https://www.youtube.com/watch?v=37cVn3LxgBY&t=2m59s)

# Conclusion

I would say it's a pity that the TinyML board binds a SAMD21 MCU. The SAMD21 here merely acts as an intermediate processor. If the TinyML board can act as a solo module and requires another MCU to be a complete project, it will be more flexible and have less power consumption. It's also okay if a development board has a Syntiant solution with another wireless MCU. Let's hope there will be more similar products in the Maker market.

