# STM32 Tips and Tricks

A collection of obscure or helpful tips, tricks, and hacks for using STM32s.

Canonical URL: <http://stm32.agg.io>

Please help: <https://github.com/adamgreig/stm32tt>

---

## Datasheets and Reference Manuals

### STM32F405 / 415 / 407 / 417
[Datasheet](http://www.st.com/resource/en/datasheet/stm32f405rg.pdf)
[Reference Manual](http://www.st.com/resource/en/reference_manual/dm00031020.pdf)

---

## STM32 and ChibiOS

### SysHalt error LED

Light up an LED when a SysHalt occurs (e.g. due to a failed assertion).

In `chconf.h`, find `CH_CFG_SYSTEM_HALT_HOOK(reason)` and replace the empty body with something like:
```
GPIOA->ODR |= 1; \
```
where GPIOA1 is an LED.

---

## STM32 and libopencm3

---

## STM32 and GDB

### Semihosting

This allows the embedded code to access stdin/stdout and files on the host computer, over the debug link.
Link with `--specs=rdimon.specs -lrdimon`, or in ChibiOS update your Makefile:
```
# user libraries
ULIBS += --specs=rdimon.specs -lrdimon```
```

Then in your `main.c` add this prototype:
```
extern void initialise_monitor_handles(void);
```

Finally call `initialise_monitor_handles()` somewhere. This will hang forever if you do not have a debugger attached, so you might want to check if the stm32 is under debug first:
```
if(CoreDebug->DHCSR & 1) {
    initialise_monitor_handles();
}
```

Now you can call `printf`, `scanf`, `fopen`, etc from your code and they will target the attached host.

### Displaying call stack after a HardFault

By default GDB doesn't know how to view the call stack that lead to a HardFault, but we can help it. Stick this anywhere:
```
/* On hard fault, copy HARDFAULT_PSP to the sp reg so gdb can give a trace */
void **HARDFAULT_PSP;
register void *stack_pointer asm("sp");
void HardFault_Handler(void) {
    asm("mrs %0, psp" : "=r"(HARDFAULT_PSP) : :);
    stack_pointer = HARDFAULT_PSP;
    while(1);
}
```

Optionally include `GPIOA->ODR |= 1;` if you have an error LED that can be lit when this occurs.

[Source](http://jpa.kapsi.fi/stuff/other/stm32-hardfault-backtrace.html)

---

## STM32 and Rust

### Useful Links
[Cargo template for Cortex-M projects](https://github.com/japaric/cortex-m-template)

[Guide to embedded devleopment in Rust](https://japaric.github.io/copper)
