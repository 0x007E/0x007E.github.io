---
layout: post
title: Use AVR0 RTC interrupt to control a systick timer
categories: [Microcontroller]
introduction: "Explanation of how to setup RTC with an OVF interrupt on a AVR0 platform to control a software systick timer within ~1 ms tick rate"
tags: [avr0, rtc, 32.768kHz, systick, attiny1606, attiny1604, atmega3208, atmega4808, microcontroller, 1ms, tick, hal, library, oscillator, time, clock, ulp]
---

Modern systems like `arm` (risc-v) are working with a `systick` instead of a `delay`. That also can be realized on a `avr0` architecture with the interal `rtc` so  `TCA` and `TCB` can be used for other things on the system. 

> First of all, it's not possible to set the `rtc` timer exactly to 1 millisecond! The internal `32.768 kHz` oscillator isn't divisible to exactly 1ms, but it's sufficient for approximate counting instead of a delay.

First of all let´s take a look into the `avr0` clock control in the family [datasheet (DS40002015B)](https://ww1.microchip.com/downloads/en/DeviceDoc/megaAVR0-series-Family-Data-Sheet-DS40002015B.pdf) on page 83.

![AVR0 CLKCTRL RTC marked]({{ '/assets/images/microcontroller/avr0_clkctrl_block_diagram_rtc_marked.png' | relative_url }})

There we can see that it it possible to route the internal `32.768 kHz` oscillator to the `rtc`. Before the timer can be switched on we need to think about the ticks to get an interrupt every `~1 ms`. Let´s do some math:

$$
T_{Period} = \frac{1}{f_{INT}} = \frac{1}{32.768 kHz} = 30.5176 \mu\text{s}
$$

Every $30.5176 \mu\text{s}$ increments $+1$. So now we can calculate the time shift to get to $1ms$.

$$
Ticks_{1ms} = \frac{1ms}{T_{Period}} = \frac{1ms}{30.5176 \mu\text{s}} = 32.77
$$

There we see that the ticks are not an integer and the interrupts can´t occur exactly at $1ms$!

$$
T_{INT,min} = (int)(32.77) * 32.768 kHz = ~0,98 ms 
$$

$$
T_{INT,max} = ((int)(32.77) + 1) * 32.768 kHz = ~1,007 ms 
$$

The best fit seems to be at $T_{INT,max}=~1,007 ms$ so the `rtc` overflow register (`PER`) should be set to $33$ ticks with a prescaler of 1.

``` c
void rtc_init(void)
{	
    RTC.INTCTRL = RTC_OVF_bm;
    RTC.CLKSEL = RTC_CLKSEL_INT32K_gc;

    // RTC Overflow
    while (RTC.STATUS & RTC_PERBUSY_bm);
    RTC.PER = 0x0021;   // 33 -> base16 -> 0x0021

    while (RTC.STATUS & RTC_CTRLABUSY_bm);
    RTC.CTRLA = RTC_PRESCALER_DIV1_gc | RTC_RTCEN_bm;
}
```

Let´s check if the generated time fit´s the calculated time. To do this we will start to toggle any available pin on an `avr0` (in this case an `ATTiny1604`).

> The initialization of the main system clock is not necessary, in cause of that we are not using any program code in `while(1)`.

``` c
#include <avr/io.h>
#include <avr/interrupt.h> 

ISR(RTC_CNT_vect)
{
    PORTA.OUTTGL = PIN0_bm;
    RTC.INTFLAGS = RTC_OVF_bm;
}

int main(void)
{
    PORTA.DIRSET = PIN0_bm;

    rtc_init();
    sei();

    while(1)
    {

    }
}
```

Now we take a look at the generated signal with an `Analog Discovery`.

**Generated signal with `0x0021` setup.**

![Diagram with 0x0021]({{ '/assets/images/microcontroller/avr0_rtc_clock_diagram_0x0021.png' | relative_url }})
![Mesasure with 0x0021]({{ '/assets/images/microcontroller/avr0_rtc_clock_measure_0x0021.png' | relative_url }})

**Generated signal with `0x0020` setup.**

![Diagram with 0x0021]({{ '/assets/images/microcontroller/avr0_rtc_clock_diagram_0x0020.png' | relative_url }})
![Mesasure with 0x0021]({{ '/assets/images/microcontroller/avr0_rtc_clock_measure_0x0020.png' | relative_url }})

There we are. In this case the value `0x0021` seems to fit better than the value `0x0020`. The drift with `0x0021` is a bit lower than with `0x0020` to $1ms$ and it seems to be better to have a positive instead of a negative timeshift.

### Generating a `systick` with the `rtc`

To keep things cleaned up, just lets use some libraries for the initialization of `rtc` and the `system` clock.

``` bash
git clone https://github.com/0x007E/hal-avr0-system.git ./hal/avr0
mv ./hal/avr0/hal-avr0-system ./hal/avr0/system

git clone https://github.com/0x007E/hal-avr0-rtc.git ./hal/avr0
mv ./hal/avr0/hal-avr0-rtc ./hal/avr0/rtc
```

Now we can just use this libraries to initialize the `system` clock to $20MHz$ and the `rtc` clock as previousely defined and add a software systick timer.

``` c
#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/atomic.h>

#include "./hal/avr0/system/system.h"
#include "./hal/avr0/rtc/rtc.h"

volatile unsigned int systick;

// Called every ~1ms
ISR(RTC_CNT_vect)
{
    systick++;
    RTC.INTFLAGS = RTC_OVF_bm;
}

int main(void)
{
    system_init();
    rtc_init();
    sei();

    PORTA.DIRSET = PIN0_bm;
    unsigned int last_value = 0;

    while(1)
    {
        unsigned int temp = 0x00;

        ATOMIC_BLOCK(ATOMIC_RESTORESTATE)
        {
            temp = systick;
        }

        if((temp - last_value) >= 1000UL)
        {   
            PORTA.OUTTGL = PIN0_bm;
            last_value = temp;
        }
    }
}
```

> The `ATOMIC_BLOCK` is necessary because the `avr0` architecture works with `8 bits` and the `systick` variable `16 bits`. The calculation therefore needs two operations on the CPU. To prevent that the interrupt get´s executed while the value is written from `systick` to `temp` and changes the value during the comparison the block is required!

Everybody knows that programmers are lazy so there is still another library to do this stuff.

``` bash
git clone https://github.com/0x007E/utlis-systick ./utils
mv ./utils/utils-systick ./utils/systick
```

``` c
#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/atomic.h>

#include "./hal/avr0/system/system.h"
#include "./hal/avr0/rtc/rtc.h"
#include "./utils/systick/systick.h"

SYSTICK_Timer systick_timer;

void systick_timer_wait_ms(unsigned int ms)
{
    systick_timer_wait(ms);
}

// Called every ~1ms
ISR(RTC_CNT_vect)
{
    systick_tick();
    RTC.INTFLAGS = RTC_OVF_bm;
}

int main(void)
{
    system_init();
    rtc_init();
    sei();
	
    systick_init();

    PORTA.DIRSET = PIN0_bm;

    while(1)
    {
        PORTA.OUTTGL = PIN0_bm;
        systick_timer_wait_ms(250UL);

        // Or just non-blocking
        if (systick_timer_elapsed(&systick_timer))
        {
            PORTA.OUTTGL = PIN0_bm;
            ystick_timer_set(&systick_timer, 250UL);
        }
    }
}
```

Feel free to use this libraries in any case of your private or comercial projects!
