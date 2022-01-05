# CMake setup
cmake_minimum_required (VERSION 3.10.3)
MESSAGE(STATUS "CMAKE_ROOT: " ${CMAKE_ROOT})

# Project name
project(HeXinSTA)

# Turn on the verbose
set(CMAKE_VERBOSE_MAKEFILE ON)


#Set executable output directory
set(HOME ${PROJECT_SOURCE_DIR})
set(EXECUTABLE_OUTPUT_PATH ${HOME}/bin)
set(COMPILEABLE_PATH ${HOME}/src)
file(MAKE_DIRECTORY "lib")


# CXX target properties
## g++
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "7.3")
    message(FATAL_ERROR "\nHeXinSta requires G++ at least v7.3 to build.")
  endif()
## clang++
elseif (CMAKE_CXX_COMPILER_ID MATCHES "Clang")   
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "6.0")
    message(FATAL_ERROR "\nHeXinSta requires clang++ at least v6.0")
  endif() 
else()
  message(FATAL_ERROR "\n\
  HeXinSta currently supports the following compilers:\n\
- g++ v7.3 or above\n\
- clang++ v6.0 or above\n\
")
endif()
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

include_directories(${COMPILEABLE_PATH})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# Set library output path
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${HOME}/lib)
message(STATUS "CMAKE_ARCHIVE_OUTPUT_DIRECTORY: ${HOME}/lib")

# Find package
# Zlib
include(FindZLIB)

#set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads REQUIRED)

# Tclsh module
include(FindTclsh)
if (NOT TCL_TCLSH)
  message(FATAL_ERROR "\n\ Require tclsh to build. Visit: https://www.tcl.tk/software/tcltk/")
endif(NOT TCL_TCLSH)

# configure a header file to pass some of the CMake settins
configure_file(
  "${COMPILEABLE_PATH}/common/config.hpp.in"
  "${COMPILEABLE_PATH}/common/config.hpp"
)


################################################################
#
# Flex/bison scanner/parsers
#
################################################################
find_package(FLEX)
find_package(BISON)

FLEX_TARGET(SpefLex 
  ${COMPILEABLE_PATH}/parasitics/spef/SpefLex.ll 
  ${CMAKE_CURRENT_BINARY_DIR}/SpefLex.cc
  COMPILE_FLAGS --prefix=SpefLex_
  )
BISON_TARGET(SpefParser 
  ${COMPILEABLE_PATH}/parasitics/spef/SpefParse.yy 
  ${CMAKE_CURRENT_BINARY_DIR}/SpefParse.cc
  COMPILE_FLAGS --name-prefix=SpefParse_
  )

ADD_FLEX_BISON_DEPENDENCY(SpefLex SpefParser)

FLEX_TARGET(VerilogLexer 
  ${COMPILEABLE_PATH}/netlist/verilog/verilog_lexer.ll 
  ${CMAKE_CURRENT_BINARY_DIR}/verilog_lexer.yy.cc
  )

BISON_TARGET(VerilogParser 
  ${COMPILEABLE_PATH}/netlist/verilog/verilog_parser.yy 
  ${CMAKE_CURRENT_BINARY_DIR}/verilog_parser.tab.cc
  )

ADD_FLEX_BISON_DEPENDENCY(VerilogLexer VerilogParser)
################################################################

add_library(Verilog
  ${FLEX_VerilogLexer_OUTPUTS}
  ${BISON_VerilogParser_OUTPUTS}
)
list(APPEND VERILOG_LINK_FLAGS Verilog)

# Add NetList library
set(NETLIST_CPP
  ${COMPILEABLE_PATH}/netlist/netlist.cpp
  ${COMPILEABLE_PATH}/netlist/module_def.cpp
  ${COMPILEABLE_PATH}/netlist/module_imp.cpp
  ${COMPILEABLE_PATH}/netlist/module_ref.cpp
  ${COMPILEABLE_PATH}/netlist/net.cpp
  ${COMPILEABLE_PATH}/netlist/pin.cpp
  ${COMPILEABLE_PATH}/netlist/NlDesign.cpp
  #netlist/NlInfoDesign.cpp
  ${COMPILEABLE_PATH}/netlist/ModuleClone.cpp
  ${COMPILEABLE_PATH}/netlist/NlPort.cpp
  ${COMPILEABLE_PATH}/netlist/NlPin.cpp
  ${COMPILEABLE_PATH}/netlist/NlCell.cpp
  ${COMPILEABLE_PATH}/netlist/NlNet.cpp
  ${COMPILEABLE_PATH}/netlist/NlHier.cpp
  ${COMPILEABLE_PATH}/netlist/ModuleCloneManager.cpp
)
add_library(NetList ${NETLIST_CPP})
list(APPEND NETLIST_LINK_FLAGS NetList)



