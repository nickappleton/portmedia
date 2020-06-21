# portmidi
# Roger B. Dannenberg
# 20 Sep 2009

cmake_minimum_required(VERSION 3.0)

project(portmidi)

if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_LIST_DIR})
  if (CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING  "Supported configuration types")
  elseif (NOT CMAKE_BUILD_TYPE)
    message(STATUS "Setting CMAKE_BUILD_TYPE type to 'Debug' as none was specified.")
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
  endif()

  if(MSVC AND WIN32)
    foreach(flag_var
      CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
      CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO
      CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
      CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif(${flag_var} MATCHES "/MD")
    endforeach(flag_var)
    message(STATUS "Note: overriding CMAKE_*_FLAGS_* to use Visual C static multithread library")
  endif(MSVC AND WIN32)

  # Keeping this here for future reference. Hopefully it can be deleted at
  # some point. It looks suspicious.
  if(FALSE)
    if(UNIX)
      # allow user to set Release or Debug
      set(CMAKE_BUILD_TYPE Release CACHE STRING 
          "Semicolon-separate list of supported configuration types")
      # set default directories but don't override cached values...
      set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CACHEFILE_DIR}/${CMAKE_BUILD_TYPE}
          CACHE STRING "libraries go here")
      set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CACHEFILE_DIR}/${CMAKE_BUILD_TYPE}
          CACHE STRING "libraries go here")
      set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CACHEFILE_DIR}/${CMAKE_BUILD_TYPE}
          CACHE STRING "executables go here")
    else(UNIX)
      # this does not seem to work for xcode:
      set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING 
          "Semicolon-separate list of supported configuration types")
    endif(UNIX)

    # Slightly concerning
    if(APPLE)
      set(CMAKE_OSX_SYSROOT /Developer/SDKs/MacOSX10.6.sdk CACHE 
          PATH "-isysroot parameter for compiler")
      set(CMAKE_C_FLAGS "-mmacosx-version-min=10.6" CACHE 
          STRING "needed in conjunction with CMAKE_OSX_SYSROOT" FORCE)
    endif(APPLE)

    # This is removed because it should be specified by the user when they
    # create project files. It breaks builds on modern platforms.
    set(CMAKE_OSX_ARCHITECTURES i386 ppc x86_64 CACHE STRING "change to needed architecture for a smaller library" FORCE)
  endif()
else()
  # checks should go here for compatibility with add_subdirectory()
endif()

# this completes the list of library sources by adding shared code
set(LIBSRC
  pm_common/pminternal.h
  pm_common/pmutil.c
  pm_common/pmutil.h
  pm_common/portmidi.c
  pm_common/portmidi.h)

# first include the appropriate system-dependent file:
if(APPLE)
  find_library(COREMIDI_LIB CoreMIDI)
  find_library(COREAUDIO_LIB CoreAudio)
  find_library(COREFOUNDATION_LIB CoreFoundation)
  find_library(CORESERVICES_LIB CoreServices)
  set(PM_NEEDED_LIBS ${COREAUDIO_LIB} ${COREFOUNDATION_LIB} ${COREMIDI_LIB} ${CORESERVICES_LIB})
  list(APPEND LIBSRC ./pm_mac/finddefault.c ./pm_mac/pmmac.c ./pm_mac/pmmac.h ./pm_mac/pmmacosxcm.c ./pm_mac/pmmacosxcm.h ./pm_mac/readbinaryplist.c ./pm_mac/readbinaryplist.h)
  list(APPEND LIBSRC ./porttime/ptmacosx_mach.c)
elseif(UNIX)
  set(PM_NEEDED_LIBS pthread asound)
  list(APPEND LIBSRC ./pm_linux/finddefault.c ./pm_linux/pmlinux.c ./pm_linux/pmlinux.h ./pm_linux/pmlinuxalsa.c ./pm_linux/pmlinuxalsa.h)
  list(APPEND LIBSRC ./porttime/ptlinux.c)
elseif(WIN32)
  set(PM_NEEDED_LIBS winmm.lib)
  list(APPEND LIBSRC ./pm_win/pmwin.c ./pm_win/pmwinmm.c ./pm_win/pmwinmm.h)
  list(APPEND LIBSRC ./porttime/ptwinmm.c)
endif()

