project(libplist)

cmake_minimum_required(VERSION 3.0)

option(BUILD_TOOLS "Build tools." ON)

include(CheckCCompilerFlag)
include(CheckCSourceCompiles)
include(CheckSymbolExists)
include(CheckLibraryExists)

cmake_policy(SET CMP0068 NEW)

######## get git version ##################################################
execute_process(COMMAND ./git-version-gen ${RELEASE_VERSION}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE git_version
)
if("${git_version}" STREQUAL "")
  if(NOT "${RELEASE_VERSION}" STREQUAL "")
    set(git_version ${RELEASE_VERSION})
  else()
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/.git)
      find_package(Git)
      execute_process(COMMAND ${GIT_EXECUTABLE} update-index -q --refresh
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
      )
      execute_process(COMMAND ${GIT_EXECUTABLE} describe --tags --dirty
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE VER
      )
    elseif(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/.tarball-version)
      file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/.tarball-version VER LIMIT_COUNT 1)
    endif()
    if(NOT "${VER}" STREQUAL "")
      string(REGEX MATCH "([^\n\r]+)" _ ${VER})
      set(git_version ${CMAKE_MATCH_1})
    endif()
  endif()
endif()
if("${git_version}" STREQUAL "")
  message(FATAL_ERROR "PACKAGE_VERSION is not defined. Make sure to configure a source tree checked out from git or that .tarball-version is present.")
endif()
set(PACKAGE_VERSION \"${git_version}\")
set(VERSION ${git_version})

if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/configure.ac)
  message(FATAL_ERROR "The file configure.ac needs to be present in the source tree.")
endif()

file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/config.h)
file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/config.h.in)

######## helper functions #################################################
function(add_to_config varname)
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/config.h.in
    "#cmakedefine ${varname} @${varname}@
"
  )
endfunction()

function(check_symbol_exists_config symbol)
  string(TOUPPER ${symbol} symbol_upper)
  set(varname "HAVE_${symbol_upper}")
  add_to_config(${varname})
  check_symbol_exists(${symbol} "stdio.h;strings.h;string.h;time.h" ${varname})
endfunction()

######## read configure.ac to parse some data from it #####################
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/configure.ac config)
string(REGEX MATCH "AC_INIT\\(\\[([^]]*)\\][ ]*,[ ]*\\[([^]]*)\\][ ]*,[ ]*\\[([^]]*)\\][ ]*,[ ]*\\[([^]]*)\\][ ]*,[ ]*\\[([^]]*)\\]\\)\n" _ ${config})
set(PACKAGE ${CMAKE_PROJECT_NAME})
set(PACKAGE_NAME ${PACKAGE})
set(PACKAGE_BUGREPORT "\"${CMAKE_MATCH_3}\"")
set(PACKAGE_URL "\"${CMAKE_MATCH_5}\"")
add_to_config(PACKAGE)
add_to_config(PACKAGE_NAME)
add_to_config(PACKAGE_URL)
add_to_config(PACKAGE_BUGREPORT)
add_to_config(PACKAGE_VERSION)
add_to_config(VERSION)

## get SO_VERSION
string(REGEX MATCH ".*_SO_VERSION=([0-9]*):([0-9]*):([0-9]*)" _ ${config})
if(NOT "${CMAKE_MATCH_2}" STREQUAL "")
  math(EXPR so_version "${CMAKE_MATCH_1}-${CMAKE_MATCH_3}")
  math(EXPR current_version "${CMAKE_MATCH_1}+1")
  math(EXPR compat_version "${CMAKE_MATCH_1}+1")
endif()

## check compiler flags
string(REGEX MATCH "AS_COMPILER_FLAGS\\(([A-Za-z_]*), \"([^\"]*)" _ ${config})
set(compiler_flags_var ${CMAKE_MATCH_1})
string(REGEX REPLACE "[ ]+" ";" flag_list ${CMAKE_MATCH_2})
set(_QUIET ${CMAKE_REQUIRED_QUIET})
message(STATUS "Checking compiler flags")
set(CMAKE_REQUIRED_QUIET 1)
set(compflags "")
foreach(flag IN LISTS flag_list)
  check_c_compiler_flag(${flag} FLAG_${flag})
  if(FLAG_${flag} EQUAL 1)
    string(APPEND compflags "${flag} ")
  endif()