#Add Parasitics library
set(PARASITICS_CC
  ${COMPILEABLE_PATH}/parasitics/spef/SpefNamespace.cc
  ${COMPILEABLE_PATH}/parasitics/spef/SpefReader.cc
  ${COMPILEABLE_PATH}/parasitics/spef/SpefReaderPvt.hh
  ${COMPILEABLE_PATH}/parasitics/Parasitics.cc
  ${COMPILEABLE_PATH}/parasitics/ConcreteParasitics.cc
)
add_library(Spef
  ${FLEX_SpefLex_OUTPUTS}
  ${BISON_SpefParser_OUTPUTS})
list(APPEND SPEF_LINK_FLAGS Spef)
add_library(Parasitics ${PARASITICS_CC})
list(APPEND PARASITICS_LINK_FLAGS Parasitics)

# Add Liberty library
set(LIBERTY_CPP
  ${COMPILEABLE_PATH}/liberty/lib.cpp
  ${COMPILEABLE_PATH}/liberty/lib_cell.cpp
  ${COMPILEABLE_PATH}/liberty/lib_pin.cpp
  ${COMPILEABLE_PATH}/liberty/library.cpp
  ${COMPILEABLE_PATH}/liberty/lut.cpp
  ${COMPILEABLE_PATH}/liberty/power.cpp
  ${COMPILEABLE_PATH}/liberty/lib_timing_arc.cpp
)
add_library(Liberty ${LIBERTY_CPP})
list(APPEND LIBERTY_LINK_FLAGS Liberty)

# Add Linker library
set(LINKER_CPP
  ${COMPILEABLE_PATH}/linker/linker.cpp
)
add_library(Linker ${LINKER_CPP})
list(APPEND LINKER_LINK_FLAGS Linker)

# Add Utility library
set(UTILITY_CPP
  ${COMPILEABLE_PATH}/utility/tokenizer.cpp
  ${COMPILEABLE_PATH}/utility/os.cpp
  ${COMPILEABLE_PATH}/utility/string_utility.cpp
  ${COMPILEABLE_PATH}/utility/PortDirection.cc
  ${COMPILEABLE_PATH}/utility/StringUtil.cc
  ${COMPILEABLE_PATH}/utility/Transition.cc
  ${COMPILEABLE_PATH}/utility/MinMax.cc
  ${COMPILEABLE_PATH}/utility/logger.cpp
)
add_library(Utility ${UTILITY_CPP})
list(APPEND UTILITY_LINK_FLAGS Utility)

# Add Unit library
set(UNIT_CPP
  ${COMPILEABLE_PATH}/unit/unit.cpp
)
add_library(Unit ${UNIT_CPP})
list(APPEND UNIT_LINK_FLAGS Unit)

# Add Shell library
set(SHELL_CPP
  ${COMPILEABLE_PATH}/shell/shell.cpp
  ${COMPILEABLE_PATH}/shell/action.cpp
  ${COMPILEABLE_PATH}/shell/builder.cpp
  ${COMPILEABLE_PATH}/shell/misc.cpp
  ${COMPILEABLE_PATH}/shell/write.cpp
)
add_library(Shell ${SHELL_CPP})
list(APPEND SHELL_LINK_FLAGS Shell)

# Add SDC library
set(SDC_CPP
  ${COMPILEABLE_PATH}/sdc/sdc.cpp
  ${COMPILEABLE_PATH}/sdc/object.cpp
  ${COMPILEABLE_PATH}/sdc/sdc_design.cpp
)
add_library(Sdc ${SDC_CPP})
list(APPEND SDC_LINK_FLAGS Sdc)

