cmake_minimum_required(VERSION 3.24)

include(CheckSymbolExists)
include(CheckIPOSupported)

# Project Metadata
project(ninja VERSION 1.0 LANGUAGES CXX)

# Options
option(NINJA_BUILD_BINARY "Build the ninja binary" ON)
option(NINJA_FORCE_PSELECT "Use pselect() even on platforms that provide ppoll()" OFF)
option(NINJA_CLANG_TIDY "Run clang-tidy on source files" OFF)
option(NINJA_PYTHON "Python interpreter to use for the browse tool" "python")

# Link-Time Optimization (IPO/LTO)
check_ipo_supported(RESULT lto_supported OUTPUT error)
if(lto_supported)
    message(STATUS "IPO / LTO enabled")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE)
else()
    message(STATUS "IPO / LTO not supported: ${error}")
endif()

# Compiler Flags
if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    add_compile_options(
        /W4 /wd4100 /wd4267 /wd4706 /wd4702 /wd4244 /GR- /Zc:__cplusplus
    )
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
else()
    include(CheckCXXCompilerFlag)
    check_cxx_compiler_flag(-Wno-deprecated flag_no_deprecated)
    if(flag_no_deprecated)
        add_compile_options(-Wno-deprecated)
    endif()

    if(CMAKE_VERSION VERSION_LESS 3.24)
        check_cxx_compiler_flag(-fdiagnostics-color flag_color_diag)
        if(flag_color_diag)
            add_compile_options(-fdiagnostics-color)
        endif()
    else()
        set(CMAKE_COLOR_DIAGNOSTICS ON)
    endif()

    # Check for ppoll() usage
    if(NOT NINJA_FORCE_PSELECT)
        include(CheckCXXSymbolExists)
        check_cxx_symbol_exists(ppoll poll.h HAVE_PPOLL)
        if(HAVE_PPOLL)
            add_compile_definitions(USE_PPOLL=1)
        endif()
    endif()
endif()

# RE2C Integration
find_program(RE2C re2c)
if(RE2C)
    execute_process(COMMAND "${RE2C}" --vernum OUTPUT_VARIABLE RE2C_RAW_VERSION)
    math(EXPR RE2C_MAJOR_VERSION "${RE2C_RAW_VERSION} / 10000")
else()
    set(RE2C_MAJOR_VERSION 0)
endif()

if(RE2C_MAJOR_VERSION GREATER 1)
    function(re2c IN OUT)
        add_custom_command(
            DEPENDS ${IN}
            OUTPUT ${OUT}
            COMMAND ${RE2C} -b -i --no-generation-date --no-version -o ${OUT} ${IN}
        )
    endfunction()

    re2c(${PROJECT_SOURCE_DIR}/src/depfile_parser.in.cc ${PROJECT_BINARY_DIR}/depfile_parser.cc)
    re2c(${PROJECT_SOURCE_DIR}/src/lexer.in.cc ${PROJECT_BINARY_DIR}/lexer.cc)
    add_library(libninja-re2c OBJECT 
        ${PROJECT_BINARY_DIR}/depfile_parser.cc 
        ${PROJECT_BINARY_DIR}/lexer.cc
    )
else()
    message(WARNING "re2c version >=2 not found; static parser files will be used.")
    add_library(libninja-re2c OBJECT src/depfile_parser.cc src/lexer.cc)
endif()

# Include directories
target_include_directories(libninja-re2c PRIVATE src)

# Browse Mode Check
function(check_platform_supports_browse_mode RESULT)
    execute_process(
        COMMAND sh -c "echo 'TEST' | src/inline.sh var"
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        RESULT_VARIABLE inline_result
        OUTPUT_QUIET
        ERROR_QUIET
    )
    if(NOT inline_result EQUAL "0")
        set(${RESULT} "0" PARENT_SCOPE)
        if(NOT WIN32)
            message(WARNING "Browse feature omitted due to inline script failure")
        endif()
        return()
    endif()

    check_symbol_exists(fork "unistd.h" HAVE_FORK)
    check_symbol_exists(pipe "unistd.h" HAVE_PIPE)

    if(HAVE_FORK AND HAVE_PIPE)
        set(${RESULT} "1" PARENT_SCOPE)
    else()
        set(${RESULT} "0" PARENT_SCOPE)
        message(WARNING "Browse feature omitted due to missing fork and pipe functions")
    endif()
endfunction()

check_platform_supports_browse_mode(platform_supports_ninja_browse)

# Main Library
add_library(libninja OBJECT
    src/build_log.cc src/build.cc src/clean.cc src/clparser.cc
    src/dyndep.cc src/dyndep_parser.cc src/debug_flags.cc
    src/deps_log.cc src/disk_interface.cc src/edit_distance.cc
    src/elide_middle.cc src/eval_env.cc src/graph.cc src/graphviz.cc
    src/json.cc src/line_printer.cc src/manifest_parser.cc src/metrics.cc
    src/missing_deps.cc src/parser.cc src/real_command_runner.cc src/state.cc
    src/status_printer.cc src/string_piece_util.cc src/util.cc src/version.cc
)
target_compile_features(libninja PUBLIC cxx_std_11)
if(WIN32)
    target_sources(libninja PRIVATE src/subprocess-win32.cc)
    target_compile_definitions(libninja PRIVATE NOMINMAX)
else()
    target_sources(libninja PRIVATE src/subprocess-posix.cc)
endif()

# Executable
if(NINJA_BUILD_BINARY)
    add_executable(ninja src/ninja.cc)
    target_link_libraries(ninja PRIVATE libninja libninja-re2c)
    if(platform_supports_ninja_browse)
        target_sources(ninja PRIVATE src/browse.cc)
        target_compile_definitions(ninja PRIVATE NINJA_HAVE_BROWSE)
    endif()
    install(TARGETS ninja)
endif()

# Tests
include(CTest)
if(BUILD_TESTING)
    enable_testing()
    find_package(GTest REQUIRED)
    add_executable(ninja_test src/test.cc)
    target_link_libraries(ninja_test PRIVATE libninja GTest::gtest)
    add_test(NAME NinjaTest COMMAND ninja_test)
endif()
