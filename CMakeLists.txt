cmake_minimum_required(VERSION 3.20)
set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_SOURCE_DIR}/cmake/arm-gcc.cmake")
project(AutonomousKobuki LANGUAGES C ASM)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
add_executable(AutonomousKobuki
  Src/main.c Src/scan.c
  Src/uart.c Src/kobuki.c Src/adc.c Src/ultrasonic.c
  Src/system_stm32g4xx.c Src/startup_stm32g474xx.S Src/syscall.c Src/sysmem.c)
target_include_directories(AutonomousKobuki PRIVATE Inc
  Drivers/CMSIS/Core/Include Drivers/CMSIS/Device/Include)
target_compile_definitions(AutonomousKobuki PRIVATE STM32G474xx)
target_compile_options(AutonomousKobuki PRIVATE
  -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard
  -ffunction-sections -fdata-sections -Wall -Wextra -Wpedantic
  $<$<CONFIG:Debug>:-Og;-g3> $<$<CONFIG:Release>:-Os>)
target_link_options(AutonomousKobuki PRIVATE
  -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard
  -T${CMAKE_CURRENT_SOURCE_DIR}/stm32g474xe_flash.ld
  --specs=nano.specs --specs=nosys.specs
  -Wl,--gc-sections -Wl,-Map=AutonomousKobuki.map -Wl,--print-memory-usage)
set_target_properties(AutonomousKobuki PROPERTIES SUFFIX ".elf")
add_custom_command(TARGET AutonomousKobuki POST_BUILD
  COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:AutonomousKobuki> AutonomousKobuki.hex
  COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:AutonomousKobuki> AutonomousKobuki.bin
  COMMAND ${CMAKE_SIZE} $<TARGET_FILE:AutonomousKobuki>)
