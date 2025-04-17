# I2C Verilog Implementation

## Plan of attack

1. Implement in Verilog & use GTK wave simulation
2. Implement Verilator to debug
3. Once debug & testing is finished, implement to FPGA
4. Measure & verify outputs of FPGA using oscilloscope

## Module Topology

There are three levels of abstraction:

(from top to down, **highest** abstraction to **lowest** abstraction)

1. `I2C master top` (this is the top module)
   1. This module abstracts all the nitty gritty stuff to some basic 8-bit registers and state registers (enabling the user to easily write/read data to and from the I2C bus)
2. `I2C master byte` (handles the byte-level operations of I²C communication)
   1. Including the handling of data transfer, acknowledgments, and bus control
3. `I2C master bit` (handles the low-level control of the bus)
   1. Manages transitions of the SDA and SCL lines
   2. Takes higher-level commands from the **`i2c_master_byte_ctrl`** module and converts them into bit-level transitions that adhere to the I²C protocol
   3. Responsible for other things such as signal filtering, clock skewing, and arbitration handling

> Using this top-down approach (similar to the 5 layer networking approach), we can easily manage high level and low-level implementations; their logic is mostly isolated. Adding or modifying features in the future would be easy and straightforward.

### Layers

1. Top module (I2C controller)
2. Byte controller
3. Bit controller

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption><p>A reference of the 5 layer networking model (analogy purposes)</p></figcaption></figure>

#### Bit controller

Think of this as the physical/data-link layer of the networking model. It handles the reading and interpreting of raw signals from the data & clock lines (SDA & SCL), ensures that the signal is meta

#### Byte controller

#### Top module (I2C controller)





## 1. Top module



## 2. Byte controller

### Transferring a single byte

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

## 3. Bit controller

They bit controller has many important responsibilities. Because the infrastructure is layers of abstraction, we need to make things easy and simple to interact with on the byte layer.

The bit controller offers a simple interface for the byte controller to read or write bits to or from the bus. It also handles things like pre-scaled clock, detecting line busy, ensuring signals are stable, etc.

### Bit-level signal basics

Here are the few basic things we must learn about how SDA & SCK work together to generate signals:

* (1) Start & (2) stop conditions
* (3) Repeated start condition
* (4) ACK & (5) NACK

> Note that conditions 1\~3 is driven by the master, while 4\~5, SDA is temporarily driven by the slave to send a ACK/NACK signal. After which, it has handed back to the master to continue driving the signal line.

> Reading: Sec.2 I2C Interface of TI's _Understanding the I2C Bus (SLVA704–June 2015)_

#### Start & stop condition

Start condition:

* SDA: HIGH to LOW
* SCL: Stays LOW

Stop condition:

* SDA: LOW to HIGH
* SCL: Stays HIGH

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption><p>From TI I2C documentation, pg. 4</p></figcaption></figure>

#### Repeated start condition

#### Acknowledge (ACK) & Not Acknowledge (NACK)

<div><figure><img src="../.gitbook/assets/image (16).png" alt="" width="129"><figcaption><p>Example of an ACK</p></figcaption></figure> <figure><img src="../.gitbook/assets/image (15).png" alt="" width="150"><figcaption><p>Example of NACK</p></figcaption></figure></div>

> Refer to explanations & examples of the on page 5\~6 of TI's _Understanding the I2C Bus (SLVA704–June 2015)_

#### Some considerations

* Because SCL line determine how fast things are being transmitted, I2C is actually has a really flexible data transfer speed (govern by the SCL signal), of course it's not blazingly fast but it is very flexible (灵活).

### Open-drain interface

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

### Tristate buffer (Three-state buffer)

You can refer to the FPGA implementation & experiments page for more information. Attached down below\~

{% content-ref url="../i2c-fpga-implementation-and-experiments.md" %}
[i2c-fpga-implementation-and-experiments.md](../i2c-fpga-implementation-and-experiments.md)
{% endcontent-ref %}

### Some Mechanisms

* Arbitration signal
  * > **Arbitration** signals are crucial for maintaining data integrity and preventing collisions on shared communication buses
  * In the case that an arbitration is detected, we raise the arbitration signal from the bit controller to the byte controller, and performa a reset in the FSM machine (stop & drop whatever we were doing)

### Four-stage command controller

(AKA the decoder/encoder?)

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

<div data-full-width="false"><figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption><p>From TI I2C documentation, pg. 4</p></figcaption></figure> <figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption><p>From OpenCores reference, pg. 15</p></figcaption></figure></div>

In order to properly identify these four unique control signals, we must have a window that is **four ticks** wide. Hopefully this gives some insight into why the pre-scale clock signal must satisfy:

