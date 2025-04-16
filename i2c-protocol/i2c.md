---
description: >-
  Here are some basic notes & primers about the standard implementation &
  inner-workings of the I2C interface.
---

# I2C

## Communication speed

* Normal
  * Fast
* High-speed

## Basic features

* Two-wire communication
* Half-duplex
  * Bidirectional communication (only one direction at a time)
* Multi-Master/Slave
* Clock stretching
  * Slave devices can hold the clock line (SCL) to allow more time for processing data
* Addressing
* Pull-up resistors
* Arbitration
  * Ensure data integrity when multiple masters attempting to use the bus simultaneously
    * Detect & resolve conflicts
    * Ensure data integrity
* ACK/NACK (acknowledge / not acknowledge)

## I2C implementation spec-sheet

### From TI (Texas Instruments)

{% file src="../.gitbook/assets/slva689 (I2C Bus Pullup Resistor Calculation).pdf" %}

{% file src="../.gitbook/assets/slva704 (Understanding the I2C Bus).pdf" %}

### From NXP

{% file src="../.gitbook/assets/UM10204.pdf" %}
[I2C-bus specification and user manual](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
{% endfile %}

## I2C primers & use-full resources

### I2C Primer

A very nice primer that explains how the I2C protocol works and some special considerations that we should take work working with it. On the surface it seems like a very simple 2 wire protocol but some features such as multi-master/slave, clock stretching, etc. (& maybe error detection/correction?) makes implementation not as straightforward as it seems.

{% embed url="https://www.i2c-bus.org/i2c-primer" %}

## Debugging I2C

### Waveform analysis

[https://www.ti.com/lit/an/slyt770/slyt770.pdf](https://www.ti.com/lit/an/slyt770/slyt770.pdf)



