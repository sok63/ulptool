ulptool v2.3.0
==================
This guide explains how to setup Arduino to use ULP coprocessor assembly files for your esp32 projects. This guide assumes you installed the esp32 core with the preferred method of the board manager.

Typically in Arduino you can compile assembly files using the '.S' extension. Using the ESP32 Arduino core framework these files would correspond to the Xtensa processors whose toolchain is incompatible with the ULP coprocessor. Luckily, Arduino provides a fairly easy albeit not that flexible build framework using series of recipes. This guide extends those esp32 recipes for building the ULP assembly files. We will use the '.s' extensions for ULP assembly files which Arduino will let you create. Remember thats a lower case 's'. I tried to keep the ulp build process the same as the esp-if framework with only a few small modifications the user needs to do in order to compile in Arduino.

Manual Setup Steps:
============
1. Download the latest release of this repository and unpack-> https://github.com/duff2013/ulptool/releases/latest
delete the release version number so the folder is just called 'ulptool'

2. Download and unpack the latest pre-compiled binutils-esp32ulp toolchain for Mac/Linux/Windows: https://github.com/espressif/binutils-esp32ulp/releases/latest

3. Find your Arduino-esp32 core directory which Arduino IDE uses:

            Typically (Mac OS) -> ~/Library/Arduino15/packages/esp32

            Typically (Windows) -> C:\Users\<USERNAME>\AppData\Local\Arduino15\packages\esp32

            Typically (Linux) ->?

4. Move the **ulptool** folder you downloaded and unpacked to the tools folder here -> **.../esp32/tools/**.

5. Copy the 'platform.local.txt' file to **.../esp32/hardware/esp32/1.0.0/**. Remember **1.0.0** has to match your esp32 core version.

6. In the **ulptool** release you downloaded, move or copy the **.../esp32/tools/ulptool/src/ulp_examples** folder to where Arduino saves your sketches.

7. Move **esp32ulp-elf-binutils** folder you downloaded and unpacked to -> **.../esp32/tools/ulptool/src/**.

Thats it, you now have all the files in place, lets look at very simple example to get you compiling ulp assembly code!

Example:
========
Open a blank Arduino sketch and copy and paste the code below into the that sketch.
```
#include "esp32/ulp.h"// Must have this!!!

// include ulp header you will create
#include "ulp_main.h"// Must have this!!!

// Custom binary loader
#include "ulptool.h"// Must have this!!!

// Unlike the esp-idf always use these binary blob names
extern const uint8_t ulp_main_bin_start[] asm("_binary_ulp_main_bin_start");
extern const uint8_t ulp_main_bin_end[]   asm("_binary_ulp_main_bin_end");

static void init_run_ulp(uint32_t usec);

void setup() {
    Serial.begin(115200);
    delay(1000);
    init_run_ulp(100 * 1000); // 100 msec
}

void loop() {
    // ulp variables data is the lower 16 bits
    Serial.printf("ulp count: %u\n", ulp_count & 0xFFFF);
    delay(100);
}

static void init_run_ulp(uint32_t usec) {
    // initialize ulp variable
    ulp_count = 0;
    ulp_set_wakeup_period(0, usec);
    // use this binary loader instead
    esp_err_t err = ulptool_load_binary(0, ulp_main_bin_start, (ulp_main_bin_end - ulp_main_bin_start) / sizeof(uint32_t));
    // ulp coprocessor will run on its own now
    err = ulp_run((&ulp_entry - RTC_SLOW_MEM) / sizeof(uint32_t));
}
```

Create a new tab named <b>ulp.s</b>, take notice that the extension is a lower case 's'. Copy the code below into that ulp assembly file tab.
```
/* Define variables, which go into .bss section (zero-initialized data) */
    .bss
/* Store count value */
    .global count
count:
    .long 0

/* Code goes into .text section */
    .text
    .global entry
entry:
    move    r3, count
    ld      r0, r3, 0
    add     r0, r0, 1
    st      r0, r3, 0
    halt
```

Create a new tab named <b>ulp_main.h</b>. This header allows your sketch to see global variables whose memory is allocated your ulp assembly file. This memory is in the SLOW RTC section. Copy the code below into that header file. As with the esp-idf you have to add 'ulp_' to the front of the variable name. Unlike esp-idf the name of this header is always this name.
```
/*
    Put your ULP globals here you want visibility
    for your sketch. Add "ulp_" to the beginning
    of the variable name and must be size 'uint32_t'
*/
#include "Arduino.h"

extern uint32_t ulp_entry;
extern uint32_t ulp_count;
```

Compile and run and you should see the variable 'ulp_count' increment every 100 msecs.

Under the Hood:
===============
All the magic happens in the python script called esp32ulp_build_recipe.py. This along with espressif's esp32ulp_mapgen.py in which both are located in the ulp directory from this repository.

Limitations:
============
While almost a complete solution to programing the ULP coprocessor in assembly, there are currently a few limitations. Once I fix these, I'll remove them from this list.

1. Not tested with Linux yet.
2. Errors can be non-informative.
3. Have to use the custom binary loader function now. (ulptool_load_binary)