# now add the shared files to make the complete list of library sources
add_library(portmidi-static ${LIBSRC})
set_target_properties(portmidi-static PROPERTIES OUTPUT_NAME "portmidi_s")
target_link_libraries(portmidi-static ${PM_NEEDED_LIBS})
target_include_directories(portmidi-static PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/pm_common>")
target_include_directories(portmidi-static PRIVATE ./porttime)
set_property(TARGET portmidi-static APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPM_CHECK_ERRORS=1 -DDEBUG")
if(UNIX AND NOT APPLE)
  set_property(TARGET portmidi-static APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPMALSA")
  set_property(TARGET portmidi-static APPEND_STRING PROPERTY COMPILE_FLAGS_RELEASE " -DPMALSA")
endif()

# add dynamic library
add_library(portmidi-dynamic SHARED ${LIBSRC})
set_target_properties(portmidi-dynamic PROPERTIES OUTPUT_NAME "portmidi")
target_link_libraries(portmidi-dynamic ${PM_NEEDED_LIBS})
target_include_directories(portmidi-dynamic PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/pm_common>")
target_include_directories(portmidi-dynamic PRIVATE ./porttime)
set_property(TARGET portmidi-dynamic APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPM_CHECK_ERRORS=1 -DDEBUG")
if(UNIX AND NOT APPLE)
  set_property(TARGET portmidi-dynamic APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPMALSA")
  set_property(TARGET portmidi-dynamic APPEND_STRING PROPERTY COMPILE_FLAGS_RELEASE " -DPMALSA")
endif()

set(PORTMIDI_TARGETS portmidi-static portmidi-dynamic)

find_package(JNI)
if(JNI_FOUND)
  list(APPEND PORTMIDI_TARGETS pmjni)
  add_library(pmjni SHARED ${LIBSRC} ./pm_java/pmjni/pmjni.c)
  target_link_libraries(pmjni "${PM_NEEDED_LIBS}" "${JNI_LIBRARIES}")
  set_target_properties(pmjni PROPERTIES EXECUTABLE_EXTENSION "jnilib")
  target_include_directories(pmjni PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/pm_common>")
  target_include_directories(pmjni PRIVATE ./porttime ${JNI_INCLUDE_DIRS})
  set_property(TARGET pmjni APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPM_CHECK_ERRORS=1 -DDEBUG")
  if(UNIX AND NOT APPLE)
    set_property(TARGET pmjni APPEND_STRING PROPERTY COMPILE_FLAGS_DEBUG " -DPMALSA")
    set_property(TARGET pmjni APPEND_STRING PROPERTY COMPILE_FLAGS_RELEASE " -DPMALSA")
  endif()
else()
  message(STATUS "JNI not found - not building JNI library")
endif()

# install the libraries (Linux and Mac OS X command line)
include(CMakePackageConfigHelpers)
configure_package_config_file(cmake_support/portmidiConfig.cmake.in ${CMAKE_BINARY_DIR}/cmake/portmidi/portmidiConfig.cmake
  INSTALL_DESTINATION "lib/cmake/portmidi"
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)
write_basic_package_version_file(${CMAKE_BINARY_DIR}/cmake/portmidi/portmidiConfigVersion.cmake
  VERSION 1
  COMPATIBILITY SameMajorVersion)
export(TARGETS ${PORTMIDI_TARGETS} FILE "${PROJECT_BINARY_DIR}/cmake/portmidi/portmidiTargets.cmake")
install(TARGETS ${PORTMIDI_TARGETS}
  EXPORT portmidi-targets
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)
install(EXPORT portmidi-targets DESTINATION lib/cmake/portmidi FILE "portmidiTargets.cmake")
install(FILES ./pm_common/portmidi.h ./porttime/porttime.h DESTINATION include)
install(FILES README.txt DESTINATION share/doc/portmidi)
install(FILES license.txt DESTINATION share/doc/portmidi)
install(FILES "${CMAKE_BINARY_DIR}/cmake/portmidi/portmidiConfig.cmake"
              "${CMAKE_BINARY_DIR}/cmake/portmidi/portmidiConfigVersion.cmake"
  DESTINATION "lib/cmake/portmidi")

add_subdirectory(pm_test)

# Cannot figure out how to make an xcode Java application with CMake
add_subdirectory(pm_java)

