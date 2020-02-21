
# Get version from core.h
file(READ ${CMAKE_CURRENT_LIST_DIR}/include/fmt/core.h core_h)
if (NOT core_h MATCHES "FMT_VERSION ([0-9]+)([0-9][0-9])([0-9][0-9])")
  message(FATAL_ERROR "Cannot get FMT_VERSION from core.h.")
endif ()

# Joins arguments and places the results in ${result_var}.
function(join result_var)
  set(result )
  foreach (arg ${ARGN})
    set(result "${result}${arg}")
  endforeach ()
  set(${result_var} "${result}" PARENT_SCOPE)
endfunction()

# Use math to skip leading zeros if any.
math(EXPR CPACK_PACKAGE_VERSION_MAJOR ${CMAKE_MATCH_1})
math(EXPR CPACK_PACKAGE_VERSION_MINOR ${CMAKE_MATCH_2})
math(EXPR CPACK_PACKAGE_VERSION_PATCH ${CMAKE_MATCH_3})
join(FMT_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.
                 ${CPACK_PACKAGE_VERSION_PATCH})
message(STATUS "Version: ${FMT_VERSION}")

message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

if (NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
endif ()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}
  "${CMAKE_CURRENT_LIST_DIR}/support/cmake")

include(cxx14)
include(CheckCXXCompilerFlag)

list(FIND CMAKE_CXX_COMPILE_FEATURES "cxx_variadic_templates" index)
if (${index} GREATER -1)
  # Use cxx_variadic_templates instead of more appropriate cxx_std_11 for
  # compatibility with older CMake versions.
  set(FMT_REQUIRED_FEATURES cxx_variadic_templates)
endif ()
message(STATUS "Required features: ${FMT_REQUIRED_FEATURES}")

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  set(PEDANTIC_COMPILE_FLAGS -pedantic-errors -Wall -Wextra -pedantic
      -Wold-style-cast -Wundef
      -Wredundant-decls -Wwrite-strings -Wpointer-arith
      -Wcast-qual -Wformat=2 -Wmissing-include-dirs
      -Wcast-align -Wnon-virtual-dtor
      -Wctor-dtor-privacy -Wdisabled-optimization
      -Winvalid-pch -Woverloaded-virtual
      -Wconversion -Wswitch-enum
      -Wno-ctor-dtor-privacy -Wno-format-nonliteral -Wno-shadow)
  if (NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.6)
      set(PEDANTIC_COMPILE_FLAGS ${PEDANTIC_COMPILE_FLAGS} -Wnoexcept
         -Wno-dangling-else -Wno-unused-local-typedefs)
  endif ()
  if (NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 5.0)
      set(PEDANTIC_COMPILE_FLAGS ${PEDANTIC_COMPILE_FLAGS} -Wdouble-promotion
          -Wtrampolines -Wzero-as-null-pointer-constant -Wuseless-cast
          -Wvector-operation-performance -Wsized-deallocation)
  endif ()
  if (NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS 6.0)
      set(PEDANTIC_COMPILE_FLAGS ${PEDANTIC_COMPILE_FLAGS} -Wshift-overflow=2
          -Wnull-dereference -Wduplicated-cond)
  endif ()
  set(WERROR_FLAG -Werror)
endif ()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  set(PEDANTIC_COMPILE_FLAGS -Wall -Wextra -pedantic -Wconversion
      -Wno-sign-conversion -Wdeprecated -Wweak-vtables)
  check_cxx_compiler_flag(-Wzero-as-null-pointer-constant HAS_NULLPTR_WARNING)
  if (HAS_NULLPTR_WARNING)
    set(PEDANTIC_COMPILE_FLAGS ${PEDANTIC_COMPILE_FLAGS}
        -Wzero-as-null-pointer-constant)
  endif ()
  set(WERROR_FLAG -Werror)
endif ()

if (MSVC)
  set(PEDANTIC_COMPILE_FLAGS /W3)
  set(WERROR_FLAG /WX)
endif ()


set(strtod_l_headers stdlib.h)

if (APPLE)
  set(strtod_l_headers ${strtod_l_headers} xlocale.h)
endif ()

include(CheckSymbolExists)
if (WIN32)
  check_symbol_exists(_strtod_l "${strtod_l_headers}" HAVE_STRTOD_L)
else ()
  check_symbol_exists(strtod_l "${strtod_l_headers}" HAVE_STRTOD_L)
endif ()

function(add_headers VAR)
  set(headers ${${VAR}})
  foreach (header ${ARGN})
    set(headers ${headers} ${CMAKE_CURRENT_LIST_DIR}/include/fmt/${header})
  endforeach()
  set(${VAR} ${headers} PARENT_SCOPE)
endfunction()

# Define the fmt library, its includes and the needed defines.
add_headers(FMT_HEADERS chrono.h color.h compile.h core.h format.h format-inl.h
                        locale.h os.h ostream.h posix.h printf.h ranges.h)
set(FMT_SOURCES ${CMAKE_CURRENT_LIST_DIR}/src/format.cc ${CMAKE_CURRENT_LIST_DIR}/src/os.cc)

add_library(fmt ${FMT_SOURCES} ${FMT_HEADERS} ${CMAKE_CURRENT_LIST_DIR}/README.rst ${CMAKE_CURRENT_LIST_DIR}/ChangeLog.rst)
add_library(fmt::fmt ALIAS fmt)

if (iOS)
  ios_add_universal_build(fmt)
endif()

if (HAVE_STRTOD_L)
  target_compile_definitions(fmt PUBLIC FMT_LOCALE)
endif ()

if (FMT_WERROR)
  target_compile_options(fmt PRIVATE ${WERROR_FLAG})
endif ()
if (FMT_PEDANTIC)
  target_compile_options(fmt PRIVATE ${PEDANTIC_COMPILE_FLAGS})
endif ()

target_compile_features(fmt INTERFACE ${FMT_REQUIRED_FEATURES})

target_include_directories(fmt PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
  $<INSTALL_INTERFACE:include>)

set(FMT_DEBUG_POSTFIX d)

set_target_properties(fmt PROPERTIES
  VERSION ${FMT_VERSION} SOVERSION ${CPACK_PACKAGE_VERSION_MAJOR}
  DEBUG_POSTFIX ${FMT_DEBUG_POSTFIX})

# Set FMT_LIB_NAME for pkg-config fmt.pc.
get_target_property(FMT_LIB_NAME fmt OUTPUT_NAME)
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(FMT_LIB_NAME ${FMT_LIB_NAME}${FMT_DEBUG_POSTFIX})
endif ()

if (BUILD_SHARED_LIBS)
  if (UNIX AND NOT APPLE AND NOT ${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
    # Fix rpmlint warning:
    # unused-direct-shlib-dependency /usr/lib/libformat.so.1.1.0 /lib/libm.so.6.
    target_link_libraries(fmt -Wl,--as-needed)
  endif ()
  target_compile_definitions(fmt PRIVATE FMT_EXPORT INTERFACE FMT_SHARED)
endif ()
if (FMT_SAFE_DURATION_CAST)
  target_compile_definitions(fmt PUBLIC FMT_SAFE_DURATION_CAST)
endif()

add_library(fmt-header-only INTERFACE)
add_library(fmt::fmt-header-only ALIAS fmt-header-only)

target_compile_definitions(fmt-header-only INTERFACE FMT_HEADER_ONLY=1)

target_compile_features(fmt-header-only INTERFACE ${FMT_REQUIRED_FEATURES})

target_include_directories(fmt-header-only INTERFACE
  $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)

