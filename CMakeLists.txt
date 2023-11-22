# STM32 Minimal CMake project for C/C++ projects
cmake_minimum_required(VERSION 3.12)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake)
include(gcc-arm-none-eabi)

project(gd32f450-demo VERSION 0.1)
enable_language(C CXX ASM)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/cmake/git-info)
include(GitInfo)
GitInfoSetup()

################################################################################
# Options
# set(build_options "")
# option(NUCLEO_BUILD "NUCLEO-devboard build configuration" OFF)
# option(SMALL_INVERTER "Small inverter" ON)
# option(UNIT_TESTS_ENABLED "On-target testing build configuration" OFF)
# option(HARDWARE_TESTS_ENABLED "Hardware testing build configuration" OFF)

# if(NUCLEO_BUILD)
#     message(STATUS "NUCLEO-devboard build configuration is ON")
#     set(mcu_devboard STM32F446_NUCLEO)
#     unset(inverter_model)
# else()
#     message(STATUS "NUCLEO-devboard build configuration is OFF")
#     unset(mcu_devboard)
#     if(SMALL_INVERTER)
#         message(STATUS "SMALL inverter build configuration is ON")
#         set(inverter_model SMALL_INVERTER)
#     else()
#         message(STATUS "BIG inverter build configuration is ON")
#         set(inverter_model BIG_INVERTER)
#     endif()
# endif()

# if(CMAKE_BUILD_TYPE STREQUAL "Debug")
# set(build_options ${build_options}-debug)
# endif()

# if(UNIT_TESTS_ENABLED)
# set(build_options ${build_options}-unittests)
# endif()

# if(HARDWARE_TESTS_ENABLED)
# set(build_options ${build_options}-hwtests)
# endif()

################################################################################
# Microcontroller Information
set(mcu_family GD32F4xx)
set(mcu_model GD32F450)
set(mcu_devboard GD32F450_CUSTOM_BOARD)
set(cpu_parameters
    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard)

set(mcu_startup_script ${PROJECT_SOURCE_DIR}/src/startup_gd32f450_470.s)
set(mcu_linker_script ${PROJECT_SOURCE_DIR}/GD32F450_xE.ld)

################################################################################
# Headers and Sources
set(sdk_incdirs
    ${PROJECT_SOURCE_DIR}/drivers/${mcu_family}_standard_peripheral
    ${PROJECT_SOURCE_DIR}/drivers/${mcu_family}_standard_peripheral/Include
    ${PROJECT_SOURCE_DIR}/drivers/CMSIS/
    ${PROJECT_SOURCE_DIR}/drivers/CMSIS/Include
    ${PROJECT_SOURCE_DIR}/drivers/CMSIS/GD/${mcu_family}/Include
    ${PROJECT_SOURCE_DIR}/drivers/CMSIS/DSP/Include
)

message(STATUS "INC DIRS: ${sdk_incdirs}")

set(project_incdirs
    ${PROJECT_SOURCE_DIR}
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_SOURCE_DIR}/src
    ${PROJECT_SOURCE_DIR}/tests
)

file(GLOB_RECURSE sdk_sources CONFIGURE_DEPENDS
    ${PROJECT_SOURCE_DIR}/drivers/${mcu_family}_StdPeriphDriver/*.c
    ${PROJECT_SOURCE_DIR}/drivers/Boards/Board_APM32F407_MINI/*.c
)

file(GLOB_RECURSE project_sources CONFIGURE_DEPENDS
    ${PROJECT_SOURCE_DIR}/include/*.c
    ${PROJECT_SOURCE_DIR}/include/*.cpp
    ${PROJECT_SOURCE_DIR}/src/*.c
    ${PROJECT_SOURCE_DIR}/src/*.cpp
    ${PROJECT_SOURCE_DIR}/tests/*.cpp
)

################################################################################
# Executable
set(executable ${CMAKE_PROJECT_NAME})

add_executable(${executable}
    ${sdk_sources}
    ${project_sources}
    ${mcu_startup_script}
    ${CMAKE_BINARY_DIR}/generated/git_info.cpp
)

target_compile_definitions(${executable} PRIVATE
    $<$<CONFIG:Debug>:DEBUG>
    $<$<CONFIG:Debug>:USE_FULL_ASSERT>
    ${mcu_model}
    ${mcu_family}
    ${mcu_devboard}
    EMBLIB_STM32
    MCUDRV_GD32
    BUILD_OPTIONS="${build_options}"
)

if(UNIT_TESTS_ENABLED)
    message(STATUS "On-target testing build configuration is ON")
    target_compile_definitions(${executable} PRIVATE
        UNIT_TESTS_ENABLED
    )
endif()

if(HARDWARE_TESTS_ENABLED)
    message(STATUS "Hardware testing build configuration is ON")
    target_compile_definitions(${executable} PRIVATE
        HARDWARE_TESTS_ENABLED
    )
endif()

target_include_directories(${executable} SYSTEM PRIVATE
    ${sdk_incdirs}
)

target_include_directories(${executable} PRIVATE
    ${project_incdirs}
)

add_dependencies(${executable} git-info)

################################################################################
# Compiler Options and Features
target_compile_options(${executable} PRIVATE
    ${cpu_parameters}
    -Wall
    -Wextra
    -Wpedantic
    -Wshadow
    -Wdouble-promotion
    -Wformat=2 -Wformat-truncation
    -Wundef
    -fno-common
    -Wno-unused-parameter
    -Wno-psabi  # suppress warning "... changed in GCC 7.1"
    $<$<COMPILE_LANGUAGE:CXX>:
        -Wconversion
        -Wsign-conversion
        -Wsign-compare
        -Wno-volatile
        -Wold-style-cast
        -Wuseless-cast
        -Wsuggest-override>
    $<$<CONFIG:Debug>:-O0 -g3 -ggdb>
    $<$<CONFIG:Release>:-O3 -g0>
)

target_compile_features(${executable} PRIVATE
    c_std_11
    cxx_std_23
)

################################################################################
# Linker Options
target_link_options(${executable} PRIVATE
    -T${mcu_linker_script}
    ${cpu_parameters}
    -Wl,-Map=${CMAKE_PROJECT_NAME}.map
    --specs=nosys.specs
    -Wl,--start-group
    -lc
    -lm
    -lstdc++
    -Wl,--end-group
    -Wl,--print-memory-usage
    -u_printf_float			# allows to use printf with float
)

target_link_directories(${executable} PRIVATE
    ${PROJECT_SOURCE_DIR}/drivers/CMSIS/DSP/Lib/GCC
)

target_link_libraries(${executable}
    libarm_cortexM4lf_math.a
)

################################################################################
# Misc Options and Post-build Tasks
set(CMAKE_VERBOSE_MAKEFILE OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_custom_command(TARGET ${executable} POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${executable}>
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${executable}> ${executable}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${executable}> ${executable}.bin
)

if (${DUMP_ASM})
    add_custom_command(TARGET ${executable} POST_BUILD
        COMMAND ${CMAKE_OBJDUMP} -D $<TARGET_FILE:${executable}> > ${executable}.s)
endif()


