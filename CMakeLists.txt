cmake_minimum_required(VERSION 3.10)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

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