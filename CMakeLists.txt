cmake_minimum_required(VERSION 3.22)
include(ExternalProject)
include(cmake/Utilities.cmake)
include(cmake/GetGitRevisionDescription.cmake)
include(cmake/ProjectVersion.cmake)
include(cmake/Littlefs.cmake)
include(cmake/Options.cmake)

project(
  Buddy
  LANGUAGES C CXX ASM
  VERSION ${PROJECT_VERSION}
  )

# Enable initialization of subprojects without their submodules
cmake_policy(SET CMP0097 NEW)

if(NOT CMAKE_CROSSCOMPILING)
  #
  # If we are not crosscompiling, include `utils` with host tools.
  #
  add_subdirectory(utils)
endif()

include(ProjectOptions.cmake)

# Check GCC Version
get_recommended_gcc_version(RECOMMENDED_TOOLCHAIN_VERSION)
if(CMAKE_CROSSCOMPILING AND NOT CMAKE_CXX_COMPILER_VERSION VERSION_EQUAL
                            ${RECOMMENDED_TOOLCHAIN_VERSION}
   )
  message(WARNING "Recommended ARM toolchain is ${RECOMMENDED_TOOLCHAIN_VERSION}"
                  ", but you have ${CMAKE_CXX_COMPILER_VERSION}"
          )

