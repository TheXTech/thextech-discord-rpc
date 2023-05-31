include_directories(${PROJECT_SOURCE_DIR}/include)

option(ENABLE_IO_THREAD "Start up a separate I/O thread, otherwise I'd need to call an update function" ON)
option(USE_STATIC_CRT "Use /MT[d] for dynamic library" OFF)
option(WARNINGS_AS_ERRORS "When enabled, compiles with `-Werror` (on *nix platforms)." OFF)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

set(CMAKE_CXX_STANDARD 14)

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
    ${PROJECT_SOURCE_DIR}/include/discord_rpc.h
    discord_rpc.cpp
    ${PROJECT_SOURCE_DIR}/include/discord_register.h
    rpc_connection.h
    rpc_connection.cpp
    serialization.h
    serialization.cpp
    connection.h
    backoff.h
    msg_queue.h
)

if (${BUILD_SHARED_LIBS})
    if(WIN32)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} dllmain.cpp)
    endif(WIN32)
endif(${BUILD_SHARED_LIBS})

if(WIN32)
    add_definitions(-DDISCORD_WINDOWS)
    set(BASE_RPC_SRC ${BASE_RPC_SRC} connection_win.cpp discord_register_win.cpp)
    add_library(discord-rpc ${BASE_RPC_SRC})
    if (MSVC)
        if(USE_STATIC_CRT)
            foreach(CompilerFlag
                    CMAKE_CXX_FLAGS
                    CMAKE_CXX_FLAGS_DEBUG
                    CMAKE_CXX_FLAGS_RELEASE
                    CMAKE_C_FLAGS
                    CMAKE_C_FLAGS_DEBUG
                    CMAKE_C_FLAGS_RELEASE)
                string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
            endforeach()
        endif(USE_STATIC_CRT)
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
    endif(MSVC)
    target_link_libraries(discord-rpc PRIVATE psapi advapi32)
endif(WIN32)

if(UNIX)
    set(BASE_RPC_SRC ${BASE_RPC_SRC} connection_unix.cpp)

    if (APPLE)
        add_definitions(-DDISCORD_OSX)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} discord_register_osx.m)
    else (APPLE)
        add_definitions(-DDISCORD_LINUX)
        set(BASE_RPC_SRC ${BASE_RPC_SRC} discord_register_linux.cpp)
    endif(APPLE)

    add_library(discord-rpc ${BASE_RPC_SRC})
    set_target_properties(discord-rpc PROPERTIES SOVERSION ${DRPC_VERSION})
    target_link_libraries(discord-rpc PUBLIC pthread)

    if (APPLE)
        target_link_libraries(discord-rpc PRIVATE "-framework AppKit, -mmacosx-version-min=10.10")
    endif (APPLE)

    target_compile_options(discord-rpc PRIVATE
        -g
        -Wall
        -Wextra
        -Wpedantic
    )

    if (${WARNINGS_AS_ERRORS})
      target_compile_options(discord-rpc PRIVATE -Werror)
    endif (${WARNINGS_AS_ERRORS})

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

    if (${BUILD_SHARED_LIBS})
        target_compile_options(discord-rpc PRIVATE -fPIC)
    endif (${BUILD_SHARED_LIBS})

    if (APPLE)
        target_link_libraries(discord-rpc PRIVATE "-framework AppKit")
    endif (APPLE)
endif(UNIX)

target_include_directories(discord-rpc PRIVATE ${RAPIDJSON}/include)

if (NOT ${ENABLE_IO_THREAD})
    target_compile_definitions(discord-rpc PUBLIC -DDISCORD_DISABLE_IO_THREAD)
endif (NOT ${ENABLE_IO_THREAD})

if (${BUILD_SHARED_LIBS})
    target_compile_definitions(discord-rpc PUBLIC -DDISCORD_DYNAMIC_LIB)
    target_compile_definitions(discord-rpc PRIVATE -DDISCORD_BUILDING_SDK)
endif(${BUILD_SHARED_LIBS})

if (CLANG_FORMAT_CMD)
    add_dependencies(discord-rpc clangformat)
endif(CLANG_FORMAT_CMD)

# install

install(
    TARGETS discord-rpc
    EXPORT "discord-rpc"
    RUNTIME
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
    LIBRARY
        DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE
        DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    INCLUDES
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
    FILES
        "../include/discord_rpc.h"
		"../include/discord_register.h"
    DESTINATION "include"
)
