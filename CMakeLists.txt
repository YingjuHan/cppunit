cmake_minimum_required(VERSION 3.16)

project(cppunit VERSION 1.15.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_DEBUG_POSTFIX "d")

foreach(output_dir ARCHIVE LIBRARY RUNTIME)
  set(CMAKE_${output_dir}_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
endforeach()

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

option(CPPUNIT_BUILD_STATIC "Build the static cppunit library" ON)
option(CPPUNIT_BUILD_SHARED "Build the shared cppunit library" ON)
option(CPPUNIT_BUILD_DLL_PLUGIN_TESTER "Build the DllPlugInTester helper" ON)

if(NOT CPPUNIT_BUILD_STATIC AND NOT CPPUNIT_BUILD_SHARED)
  message(FATAL_ERROR "Enable at least one of CPPUNIT_BUILD_STATIC or CPPUNIT_BUILD_SHARED")
endif()

set(cppunit_core_sources
  src/cppunit/AdditionalMessage.cpp
  src/cppunit/Asserter.cpp
  src/cppunit/BriefTestProgressListener.cpp
  src/cppunit/CompilerOutputter.cpp
  src/cppunit/DefaultProtector.cpp
  src/cppunit/DynamicLibraryManager.cpp
  src/cppunit/DynamicLibraryManagerException.cpp
  src/cppunit/Exception.cpp
  src/cppunit/Message.cpp
  src/cppunit/PlugInManager.cpp
  src/cppunit/PlugInParameters.cpp
  src/cppunit/Protector.cpp
  src/cppunit/ProtectorChain.cpp
  src/cppunit/RepeatedTest.cpp
  src/cppunit/ShlDynamicLibraryManager.cpp
  src/cppunit/SourceLine.cpp
  src/cppunit/StringTools.cpp
  src/cppunit/SynchronizedObject.cpp
  src/cppunit/Test.cpp
  src/cppunit/TestAssert.cpp
  src/cppunit/TestCase.cpp
  src/cppunit/TestCaseDecorator.cpp
  src/cppunit/TestComposite.cpp
  src/cppunit/TestFailure.cpp
  src/cppunit/TestFactoryRegistry.cpp
  src/cppunit/TestLeaf.cpp
  src/cppunit/TestNamer.cpp
  src/cppunit/TestPath.cpp
  src/cppunit/TestDecorator.cpp
  src/cppunit/TestPlugInDefaultImpl.cpp
  src/cppunit/TestResult.cpp
  src/cppunit/TestResultCollector.cpp
  src/cppunit/TestRunner.cpp
  src/cppunit/TestSetUp.cpp
  src/cppunit/TestSuccessListener.cpp
  src/cppunit/TestSuite.cpp
  src/cppunit/TestSuiteBuilderContext.cpp
  src/cppunit/TextOutputter.cpp
  src/cppunit/TextTestProgressListener.cpp
  src/cppunit/TextTestResult.cpp
  src/cppunit/TextTestRunner.cpp
  src/cppunit/TypeInfoHelper.cpp
  src/cppunit/UnixDynamicLibraryManager.cpp
  src/cppunit/Win32DynamicLibraryManager.cpp
  src/cppunit/XmlDocument.cpp
  src/cppunit/XmlElement.cpp
  src/cppunit/XmlOutputter.cpp
  src/cppunit/XmlOutputterHook.cpp
)

set(cppunit_install_targets)

if(CPPUNIT_BUILD_STATIC)
  add_library(cppunit STATIC ${cppunit_core_sources})
  target_include_directories(cppunit PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )
  target_compile_definitions(cppunit PRIVATE _LIB)
  list(APPEND cppunit_install_targets cppunit)
  add_library(CppUnit::cppunit ALIAS cppunit)
endif()

if(CPPUNIT_BUILD_SHARED)
  add_library(cppunit_dll SHARED ${cppunit_core_sources} src/cppunit/DllMain.cpp)
  target_include_directories(cppunit_dll PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )
  target_compile_definitions(cppunit_dll PRIVATE CPPUNIT_BUILD_DLL)
  set_target_properties(cppunit_dll PROPERTIES OUTPUT_NAME cppunit_dll)
  list(APPEND cppunit_install_targets cppunit_dll)
  add_library(CppUnit::cppunit_dll ALIAS cppunit_dll)
endif()

if(CPPUNIT_BUILD_DLL_PLUGIN_TESTER)
  set(_cppunit_consumer_target)
  if(TARGET cppunit_dll)
    set(_cppunit_consumer_target cppunit_dll)
  elseif(TARGET cppunit)
    set(_cppunit_consumer_target cppunit)
  endif()

  if(NOT _cppunit_consumer_target)
    message(FATAL_ERROR "DllPlugInTester requires cppunit or cppunit_dll to be built.")
  endif()

  add_executable(DllPlugInTester
    src/DllPlugInTester/CommandLineParser.cpp
    src/DllPlugInTester/DllPlugInTester.cpp
  )
  target_include_directories(DllPlugInTester PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include")
  target_link_libraries(DllPlugInTester PRIVATE ${_cppunit_consumer_target})

  if(_cppunit_consumer_target STREQUAL "cppunit_dll")
    target_compile_definitions(DllPlugInTester PRIVATE CPPUNIT_DLL)
  endif()

  if(WIN32)
    target_link_libraries(DllPlugInTester PRIVATE odbc32 odbccp32)
  endif()

  list(APPEND cppunit_install_targets DllPlugInTester)
  add_executable(CppUnit::DllPlugInTester ALIAS DllPlugInTester)
endif()

set(cppunit_install_cmakedir "${CMAKE_INSTALL_LIBDIR}/cmake/CppUnit")

install(TARGETS ${cppunit_install_targets}
  EXPORT CppUnitTargets
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(EXPORT CppUnitTargets
  NAMESPACE CppUnit::
  DESTINATION ${cppunit_install_cmakedir}
  FILE CppUnitTargets.cmake
)

configure_package_config_file(
  cmake/CppUnitConfig.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/CppUnitConfig.cmake"
  INSTALL_DESTINATION ${cppunit_install_cmakedir}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/CppUnitConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/CppUnitConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/CppUnitConfigVersion.cmake"
  DESTINATION ${cppunit_install_cmakedir}
)
