cmake_minimum_required(VERSION 3.12)

# Specify the toolchain
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR cortex-m4)
set(CMAKE_SYSTEM_VERSION 99)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_AR arm-none-eabi-ar)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_OBJDUMP arm-none-eabi-objdump)
set(CMAKE_SIZE_UTIL arm-none-eabi-size)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# Add an option to select the compiler
option(USE_ARMCLANG "Use armclang as the compiler" OFF)

# Set the compiler based on the option
if(USE_ARMCLANG)
    set(CMAKE_C_COMPILER armclang)
    set(CMAKE_CXX_COMPILER armclang++)
endif()

add_definitions(-DAT32F425K8U7_4)
# for HID
add_definitions(-DAP_FW_V1) # for HID
# Set the compiler flags for Cortex-M4
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=cortex-m4 -mthumb -mfloat-abi=soft --specs=nosys.specs -specs=nano.specs -O2")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=cortex-m4 -mthumb -mfloat-abi=soft --specs=nosys.specs -specs=nano.specs -O2")

# Set the project name and version
project(DAP_F425 VERSION 1.0 LANGUAGES C CXX ASM)

# Include directories
include_directories(
    App
    DAP/Include
    Libraries/cmsis/cm4/core_support
    Libraries/cmsis/cm4/core_support/dsp
    Libraries/cmsis/cm4/device_support
    Libraries/drivers/inc
    Libraries/usb_drivers/inc
)

set(SOURCES
    App/DAP_config.h
    App/at32f425_conf.h
    App/descriptor.c
    App/hid_transfer.c
    App/main.c
    App/usb_conf.h
    App/vcom_serial.c
    DAP/Source/DAP.c
    DAP/Source/DAP_vendor.c
    DAP/Source/JTAG_DP.c
    DAP/Source/SWO.c
    DAP/Source/SW_DP.c
    DAP/Source/UART.c
)

# Gather all source files for drivers
file(GLOB DRIVER_SOURCES 
    Libraries/drivers/src/*.c
)
# Create the drivers library from source files
add_library(drivers 
   ${DRIVER_SOURCES}
)

# Create the CMSIS library from source files
add_library(cmsis
    Libraries/cmsis/cm4/device_support/startup/gcc/startup_at32f425.s
    Libraries/cmsis/cm4/device_support/system_at32f425.c
)

# Gather all source files for usb_drivers
file(GLOB USB_DRIVERS_SOURCES 
    Libraries/usb_drivers/src/*.c
)
# Create the usb_drivers library from source files
add_library(usb_drivers 
    ${USB_DRIVERS_SOURCES}
)

# Create the executable
add_executable(${PROJECT_NAME}.elf ${SOURCES})

# Set the linker script
set_target_properties(${PROJECT_NAME}.elf PROPERTIES
    LINK_FLAGS
    "-T ${CMAKE_SOURCE_DIR}/AT32F425x8_FLASH.ld -Wl,--gc-sections -Wl,-Map=${PROJECT_NAME}.map,--cref -Xlinker -print-memory-usage -Xlinker"
)

# Link libraries if needed
target_link_libraries(${PROJECT_NAME}.elf 
    # Add library names here, e.g., CMSIS, drivers
    cmsis
    usb_drivers
    drivers
)

# Optional: Create hex, bin and S-Record files after the build
add_custom_command(TARGET ${PROJECT_NAME}.elf 
  POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -O ihex ${PROJECT_NAME}.elf  ${PROJECT_NAME}.hex
  COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf  ${PROJECT_NAME}.bin
  COMMENT "Generating HEX and BIN files"
)

# Optional: Print executable size as part of the post build process
add_custom_command(TARGET ${PROJECT_NAME}.elf
  POST_BUILD
  COMMAND ${CMAKE_SIZE_UTIL} ${PROJECT_NAME}.elf 
)