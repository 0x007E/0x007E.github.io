---
layout: post
title: How to use printf/scanf with floating-point in Microchip Studio
categories: [Microcontroller]
introduction: "How to use printf with floating-point in Microchip Studio"
tags: [printf, floating-point, microchip studio, avr, atmega, attiny, microcontroller]
---

Floating point numbers are disabled in Microchip Studio on small AVR platforms because this causes the controller's memory to be exceeded under heavy use. But it is possible to enable floating-point numbers if needed (on platforms that have sufficient program memory space).

> To activate the `printf`/`scanf` functionality for floating-point numbers the linker arguments need to be extended!

Settings can be done in `Microchip Studio` within the `Project Properties`:

![Project Properties]({{ '/assets/images/microcontroller/microchip-studio-project-properties.png' | relative_url }})

> In the submenu `Linker -> General`, check the box `Use vprintf library`.

![Use vprintf]({{ '/assets/images/microcontroller/microchip-studio-linker-general-use-vprintf.png' | relative_url }})

The printf and/or scanf libraries must then be added to the `Linker -> Libraries`.

![Use vprintf]({{ '/assets/images/microcontroller/microchip-studio-linker-libraries-libs.png' | relative_url }})

```
# -> for printf floating point operations
libprintf_flt

# -> for scanf floating point operations
libscanf_flt
```

In the `Linker -> Miscellaneous` the following `flags` have to be added:

![Use vprintf]({{ '/assets/images/microcontroller/microchip-studio-linker-miscellaneous-flags.png' | relative_url }})

```
# -> for printf floating point operations
-Wl,-u,vfprintf -lprintf_flt -lm

# -> for scanf floating point operations
-Wl,-u,vfscanf -lscanf_flt -lm

# -> for printf and scanf floating point operations
-Wl,-u,vfprintf -lprintf_flt -lm -Wl,-u,vfscanf -lscanf_flt -lm
```

> To get things work, save the modifications and rebuild the project!

After all it should now be possible to work with `printf`/`scanf`:

```c
printf("Please enter a floating-point number: ");

float number;

if(scanf("%f", &number) == 1)
{
    printf("\n\n\rThe result of %f * 5.23 equals: %f\n\n\r", number, (number * 5.23));
}
```
