cmake_minimum_required(VERSION 2.8)

# Set CMake policy CMP0053 to use new variable reference evaluation rules
# This is the standard way to handle CMP0053 warnings in modern CMake projects
if(POLICY CMP0053)
    cmake_policy(SET CMP0053 NEW)
endif()

if (NOT USE_SYSTEM_ARCH)
    # select use platform 'LINUX' or 'RTOS' here, reset cache and reload cmake project
    set(USE_SYSTEM_ARCH LINUX)
endif ()

if (USE_SYSTEM_ARCH MATCHES RTOS)
    cmake_minimum_required(VERSION 3.15)
    set(CMAKE_C_COMPILER arm-none-eabi-gcc)
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
    set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
    set(CMAKE_AR arm-none-eabi-ar)
    set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
    set(CMAKE_OBJDUMP arm-none-eabi-objdump)
    set(SIZE arm-none-eabi-size)
    set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
endif ()

project(entry)

# Disable in-source builds to prevent source tree corruption.
if (" ${CMAKE_SOURCE_DIR}" STREQUAL " ${CMAKE_BINARY_DIR}")
    message(FATAL_ERROR "FATAL: In-source builds are not allowed.
       You should create a separate directory for build files.")
endif ()

if (USE_SYSTEM_ARCH MATCHES LINUX)
    add_definitions(-DSYSTEM_ARCH_LINUX)
    
    # Detect Raspberry Pi
    # /proc/device-tree/model is the standard way to detect Raspberry Pi on ARM systems
    # It exists on all modern Raspberry Pi models and provides accurate model information
    set(IS_RASPBERRY_PI FALSE)
    if (EXISTS "/proc/device-tree/model")
        file(READ "/proc/device-tree/model" RPI_MODEL)
        string(STRIP "${RPI_MODEL}" RPI_MODEL)
        if (RPI_MODEL MATCHES "Raspberry Pi")
            set(IS_RASPBERRY_PI TRUE)
            message(STATUS "Detected Raspberry Pi: ${RPI_MODEL}")
        endif ()
    endif ()
    
    execute_process(COMMAND uname -m OUTPUT_VARIABLE DEVICE_SYSTEM_ID)
    if (DEVICE_SYSTEM_ID MATCHES x86_64)
        set(LIBRARY_PATH psdk_lib/lib/x86_64-linux-gnu-gcc)
        # x86_64 is always manifold2
        add_subdirectory(samples/sample_c/platform/linux/manifold2)
        add_subdirectory(samples/sample_c++/platform/linux/manifold2)
    elseif (DEVICE_SYSTEM_ID MATCHES aarch64)
        set(LIBRARY_PATH psdk_lib/lib/aarch64-linux-gnu-gcc)
        # For aarch64, check if it's Raspberry Pi
        if (IS_RASPBERRY_PI)
            message(STATUS "Building for Raspberry Pi platform")
            add_subdirectory(samples/sample_c/platform/linux/raspberry_pi)
            add_subdirectory(samples/sample_c++/platform/linux/raspberry_pi)
        else ()
            message(STATUS "Building for Manifold2 platform")
            add_subdirectory(samples/sample_c/platform/linux/manifold2)
            add_subdirectory(samples/sample_c++/platform/linux/manifold2)
        endif ()
    else ()
        message(FATAL_ERROR "FATAL: Please confirm your platform.")
    endif ()

    install(FILES ${LIBRARY_PATH}/libpayloadsdk.a
            DESTINATION "${CMAKE_INSTALL_PREFIX}/lib"
            )

    install(DIRECTORY psdk_lib/include
            DESTINATION "${CMAKE_INSTALL_PREFIX}"
            )
elseif (USE_SYSTEM_ARCH MATCHES RTOS)
    add_definitions(-DSYSTEM_ARCH_RTOS)
    add_subdirectory(samples/sample_c/platform/rtos_freertos/stm32f4_discovery/project/armgcc)
endif ()

add_custom_target(${PROJECT_NAME} ALL)
