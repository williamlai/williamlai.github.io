---
title: Run WASM application on ESP32
date: 2022-11-17 14:29:22
categories: tech
tags:
- wasm
- esp32
---

In this article, I'm going to set up and run an simple Wasm application on ESP32.

<!-- more -->

# What is WebAssembly

WebAssembly (abbreviated WASM) is a binary instruction format for a stack-based virtual machine. It's known for running applications on the web. The WASM runtime can load and run an application, unload the application, and wait for the next application. It creates a virtual environment and provides flexibility and security.

However, it can also run on embedded devices with FreeRTOS kernel. It makes this kind of embedded device able to run different types of applications by replacing the WASM application binary. Before, changing the application at runtime was hard because there is no dynamic linking on devices that run RTOS. It then requires some propriety ways to do so. OTA update is one example. But OTA usually requires a reboot to make it effective.

# WebAssembly Micro Runtime

There are different types of WASM runtime. [WebAssembly Micro Runtime (WAMR)](https://github.com/bytecodealliance/wasm-micro-runtime) is a lightweight standalone WASM runtime. It supports many platforms, including ESP-IDF.

The VM core of WAMR is called iwasm. The main task of iwasm is to run a WASM application.

There are two significant tasks to take care of to run a simple WASM application on an embedded device.

* iwasm: Launch WASM runtime so that it can load a WASM application.
* WASM application: It needs to convert an application into WASM binary.

# Official Example: ESP32

ESP32 is one of the official examples. Before running any example, it needs to set up the [ESP32 building environment](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/#get-started-get-prerequisites).

The example is here at [https://github.com/bytecodealliance/wasm-micro-runtime/tree/main/product-mini/platforms/esp-idf](https://github.com/bytecodealliance/wasm-micro-runtime/tree/main/product-mini/platforms/esp-idf). To run the example, we first download the repository.

```Bash
git clone https://github.com/bytecodealliance/wasm-micro-runtime.git
```

Go to the example folder, build and run it.

```Bash
cd wasm-micro-runtime/product-mini/platforms/esp-idf
./build_and_run.sh
```

In the `build_and_run.sh` script, we can see it sets an environment variable `WAMR_PATH`.

```Bash
if [[ -z "${WAMR_PATH}" ]]; then
        export WAMR_PATH=$PWD/../../..
fi
```

The `WAMR_PATH` points to the repository root that we just cloned. So we can set it in `.bashrc` so that we can still use it in other WASM projects.

We can check the log with the following command.

```Bash
idf.py monitor
```

We can see the following logs from the sample WASM application. The binary is built in a C-array format in [https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/product-mini/platforms/esp-idf/main/test_wasm.h](https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/product-mini/platforms/esp-idf/main/test_wasm.h). We can see it's not a big application.

```
Hello world!
buf ptr: 0x1458
buf: 1234
```

# Create our own WASM application on ESP32

Now we can create a WASM application on ESP32 from scratch. Before starting, we need to check the following prerequisites.
* [ESP32 building environment](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/#get-started-get-prerequisites): Ensure the related environment variables for ESP32 has been set.
* Setup WAMR_PATH: Set environment variable `WAMR_PATH` to the folder of wasm-micro-runtime.

## Build a WASM application

The following section is a brief version of the doc [Build WASM applications](https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/doc/build_wasm_app.md).

### Download wasi SDK

The wasi SDK is for building C applications into WASM applications. Download wasi SDK in [wasi-sdk release](https://github.com/WebAssembly/wasi-sdk/releases). Extract it into the default location.

```
curl -LO https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-16/wasi-sdk-16.0-linux.tar.gz
tar -zxf wasi-sdk-16.0-linux.tar.gz
sudo mv wasi-sdk-16.0-linux /opt/wasi-sdk
```

### Create ESP32 project

```Bash
idf.py create-project esp32-wasm-test
```

It'll create the following folders and files.

```Bash
$ tree esp32-wasm-test/
esp32-wasm-test/
├── CMakeLists.txt
└── main
    ├── CMakeLists.txt
    └── esp32-wasm-test.c
```

Modify `esp32-wasm-test/CMakeLists.txt` as follows.

```
cmake_minimum_required(VERSION 3.5)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

set (COMPONENTS ${IDF_TARGET} main freertos wamr)
list(APPEND EXTRA_COMPONENT_DIRS "$ENV{WAMR_PATH}/build-scripts/esp-idf")

project(esp32-wasm-test)
```

We specify the needed components are `main`, `freertos`, and `wamr`.

In the folder `${WAMR_PATH}/build-scripts/esp-idf`, there is a ready-to-use `wamr` ESP32 component. We add this folder to the extra component directories.

Then we modify `esp32-wasm-test/main/CMakeLists.txt` as follows.

```
idf_component_register(SRCS "esp32-wasm-test.c"
                    INCLUDE_DIRS "."
                    REQUIRES wamr)
```

The change we made is adding `wamr` into the dependent component.

### Build WASM application

We use the following C sample code as an example and save it as `test.c`. It's actually the same sample code in the official example.

```C
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    char *buf;

    printf("Hello world!\n");

    buf = malloc(1024);
    if (!buf) {
        printf("malloc buf failed\n");
        return -1;
    }

    printf("buf ptr: %p\n", buf);

    sprintf(buf, "%s", "1234\n");
    printf("buf: %s", buf);

    free(buf);
    return 0;
}
```

Then we compile it into WASM binary with the following command.

```
/opt/wasi-sdk/bin/clang -O3 -o test.wasm test.c
```

However, the built WASM binary `test.wasm` is too big because it brings too many dependencies.

```
$ ls -l
total 36
-rw-rw-r-- 1 william william   355 Nov 17 11:40 test.c
-rwxrwxr-x 1 william william 28822 Nov 17 16:45 test.wasm
```

We can reduce the footprint with the following command.

```
/opt/wasi-sdk/bin/clang -O3 -nostdlib \
    -z stack-size=8192 -Wl,--initial-memory=65536 \
    -o test.wasm test.c \
    -Wl,--export=main -Wl,--export=__main_argc_argv \
    -Wl,--export=__heap_base -Wl,--export=__data_end \
    -Wl,--no-entry -Wl,--strip-all -Wl,--allow-undefined
```

The `--no-entry` tells the compiler that it does not follow traditional entry point has its own entry function. In WASM, the default expected entry point is `main` instead of `_start` in Linux.

The `--strip-all` strips all symbols which is not needed for running a WASM application. The symbols are helpful when debugging.

The exported symbols are somewhat necessary because there is no entry point, so we have to pick up some symbols so that the linker can at least know which functions are needed.

Now the binary size is similar to the official example.

```
$ ls -l
total 8
-rw-rw-r-- 1 william william 355 Nov 17 11:40 test.c
-rwxrwxr-x 1 william william 415 Nov 17 16:47 test.wasm
```

Then we can use `xxd` to convert the binary into C-array format.

```
xxd -i test.wasm esp32-wasm-test/main/test_wasm.h
```

We suppost to ensure to align this array with `__aligned(4)`, but ESP32 by default already aligns it, so we can ignore it.

## Write ESP32 code to launch iwasm

I simplify the example code `esp32-wasm-test/main/esp32-wasm-test.c` as follows. It only loads one WASM application in interpreter mode.

```C
#include <stdio.h>
#include "wasm_export.h"
#include "bh_platform.h"
#include "test_wasm.h"

#include "esp_log.h"

#define LOG_TAG "wamr"

static int app_instance_main(wasm_module_inst_t module_inst)
{
    int res = 0;
    const char *exception;

    wasm_application_execute_main(module_inst, 0, NULL);
    if ((exception = wasm_runtime_get_exception(module_inst))) {
        printf("%s\n", exception);
        res = -1;
    }
        
    return res;
}

void *iwasm_main(void *arg)
{
    wasm_module_t wasm_module = NULL;
    wasm_module_inst_t wasm_module_inst = NULL;
    char error_buf[128];
    RuntimeInitArgs init_args;

    memset(&init_args, 0, sizeof(RuntimeInitArgs));
    init_args.mem_alloc_type = Alloc_With_Allocator;
    init_args.mem_alloc_option.allocator.malloc_func = (void *)os_malloc;
    init_args.mem_alloc_option.allocator.realloc_func = (void *)os_realloc;
    init_args.mem_alloc_option.allocator.free_func = (void *)os_free;

    if (wasm_runtime_full_init(&init_args) == false) {
        ESP_LOGE(LOG_TAG, "Init runtime failed.");
    } else if ((wasm_module = wasm_runtime_load(test_wasm, sizeof(test_wasm), error_buf, sizeof(error_buf))) == NULL) {
        ESP_LOGE(LOG_TAG, "Error in wasm_runtime_load: %s", error_buf);
    } else if ((wasm_module_inst = wasm_runtime_instantiate(wasm_module, 32 * 1024, 32 * 1024, error_buf, sizeof(error_buf))) == NULL) {
        ESP_LOGE(LOG_TAG, "Error while instantiating: %s", error_buf);
    } else if ((app_instance_main(wasm_module_inst)) != 0) {
        ESP_LOGE(LOG_TAG, "Error while app_instance_main");
    }

    if (wasm_module_inst != NULL) {
        wasm_runtime_deinstantiate(wasm_module_inst);
    }
    if (wasm_module != NULL) {
        wasm_runtime_unload(wasm_module);
    }
    wasm_runtime_destroy();

    return NULL;
}

void app_main(void) {
    pthread_t t;
    int res;

    pthread_attr_t tattr;
    pthread_attr_init(&tattr);
    pthread_attr_setdetachstate(&tattr, PTHREAD_CREATE_JOINABLE);
    pthread_attr_setstacksize(&tattr, 4096);

    res = pthread_create(&t, &tattr, iwasm_main, (void *)NULL);
    assert(res == 0);

    res = pthread_join(t, NULL);
    assert(res == 0);

    ESP_LOGI(LOG_TAG, "Exiting...");
}
```

Use the following command to build, flash , and check the logs.

```
cd esp32-wasm-test
idf.py build
idf.py flash monitor
```

### code study

We can see the memory that iwasm uses is from the allocator. We can provide our allocator instead of a system allocator if we want.

The following are the needed API calls.

* `wasm_runtime_full_init` & `wasm_runtime_destroy`: These are two APIs that init the iwasm.
* `wasm_runtime_load` & `wasm_runtime_unload`: Loading and unloading the WASM application into a model.
* `wasm_runtime_instantiate` & `wasm_runtime_deinstantiate`: It creates and destroys instances from a model.
* `wasm_application_execute_main` & `wasm_runtime_get_exception`: It runs the instance and gets the results.

# Conclusion

Before studying any WASM, I thought WASM was not a practical solution because the byte codes of a WASM application may need to be bigger. After all, it has to bind many features to make it work within the WASM engine. But I was wrong. The byte code size of the example is similar to the size in the native environment. It even includes printf and malloc, and the two functions usually cost lots of code space. And WASM gets a good balance by taking advantage of the native environment support.

So I think WASM is pretty cool and makes MCU have a flexible running environment.