# Add Timing Graph library
set(TG_CPP
  ${COMPILEABLE_PATH}/timer/graph/tg_pin.cpp
  ${COMPILEABLE_PATH}/timer/graph/tg_arc.cpp
  ${COMPILEABLE_PATH}/timer/graph/tg_stage.cpp
  ${COMPILEABLE_PATH}/timer/graph/timing_graph.cpp
  ${COMPILEABLE_PATH}/timer/graph/nl_tg_map.cpp
  ${COMPILEABLE_PATH}/timer/graph/lib_tg_map.cpp
  ${COMPILEABLE_PATH}/timer/graph/dc_algorithm.cpp
  ${COMPILEABLE_PATH}/timer/data/timing_data.cpp
  ${COMPILEABLE_PATH}/timer/data/timing_tag.cpp
  ${COMPILEABLE_PATH}/timer/data/timing_annotation.cpp
  ${COMPILEABLE_PATH}/timer/data/timing_derate.cpp
  #timer/graph/test.cpp
)
add_library(TimingGraph ${TG_CPP})
list(APPEND TIMINGGRAPH_LINK_FLAGS TimingGraph)

# Add Timer library
set(TIMER_CPP
  ${COMPILEABLE_PATH}/timer/timer.cpp
  ${COMPILEABLE_PATH}/timer/clock/clock.cpp
)
add_library(Timer ${TIMER_CPP})
list(APPEND TIMER_LINK_FLAGS Timer)

# Add Report
set(REPORT_CPP
  ${COMPILEABLE_PATH}/report/report.cpp
  ${COMPILEABLE_PATH}/report/endpoint.cpp
)
add_library(Report ${REPORT_CPP})
list(APPEND REPORT_LINK_FLAGS Report)

find_package(TCL)
find_package(SWIG 3.0 REQUIRED)
include(UseSWIG)

set(STA_SWIG_FILE ${COMPILEABLE_PATH}/main/sta.i)

set_property(SOURCE ${STA_SWIG_FILE}
  PROPERTY CPLUSPLUS ON
)

set_property(SOURCE ${STA_SWIG_FILE}
  PROPERTY SWIG_FLAGS
  -module sta
  -Isdc
)

swig_add_library(sta_swig
  LANGUAGE tcl
  TYPE STATIC
  SOURCES ${STA_SWIG_FILE}
)

target_include_directories(sta_swig
  PRIVATE
  sdc
  ${TCL_INCLUDE_PATH}
)



#mode of use ? cmake .. -DTARGET_CPU:STRING=nl_test  or  cmake ..
if(TARGET_CPU STREQUAL "nl_test")
  # For internal test 
  set(MAIN nl_test)
  add_executable(${MAIN} test/netlist_test.cpp)
  
elseif(TARGET_CPU STREQUAL "tg_test")
  # For internal tg_test 
  set(MAIN tg_test)
  add_executable(${MAIN} ${COMPILEABLE_PATH}/timer/graph/test.cpp)
  
else()
  set(MAIN hx_timer)
  add_executable(${MAIN} ${COMPILEABLE_PATH}/main/hx_timer.cpp)
  
endif()

target_link_libraries(${MAIN} ${SHELL_LINK_FLAGS})
target_link_libraries(${MAIN} ${REPORT_LINK_FLAGS})
target_link_libraries(${MAIN} ${SDC_LINK_FLAGS})
target_link_libraries(${MAIN} ${TIMER_LINK_FLAGS})
target_link_libraries(${MAIN} ${TIMINGGRAPH_LINK_FLAGS})
target_link_libraries(${MAIN} ${LINKER_LINK_FLAGS})
target_link_libraries(${MAIN} ${NETLIST_LINK_FLAGS})
target_link_libraries(${MAIN} ${PARASITICS_LINK_FLAGS})
target_link_libraries(${MAIN} ${VERILOG_LINK_FLAGS})
target_link_libraries(${MAIN} ${LIBERTY_LINK_FLAGS})
target_link_libraries(${MAIN} ${UNIT_LINK_FLAGS})
target_link_libraries(${MAIN} ${UTILITY_LINK_FLAGS})
target_link_libraries(${MAIN} ${TCL_LIBRARY})
target_link_libraries(${MAIN} ${TK_LIBRARY})
target_link_libraries(${MAIN}
    sta_swig
    ${CMAKE_THREAD_LIBS_INIT}
    )
  if (ZLIB_LIBRARIES)
    target_link_libraries(${MAIN} ${ZLIB_LIBRARIES})
  endif()
  
  message(STATUS "hx_timer executable: bin/${MAIN}")
