cmake_minimum_required (VERSION 3.13)

project (brainflow)

SET (CMAKE_CXX_STANDARD 11)
SET (CMAKE_VERBOSE_MAKEFILE ON)
SET (BRAINFLOW_VERSION "0.0.1" CACHE STRING "BrainFlow Version")
SET (VERSION ${BRAINFLOW_VERSION})

SET (CMAKE_CXX_VISIBILITY_PRESET hidden)
SET (CMAKE_C_VISIBILITY_PRESET hidden)
SET (CMAKE_POSITION_INDEPENDENT_CODE ON)
SET (VERSION_CONFIG ${CMAKE_CURRENT_SOURCE_DIR}/compiled/brainflowConfigVersion.cmake)
SET (CONFIG_INSTALL_DIR lib/cmake/brainflow)
SET (PROJECT_CONFIG ${CMAKE_CURRENT_SOURCE_DIR}/compiled/brainflowConfig.cmake)
SET (TARGETS_EXPORT_NAME brainflowTargets)
SET (PACKAGE_LIB_INSTALL_DIR ${CMAKE_INSTALL_PREFIX}/lib)
SET (PACKAGE_INC_INSTALL_DIR ${CMAKE_INSTALL_PREFIX}/inc)

# add option to build only Muse boards
option (BUILD_ONLY_MUSE "Build only Muse boards" OFF)

option (USE_LIBFTDI "USE_LIBFTDI" OFF)
option (USE_OPENMP "USE_OPENMP" OFF)
option (WARNINGS_AS_ERRORS "WARNINGS_AS_ERRORS" OFF)
option (BUILD_OYMOTION_SDK "BUILD_OYMOTION_SDK" OFF)
option (BUILD_SYNCHRONI_SDK "BUILD_SYNCHRONI_SDK" ON)
option (BUILD_BLUETOOTH "BUILD_BLUETOOTH" OFF)
option (BUILD_BLE "BUILD_BLE" OFF)
option (BUILD_ONNX "BUILD_ONNX" OFF)
option (BUILD_TESTS "BUILD_TESTS" OFF)
option (BUILD_PERIPHERY "BUILD_PERIPHERY" OFF)

include (${CMAKE_CURRENT_SOURCE_DIR}/cmake/macros.cmake)
configure_msvc_runtime ()

add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")

if (WARNINGS_AS_ERRORS)
    if (MSVC)
        add_compile_options (/WX)
        if (CMAKE_BUILD_TYPE EQUAL "DEBUG")
            add_compile_options (/bigobj)
        endif (CMAKE_BUILD_TYPE EQUAL "DEBUG")
    else ()
        add_compile_options (-Werror -Wno-varargs -Wno-error=deprecated-declarations)
    endif ()
endif (WARNINGS_AS_ERRORS)

# Muse-specific build option
if (BUILD_ONLY_MUSE)
    message (STATUS "Building only Muse boards")
    add_definitions (-DBUILD_ONLY_MUSE)
    set (BOARD_CONTROLLER_SRC
        ${CMAKE_HOME_DIRECTORY}/src/utils/timestamp.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/data_buffer.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/os_serial.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/os_serial_ioctl.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/serial.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/libftdi_serial.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/socket_client_tcp.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/socket_client_udp.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/socket_server_tcp.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/socket_server_udp.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/multicast_client.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/multicast_server.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/broadcast_client.cpp
        ${CMAKE_HOME_DIRECTORY}/src/utils/broadcast_server.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/board_controller.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/board_info_getter.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/brainflow_boards.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/streaming_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/synthetic_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/dyn_lib_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/bt_lib_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/playback_file_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/file_streamer.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/multicast_streamer.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/plotjuggler_udp_streamer.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/ble_lib_board.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/muse/muse_bled.cpp
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/muse/muse.cpp
    )
    include (${CMAKE_HOME_DIRECTORY}/src/board_controller/muse/muse_bglib/build.cmake)
else ()
    # Include regular build.cmake that builds all boards
    include (${CMAKE_HOME_DIRECTORY}/src/board_controller/build.cmake)
endif (BUILD_ONLY_MUSE)

include (${CMAKE_CURRENT_SOURCE_DIR}/src/data_handler/build.cmake)
include (${CMAKE_CURRENT_SOURCE_DIR}/src/ml/build.cmake)
include (${CMAKE_CURRENT_SOURCE_DIR}/cpp_package/build.cmake)
if (BUILD_TESTS) 
    include (${CMAKE_CURRENT_SOURCE_DIR}/src/tests/build.cmake)
endif (BUILD_TESTS)

#add_subdirectory (${CMAKE_CURRENT_SOURCE_DIR}/cpp_package/examples/get_data)

include (CMakePackageConfigHelpers)

write_basic_package_version_file (
    ${VERSION_CONFIG}
    VERSION ${BRAINFLOW_VERSION}
    COMPATIBILITY AnyNewerVersion
)
configure_package_config_file (
    cmake/Config.cmake.in
    ${PROJECT_CONFIG}
    INSTALL_DESTINATION ${CONFIG_INSTALL_DIR}
)

install (
    FILES ${PROJECT_CONFIG} ${VERSION_CONFIG}
    DESTINATION ${CONFIG_INSTALL_DIR}
)
install (
    EXPORT ${TARGETS_EXPORT_NAME}
    NAMESPACE brainflow::
    DESTINATION ${CONFIG_INSTALL_DIR}
)