$$\text{Prescale} = \frac{\text{32Mhz}}{4 \cdot \text{100Khz} } = 80_{10} = 50_{16}$$ (assuming that the clock is running at 32MHz, and the desired I2C clock rate is 100KHz).

> Here is something to consider: the Rabbit application and the SMIMS driver runs at a maximum refresh rate of 50KHz. That is not enough to satisfy the slow operation mode of the I2C standard.\
> \
> If we were to use the onboard 50MHz signal, how can we implement a solution that is able to stay in sync with the virtual component platform Rabbit? Is it possible to "synchronize"? Maybe we can build a custom driver and interface for it?\
> \
> This would be a practice problem to solve/overcome if we want to be able to reach the standard speed. As I am writing this planning/documentation, I am still contemplating how to solve this problem and hopefully I will have a follow up section regarding how to solve it. Perhaps you. as the future audience, might be inspired to create a faster and or more efficient solution!\
> \
> If we I am to implement using the maximum possible frequency of the Rabbit software (@ 50KHz), we can achieve a I2C clock frequency of: $$\text{max} = \frac{\text{clock}}{4} = \frac{\text{50KHz}}{4} = 12.5 \text{KHz}$$, with the pre-scale value being $$1$$. Therefore, the max SCL freq should be 12.5 KHz without any modifications to the Rabbit interface or driver code & using the SMIMS driver generated clock signal.

> Reference: Page 4 of OpenCores module documentation.

### The proposed FSM

At the bit level, there are 4 kinds of operations that I2C can do:

* Start operation
* Stop operation
* Write operation (drive the SDA line)
* Read operation (let go of the SDA line, and read its value)

Each of these operations can be divided in signal level operations (pulling down or letting go), except the start operation which has 5 steps.

By doing some simple math, `5 + 4 + 4 + 4 = 17` , there can be a total of 17 states. With the addition of the idle state, there are **18 states**.

<div><figure><img src="../.gitbook/assets/image (18).png" alt="" width="546"><figcaption><p>Each operation split into ⅘ signal level operations</p></figcaption></figure> <figure><img src="../.gitbook/assets/image (17).png" alt="" width="563"><figcaption><p>FSM flow chart</p></figcaption></figure></div>

### Plan of attack

Let's first assume a max clock rate of 50kHz (50,000 cycles/second), then let's calculate some things:

I'll make another assumption that I would like to achieve the fastest possible I2C clock rate given my current situation. Now we have to choose a pre-scale value.

> Note that the pre-scale value will be divided by 4 in the tristate module, so 4 must be a common denominator of the pre-scale value.

$$
\text{prescale} = 
\frac
{ \text{CLK} }
{ 4 \cdot \text{SCL} }
=
\frac
{50 \text{KHz}}
{4 \cdot \text{SCL}_f }
$$

If we choose $$\text{prescale} = 4$$, then $$\text{SCL}_f = 3.125 \text{KHz}$$.

Given that $$\text{filter\_cnt} = \text{prescale} / 4 = 1$$, the rate at which the raw signal is sampled within the tristate module is $$12.5 \text{KHz}$$.

> Some questions:
>
> 1. Why is the sampling rate of the tristate module faster than the I2C FSM logic?
> 2. The chosen SCL value and the resulting pre-scale value must be a whole number and a common divisor of 4. Why?

## Asynchronous FIFO Buffer

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

There are actually two FPGA's on the FDE board, a Xilinx Spartan & the FDP3P7.

The SIMIS engine is implemented on the Xilinx FPGA which is basically like the broker for the USB communication interface and the actual FDP FPGA. It is responsible for programming the FGPA, providing IO between the PC software & pins on the FPGA, and generating an artificial clock signal that drives the FPGA.

Because of the architecture of the FDE board, we can have two independent clock sources. But note that only one of the clocks is from a oscillator, the other clock source is artificially generated from the SMIMS engine and it is incredibly unstable in the sense that that timings are not consistent.

If we wish to use the internal 30MHz crystal oscillator, then that means our FPGA would be running asyncronously from the SMIMS engine and that would present a few challenges. One of which is how to exchange information between to two mediums? The answer lies in **asynchronous FIFO buffers**.

{% content-ref url="../buffers/asynchronous-fifo.md" %}
[asynchronous-fifo.md](../buffers/asynchronous-fifo.md)
{% endcontent-ref %}

## Some new terms to learn

### Metastability

Metastability occurs when a flip-flop or a latch in a digital circuit fails to settle into a stable state within the specified setup and hold time constraint.

Metastability is the propagation of the metastable state. Metastability is inherent in any system handling bistable states of 1 and 0 or high and low. The output becomes incapable of reaching the confirmed state of either 1 or 0 within a specified period of time.

