---
title: STM32 Tips and Tricks
---
<link rel="stylesheet" href="/style.css">

# STM32 Tips and Tricks

A collection of obscure or helpful tips, tricks, and hacks for using STM32s.

Canonical URL: <https://stm32.agg.io>

Please help: <https://github.com/adamgreig/stm32tt>

* Table of Contents
{:toc}

---

## Datasheets and Reference Manuals

STM32F405 / 415 / 407 / 417:
[Datasheet](http://www.st.com/resource/en/datasheet/stm32f405rg.pdf),
[Reference Manual](http://www.st.com/resource/en/reference_manual/dm00031020.pdf)

---

## General Reminders

### Watchdog
During initialisation:
```
    /* Turn on the watchdog timer, stopped in debug halt */
    DBGMCU->APB1FZ |= DBGMCU_APB1_FZ_DBG_IWDG_STOP;
    IWDG->KR = 0x5555;
    IWDG->PR = 3;
    IWDG->KR = 0xCCCC;
```

Somewhere in your main loop:
```
        /* Clear the watchdog timer */
        IWDG->KR = 0xAAAA;
```

### Debug access during sleep
Before sleeping or during initialisation:

STM32F4:
```
    /* Allow debug access during WFI sleep */
    DBGMCU->CR |= DBGMCU_CR_DBG_SLEEP;
```

### DCACHE on STM32F7
The F7 has a DCACHE that needs careful management.

Either disable it entirely: in initialisation,
```
SCB_DisableDCache();
```

Or place all DMA buffers into DTCM (the default in ChibiOS, no need to specify the section normally):
```
uint8_t mybuf[1024] __attribute__((section(".ram3")));
```


Or place all DMA buffers into SRAM2 and disable the DCACHE on SRAM2:
```
uint8_t mybuf[1024] __attribute__((section(".ram2")));
```

In ChibiOS, add `#define STM32_SRAM2_NOCACHE TRUE` to `mcuconf.h`, or during initialisation:
```
/* Enable the memory protection unit and set the
 * first region to SRAM2 with DCACHE disabled.
 */
MPU->RNR = 7;
MPU->RBAR = 0x2007C000;
MPU->RASR =
    (3<<24) |       /* (AP) Access permission: RW, RW */
    (4<<19) |       /* (TEX) outer cache policy: no cache */
    (0<<16) |       /* (B) inner cache policy (1/2): no cache */
    (0<<17) |       /* (C) inner cache policy (2/2): no cache */
    (1<<18) |       /* (S) shared */
    (13<<1) |       /* (SIZE) 2^(13+1) = 2^14 = 16K bytes */
    (1<<0);         /* (ENABLE) */
MPU->CTRL = (1<<2) | (1<<0);
SCB_CleanInvalidateDCache();
```

Or, manage the cache manually with the following commands:
```
/* Write-through: commit anything written to cache into RAM.
 * Call before a DMA transfer from RAM.
 */
SCB_CleanDCache();

/* Invalidate: purge anything in the cache so it will be reloaded from RAM.
 * Call before a DMA transfer to RAM.
 */
SCB_InvalidateDCache();

/* Clean and invalidate the DCACHE. */
SCB_CleanInvalidateDCache();
```

Helpful references:
[NuttX](http://www.nuttx.org/doku.php?id=wiki:howtos:port-drivers_stm32f7),
[ChibiOS](http://www.chibios.org/dokuwiki/doku.php?id=chibios:articles:cortexm7_dma_guide)


---

## STM32 and ChibiOS

### SysHalt error LED

Light up an LED when a SysHalt occurs (e.g. due to a failed assertion).

In `chconf.h`, find `CH_CFG_SYSTEM_HALT_HOOK(reason)` and replace the empty body with something like:
```
#define CH_CFG_SYSTEM_HALT_HOOK(reason) { \
    GPIOA->ODR |= (1<<5); \
}

```
where GPIOA5 (in this case) is an LED.

### Low power sleep during idle

Go to WFI sleep mode when no threads are running (requires idle thread). In `chconf.h`:
```
/* Enable wait-for-interrupt sleep during idle */
#define CORTEX_ENABLE_WFI_IDLE TRUE
```

Consider enabling debug access during sleep (see above) when this is enabled.

### STM32F7: Disable DCACHE on SRAM2
In `mcuconf.h`:
```
#define STM32_SRAM2_NOCACHE                 TRUE
```

### Reset on clock security system failure
If clocking from some external source, it can be nice to enable the CSS so if the external source fails you can do something about it. Best thing to do is perhaps a reset:
```
/* The clock security system calls this if HSE goes away.
 * Best bet is probably to reset everything.
 */
void NMI_Handler(void) {
    NVIC_SystemReset();
}
```

To enable CSS:
```
RCC->CR |= RCC_CR_CSSON;
```

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

<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/anchor-js/3.2.2/anchor.min.js"></script>
<script type="text/javascript">
anchors.add("h2, h3, h4, h5, h6");
</script>
