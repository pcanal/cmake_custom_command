include_directories(${CMAKE_SOURCE_DIR})

add_custom_command(OUTPUT gensource.cxx genhelper.txt
   COMMAND touch genhelper.txt && sleep 2 && touch gensource.cxx
   IMPLICIT_DEPENDS CXX ${CMAKE_SOURCE_DIR}/sub1/topheader.h
   DEPENDS ${CMAKE_SOURCE_DIR}/sub1/topheader.h
   COMMAND_EXPAND_LISTS
)

add_library(main SHARED gensource.cxx)