endforeach()
set(CMAKE_REQUIRED_QUIET ${_QUIET})
set(${compiler_flags_var} ${compflags})

## check AC_CHECK_FUNCS
string(REGEX MATCH "AC_CHECK_FUNCS\\(\\[([^]]*)" _ ${config})
string(REGEX REPLACE "[ ]+" ";" func_list ${CMAKE_MATCH_1})
foreach(func IN LISTS func_list)
  check_symbol_exists_config(${func})
endforeach()

include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
  set(__BIG_ENDIAN__ 1)
  add_to_config("__BIG_ENDIAN__")
else()
  set(__LITTLE_ENDIAN__ 1)
  add_to_config("__LITTLE_ENDIAN__")
endif()

if(NOT WIN32)
  set(THREADS_PREFER_PTHREAD_FLAG ON)
  find_package(Threads REQUIRED)
endif()

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
set(libdir "\${exec_prefix}/lib")
set(includedir "\${prefix}/include")
set(PACKAGE_NAME ${PROJECT_NAME})

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_SOURCE_DIR}/cmake/modules)

if(NOT DEFINED CMAKE_INSTALL_LIBDIR)
   set(CMAKE_INSTALL_LIBDIR lib CACHE PATH "Output directory for libraries")
endif()

####### check if struct tm has a tm_gmtoff member
check_c_source_compiles("
#include <time.h>
int main() {
      struct tm tm;
      tm.tm_gmtoff = 1;
}
"
HAVE_TM_TM_GMTOFF
)
add_to_config(HAVE_TM_TM_GMTOFF)

check_c_source_compiles("
#include <time.h>
int main() {
      struct tm tm;
      tm.tm_zone = 1;
}
"
HAVE_TM_TM_ZONE
)
add_to_config(HAVE_TM_TM_ZONE)

find_library(_LIB_PATH "m")
check_library_exists("m" "fmin" ${_LIB_PATH} fmin_FOUND)
if(${fmin_FOUND})
  set(LIBM_PATH ${_LIB_PATH})
endif()

option(ENABLE_DEBUG "build debug message output code (default is no)" OFF)
if(ENABLE_DEBUG)
  add_to_config(DEBUG)
  set(GLOBAL_CFLAGS "${GLOBAL_CFLAGS} -g")
  set(DEBUG 1)
endif()

option(ENABLE_CYTHON "Enable Cython Python bindings (needs Cython)" ON)
if(ENABLE_CYTHON)
  find_package(PythonInterp)
  if(PYTHONINTERP_FOUND)
    get_filename_component(_python_path ${PYTHON_EXECUTABLE} PATH)
    find_program(CYTHON_EXECUTABLE
      NAMES cython cython.bat cython3
      HINTS ${_python_path}
    )
  else()
    find_program(CYTHON_EXECUTABLE
      NAMES cython cython.bat cython3
    )
  endif()
  if(CYTHON_EXECUTABLE)
    message(STATUS "Found Cython: ${CYTHON_EXECUTABLE}")
  else()
    message(FATAL_ERROR "ENABLE_CYTHON is set but Cython executable could not be found")
  endif()
endif()

include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/include
                     ${CMAKE_CURRENT_SOURCE_DIR}/libcnary/include
                     ${CMAKE_CURRENT_SOURCE_DIR}/src
                     ${CMAKE_CURRENT_BINARY_DIR}
                     ${CMAKE_INCLUDE_PATH})

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${GLOBAL_CFLAGS} -DHAVE_CONFIG_H")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHAVE_CONFIG_H")

if(WIN32 AND (NOT CMAKE_CXX_COMPILER_ID STREQUAL "MSVC"))
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -avoid-version -static-libgcc")
endif()

