---
title: Measure Power Consumption With Nordic PPK2 on AmebaD
date: 2022-10-14 10:32:46
categories: tech
tags:
- Nordic
- PPK2
- Aemba
- AmebaD
- MCU
- Power Consumption
---

I bought a Nordic Power Profiler kit II for measuring power consumption for MCUs.

<!-- more -->

# Why I bought Nordic PPK2

I've used Keysight 34465A and Keithley DMM6500 digital multimeter to measure MCU's power consumption. Those devices have ampere precision to uA and a sample rate of around 1K (one millisecond per sample).

The ampere precision is essential for MUCs with a deep sleep mode that costs less than 100uW. And the sample rate is important to integrate a period of power consumption, especially for MCUs with wireless capability.

These multimeters can do the job. But they are too expensive for individual makers or developers and too heavy to carry around.

So, one day, when I was browsing Nordic products, I found the [**Power Profiler Kit II**](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2). It immediately caught my attention because it has a high sample rate (10 microseconds per sample) and high precision (200nA). The most important is, IT'S CHEAP. (Well, compared to the expensive digital multimeter).

Then I bought it immediately.

# How to use it

In the following section, I'm gonna use PPK2 to measure the power consumption of [Realtek AmebaD](https://www.amebaiot.com/en/amebad/#rtk_amb21).

## Install nRF Connect for Desktop

Install ["nRF Connect for Desktop"](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-desktop). It supports different OS, including Windows, Linux, and macOS.

After installation, open "nRF Connect for Desktop," and select "Install" for "Power Profiler."

{% asset_img install_power_profiler.png %}

Then open "Power Profiler."

{% asset_img open_power_profiler.png %}

## Set up PPK2 configuration

Make sure our PPK2 has connected to the PC/laptop. Click "SELECT DEVICE" at the top left corner, and select "PPK2."

{% asset_img nrf_connect_select_device.png %}

On the left panel, we need to put the following settings.

{% asset_img ppk2_configuration.png %}

* Source meter: PPK2 can be a pure Ampere meter or a power supply with the meter. Here we choose "Source meter."
* Set supply voltage to 3300mV.
* Enable power output: **TURN THIS OPTION OFF NOW**. We'll turn it on when we start measuring.
* Timestamps: Turning it on will show a timestamp on the X-axis, which is easier to know the time/current relation.
* Digital channels: PPK2 can be a simple logic analyzer with 8 channels. If we want to know how a GPIO interrupt wakes up a system, it can show the GPIO signal and the current changes in the same diagram. We turn if off since we won't use it in this test.

## Connect boards

Now we can connect Nordic PPK2, AmebaD, and an UART adapter as in the following picture.

{% asset_img connect_boards.png %}

* PPK2: There are 2 micro USB slots on PPK2. Plug a micro USB from the PC/laptop to "USB DATA/POWER" on PPK2. (NOTE: If the target device consumes more than 500mA, then we also need to plug the "POWER ONLY")
* AmebaD: Please note that we don't plug in any micro USB to it because there are other components (e.q. FT232) that may consume power.
    * UART Adapter: There are 2 jumpers for UART adapters. The original setting is to connect the UART to the onboard FT232. We're gonna use our UART adapter for issuing AT commands, so remove these 2 jumpers and connect them to our UART adapter.
    * VIN: There is a jumper at the VIN pin for connecting to the onboard power source. We'll use PPK2's power source instead. So remove the jumper, and connect the VIN of AmebaD to the VOUT of PPK2.
    * GND: Connect the GND of the UART adapter to AmebaD. We also need to connect the GND of PPK2 to AmebaD to complete a whole circuit cycle.

Here is a more detailed picutre of AmebaD connection.

{% asset_img connection_of_amebad.png %}

## Start measuring

We can start the test by clicking "Start" on the left panel. It'll begin sampling. Then turn on the "Enable power output" option to power on AmebaD.

Here is my measurement result and its current changes.

{% asset_img overall.png %}

1. At 1st bootup, it connected to WiFi AP with a fast connection. Then it idled until entering WiFi LPS.
2. I typed a command over UART Adapter and made AmebaD enter deep sleep for around 10 seconds.
3. As it woke up from a timeout of deep sleep, it connected to WiFi AP again.

Here are some details by zoom in the current diagram. We can see the WiFi RX idle costs around 67 mA.

{% asset_img bootup.png %}

### WiFi LPS

Here are the details of WiFi LPS. We can see it costs 30.17 mA when WiFi RF is off. We can select a range to see its average current. And we can see the WiFi LPS cost 30.81 mA in average.

{% asset_img wifi_lps.png %}

### Deep Sleep

Here is the result of deep sleep. It's 39.75 uA in average, which is different from the official results. It might be caused by the power leakage from the UART adapter, or from IC deviation.

{% asset_img deep_sleep.png %}

### WiFi TX

Here is the detail of when AmebaD sends a WiFi TX package. We can see PPK2 makes several samples for it. The sample interval is 10 microseconds.

{% asset_img high_sampling_rate.png %}

# Conclusion

I believe Nordic PPK2 is an excellent ampere meter with a good SW. But it wonders me that there is not much discussion on the Internet since it was released last year. I can only see a few Youtube videos that introduce it. I hope this article helps you if you're looking for an ampere meter for MCUs.

At least I'm glad I finally have a good ampere meter.
