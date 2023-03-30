cmake_minimum_required(VERSION 3.10)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

option(TINY_REGEX_C_DOT_MATCHES_NEWLINE "'.' matches '\r' + '\n'"   ON)
option(TINY_REGEX_C_RECURSIVE_MATCHING  "Enable recursive matching" OFF)

if(!TINY_REGEX_C_DOT_MATCHES_NEWLINE)
    add_definitions(-DRE_DOT_MATCHES_NEWLINE=0)
endif()

if(TINY_REGEX_C_RECURSIVE_MATCHING)
    add_definitions(-DRECURSIVE_RE)
endif()

project(TinyRegexC 
    VERSION 
        "0.0.0"
    DESCRIPTION 
        "Small, portable regex pattern matcher in C."
    HOMEPAGE_URL
        "https://github.com/kokke/tiny-regex-c"
    LANGUAGES C)

add_library(TinyRegexC STATIC
            re.h 
            re.c)

target_include_directories(libGimbal
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)