#ENABLE_TESTING()

set(libcnary_SRC
  libcnary/node.c
  libcnary/node_list.c
  libcnary/include/node.h
  libcnary/include/node_list.h
  libcnary/include/object.h
)
add_library(cnary STATIC ${libcnary_SRC})
set_target_properties(cnary PROPERTIES
  ARCHIVE_OUTPUT_DIRECTORY internal
)

set(libplist_HDR
  include/plist/plist.h
)
set(libplist_SRC
  src/base64.c src/base64.h
  src/bytearray.c src/bytearray.h
  src/strbuf.h
  src/hashtable.c src/hashtable.h
  src/ptrarray.c src/ptrarray.h
  src/time64.c src/time64.h
  src/time64_limits.h
  src/xplist.c
  src/bplist.c
  src/jsmn.c src/jsmn.h
  src/jplist.c
  src/oplist.c
  src/out-default.c
  src/out-plutil.c
  src/out-limd.c
  src/plist.c src/plist.h
  ${libplist_HDR}
)
set(libplist++_HDR
  include/plist/plist++.h
  include/plist/Array.h
  include/plist/Boolean.h
  include/plist/Data.h
  include/plist/Date.h
  include/plist/Dictionary.h
  include/plist/Integer.h
  include/plist/Node.h
  include/plist/Real.h
  include/plist/String.h
  include/plist/Key.h
  include/plist/Uid.h
  include/plist/Structure.h
)
set(libplist++_SRC
  src/Node.cpp
  src/Structure.cpp
  src/Array.cpp
  src/Boolean.cpp
  src/Data.cpp
  src/Date.cpp
  src/Dictionary.cpp
  src/Integer.cpp
  src/Key.cpp
  src/Real.cpp
  src/String.cpp
  src/Uid.cpp
  ${libplist++_HDR}
)

add_library(plist SHARED ${libplist_SRC})
add_library(plist_a STATIC ${libplist_SRC})
add_library(plist++ SHARED ${libplist++_SRC})
add_library(plist++_a STATIC ${libplist_SRC})
add_executable(plistutil tools/plistutil.c)

set_target_properties(plist PROPERTIES
  OUTPUT_NAME plist-2.0
  LIBRARY_OUTPUT_DIRECTORY lib
  SOVERSION ${so_version}
)
set_target_properties(plist++ PROPERTIES
  OUTPUT_NAME plist++-2.0
  LIBRARY_OUTPUT_DIRECTORY lib
  SOVERSION ${so_version}
)
set_target_properties(plist_a PROPERTIES
  OUTPUT_NAME plist-2.0
  ARCHIVE_OUTPUT_DIRECTORY lib
)
set_target_properties(plist++_a PROPERTIES
  OUTPUT_NAME plist++-2.0
  ARCHIVE_OUTPUT_DIRECTORY lib
)
set_target_properties(plistutil PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY tools
)

