# CMake file to build tests in this directory: pm_test

# set the build directory to be in portmidi, not in portmidi/pm_test
# this is required for Xcode:
if(APPLE)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})
endif(APPLE)

macro(make_a_test name)
  add_executable(${name} ${name}.c)
  target_link_libraries(${name} portmidi-static ${PM_NEEDED_LIBS})
  add_dependencies(${name} portmidi-static)
  target_include_directories(${name} PRIVATE ../porttime)
endmacro(make_a_test)

add_executable(test1 test.c)
target_link_libraries(test1 portmidi-static ${PM_NEEDED_LIBS})
add_dependencies(test1 portmidi-static)
target_include_directories(test1 PRIVATE ../porttime)

make_a_test(midithread)
make_a_test(midithru)
make_a_test(sysex)
make_a_test(latency)
make_a_test(mm)
make_a_test(midiclock)
make_a_test(qtest)
make_a_test(fast)
