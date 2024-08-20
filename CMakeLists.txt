cmake_minimum_required(VERSION 3.2...3.5)
project(DiscordRPC LANGUAGES C CXX)
set(DRPC_VERSION 3.4.0)

include(GNUInstallDirs)
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

set(CMAKE_CXX_STANDARD 14)

option(ENABLE_IO_THREAD "Start up a separate I/O thread, otherwise I'd need to call an update function" ON)
option(BUILD_EXAMPLES "Build example apps" ON)

# Warning flags macros
macro(drpc_add_cxx_warning_flag CTARGET WARNINGFLAG WARNING_VAR)
    check_cxx_compiler_flag("-W${WARNINGFLAG}" HAVE_CXXW_${WARNING_VAR})
    if(HAVE_CXXW_${WARNING_VAR})
        target_compile_options(${CTARGET} PRIVATE -W${WARNINGFLAG})
    endif()
endmacro()

macro(drpc_disable_cxx_warning_flag CTARGET WARNINGFLAG WARNING_VAR)
    check_cxx_compiler_flag("-W${WARNINGFLAG}" HAVE_CXXW_${WARNING_VAR})
    if(HAVE_CXXW_${WARNING_VAR})
        target_compile_options(${CTARGET} PRIVATE -Wno-${WARNINGFLAG})
    endif()
endmacro()

# Add compiler argument(s)
macro(drpc_add_cxx_compiler_flag CTARGET COMPILERFLAG COMPILERFLAG_VAR)
    check_cxx_compiler_flag("${COMPILERFLAG}" HAVE_CXX_${COMPILERFLAG_VAR})
    if(HAVE_CXX_${COMPILERFLAG_VAR})
       target_compile_options(${CTARGET} PRIVATE ${COMPILERFLAG})
    endif()
endmacro()

set(BASE_RPC_SRC
    include/discord_rpc.h
    include/discord_register.h
    src/discord_rpc.cpp
    src/rpc_connection.h
    src/rpc_connection.cpp
    src/serialization.h
    src/serialization.cpp
    src/connection.h
    src/backoff.h
    src/msg_queue.h
)

if(WIN32)
    list(APPEND BASE_RPC_SRC
        src/connection_win.cpp
        src/discord_register_win.cpp
    )
else()
    list(APPEND BASE_RPC_SRC
        src/connection_unix.cpp
    )

    if(APPLE)
        add_definitions(-DDISCORD_OSX)
        list(APPEND BASE_RPC_SRC
            src/discord_register_osx.m
        )
    else()
        add_definitions(-DDISCORD_LINUX)
        list(APPEND BASE_RPC_SRC
            src/discord_register_linux.cpp
        )
    endif()
endif()

add_library(discord-rpc ${BASE_RPC_SRC})

target_include_directories(discord-rpc PRIVATE include)

if(MSVC)
    target_compile_options(discord-rpc PRIVATE /EHsc
        /Wall
        /wd4100 # unreferenced formal parameter
        /wd4514 # unreferenced inline
        /wd4625 # copy constructor deleted
        /wd5026 # move constructor deleted
        /wd4626 # move assignment operator deleted
        /wd4668 # not defined preprocessor macro
        /wd4710 # function not inlined
        /wd4711 # function was inlined
        /wd4820 # structure padding
        /wd4946 # reinterpret_cast used between related classes
        /wd5027 # move assignment operator was implicitly defined as deleted
    )
else()
    set_target_properties(discord-rpc PROPERTIES SOVERSION ${DRPC_VERSION})

    drpc_add_cxx_warning_flag(discord-rpc all ALL)
    drpc_add_cxx_warning_flag(discord-rpc extra EXTRA)
    drpc_add_cxx_warning_flag(discord-rpc pedantic PEDANTIC)

    drpc_disable_cxx_warning_flag(discord-rpc unknown-pragmas UNKNOWN_PRAGMAS) # pragma push thing doesn't work on clang
    drpc_disable_cxx_warning_flag(discord-rpc old-style-cast OLD_STYLE_CAST) # it's fine
    drpc_disable_cxx_warning_flag(discord-rpc c++98-compat CPP98COMPAT) # that was almost 2 decades ago
    drpc_disable_cxx_warning_flag(discord-rpc c++98-compat-pedantic CPP98COMPAT_PEDANTIC)
    drpc_disable_cxx_warning_flag(discord-rpc missing-noreturn MISSING_NORETURN)
    drpc_disable_cxx_warning_flag(discord-rpc padding PADDING) # structure padding
    drpc_disable_cxx_warning_flag(discord-rpc covered-switch-default COVERED_SWITCH_DEFAULT)
    drpc_disable_cxx_warning_flag(discord-rpc exit-time-destructors EXIT_TIME_DESTRUCTORS) # not sure about these
    drpc_disable_cxx_warning_flag(discord-rpc global-constructors GLOBAL_CONSTRUCTORS) # not sure about these
    drpc_disable_cxx_warning_flag(discord-rpc class-memaccess CLASSMEMACCESS)
endif()

if(NOT ENABLE_IO_THREAD)
    target_compile_definitions(discord-rpc PUBLIC -DDISCORD_DISABLE_IO_THREAD)
endif()

if(WIN32)
    target_link_libraries(discord-rpc PRIVATE psapi advapi32)
endif()

if(APPLE)
    target_link_libraries(discord-rpc PRIVATE "-framework AppKit")
endif()

if(BUILD_SHARED_LIBS)
    target_compile_definitions(discord-rpc PUBLIC -DDISCORD_DYNAMIC_LIB)
    target_compile_definitions(discord-rpc PRIVATE -DDISCORD_BUILDING_SDK)
endif()

if(BUILD_EXAMPLES)
    add_subdirectory(test/send-presence)
endif()

install(
    TARGETS discord-rpc
    EXPORT "discord-rpc"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
    FILES
        "include/discord_rpc.h"
        "include/discord_register.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
