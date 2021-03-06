cmake_minimum_required(VERSION 2.8)
project(mcsimport)
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# Provides things like lib64 dir (and Debian equiv)
include(GNUInstallDirs)

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=gnu++11" COMPILER_SUPPORTS_CXX11)
if(NOT COMPILER_SUPPORTS_CXX11)
    message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no GNU++11 support. Please use a different C++ compiler.")
endif()

IF(UNIX)
  INCLUDE(Findmcsapi)
ENDIF(UNIX)
IF(WIN32)
  INCLUDE(Findmcsapi_windows)
ENDIF(WIN32)

if (NOT MCSAPI_FOUND)
    IF(UNIX)
        MESSAGE(FATAL_ERROR "Could not find mcsapi")
    ENDIF(UNIX)
    IF(WIN32)
        MESSAGE(FATAL_ERROR "Could not find mcsapi.\nYou can hint cmake by setting the environment variable MCSAPI_INSTALL_DIR")
    ENDIF(WIN32)
endif()
include_directories(${MCSAPI_INCLUDE_DIRS})

IF(UNIX)
  INCLUDE(FindYamlCpp)
ENDIF(UNIX)
IF(WIN32)
  INCLUDE(FindYamlCpp)
ENDIF(WIN32)

IF (NOT YAML_FOUND)
  IF(UNIX)
    MESSAGE(FATAL_ERROR "Could not find libyaml-cpp")
  ENDIF(UNIX)
  IF(WIN32)
    MESSAGE(FATAL_ERROR "Could not find libyaml-cppmd.\nYou can hint cmake by setting the environment variable YAML_CPP_INSTALL_DIR")
  ENDIF(WIN32)
ENDIF()
include_directories(${YAML_INCLUDE_DIRS})

IF(UNIX)
  # Disable no-deprecated-declarations
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DDEBUG -Wall -Wunused -Wwrite-strings -Wno-strict-aliasing -Wextra -Wshadow -Werror -Wno-deprecated-declarations")

  # Disable format-truncation since this triggers in mcsapi_types for something that isn't a problem
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11 -Wformat -Wformat-security -fstack-protector-all -fstack-check -Wno-format-truncation -Wno-unused-variable -Wno-unused-parameter -fno-omit-frame-pointer -g")
ENDIF(UNIX)

SET(MCSIMPORT_FILES
    mcsimport.cpp
)

add_executable(mcsimport ${MCSIMPORT_FILES})

target_link_libraries(mcsimport ${MCSAPI_LIBRARIES} ${YAML_LIBRARIES})

IF(WIN32)
    INSTALL(TARGETS mcsimport RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT mcsimport)
    INSTALL(FILES "${CMAKE_CURRENT_SOURCE_DIR}/README.md" DESTINATION "${CMAKE_INSTALL_DOCDIR}" COMPONENT mcsimport)
ELSE(WIN32)
    INSTALL(TARGETS mcsimport RUNTIME DESTINATION "${MCSIMPORT_DIR}" COMPONENT mcsimport)
    INSTALL(FILES "${CMAKE_CURRENT_SOURCE_DIR}/README.md" DESTINATION "${MCSIMPORT_DIR}" COMPONENT mcsimport)
    INSTALL(FILES "${CMAKE_CURRENT_SOURCE_DIR}/columnstoreToolsAlias.sh" DESTINATION "/etc/profile.d" COMPONENT mcsimport)
ENDIF(WIN32)

IF(TEST_RUNNER)
    execute_process(COMMAND python -c "import sys; print(sys.version_info[0]);" OUTPUT_VARIABLE PYTHON_VERSION_AVAILABLE OUTPUT_STRIP_TRAILING_WHITESPACE)
    IF(PYTHON_VERSION_AVAILABLE EQUAL 2)
        execute_process(COMMAND python -c "import sys; print(sys.executable);" OUTPUT_VARIABLE PYTHON2_EXECUTABLE OUTPUT_STRIP_TRAILING_WHITESPACE)
    ELSE(PYTHON_VERSION_AVAILABLE EQUAL 2)
         MESSAGE(FATAL_ERROR "Python2 is required to test mcsapi")
    ENDIF(PYTHON_VERSION_AVAILABLE EQUAL 2)
    # On Windows copy the required shared libraries to execute the tests.
    IF(WIN32)
        ADD_CUSTOM_TARGET(copy_required_shared_test_libraries ALL
            COMMAND ${CMAKE_COMMAND} -E copy ${MCSAPI_LIBXML2_RUNTIME_LIBRARY} "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>"
            COMMAND ${CMAKE_COMMAND} -E copy ${MCSAPI_LIBICONV_RUNTIME_LIBRARY} "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>"
            COMMAND ${CMAKE_COMMAND} -E copy ${MCSAPI_LIBUV_RUNTIME_LIBRARY} "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>"
            COMMAND ${CMAKE_COMMAND} -E copy ${MCSAPI_RUNTIME_LIBRARY} "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>"
            DEPENDS mcsimport
            COMMENT "Copy the shared libraries for executing the tests"
        )
        add_test(NAME mcsimport_test_suite COMMAND "${PYTHON2_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/test/test.py" "${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/mcsimport.exe")
    ENDIF(WIN32)
    IF(UNIX)
        add_test(NAME mcsimport_test_suite COMMAND "${PYTHON2_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/test/test.py" "${CMAKE_CURRENT_BINARY_DIR}/mcsimport")
    ENDIF(UNIX)
    set_tests_properties(mcsimport_test_suite PROPERTIES TIMEOUT 3600)
ENDIF(TEST_RUNNER)

# Install the mcsapi DLLs with the mcsimport binary
IF(WIN32)
    INSTALL(FILES ${MCSAPI_RUNTIME_LIBRARY} DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT mcsimport)
    INSTALL(FILES ${MCSAPI_LIBXML2_RUNTIME_LIBRARY} DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT mcsimport)
    INSTALL(FILES ${MCSAPI_LIBICONV_RUNTIME_LIBRARY} DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT mcsimport)
    INSTALL(FILES ${MCSAPI_LIBUV_RUNTIME_LIBRARY} DESTINATION "${CMAKE_INSTALL_BINDIR}" COMPONENT mcsimport)
    install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/3rd party licenses.txt" DESTINATION "${CMAKE_INSTALL_DOCDIR}" COMPONENT mcsimport)
ENDIF(WIN32)

