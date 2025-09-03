cmake_minimum_required(VERSION 3.15)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# 设置工具链
set(TOOLPATH "${CMAKE_CURRENT_SOURCE_DIR}/ToolChain/gcc/bin/riscv-nuclei-elf-")

set(CMAKE_C_COMPILER ${TOOLPATH}gcc.exe)
set(CMAKE_CXX_COMPILER ${TOOLPATH}g++.exe)
set(CMAKE_ASM_COMPILER ${TOOLPATH}gcc.exe)
set(CMAKE_AR ${TOOLPATH}ar.exe)
set(CMAKE_OBJCOPY ${TOOLPATH}objcopy.exe)
set(CMAKE_OBJDUMP ${TOOLPATH}objdump.exe)
set(CMAKE_SIZE ${TOOLPATH}size.exe)

# 定义项目
project(VW553_Camera VERSION 1.0 LANGUAGES C CXX ASM)

# 编译标准
set(CMAKE_C_STANDARD 99)
set(CMAKE_CXX_STANDARD 11)

# 链接脚本
set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/LinkScript/gd32vw55x.lds)

# 源文件
file(GLOB_RECURSE SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/env_Eclipse/entry.S
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/env_Eclipse/start.S
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/GD32VW55x_standard_peripheral/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/stubs/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/env_Eclipse/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/App/src/*.c
)

add_executable(${PROJECT_NAME}.elf ${SOURCES})

# 头文件
target_include_directories(${PROJECT_NAME}.elf PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/GD32VW55x_standard_peripheral
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/GD32VW55x_standard_peripheral/Include
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/env_Eclipse
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/stubs
    ${CMAKE_CURRENT_SOURCE_DIR}/Firmware/RISCV/drivers
    ${CMAKE_CURRENT_SOURCE_DIR}/App/inc
)

# 彩色编译输出
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(
        -fdiagnostics-color=always
    )
endif()

target_compile_options(${PROJECT_NAME}.elf PRIVATE
    -march=rv32imafcbp
    -mcmodel=medlow
    -msmall-data-limit=8
    -msave-restore
    -mabi=ilp32f
    -O2
    -fmessage-length=0
    -fsigned-char
    -ffunction-sections
    -fdata-sections
    -fno-common
    -Wuninitialized
    -g
)

target_link_options(${PROJECT_NAME}.elf PRIVATE
    -march=rv32imafcbp
    -mcmodel=medlow
    -msmall-data-limit=8
    -msave-restore
    -mabi=ilp32f
    -Wl,-Map=${CMAKE_BINARY_DIR}/${PROJECT_NAME}.map
    -T ${LINKER_SCRIPT}
    -nostartfiles
    -Xlinker --gc-sections
    --specs=nano.specs
    --specs=nosys.specs
    -Wl,--print-memory-usage
)

target_compile_definitions(${PROJECT_NAME}.elf PRIVATE
    GD_ECLIPSE_GCC
    USE_STDPERIPH_DRIVER
)

# 确保 out 目录存在
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/out)

# 生成 HEX / BIN / LST / SIZE 到 out 目录
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/out
    COMMAND ${CMAKE_SIZE} --format=berkeley $<TARGET_FILE:${PROJECT_NAME}.elf>
    COMMAND ${CMAKE_SIZE} --format=berkeley $<TARGET_FILE:${PROJECT_NAME}.elf> > ${CMAKE_BINARY_DIR}/out/${PROJECT_NAME}.size
    COMMAND ${CMAKE_OBJDUMP} --source --all-headers --demangle --line-numbers --wide $<TARGET_FILE:${PROJECT_NAME}.elf> > ${CMAKE_BINARY_DIR}/out/${PROJECT_NAME}.lst
    COMMAND ${CMAKE_OBJCOPY} -Oihex   $<TARGET_FILE:${PROJECT_NAME}.elf> ${CMAKE_BINARY_DIR}/out/${PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${CMAKE_BINARY_DIR}/out/${PROJECT_NAME}.bin
)