elseif(NOT CMAKE_CROSSCOMPILING AND NOT CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  message(
    WARNING
      "Recommended compiler for host tools and unittests is GCC, you have ${CMAKE_CXX_COMPILER_ID}."
    )
endif()

# eclipse sets those variables, so lets just use them so we don't get a warning about unused
# variables
set(unused "${CMAKE_VERBOSE_MAKEFILE} ${CMAKE_RULE_MESSAGES}")
set(CMAKE_CXX_STANDARD 20)

# enable all warnings (well, not all, but some)
add_compile_options(-Wall -Wsign-compare)

# make all paths stored inside firmware relative
add_compile_options(-ffile-prefix-map=${CMAKE_SOURCE_DIR}=.)

add_compile_options(
  $<$<COMPILE_LANGUAGE:CXX>:-Wno-register> $<$<COMPILE_LANGUAGE:CXX>:-Wno-volatile>
  $<$<COMPILE_LANGUAGE:CXX>:-fno-rtti>
  )

# append custom C/C++ flags
if(CUSTOM_COMPILE_OPTIONS)
  string(REPLACE " " ";" CUSTOM_COMPILE_OPTIONS "${CUSTOM_COMPILE_OPTIONS}")
  add_compile_options(${CUSTOM_COMPILE_OPTIONS})
endif()
if(PRESET_COMPILE_OPTIONS)
  string(REPLACE " " ";" PRESET_COMPILE_OPTIONS "${PRESET_COMPILE_OPTIONS}")
  add_compile_options(${PRESET_COMPILE_OPTIONS})
endif()

# find bootloader.bin location
if(BOOTLOADER STREQUAL "YES")
  string(TOLOWER ${PRINTER} printer_low)
  set(BOOTLOADER_VARIANT "${printer_low}")
endif()

#
# BuddyHeaders
#
# This library provides headers in the /include directory. When a library requires a configuration
# header, e.g. STM32::USBHost requires usbh_conf.h, we can just place the header to /include and
# then add BuddyHeaders as a dependency to STM32::USBHost.
#
# TODO: Refactor this to make it clear what header files are associated with which targets.
#

add_library(BuddyHeaders INTERFACE)
target_include_directories(
  BuddyHeaders
  INTERFACE include
            include/usb_host
            include/usb_device
            include/marlin
            include/freertos
            src/hw
            src/common
            src/persistent_stores
            src/persistent_stores/store_instances
            src/guiconfig
            src/lang
            src/common/utils
            ${OPTIONS_INCLUDE_DIR}
  )

if(MCU MATCHES "STM32G0")
  target_include_directories(
    BuddyHeaders BEFORE INTERFACE include/stm32g0_hal include/device/stm32g0
    )
  target_link_libraries(BuddyHeaders INTERFACE STM32G0::HAL)
elseif(MCU MATCHES "STM32F4")
  target_include_directories(
    BuddyHeaders BEFORE INTERFACE include/stm32f4_hal include/device/stm32f4
    )
  target_link_libraries(BuddyHeaders INTERFACE STM32F4::HAL)
endif()

if(BOARD MATCHES "BUDDY")
  target_include_directories(BuddyHeaders INTERFACE include/buddy)
elseif(BOARD MATCHES "DWARF")
  target_include_directories(BuddyHeaders INTERFACE include/dwarf)
elseif(BOARD MATCHES "MODULARBED")
  target_include_directories(BuddyHeaders INTERFACE include/modularbed)
endif()

target_include_directories(BuddyHeaders INTERFACE include)

target_link_libraries(BuddyHeaders INTERFACE FreeRTOS::FreeRTOS)

# Define printer type and version
set(PRINTER_VERSION "1") # default version
set(PRINTER_SUBVERSION "0") # default subversion

# The PRINTER_TYPE is (besides being used in the code) being stored in the .bbf and checked by the
# bootloader (which contains the same "enum" as below). Preferably do not use it on an interface
# though, there should be a better number to use there, e.g. the PRINTER_CODE (USB PID)
if(PRINTER STREQUAL "MK4")
  set(PRINTER_TYPE "1")
  set(PRINTER_VERSION "4")
  set(PRINTER_CODE "13")
elseif(PRINTER STREQUAL "MK3.5")
  set(PRINTER_TYPE "1")
  set(PRINTER_VERSION "3")
  set(PRINTER_SUBVERSION "5")
  set(PRINTER_CODE "20")
elseif(PRINTER STREQUAL "MINI")
  set(PRINTER_TYPE "2")
  set(PRINTER_CODE "12")
elseif(PRINTER STREQUAL "XL")
  set(PRINTER_TYPE "3")
  set(PRINTER_CODE "17")
elseif(PRINTER STREQUAL "iX")
  set(PRINTER_TYPE "4")
  set(PRINTER_CODE "16")
else()
  message(FATAL_ERROR "Unknown printer \"${PRINTER}\".")
endif()

if(${BOARD} STREQUAL "DWARF" OR ${BOARD} STREQUAL "MODULARBED")
  set(HAS_MARLIN FALSE)
else()
  set(HAS_MARLIN TRUE)
endif()

if(BOARD STREQUAL "BUDDY" OR BOARD STREQUAL "XBUDDY")
  set(MARLIN_BOARD_NUM 1823)
elseif(BOARD STREQUAL "XLBUDDY")
  set(MARLIN_BOARD_NUM 1824)
elseif(BOARD MATCHES "DWARF")
  set(MARLIN_BOARD_NUM 1825)
elseif(BOARD MATCHES "MODULARBED")
  set(MARLIN_BOARD_NUM -1)
else()
  message(FATAL_ERROR "Unknown board")
endif()

target_compile_definitions(
  BuddyHeaders
  INTERFACE MOTHERBOARD=${MARLIN_BOARD_NUM}
            PRINTER_MODEL="${PRINTER}"
            PRINTER_TYPE=${PRINTER_TYPE}
            PRINTER_VERSION=${PRINTER_VERSION}
            PRINTER_SUBVERSION=${PRINTER_SUBVERSION}
            PRINTER_CODE=${PRINTER_CODE}
            BOARD=BOARD_${BOARD}
            BOARD_VERSION_MAJOR=${BOARD_VERSION_MAJOR}
            BOARD_VERSION_MINOR=${BOARD_VERSION_MINOR}
            BOARD_VERSION_PATCH=${BOARD_VERSION_PATCH}
            MARLIN_DISABLE_INFINITE_LOOP
            PROCESS_CUSTOM_GCODE
            HAS_MARLIN=$<BOOL:${HAS_MARLIN}>
            _EXTUI
  )

if(MCU MATCHES "STM32")
  target_compile_definitions(BuddyHeaders INTERFACE STM32GENERIC)
endif()

if(MCU MATCHES "STM32F4")
  target_compile_definitions(BuddyHeaders INTERFACE STM32F4 STM32F4xx)
elseif(MCU MATCHES "STM32G0")
  target_compile_definitions(BuddyHeaders INTERFACE STM32G0 STM32G0xx)
endif()

add_library(BuddyLogging INTERFACE)
target_include_directories(BuddyLogging INTERFACE src/logging)
target_link_libraries(BuddyHeaders INTERFACE BuddyLogging)

#
# Configure Arduino Core
#

if(MCU MATCHES "STM32F4")
  set(ARDUINO_CORE_BUDDY_USB_SUPPORT TRUE)
else()
  set(ARDUINO_CORE_BUDDY_USB_SUPPORT FALSE)
endif()
add_library(CoreBuddy_Config ALIAS BuddyHeaders)

#
# Configure STMicroelectronics Libraries
#

# HAL
if(MCU MATCHES "STM32F4")
  if(MCU MATCHES "STM32F40")
    set(STM32F4_HAL_TARGET "STM32F407xx")
  elseif(MCU MATCHES "STM32F42")
    set(STM32F4_HAL_TARGET "STM32F427xx")
  else()
    message(FATAL_ERROR "Don't know how to configure STM32F4::HAL for mcu ${MCU}")
  endif()
  add_library(STM32F4_HAL_Config INTERFACE)
  target_compile_definitions(STM32F4_HAL_Config INTERFACE ${STM32F4_HAL_TARGET} USE_HAL_DRIVER)
  target_include_directories(STM32F4_HAL_Config INTERFACE include/stm32f4_hal)
elseif(MCU MATCHES "STM32G0")
  add_library(STM32G0_HAL_Config INTERFACE)
  target_compile_definitions(STM32G0_HAL_Config INTERFACE STM32G070xx USE_HAL_DRIVER)
  target_include_directories(STM32G0_HAL_Config INTERFACE include/stm32g0_hal)
else()
  message(FATAL_ERROR "Unknown hardware platform of mcu ${MCU}")
endif()

# STM32::USBHost
add_library(STM32_USBHost_Config ALIAS BuddyHeaders)

# STM32::Utilities::CPU
add_library(STM32_Utilities_CPU_Config ALIAS BuddyHeaders)

#
# Configure SEGGER
#
add_library(Segger_Config INTERFACE)
target_include_directories(Segger_Config INTERFACE include/segger src/segger)
target_link_libraries(Segger_Config INTERFACE BuddyHeaders)

#
# Configure FreeRTOS
#

add_library(FreeRTOS_Config INTERFACE)
target_include_directories(FreeRTOS_Config INTERFACE include/freertos)
target_link_libraries(FreeRTOS_Config INTERFACE Segger BuddyHeaders)

#
# Configure LwIP
#

add_library(LwIP_Config INTERFACE)
target_link_libraries(LwIP_Config INTERFACE BuddyHeaders BuddyLogging)

#
# Configure FatFs
#

add_library(FatFs_Config INTERFACE)
target_link_libraries(FatFs_Config INTERFACE BuddyHeaders BuddyLogging STM32::USBHost)

#
# Configure Marlin
#

add_library(Marlin_Config INTERFACE)
# TODO: fix dependency on src/common and src/gui
target_include_directories(
  Marlin_Config INTERFACE include/marlin src/common src/gui src/marlin_stubs/include
  )
target_include_directories(Marlin_Config INTERFACE src/common/selftest)
target_link_libraries(Marlin_Config INTERFACE BuddyHeaders FreeRTOS::FreeRTOS)

#
# Configure tinyusb
#
add_library(tinyusb_dependencies INTERFACE)
target_include_directories(tinyusb_dependencies INTERFACE include/tinyusb)
target_link_libraries(tinyusb_dependencies INTERFACE FreeRTOS::FreeRTOS)

#
# Global Compiler & Linker Configuration
#

# Thread safe newlib
add_compile_options(-DconfigUSE_NEWLIB_REENTRANT=1)

# include symbols
add_compile_options(-g)

# optimizations
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  if(MCU MATCHES "STM32F40"
     OR MCU MATCHES "STM32G0"
     OR PRINTER STREQUAL "MK4"
     OR PRINTER STREQUAL "MK3.5"
     OR PRINTER STREQUAL "XL"
     OR PRINTER STREQUAL "iX"
     )
    add_compile_options(-Og)
  else()
    # O0 should be compiled with -fomit-frame-pointer to make sure there is enough registers for asm
    # functions
    add_compile_options(-O0 -fomit-frame-pointer)
  endif()
else()
  add_compile_options(-Os)
endif()

if(CMAKE_CROSSCOMPILING)
  # mcu related settings
  set(MCU_FLAGS -mthumb)
  if(MCU MATCHES "STM32F4")
    list(APPEND MCU_FLAGS -mcpu=cortex-m4 -mfloat-abi=hard -mfpu=fpv4-sp-d16)
  elseif(MCU MATCHES "STM32G0")
    list(APPEND MCU_FLAGS -mcpu=cortex-m0plus -mfloat-abi=soft)
  endif()
  add_compile_options(${MCU_FLAGS})
  add_link_options(${MCU_FLAGS})

  # better FreeRTOS support
  add_link_options(-Wl,--undefined=uxTopUsedPriority,--undefined=init_task)

  # split and gc sections
  add_compile_options(-ffunction-sections -fdata-sections)
  add_link_options(-Wl,--gc-sections)

  # disable exceptions and related metadata
  add_compile_options(-fno-exceptions -fno-unwind-tables)
  add_link_options(-Wl,--defsym,__exidx_start=0,--defsym,__exidx_end=0)

  # wrap _dtoa to ensure formatting of floats isn't being called from ISR
  add_link_options(-Wl,--wrap=_dtoa_r)

  # use custom printf implementation instead of the one in newlib (see lib/printf)
  add_link_options(
    -Wl,--defsym=printf=printf_,--defsym=sprintf=sprintf_,--defsym=snprintf=snprintf_,--defsym=vprintf=vprintf_,--defsym=vsnprintf=vsnprintf_
    )

  # wrap _dtoa to ensure formatting of floats isn't being called from ISR
  add_link_options(-Wl,--wrap=_dtoa_r)
endif()

# support _DEBUG macro (some code uses to recognize debug builds)
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_compile_definitions(_DEBUG)
endif()

# disable unaligned access
#
# * Otherwise, with optimizations turned on, the firmware crashes on startup.
#
# The main problem was caused by zlib, thus this switch was propagated directly to it, it seems to
# be solved now And let's keep this line commented here for emergency situations ;)
#
# add_compile_options(-mno-unaligned-access)

if(CMAKE_CROSSCOMPILING)
  # configure linker script
  if(BOOTLOADER STREQUAL "EMPTY" OR BOOTLOADER)
    set(BOOT_SUFFIX "_boot")
  else()
    set(BOOT_SUFFIX "")
  endif()
  if(MCU MATCHES "STM32F42")
    set(LINKER_SCRIPT "src/device/stm32f4/linker/stm32f42x${BOOT_SUFFIX}.ld")
  elseif(MCU MATCHES "STM32F407")
    set(LINKER_SCRIPT "src/device/stm32f4/linker/stm32f407vg${BOOT_SUFFIX}.ld")
  elseif(MCU MATCHES "STM32G070")
    set(LINKER_SCRIPT "src/device/stm32g0/linker/stm32g070rb${BOOT_SUFFIX}.ld")
  endif()
  set(LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/${LINKER_SCRIPT}")
  add_link_options("-Wl,-T,${LINKER_SCRIPT}")
endif()

#
# Import definitions of all libraries
#

add_subdirectory(lib)

#
# Build Puppy Firmwares
#

if(BOARD MATCHES ".*BUDDY" AND HAS_PUPPIES_BOOTLOADER)
  string(TOLOWER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE_lower)
  string(TOLOWER "${PRINTER}" PRINTER_lower)

  if(PRINTER STREQUAL "XL" AND BOARD_VERSION VERSION_EQUAL "0.4.0")
    set(dwarf_version_suffix "-04")
    set(modular_bed_version_suffix "-06")
  endif()

  if(NOT DWARF_BINARY_PATH AND HAS_DWARF)
    get_filename_component(
      DWARF_BINARY_DIR "${DWARF_BINARY_DIR}" REALPATH BASE_DIR "${CMAKE_SOURCE_DIR}"
      )
    get_filename_component(DWARF_BINARY_PATH "firmware.bin" REALPATH BASE_DIR "${DWARF_BINARY_DIR}")
    ExternalProject_Add(
      dwarf
      SOURCE_DIR "${DWARF_SOURCE_DIR}"
      BINARY_DIR "${DWARF_BINARY_DIR}"
      CMAKE_ARGS --preset xl-dwarf${dwarf_version_suffix}_${CMAKE_BUILD_TYPE_lower}_boot -B
                 "${DWARF_BINARY_DIR}"
      INSTALL_COMMAND ${CMAKE_COMMAND} -E echo "Skipping install step"
      BUILD_BYPRODUCTS "${DWARF_BINARY_PATH}" "${DWARF_BINARY_DIR}/firmware"
      STEP_TARGETS build
      USES_TERMINAL_BUILD TRUE
      USES_TERMINAL_INSTALL FALSE
      BUILD_ALWAYS TRUE
      )
  endif()

  if(NOT MODULARBED_BINARY_PATH)
    get_filename_component(
      MODULARBED_BINARY_DIR "${MODULARBED_BINARY_DIR}" REALPATH BASE_DIR "${CMAKE_SOURCE_DIR}"
      )
    get_filename_component(
      MODULARBED_BINARY_PATH "firmware.bin" REALPATH BASE_DIR "${MODULARBED_BINARY_DIR}"
      )
    ExternalProject_Add(
      modularbed
      SOURCE_DIR "${MODULARBED_SOURCE_DIR}"
      BINARY_DIR "${MODULARBED_BINARY_DIR}"
      CMAKE_ARGS
        --preset
        ${PRINTER_lower}-modularbed${modular_bed_version_suffix}_${CMAKE_BUILD_TYPE_lower}_boot -B
        "${MODULARBED_BINARY_DIR}"
      INSTALL_COMMAND ${CMAKE_COMMAND} -E echo "Skipping install step"
      BUILD_BYPRODUCTS "${MODULARBED_BINARY_PATH}" "${MODULARBED_BINARY_DIR}/firmware"
      STEP_TARGETS build
      USES_TERMINAL_BUILD TRUE
      USES_TERMINAL_INSTALL FALSE
      BUILD_ALWAYS TRUE
      )
  endif()
endif()

# Unittests
if(NOT CMAKE_CROSSCOMPILING)
  option(UNITTESTS_ENABLE "Enable building of unittest" ON)
  if(UNITTESTS_ENABLE)
    enable_testing()
    add_subdirectory(tests)
  endif()
endif()

# std::rand/random
if(BOARD MATCHES ".*BUDDY")
  set(ENABLE_HW_STD_RAND TRUE)
  set(RANDOM_CPP random_hw.cpp)
else()
  set(ENABLE_HW_STD_RAND FALSE)
  set(RANDOM_CPP random_sw.cpp)
endif()

#
# Buddy firmware
#

add_executable(firmware)
add_subdirectory(src)

target_compile_options(
  firmware
  PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-Wextra>
          $<$<COMPILE_LANGUAGE:CXX>:-Wno-expansion-to-defined>
          $<$<COMPILE_LANGUAGE:CXX>:-Wnon-virtual-dtor>
          $<$<COMPILE_LANGUAGE:CXX>:-Wduplicated-cond>
          $<$<COMPILE_LANGUAGE:CXX>:-Wlogical-op>
          $<$<COMPILE_LANGUAGE:CXX>:-Wnull-dereference>
          $<$<COMPILE_LANGUAGE:CXX>:-Wshadow=compatible-local>
          -Wdouble-promotion
  )

if(ENABLE_HW_STD_RAND)
  # Wrap rand and replace the implementation with one using HW RNG
  target_link_options(firmware PUBLIC -Wl,--wrap=rand)

  # Wrap srand, do not implement the wrapper -> throws error when referenced in the code. rand()
  # uses HW RNG now, so it cannot be seeded
  target_link_options(firmware PUBLIC -Wl,--wrap=srand)
endif()

# Appending fw descriptor (with calculated fingerprint) to the ELF
if(BOARD STREQUAL "DWARF" OR BOARD STREQUAL "MODULARBED")
  add_custom_command(
    TARGET firmware
    POST_BUILD
    # Generate a binary without fingerprint
    COMMAND "${CMAKE_OBJCOPY}" -O binary -S "$<TARGET_FILE:firmware>" "firmware_no_descriptor.bin"
    # Create the fw descriptor with calculated fingerprint of binary
    COMMAND "${Python3_EXECUTABLE}" "${CMAKE_SOURCE_DIR}/utils/gen_puppies_descriptor.py"
            "firmware_no_descriptor.bin" "fw_descriptor.bin"
    # Set the firmware's fw descriptor in the firmware's ELF file
    COMMAND "${CMAKE_OBJCOPY}" "--update-section" ".fw_descriptor=fw_descriptor.bin"
            "$<TARGET_FILE:firmware>" "$<TARGET_FILE:firmware>"
    VERBATIM
    )
endif()

# generate firmware.bin file
objcopy(firmware "binary" ".bin")

# generate linker map file
target_link_options(firmware PUBLIC -Wl,-Map=firmware.map,--print-memory-usage)

# link time optimizations
if(CMAKE_BUILD_TYPE STREQUAL "Release")
  if(MCU MATCHES "STM32F40")
    set_target_properties(firmware PROPERTIES INTERPROCEDURAL_OPTIMIZATION True)
  endif()
endif() # "Release"

# inform about the firmware's size in terminal
report_size(firmware)

add_link_dependency(firmware "${LINKER_SCRIPT}")

# generate .bbf version if requested
if(GENERATE_BBF)
  message(STATUS "Configured to generate .bbf version of the firmware.")
  message(STATUS "Signing Key: ${SIGNING_KEY}")

  if(PROJECT_VERSION_SUFFIX)
    if(NOT "${PROJECT_VERSION_SUFFIX}" MATCHES "\\+${BUILD_NUMBER}")
      message(WARNING "The generated .bbf is configured to use the build number ${BUILD_NUMBER},
      but the version suffix (${PROJECT_VERSION_SUFFIX}) does not contain +${BUILD_NUMBER}.
      Are you sure you know what you are doing?"
              )
    endif()
  endif()

  set(resource_images)
  set(resource_image_names)

  if(RESOURCES)
    list(APPEND resource_images resources-image)
    list(APPEND resource_image_names "RESOURCES_IMAGE")
  endif()

  if(BOOTLOADER_UPDATE)
    list(APPEND resource_images resources-bootloader-image)
    list(APPEND resource_image_names "RESOURCES_BOOTLOADER_IMAGE")
  endif()

  pack_firmware(
    firmware
    FW_VERSION
    "${PROJECT_VERSION}${PROJECT_VERSION_SUFFIX_SHORT}"
    BUILD_NUMBER
    "${BUILD_NUMBER}"
    PRINTER_TYPE
    "${PRINTER_TYPE}"
    PRINTER_VERSION
    "${PRINTER_VERSION}"
    PRINTER_SUBVERSION
    "${PRINTER_SUBVERSION}"
    SIGNING_KEY
    "${SIGNING_KEY}"
    BBF_VERSION
    "2"
    RESOURCE_IMAGES
    ${resource_images}
    RESOURCE_IMAGE_NAMES
    ${resource_image_names}
    )

  if(PRINTER STREQUAL "MINI")
    pack_firmware(
      firmware
      FW_VERSION
      "${PROJECT_VERSION_FULL}"
      BUILD_NUMBER
      "${BUILD_NUMBER}"
      PRINTER_TYPE
      "${PRINTER_TYPE}"
      PRINTER_VERSION
      "${PRINTER_VERSION}"
      PRINTER_SUBVERSION
      "${PRINTER_SUBVERSION}"
      SIGNING_KEY
      "${SIGNING_KEY}"
      BBF_VERSION
      "1"
      OUTPUT_PATH
      "firmware_update_pre_4.4.bbf"
      )
  endif()

elseif(SIGNING_KEY)
  message(WARNING "SIGNING_KEY specified but BBF generation is not enabled.")
endif()

# generate .dfu version if requested
if(GENERATE_DFU)
  if(BOOTLOADER)
    set(firmware_addr "0x08020000")
    if(BOOTLOADER STREQUAL "YES")
      get_dependency_directory("bootloader-${BOOTLOADER_VARIANT}" bootloader_dir)
      set(bootloader_input "0x08000000:${bootloader_dir}/bootloader.bin")
    endif()
    set(firmware_input "0x08020000:${CMAKE_BINARY_DIR}/firmware.bbf")
  else()
    set(firmware_input "0x08000000:${CMAKE_BINARY_DIR}/firmware.bin")
  endif()

  create_dfu(
    TARGET
    firmware
    INPUT
    "${bootloader_input}"
    "${firmware_input}"
    OUTPUT
    "${CMAKE_CURRENT_BINARY_DIR}/firmware.dfu"
    )
endif()

target_include_directories(
  firmware
  PRIVATE src/lang
          src/hw
          src/segger
          src/logging
          src/syslog
          src/mmu2
          include/mbedtls
  )

target_link_libraries(firmware PRIVATE BuddyHeaders Segger printf::printf CrashCatcher)

if(BOARD MATCHES "BUDDY")
  target_link_libraries(
    firmware
    PRIVATE BuddyHeaders
            Marlin
            Arduino::Core
            Arduino::TMCStepper
            Arduino::LiquidCrystal
            LwIP
            FatFs
            flasher
            littlefs
            STM32::USBHost
            inih::inih
            $<$<BOOL:${WUI}>:WUI>
            jsmn::jsmn
            QR
            Buddy::Lang
            CppStdExtensions
            printf::printf
            sysbase
            Segger
            tinyusb::tinyusb
            $<$<BOOL:${HAS_MMU2}>:MMU2::MMU2>
            mbedTLS
            $<$<BOOL:${HAS_PUPPIES}>:lightmodbus>
            error_codes
            error_codes_mmu # TODO once MMU is not necessary for the build, replace with:
                            # $<$<BOOL:${HAS_MMU2}>:error_codes_mmu>
            bgcode_core
            heatshrink_decoder
    )
elseif(BOARD STREQUAL "DWARF")
  target_link_libraries(firmware PRIVATE BuddyHeaders Marlin Arduino::Core Arduino::TMCStepper)
elseif(BOARD STREQUAL "MODULARBED")
  target_link_libraries(firmware PRIVATE BuddyHeaders)
endif()

set_property(
  SOURCE src/common/version.cpp
  APPEND
  PROPERTY COMPILE_DEFINITIONS
           FW_BUILD_NUMBER=${BUILD_NUMBER}
           FW_VERSION_FULL=${PROJECT_VERSION_FULL}
           FW_VERSION=${PROJECT_VERSION}
           FW_VERSION_MAJOR=${PROJECT_VERSION_MAJOR}
           FW_VERSION_MINOR=${PROJECT_VERSION_MINOR}
           FW_VERSION_PATCH=${PROJECT_VERSION_PATCH}
           FW_VERSION_SUFFIX=${PROJECT_VERSION_SUFFIX}
           FW_VERSION_SUFFIX_SHORT=${PROJECT_VERSION_SUFFIX_SHORT}
  )
