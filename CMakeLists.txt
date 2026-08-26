cmake_minimum_required(VERSION 3.20)

set(PICO_BOARD pico2_w CACHE STRING "Board type")
include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)
include(${CMAKE_CURRENT_LIST_DIR}/FreeRTOS_Kernel_import.cmake)

project(cmdr-pico-breadboard-kit C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
pico_sdk_init()

include(FetchContent)
FetchContent_Declare(commander
    GIT_REPOSITORY https://github.com/gbryant/commander.git
    GIT_TAG        v1.1
)
# Local commander source override — managed by `cmdr link` / `cmdr unlink`.
include(${CMAKE_SOURCE_DIR}/commander_local.cmake OPTIONAL)
add_compile_definitions(MAX_COMMANDS=16)
FetchContent_MakeAvailable(commander)

add_executable(cmdr-pico-breadboard-kit main.cpp)
commander_generate_scripts(cmdr-pico-breadboard-kit)
target_link_libraries(cmdr-pico-breadboard-kit PRIVATE commander::pico_runner)
pico_enable_stdio_usb(cmdr-pico-breadboard-kit 1)
pico_enable_stdio_uart(cmdr-pico-breadboard-kit 0)
pico_add_extra_outputs(cmdr-pico-breadboard-kit)
