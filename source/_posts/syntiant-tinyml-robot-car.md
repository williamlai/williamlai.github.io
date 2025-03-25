---
title: '[tech] Syntiant TinyML Robot Car'
date: 2022-11-01 17:15:01
categories: tech
tags:
- Syntiant
- NDP101
- AI
- Car
- Robot
- TB6612FNG
---

I built a small robot car that can be controlled by voice. I use the Syntiant TinyML development board to recognize voice commands "go" and "stop," and then control the TB6612FNG DC motor drive accordingly.

<!-- more -->

# Syntiant TinyML

[Syntiant](https://www.syntiant.com/) is known for a provider of edge AI processor soltuion with always-on and low-power. TinyML board is their evaluation board that we can buy on [Digi-Keys](https://www.digikey.com/en/products/detail/syntiant-corp./SYNTIANT%2520TINYML/15293343?utm_adgroup=Evaluation%20Boards%20-%20Embedded%20-%20MCU%2C%20DSP&utm_source=google&utm_medium=cpc&utm_campaign=Shopping_Product_Development%20Boards%2C%20Kits%2C%20Programmers_NEW&utm_term=&utm_content=Evaluation%20Boards%20-%20Embedded%20-%20MCU%2C%20DSP&gclid=CjwKCAjwh4ObBhAzEiwAHzZYU41yvERYEPr_n76biaxilYPi7i3pOVApiZlteuIWdJEWnDL6IRch2RoCaTIQAvD_BwE).

On the TinyML board is NDP101, a deep learning inference processor that costs less than **140uW** and is more efficient than traditional CPU and DSP. It's suitable for audio and sensor applications. The following picture is its pinout.

{% asset_img tinyml_pinout.png %}

In the picture, there is another MCU. It's Microchip SAMD21 which is the same one as Arduino MKR ZERO. It plays a role in the application process, providing more application peripherals.

Most of the time, SAMD21, the application processor, can stay in sleep or deep sleep mode until NDP101 inference a different result, then NDP101 wakes SAMD21 up to handle the change.

The NDP101 is also tiny, and so is the TinyML board. So it can fit into small devices.

# Edge Impulse

Machine learning and deep learning are complicated and require specialties. MCU is another domain that is also needed specialties. Edge ML combines these two domains, which makes it harder to build excellent applications.

[Edge Impulse](https://www.edgeimpulse.com/) is a development platform for machine learning on edge devices and tries to make things easier. Edge Impulse provides a whole development process, including data collection, model selection, training model, and SDK for deploying to the target edge device. The edge device I'm going to use is Syntiant TinyML.

# Robot Car Example

In the example, I want to build a robot car that can be controlled by voice, and it should be small.

The TinyML tutorial uses Edge Impulse and makes it able to recognize audio in two words, "go" and "stop." So I'm gonna apply this tutorial to build a small robot car. Here are the steps.

1. Follow the TinyML tutorial and upload code to TinyML.
2. Find a DC motor controller.
3. Add code to control motors.
4. Assemble parts and test them.

## TinyML tutorial

On the [TinyML web page](https://www.syntiant.com/tinyml), there are [TinyML Board Tutorial (for Mac)](https://www.syntiant.com/_files/ugd/64a391_12ad538fb5864834854209cd8fa7aa78.pdf) and [TinyML Board Tutorial (for Windows)](https://www.syntiant.com/_files/ugd/64a391_8acb0f3c9faa49c9a9257f1a01beb95a.pdf).

In the tutorial, there is a project on Edge Impulse with a fine dataset and a trained model. We can just clone it and use it on our project.

The dataset is an audio dataset with three labels, *"go," "stop," and "z_openset."* The *"z_openset"* category is for audio that does not belong to *"go"* and *"stop."* The training/testing dataset ratio is around 80/20, which is the recommended ratio from Edge Impulse.

The model in the project follows Edge Impules' pipeline template. The following picture shows the pipeline in the project.

{% asset_img rc_model.png %}

The first stage is for the dataset, and the last state is for output features. The second stage is "Audio (Syntiant)," which is the log of MFE (Mel-Filterbank Energy). In this stage, it extracts features from raw samples, which are the inputs of the third stage. The third stage, it's a neural network classification model, which is a model in Keras.

{% asset_img nn_classifier_keras.png %}

The inference requires large computations. In real-time recognition, the device has to complete the inference within the window size. For example, to complete an inference on a 968ms audio sample, the device has to make an inference within a 484ms window size. And the TinyML board is capable of completing the inference within a window size.

In the deployment, Edge Impulse provides either binaries or source code. Both use Arduino CLI for uploading binaries through a serial interface. The SAMD21 has a USB host HW and provides a serial interface when a USB connects to the TinyML board. SAMD21 is also responsible for uploading the trained model to NDP101. Here I upload the binary first to check if everything works fine.

So when the tutorial finishes, it'll show a green light when it recognizes "go" and a red light when it recognizes "stop."

## DC motor controller

In this model, the robot car can only go or stop, and I can select a DC motor controller that supports only one motor and then apply the same action on two motors. But I prefer to use a motor controller that controls two motors for different actions so that it can do more if I have a model that supports "turn left," "turn right," and "backward."

L298N is a common choice for controlling 2 DC motors because it can drive a motor that consumes a large current, which means this motor controller is for applications with heavy loading.

However, the TinyML board is quite small because the NDP101 is small so it can fit into a tiny space. Here I choose **TB6612FNG** as my DC motor controller. TB6612FNG is small, too. And it can drive two motors with a 1.2A maximum each.

{% asset_img TB6612FNG.jpg %}

{% raw %}
<div style="text-align: center;">TB6612FNG image from Sparkfun</div>
{% endraw %}

These are input control pins.

* PWMA, PWMB: it controls the speed of the motor A and motor B.
* AIN1, AIN2, BIN1, BIN2: These pins control the motor's rotation.
    * AIN1=0, AIN2=0: Motor stops.
    * AIN1=1, AIN2=0: Motor rotates in clockwise.
    * AIN1=0, AIN2=1: Motor rotates in counter-clockwise.
    * AIN1=1, AIN2=1: Motor short breaks.

These are output control pins.

* A01, A02, B01, B02: These pins output signals to motors.
    * A01=0, A02=0: Motor stops.
    * A01=1, A02=0: Motor rotates in clockwise.
    * A01=0, A02=1: Motor rotates in counter-clockwise.

These are power-related pins.
* VM: Power source of motors and motor controller. It accepts 2.5V~13.5V. Typically we'll use a voltage larger than 3.7V.
* VCC: It provides the power source for the application.
* STBY: It's a standby pin. When STBY=0, it'll turn off VCC. TB6612FNG checks the STBY after boot up. If the application cannot set STBY to 1 within this window, then TB6612FNG will turn off VCC.
* GND

## Add code to control motors

There are five pins available on TinyML for peripheral usage. These pins are connected to SAMD21 so we can program them just like Arduino MKR ZERO.

There are also pins on test points. But I don't want to weld the TinyML board and make it look unpleasant, so I won't use these pins.

{% asset_img tinyml_pinout2.jpg %}

We can check the [MKR ZERO variant table](https://github.com/arduino/ArduinoCore-samd/blob/master/variants/mkrzero/variant.cpp) to use these pins in Arduino API.

* PA02: It's pin 15 in Arduino API, and it's a analog pin which can be ADC or DAC.
* PA21, PA20: They are pin 7 and pin 6 in Arduino API. They can be programmed as GPIO, PWM, and UART. In Edge Impulse SDK, they have been programmed as a UART2 instance for printing debug information.
* PA23, PA22: They are pin 1 and pin 0 in Arduino API. They can be programmed as GPIO, PWM, and UART.

Since there needed to be more pins to control the 2 DC motor controller fully, my first try was adding another MCU to expand the pins. So I picked another Arduino MKR ZERO board and wrote a log parser to parse the message from TinyML's UART. In this way, I can keep Edge Impulse SDK unchanged. Then I added code for motor control. I put TinyML, TB6612FNG, Arduino MKR ZERO, 2 motors with wheels, and battry all together, and it worked well on the breadboard. Then I assembled them as a robot car. But...they're too heavy so the small motors couldn't move. That means I need to remove parts, including the expanded MCU (Arduino MKR Zero).

I then considered using TinyML only and using two pins only for the "go" and "stop" movements. I made the following arrangement for the DC motor controller.

* STBY=1: Connect the STBY to constantly HIGH value.
* AIN1=1, AIN2=0, BIN1=1, BIN2=0: Make both two motors always rotate clockwise.
* PWMA, PWMB: Connect these two pins to TinyML. PWMA=0 & PWMB=0 means stop, and PWMA>0 & PWMB>0 means go.

In this way, I can't make it turn left/right or backward, but at least the motors may afford the loading.

I attempted to use pin PA23 and pin PA22, but it seems it has been used by the USB host. If I configure PA23 and PA22 successfully, the USB host won't work. From the logic analyzer, there are some periodical signals. So give up using these two pins.

To use PA21, and PA20, which are the UART pins, I need to find a way to bypass or disable `Serial2`. The source code `syntiant.cpp` configures Serial2 with pins 6 and 7 in the following way.

```cpp
            pinPeripheral(6, PIO_SERCOM_ALT);
            pinPeripheral(7, PIO_SERCOM_ALT);
```

I modified it to use other pins.

```cpp
            pinPeripheral(0, PIO_SERCOM_ALT);
            pinPeripheral(1, PIO_SERCOM_ALT);
```

So the `Serial2` now using pins 0 and 1. Then I can use pins 6 and 7 as PWM with following sample code.

```cpp
#define MOTOR_A_PWM_PIN 6
#define MOTOR_B_PWM_PIN 7

static const int speed = 255;

void on_classification_changed(const char *event, float confidence, float anomaly_score) {

    // here you can write application code, e.g. to toggle LEDs based on keywords
    if (strcmp(event, "stop") == 0) {
        // Toggle LED
        digitalWrite(LED_RED, HIGH);
        analogWrite(MOTOR_A_PWM_PIN, 0);
        analogWrite(MOTOR_B_PWM_PIN, 0);
    }

    if (strcmp(event, "go") == 0) {
        // Toggle LED
        digitalWrite(LED_GREEN, HIGH);
        analogWrite(MOTOR_A_PWM_PIN, speed);
        analogWrite(MOTOR_B_PWM_PIN, speed);
    }
}

void setup(void)
{
    pinMode(MOTOR_A_PWM_PIN, OUTPUT);
    pinMode(MOTOR_B_PWM_PIN, OUTPUT);
    analogWrite(MOTOR_A_PWM_PIN, 0);
    analogWrite(MOTOR_B_PWM_PIN, 0);

    syntiant_setup();
}

void loop(void)
{
    syntiant_loop();
}
```

## Assemble parts and test them

Here is the list of parts.

* Syntiant TinyML board
* TB6612FNG
* 2 '130 size' DC hobby motors
* 2 29.5mm wheels
* 1 Caster Bearing Wheel
* Popsicle sticks - I use popsicle sticks as the body of the robot car. They are light and easy to assemble and reshape.
* 500mAH LiPo battery
* 1 2P Rocker switch

The following diagram shows the connection logic.

{% asset_img rc_connection.jpg %}

Here are some pictures of my robot car. The wiring is ugly, but putting all the wires in a small place is pretty hard. I used 3 popsicle sticks to make a triangle-shaped body and white glue to glue the motors. I also use tapes to stable other parts.

{% asset_img car_side.jpg %}

{% asset_img car_front.jpg %}

So the car is pretty small. The heaviest loading would be the battery and the Dupont wires. It'll be lighter if I use single-core wires, but it'll make the parts hard to reuse.

And here is the demo video.

[https://youtube.com/shorts/gU9Gy3q3VfY](https://youtube.com/shorts/gU9Gy3q3VfY)

[![IMAGE ALT TEXT](http://img.youtube.com/vi/gU9Gy3q3VfY/0.jpg)](https://youtube.com/shorts/gU9Gy3q3VfY)

I tried to play it on carpet and texture wooden floor, but it won't move due to friction.

# Conclusion

It's a pity that I can't control it to do backward, left, and right turn. But it's hard to train a voice model, especially with motor noise. Even if I have the model, getting enough pins is still a problem.

I have lots of fun studying this project and making it works. And TinyML is an excellent device for realizing edge ML. Good job, Syntiant.

