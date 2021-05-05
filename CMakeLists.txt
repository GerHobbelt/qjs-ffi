cmake_minimum_required(VERSION 3.0)
project(qjs-ffi)

set(DISABLE_WERROR ON)

if(NOT quickjs_libdir)
  execute_process(
    COMMAND cc -dumpmachine
    OUTPUT_VARIABLE HOST_SYSTEM_NAME
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  execute_process(
    COMMAND ${CMAKE_C_COMPILER} -dumpmachine
    OUTPUT_VARIABLE SYSTEM_NAME
    OUTPUT_STRIP_TRAILING_WHITESPACE)

  if(NOT "${HOST_SYSTEM_NAME}" STREQUAL "${SYSTEM_NAME}")
    string(REGEX REPLACE i686 i386 quickjs_cross_arch "${SYSTEM_NAME}")
    message("quickjs_cross_arch = ${quickjs_cross_arch}")
  endif()

  if(quickjs_cross_arch)
    set(quickjs_libdir lib/${quickjs_cross_arch})
  else(quickjs_cross_arch)
    set(quickjs_libdir lib)
  endif(quickjs_cross_arch)

endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/..
                    ${CMAKE_CURRENT_SOURCE_DIR}/../..)
if(CMAKE_SYSTEM_INCLUDE_PATH)
  include_directories(${CMAKE_SYSTEM_INCLUDE_PATH})
endif(CMAKE_SYSTEM_INCLUDE_PATH)

set(DISABLE_WERROR TRUE)

function(make_shared_module FNAME)
  string(REGEX REPLACE "_" "-" NAME "${FNAME}")
  string(TOUPPER "${FNAME}" UNAME)

  message("Module: ${NAME}")
  set(TARGET_NAME quickjs-${NAME})

  add_library(${TARGET_NAME} SHARED ${FNAME}.c)

  target_link_libraries(${TARGET_NAME} ffi dl)
  set_target_properties(
    ${TARGET_NAME}
    PROPERTIES
      PREFIX ""
      BUILD_RPATH "${CMAKE_CURRENT_BINARY_DIR}"
      INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib"
      OUTPUT_NAME "${NAME}" # COMPILE_FLAGS "-fvisibility=hidden"
      BUILD_RPATH
      "${CMAKE_BINARY_DIR};${CMAKE_CURRENT_BINARY_DIR};${CMAKE_BINARY_DIR}/quickjs;${CMAKE_CURRENT_BINARY_DIR}/quickjs"
  )
  target_compile_definitions(${TARGET_NAME} PRIVATE -DJS_${UNAME}_MODULE=1)
  install(
    TARGETS ${TARGET_NAME}
    DESTINATION ${quickjs_libdir}/quickjs
    PERMISSIONS
      OWNER_READ
      OWNER_WRITE
      OWNER_EXECUTE
      GROUP_READ
      GROUP_EXECUTE
      WORLD_READ
      WORLD_EXECUTE)

endfunction()

file(GLOB JS_BINDINGS_SOURCES ffi.c)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(CheckLibraryExists)

include(FindPkgConfig)
if(NOT FFI_FOUND)
  pkg_search_module(FFI libffi)

  #[[if(NOT FFI_FOUND)
    find_library(FFI NAMES libffi ffi PATHS "${CMAKE_INSTALL_PREFIX}/lib")
  endif(NOT FFI_FOUND)]]
endif(NOT FFI_FOUND)

if(pkgcfg_lib_FFI_ffi)
  set(FFI_LIBRARIES "${pkgcfg_lib_FFI_ffi}")
  set(FFI_LINK_LIBRARIES "${pkgcfg_lib_FFI_ffi}")
endif(pkgcfg_lib_FFI_ffi)

if(FFI_FOUND)
  set(HAVE_LIBFFI TRUE)
endif(FFI_FOUND)
# check_library_exists(ffi ffi_call "" HAVE_LIBFFI)

if(HAVE_LIBFFI)
  link_libraries(${FFI_LINK_LIBRARIES})
endif(HAVE_LIBFFI)

if(BUILD_SHARED_LIBS)
  add_definitions(-DJS_SHARED_LIBRARY)
endif(BUILD_SHARED_LIBS)

# Main
add_library(quickjs-ffi MODULE ${JS_BINDINGS_SOURCES})
target_include_directories(quickjs-ffi PRIVATE)
set_target_properties(quickjs-ffi PROPERTIES PREFIX "" OUTPUT_NAME "ffi")
target_compile_definitions(quickjs-ffi PRIVATE -DJS_SHARED_LIBRARY=1 -DJS_BINDINGS_INIT_MODULE=1)
target_link_libraries(quickjs-ffi PUBLIC ${FFI_LIBRARIES} dl)

install(
  TARGETS quickjs-ffi
  DESTINATION ${quickjs_libdir}/quickjs
  PERMISSIONS
    OWNER_READ
    OWNER_WRITE
    OWNER_EXECUTE
    GROUP_READ
    GROUP_EXECUTE
    WORLD_READ
    WORLD_EXECUTE)

add_library(test MODULE test.c)
set_target_properties(test PROPERTIES PREFIX "" OUTPUT_NAME "test")
target_link_libraries(test)
