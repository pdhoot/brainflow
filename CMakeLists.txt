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
    
    # For iOS builds, disable deprecated warnings treated as errors
    if (APPLE AND CMAKE_OSX_DEPLOYMENT_TARGET)
        message (STATUS "Configuring for iOS build with deployment target ${CMAKE_OSX_DEPLOYMENT_TARGET}")
        add_compile_options (-Wno-deprecated-declarations)
        add_compile_options (-Wno-error=deprecated-declarations)
        
        # iOS-specific definitions to handle platform differences
        add_definitions (-DIOS_PLATFORM)
        add_definitions (-DNO_FILESYSTEM)
        # Prevent unsupported thread operations on iOS
        add_definitions (-DSIMPLE_THREADING)
        # Fix for uint8 type issue
        add_definitions (-Duint8=unsigned\ char)
        # Disable missing declaration warnings
        add_compile_options (-Wno-missing-declarations)
    endif ()
    
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
    
    # Set up library name based on architecture
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        SET (BOARD_CONTROLLER_NAME "BoardController")
        if (APPLE)
            SET (BOARD_CONTROLLER_COMPILED_NAME "libBoardController.dylib")
        elseif (UNIX)
            SET (BOARD_CONTROLLER_COMPILED_NAME "libBoardController.so")
        else ()
            SET (BOARD_CONTROLLER_COMPILED_NAME "BoardController.dll")
            SET (BOARD_CONTROLLER_COMPILED_NAME_DOT_LIB "BoardController.lib")
        endif (APPLE)
    else (CMAKE_SIZEOF_VOID_P EQUAL 8)
        if (APPLE)
            SET (BOARD_CONTROLLER_NAME "BoardController")
            SET (BOARD_CONTROLLER_COMPILED_NAME "libBoardController.dylib")
        elseif (UNIX)
            SET (BOARD_CONTROLLER_NAME "BoardController")
            SET (BOARD_CONTROLLER_COMPILED_NAME "libBoardController.so")
        else ()
            SET (BOARD_CONTROLLER_NAME "BoardController32")
            SET (BOARD_CONTROLLER_COMPILED_NAME "BoardController32.dll")
            SET (BOARD_CONTROLLER_COMPILED_NAME_DOT_LIB "BoardController32.lib")
        endif (APPLE)
    endif (CMAKE_SIZEOF_VOID_P EQUAL 8)
    
    # Create the library
    add_library (
        ${BOARD_CONTROLLER_NAME} SHARED
        ${BOARD_CONTROLLER_SRC}
    )

    # Set include directories
    target_include_directories (
        ${BOARD_CONTROLLER_NAME} PRIVATE
        ${CMAKE_HOME_DIRECTORY}/third_party/
        ${CMAKE_HOME_DIRECTORY}/third_party/json
        ${CMAKE_HOME_DIRECTORY}/third_party/http
        ${CMAKE_HOME_DIRECTORY}/third_party/oscpp/include
        ${CMAKE_HOME_DIRECTORY}/src/utils/inc
        ${CMAKE_HOME_DIRECTORY}/src/utils/bluetooth/inc
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/muse/inc
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/muse/muse_bglib/inc
    )

    # Set compile definitions
    target_compile_definitions(${BOARD_CONTROLLER_NAME} PRIVATE NOMINMAX BRAINFLOW_VERSION=${BRAINFLOW_VERSION})

    # Set output directories
    set_target_properties (${BOARD_CONTROLLER_NAME}
        PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_HOME_DIRECTORY}/compiled
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_HOME_DIRECTORY}/compiled
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_HOME_DIRECTORY}/compiled
    )

    # Handle system-specific linking
    if (UNIX AND NOT ANDROID)
        # For iOS builds, we need different linking options
        if (APPLE AND CMAKE_OSX_DEPLOYMENT_TARGET)
            # iOS doesn't need pthread and dl linking
            target_link_libraries (${BOARD_CONTROLLER_NAME} PRIVATE)
        else()
            # Regular macOS or Linux builds
            target_link_libraries (${BOARD_CONTROLLER_NAME} PRIVATE pthread dl)
        endif()
    endif (UNIX AND NOT ANDROID)
    if (ANDROID)
        find_library (log-lib log)
        target_link_libraries (${BOARD_CONTROLLER_NAME} PRIVATE log)
    endif (ANDROID)

    # Install the library
    install (
        FILES
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h
        ${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_constants.h
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h
        ${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/brainflow_input_params.h
        ${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export.h
        ${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_array.h
        ${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_exception.h
        ${CMAKE_HOME_DIRECTORY}/third_party/json/json.hpp
        DESTINATION inc
    )

    install (
        TARGETS ${BOARD_CONTROLLER_NAME}
        EXPORT ${TARGETS_EXPORT_NAME}
        RUNTIME DESTINATION lib
        LIBRARY DESTINATION lib
        INCLUDES DESTINATION inc
        ARCHIVE DESTINATION lib
    )
    
    # Copy library to package directories
    if (MSVC)
        add_custom_command (TARGET ${BOARD_CONTROLLER_NAME} POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/nodejs_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java_package/brainflow/src/main/resources/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp_package/brainflow/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/board_controller.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/board_info_getter.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/shared_export.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/$<CONFIG>/${BOARD_CONTROLLER_COMPILED_NAME_DOT_LIB}" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME_DOT_LIB}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/board_controller.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/board_info_getter.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_constants.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/brainflow_constants.h"
        )
    endif (MSVC)
    if (UNIX AND NOT ANDROID)
        add_custom_command (TARGET ${BOARD_CONTROLLER_NAME} POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/nodejs_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/python_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/julia_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/java_package/brainflow/src/main/resources/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/csharp_package/brainflow/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/board_controller.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/board_info_getter.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/shared_export_matlab.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/shared_export.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_constants.h" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/inc/brainflow_constants.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/matlab_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/compiled/${BOARD_CONTROLLER_COMPILED_NAME}" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/lib/${BOARD_CONTROLLER_COMPILED_NAME}"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_controller.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/board_controller.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/board_controller/inc/board_info_getter.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/board_info_getter.h"
            COMMAND "${CMAKE_COMMAND}" -E copy_if_different "${CMAKE_HOME_DIRECTORY}/src/utils/inc/brainflow_constants.h" "${CMAKE_HOME_DIRECTORY}/rust_package/brainflow/inc/brainflow_constants.h"
        )
    endif (UNIX AND NOT ANDROID)
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