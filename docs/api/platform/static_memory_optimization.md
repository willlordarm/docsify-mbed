# Static memory optimization

## Removing unused modules

For a simple program like [Blinky](https://github.com/ARMmbed/mbed-os-example-blinky), a program that flashes an LED, typical memory usage is split among the following modules:

```
+---------------------+-------+-------+-------+
| Module              | .text | .data |  .bss |
+---------------------+-------+-------+-------+
| Fill                |   132 |     4 |  2377 |
| Misc                | 28807 |  2216 |    88 |
| features/frameworks |  4236 |    52 |   744 |
| hal/common          |  2745 |     4 |   325 |
| hal/targets         | 12172 |    12 |   200 |
| rtos/rtos           |   119 |     4 |     0 |
| rtos/rtx            |  5721 |    20 |  6786 |
| Subtotals           | 53932 |  2312 | 10520 |
+---------------------+-------+-------+-------+
```

The `features/frameworks` module includes the Mbed OS test tools, even if you are no longer testing your program. Because of this, you are building one of our test harnesses into every binary. [Removing this module](https://github.com/ARMmbed/mbed-os/pull/2559) saves a significant amount of RAM and flash memory.

### `Printf` and UART

The linker can also remove other modules that your program does not use. For example, [Blinky's](https://github.com/ARMmbed/mbed-os-example-blinky) `main` program doesn't use `printf` or UART drivers. However, every Mbed OS module handles traces and assertions by redirecting their error messages to `printf` on serial output. This forces the `printf` and UART drivers to compile and requires a large amount of flash memory.

To disable error logging to serial output, set the `NDEBUG` macro and the following configuration parameter in your program's `mbed_app.json` file:

```
{
    "macros": [
        "NDEBUG=1"
    ],
    "target_overrides": {
        "*": {
            "platform.stdio-flush-at-exit": false
        }
    }
}
```

<span class="notes">**Note:** Different compilers yield different memory savings.</span>

### Embedded targets

You can also take advantage of the fact that these programs only run on embedded targets. When you run a C++ application on a desktop computer, the operating system constructs every global C++ object before calling `main`. It also registers a handle to destroy these objects when the program ends. The code the compiler injects has some implications for the application:

- The code that the compiler injects consumes memory.
- It implies dynamic memory allocation and thus requires the binary to include `malloc`, even when the application does not use it.

When you run an application on an embedded device, you don't need handlers to destroy objects when the program exits, because the application will never end. To save more RAM and flash memory, [remove destructor registration](https://github.com/ARMmbed/mbed-os/pull/2745) on application startup and eliminate the code to destruct objects when the operating system calls `exit()` at runtime.