target_link_directories(plist++ PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_directories(plistutil PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)

target_link_libraries(plist ${CMAKE_CURRENT_BINARY_DIR}/internal/libcnary.a ${CMAKE_THREAD_LIBS_INIT} ${LIBM_PATH})
target_link_libraries(plist_a ${CMAKE_CURRENT_BINARY_DIR}/internal/libcnary.a ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(plist++ plist-2.0)
target_link_libraries(plistutil plist-2.0)
if(APPLE)
  set_target_properties(plist PROPERTIES
    MACHO_COMPATIBILITY_VERSION ${compat_version}
    MACHO_CURRENT_VERSION ${current_version}
    INSTALL_NAME_DIR ${CMAKE_INSTALL_PREFIX}/lib
  )
  set_target_properties(plist++ PROPERTIES
    MACHO_COMPATIBILITY_VERSION ${compat_version}
    MACHO_CURRENT_VERSION ${current_version}
    INSTALL_NAME_DIR ${CMAKE_INSTALL_PREFIX}/lib
  )
endif()

#IF ( CYTHON_FOUND AND PYTHONLIBS_FOUND AND PYTHONINTERP_FOUND )
#       ADD_SUBDIRECTORY( cython )
#ENDIF ( CYTHON_FOUND AND PYTHONLIBS_FOUND AND PYTHONINTERP_FOUND )

######## build config.h ###################################################
configure_file(
  ${CMAKE_CURRENT_BINARY_DIR}/config.h.in
  ${CMAKE_CURRENT_BINARY_DIR}/config.h
)

######## Tests ############################################################
set(plist_test_SRC
  test/plist_test.c
)
add_executable(plist_test ${plist_test_SRC})
target_link_directories(plist_test PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_test plist-2.0)

set(plist_cmp_SRC
  test/plist_cmp.c
)
add_executable(plist_cmp ${plist_cmp_SRC})
target_link_directories(plist_cmp PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_cmp plist-2.0)

set(plist_test++_SRC
  test/plist_test++.cpp
)
add_executable(plist_test++ ${plist_test++_SRC})
target_link_directories(plist_test++ PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_test++ plist++-2.0)

set(integer_set_SRC
  test/integer_set.c
)
add_executable(integer_set_test ${integer_set_SRC})
target_link_directories(integer_set_test PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(integer_set_test plist-2.0)

set(plist_btest_SRC
  test/plist_btest.c
)
add_executable(plist_btest ${plist_btest_SRC})
target_link_directories(plist_btest PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_btest plist-2.0)

set(plist_jtest_SRC
  test/plist_jtest.c
)
add_executable(plist_jtest ${plist_jtest_SRC})
target_link_directories(plist_jtest PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_jtest plist-2.0)

set(plist_otest_SRC
  test/plist_otest.c
)
add_executable(plist_otest ${plist_otest_SRC})
target_link_directories(plist_otest PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/lib)
target_link_libraries(plist_otest plist-2.0)

set_target_properties(plist_test plist_test++ plist_cmp integer_set_test plist_btest plist_jtest plist_otest PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY test
)

include(CTest)

set(TESTSRC ${CMAKE_CURRENT_SOURCE_DIR}/test)
set(TESTWD ${CMAKE_CURRENT_BINARY_DIR}/test)

function(add_testcase testpath)
  get_filename_component(testname ${testpath} NAME_WE)
  add_test(NAME ${testname} COMMAND sh -c ${testpath} WORKING_DIRECTORY "${TESTWD}")
  set_property(TEST ${testname} PROPERTY ENVIRONMENT "top_srcdir=${CMAKE_CURRENT_SOURCE_DIR};top_builddir=${CMAKE_CURRENT_BINARY_DIR}")
endfunction()

add_testcase(${TESTSRC}/empty.test)
add_testcase(${TESTSRC}/small.test)
add_testcase(${TESTSRC}/medium.test)
add_testcase(${TESTSRC}/large.test)
add_testcase(${TESTSRC}/huge.test)
add_testcase(${TESTSRC}/bigarray.test)
add_testcase(${TESTSRC}/empty++.test)
add_testcase(${TESTSRC}/small++.test)
add_testcase(${TESTSRC}/medium++.test)
add_testcase(${TESTSRC}/large++.test)
add_testcase(${TESTSRC}/huge++.test)
add_testcase(${TESTSRC}/bigarray++.test)
add_testcase(${TESTSRC}/dates.test)
add_testcase(${TESTSRC}/timezone1.test)
add_testcase(${TESTSRC}/timezone2.test)
add_testcase(${TESTSRC}/signedunsigned1.test)
add_testcase(${TESTSRC}/signedunsigned2.test)
add_testcase(${TESTSRC}/signedunsigned3.test)
add_testcase(${TESTSRC}/hex.test)
add_testcase(${TESTSRC}/order.test)
add_testcase(${TESTSRC}/recursion.test)
add_testcase(${TESTSRC}/entities.test)
add_testcase(${TESTSRC}/empty_keys.test)
add_testcase(${TESTSRC}/amp.test)
add_testcase(${TESTSRC}/invalid_tag.test)
add_testcase(${TESTSRC}/cdata.test)
add_testcase(${TESTSRC}/offsetsize.test)
add_testcase(${TESTSRC}/refsize.test)
add_testcase(${TESTSRC}/malformed_dict.test)
add_testcase(${TESTSRC}/uid.test)
add_testcase(${TESTSRC}/integer_set.test)
add_testcase(${TESTSRC}/json1.test)
add_testcase(${TESTSRC}/json2.test)
add_testcase(${TESTSRC}/json3.test)
add_testcase(${TESTSRC}/json-invalid-types.test)
add_testcase(${TESTSRC}/json-int64-min-max.test)
add_testcase(${TESTSRC}/ostep1.test)
add_testcase(${TESTSRC}/ostep2.test)
add_testcase(${TESTSRC}/ostep-strings.test)
add_testcase(${TESTSRC}/ostep-comments.test)
add_testcase(${TESTSRC}/ostep-invalid-types.test)

######## PkgConfig files###################################################
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/src/libplist-2.0.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/lib/libplist-2.0.pc
  @ONLY
)
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/src/libplist++-2.0.pc.in
  ${CMAKE_CURRENT_BINARY_DIR}/lib/libplist++-2.0.pc
  @ONLY
)

######## install ##########################################################
install(TARGETS plist RUNTIME DESTINATION lib COMPONENT plist)
install(TARGETS plist++ RUNTIME DESTINATION lib COMPONENT plist++)
install(TARGETS plistutil RUNTIME DESTINATION bin COMPONENT plistutil)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lib/libplist-2.0.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/ )
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lib/libplist++-2.0.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig/ )
install(FILES ${libplist_HDR} DESTINATION include/plist COMPONENT dev)
install(FILES ${libplist++_HDR} DESTINATION include/plist COMPONENT dev)

