#
# Copyright 2019, Data61
# Commonwealth Scientific and Industrial Research Organisation (CSIRO)
# ABN 41 687 119 230.
#
# This software may be distributed and modified according to the terms of
# the BSD 2-Clause license. Note that NO WARRANTY is provided.
# See "LICENSE_BSD2.txt" for details.
#
# @TAG(DATA61_BSD)
#

cmake_minimum_required(VERSION 3.7.2)
include(settings.cmake)
project(picoserver C CXX ASM)

find_package(camkes-tool REQUIRED)
find_package(global-components REQUIRED)
find_package(sel4_projects_libs REQUIRED)

# Set some options we know we need here. Applications can override them
set(CapDLLoaderMaxObjects 20000 CACHE STRING "")
set(KernelRootCNodeSizeBits 17 CACHE STRING "")
if(KernelArchARM)
    ApplyData61ElfLoaderSettings(${KernelPlatform} ${KernelSel4Arch})
endif()
camkes_tool_setup_camkes_build_environment()

sel4_projects_libs_import_libraries()

function(includeGlobalComponents)
    global_components_import_project()
endfunction()

enable_testing()
macro(add_simulate_test test_script)
    set(TEST_SCRIPT ${test_script})
    configure_file(${CMAKE_SOURCE_DIR}/tests/test_script.tcl.in ${CMAKE_BINARY_DIR}/test.tcl @ONLY)
    add_test(${CMAKE_PROJECT_NAME} expect -f ${CMAKE_BINARY_DIR}/test.tcl)
endmacro()

includeGlobalComponents()

set(LibPicotcp ON CACHE BOOL "" FORCE)
set(LibPicotcpBsd OFF CACHE BOOL "" FORCE)
set(LibEthdriverNumPreallocatedBuffers 32 CACHE STRING "" FORCE)

# For x86, we map DMA frames into the IOMMU to use as buffers for the
# Ethernet device. The VKA and VSpace libraries do not like pages that
# are not 4K in size.
set(CAmkESDMALargeFramePromotion OFF CACHE BOOL "" FORCE)

# The app has only been tested on hardware, and not on QEMU
set(SIMULATION OFF CACHE BOOL "" FORCE)
if("${KernelArch}" STREQUAL "x86")
    # The IOMMU is required for the Ethdriver component on x86
    set(KernelIOMMU ON CACHE BOOL "" FORCE)
endif()

set(CAmkESCPP ON CACHE BOOL "" FORCE)
if("${KernelArch}" STREQUAL "x86")
    set(cpp_define -DKernelArchX86)
elseif("${KernelArch}" STREQUAL "arm")
    set(cpp_define -DKernelArchArm)
endif()

set(PICOSERVER_IP_ADDR "" CACHE STRING "IP address for the Picoserver component")

DeclareCAmkESComponent(
    Echo
    SOURCES
    components/Echo/src/echo.c
    INCLUDES
    components/include/
    LIBS
    picotcp
    PicoServer-client
)
DeclareCAmkESComponent(
    Listener
    SOURCES
    components/Listener/src/listener.c
    INCLUDES
    components/include/
    LIBS
    picotcp
    PicoServer-client
)

DeclareCAmkESRootserver(
    picoserver.camkes
    CPP_FLAGS
    ${cpp_define}
    -DPICOSERVER_IP_ADDR=\"${PICOSERVER_IP_ADDR}\"
)

GenerateCAmkESRootserver()