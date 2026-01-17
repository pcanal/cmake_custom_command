cmake_minimum_required(VERSION 3.30)
project(CustomCommands)

include_directories(${CMAKE_SOURCE_DIR})

add_custom_command(OUTPUT othergensource.cxx
   COMMAND sleep 2 && touch othergensource.cxx
   DEPENDS ${CMAKE_BINARY_DIR}/sub1/genhelper.txt)

add_library(sub SHARED othergensource.cxx)
target_link_libraries(sub PUBLIC main)