######## add uninstall target #############################################
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
"if(NOT EXISTS \"${CMAKE_CURRENT_BINARY_DIR}/install_manifest.txt\")
  message(FATAL_ERROR \"Cannot find install manifest: \\\"${CMAKE_CURRENT_BINARY_DIR}/install_manifest.txt\\\"\")
endif()
file(READ \"${CMAKE_CURRENT_BINARY_DIR}/install_manifest.txt\" files)
string(REGEX REPLACE \"\\n\" \";\" files \"\${files}\")
foreach(file \${files})
  message(STATUS \"Uninstalling \\\"\$ENV{DESTDIR}\${file}\\\"\")
  if((EXISTS \"\$ENV{DESTDIR}\${file}\") OR (IS_SYMLINK \"\$ENV{DESTDIR}\${file}\"))
    exec_program(
      \"${CMAKE_COMMAND}\" ARGS \"-E remove \\\"\$ENV{DESTDIR}\${file}\\\"\"
      OUTPUT_VARIABLE rm_out
      RETURN_VALUE rm_retval
    )
    if(NOT \"\${rm_retval}\" STREQUAL 0)
      message(FATAL_ERROR \"Problem when removing \\\"\$ENV{DESTDIR}\${file}\\\"\")
    endif()
  else()
    message(STATUS \"File \\\"\$ENV{DESTDIR}\${file}\\\" does not exist.\")
  endif()
endforeach()
"
)
add_custom_target(uninstall "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")

########## CPack ################################
#INCLUDE( libplistPackaging )
#LIBPLIST_PACKAGE(LIBPLIST_VERSION_MAJOR LIBPLIST_VERSION_MINOR)

message("
Configuration for ${PACKAGE} ${VERSION}:
-------------------------------------------

  Install prefix ..........: ${CMAKE_INSTALL_PREFIX}
  Debug code ..............: ${debug}
  Python bindings .........: ${cython_python_bindings}
${EXTRA_CONF}
  Now type 'make' to build ${PACKAGE} ${VERSION},
  and then 'make install' for installation.
")
