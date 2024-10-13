#==============================================================================#
# CMake build file for Little-CMS
#==============================================================================#
cmake_minimum_required( VERSION 3.5 )
#
# Read LCMS VERSION from configure file
# - if VERSION not found in the configure file it is set to 0.0
file( READ "${CMAKE_CURRENT_LIST_DIR}/../../configure" lcms2_configure )
string( REGEX MATCH "PACKAGE_VERSION='([0-9]+)\\.([0-9]+)'" 
                   _ ${lcms2_configure} )
set( LCMS2_VERSION_MAJOR "${CMAKE_MATCH_1}" )
set( LCMS2_VERSION_MINOR "${CMAKE_MATCH_2}" )
set( LCMS2_VERSION_MICRO 0                  )
string( COMPARE EQUAL "${CMAKE_MATCH_1}" "" result )
if( result )
  set( LCMS2_VERSION_MAJOR "0" )
  set( LCMS2_VERSION_MINOR "0" )
endif()
set( LCMS2_VERSION "${LCMS2_VERSION_MAJOR}.${LCMS2_VERSION_MINOR}" )

message( "===================================================================" )
message( "  Little CMS - version ${LCMS2_VERSION}" )
message( "===================================================================" )
message( "" )
project( lcms2 VERSION ${LCMS2_VERSION} LANGUAGES C CXX )
#
# Build options
option( BUILD_TESTING     "build test executables" OFF )
option( BUILD_STATIC_LIBS "build static library"   OFF )
option( BUILD_SHARED_LIBS "build shared library"   ON  )

option( LCMS_INSTALL_DOCS           "Install documentation"          OFF )
option( LCMS_INSTALL_MANPAGES       "Install manpages for CLI tools" OFF )
option( LCMS_BUILD_TOOLS            "Build CLI tools"                OFF )
option( LCMS_BUILD_JPG_TOOLS        "Build jpeg CLI tools"           OFF )
option( LCMS_BUILD_TIF_TOOLS        "Build tiff CLI tools"           OFF )
option( LCMS_WITH_PLUGIN_FAST_FLOAT "Build fast float plugin"        OFF )
option( LCMS_WITH_PLUGIN_THREADED   "Build threaded plugin"          OFF )
option( LCMS_USE_CPP_API            "Compile with a C++ API"         OFF )

if (NOT BUILD_STATIC_LIBS AND NOT BUILD_SHARED_LIBS )
  message( WARNING "No library build type selected, force to shared libs")
  message( "" )
  set(BUILD_SHARED_LIBS ON)
endif()
message( STATUS "Build configuration summary" )
message( STATUS "----------------------------------" )
message( STATUS "Build static libraries:    " ${BUILD_SHARED_LIBS} )
message( STATUS "Build shared libraries:    " ${BUILD_STATIC_LIBS} )
message( STATUS "Build tests:               " ${BUILD_TESTING}     )
message( STATUS "Compile with C++ API:      " ${LCMS_USE_CPP_API} )
message( STATUS "Documentation" )
message( STATUS "  Install documentation:   " ${LCMS_INSTALL_DOCS} )
message( STATUS "  Install manpages:        " ${LCMS_INSTALL_MANPAGES} )
message( STATUS "Tools ")
message( STATUS "  Build CLI tools:         " ${LCMS_BUILD_TOOLS} )
message( STATUS "  Build JPEG tools:        " ${LCMS_BUILD_JPG_TOOLS} )
message( STATUS "  Build TIFF tools:        " ${LCMS_BUILD_TIF_TOOLS} )
message( STATUS "Plugins" )
message( STATUS "  Build fast float plugin: " ${LCMS_WITH_PLUGIN_FAST_FLOAT} )
message( STATUS "  Build threaded plugin:   " ${LCMS_WITH_PLUGIN_THREADED} )
message( "" )
#
# set installation directories
#
include(GNUInstallDirs)
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin )
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin )
if( NOT CMAKE_DEBUG_POSTFIX )
  set(CMAKE_DEBUG_POSTFIX "d" )
endif()
#==============================================================================#
# set additional compiler definitions
#
set( opt_compiler_definitions -DUNICODE -D_UNICODE ) # additional compiler definitions
include( CheckTypeSize )
include( TestBigEndian )
# check if don't use int64 must be set
check_type_size("long long" SIZEOF_LONG_LONG)
check_type_size("long"      SIZEOF_LONG     )
if( NOT( SIZEOF_LONG EQUAL "8" OR SIZEOF_LONG_LONG EQUAL "8" ) )
  list( APPEND opt_compiler_definitions -DCMS_DONT_USE_INT64=1 )
endif()
# check if use big endian must be set
TEST_BIG_ENDIAN(LCMS_USE_BIG_ENDIAN)
if( LCMS_USE_BIG_ENDIAN )
  list( APPEND opt_compiler_definitions -DCMS_USE_BIG_ENDIAN=1 )
endif()
# check if use cpp api must be set
if ( LCMS_USE_CPP_API )
  list( APPEND opt_compiler_definitions -DCMS_USE_CPP_API )
endif()
# general compiler definitions if compiled with MSVC
if(MSVC)
  add_definitions( -D_CRT_SECURE_NO_DEPRECATE  )
  add_definitions( -D_CRT_NONSTDC_NO_DEPRECATE )
endif()
#==============================================================================#
# build libraries
#==============================================================================#
set( lcms_lib_targets ) # targets for installation
#
# build lcms2
#
file( GLOB SRCS_LCMS ../../src/*.c )

if( WIN32 )
  configure_file( ../../src/lcms2.rc.in
                  ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2.rc
                  @ONLY
                )
  set( RC_LCMS_SRC ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2.rc )
endif()

if( BUILD_SHARED_LIBS )
  add_library( lcms2 SHARED ${SRCS_LCMS} ${RC_LCMS_SRC} )

  target_compile_options( lcms2 PRIVATE -DCMS_DLL_BUILD )
  target_compile_options( lcms2 PUBLIC  -DCMS_DLL       )
  target_compile_options( lcms2 PRIVATE ${opt_compiler_definitions} )
  target_compile_options( lcms2 PUBLIC ${opt_compiler_definitions}  )

  target_include_directories( lcms2 PUBLIC 
                    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../include> 
                    $<INSTALL_INTERFACE:include> )

  set_target_properties( lcms2 PROPERTIES PUBLIC_HEADER 
    "${CMAKE_CURRENT_LIST_DIR}/../../include/lcms2.h;${CMAKE_CURRENT_LIST_DIR}/../../include/lcms2_plugin.h")

  list( APPEND lcms_lib_targets lcms2 )
endif()

if( BUILD_STATIC_LIBS )
  add_library( lcms2_static STATIC ${SRCS_LCMS} )

  target_compile_options( lcms2_static PRIVATE ${opt_compiler_definitions} )
  target_compile_options( lcms2_static PUBLIC  ${opt_compiler_definitions} )

  target_include_directories( lcms2_static PUBLIC 
                      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../include> 
                      $<INSTALL_INTERFACE:include> )

  set_target_properties(lcms2_static PROPERTIES PUBLIC_HEADER 
    "${CMAKE_CURRENT_LIST_DIR}/../../include/lcms2.h;${CMAKE_CURRENT_LIST_DIR}/../../include/lcms2_plugin.h")

  list( APPEND lcms_lib_targets lcms2_static )
endif()
#
# build plugin lcms2_fast_float
#
if( LCMS_WITH_PLUGIN_FAST_FLOAT )
  file( GLOB SRCS_FAST_FLOAT ../../plugins/fast_float/src/*.c )
  if( WIN32 )
    configure_file( ../../plugins/fast_float/src/lcms2_fast_float.rc.in
                    ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2_fast_float.rc
                    @ONLY
                  )
    set( RC_FAST_FLOAT_SRC 
         ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2_fast_float.rc )
  endif()
  if( BUILD_SHARED_LIBS )
    add_library( lcms2_fast_float SHARED ${SRCS_FAST_FLOAT} ${RC_FAST_FLOAT_SRC} )

    target_compile_options( lcms2_fast_float PRIVATE -DCMS_DLL_BUILD )
    target_compile_options( lcms2_fast_float PUBLIC  -DCMS_DLL       )
    target_compile_options( lcms2_fast_float PRIVATE ${opt_compiler_definitions} )
    target_compile_options( lcms2_fast_float PUBLIC  ${opt_compiler_definitions} )

    target_link_libraries( lcms2_fast_float lcms2 )
    target_include_directories( lcms2_fast_float PUBLIC 
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../plugins/fast_float/include> 
      $<INSTALL_INTERFACE:include> )

    set_target_properties( lcms2_fast_float PROPERTIES PUBLIC_HEADER 
      "${CMAKE_CURRENT_LIST_DIR}/../../plugins/fast_float/include/lcms2_fast_float.h" )

    list( APPEND lcms_lib_targets lcms2_fast_float )
  endif()
  if( BUILD_STATIC_LIBS )
    add_library( lcms2_fast_float_static STATIC ${SRCS_FAST_FLOAT} ${RC_FAST_FLOAT_SRC} )

    target_compile_options( lcms2_fast_float_static PRIVATE ${opt_compiler_definitions} )
    target_compile_options( lcms2_fast_float_static PUBLIC  ${opt_compiler_definitions} )

    target_link_libraries( lcms2_fast_float_static lcms2_static )
    target_include_directories( lcms2_fast_float_static PUBLIC 
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../plugins/fast_float/include> 
      $<INSTALL_INTERFACE:include> )

    set_target_properties( lcms2_fast_float_static PROPERTIES PUBLIC_HEADER 
      "${CMAKE_CURRENT_LIST_DIR}/../../plugins/fast_float/include/lcms2_fast_float.h" )

    list( APPEND lcms_lib_targets lcms2_fast_float_static )
  endif()
endif()
#
# build plugin lcms2_threaded
#
if( LCMS_WITH_PLUGIN_THREADED )
  file( GLOB SRCS_THREADED ../../plugins/threaded/src/*.c )
  if( WIN32 )
    configure_file( ../../plugins/threaded/src/lcms2_threaded.rc.in
                    ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2_threaded.rc
                    @ONLY
                  )
    set( RC_THREADED_SRC ${CMAKE_CURRENT_BINARY_DIR}/generated/lcms2_threaded.rc )
  endif()
  if( BUILD_SHARED_LIBS )
    add_library( lcms2_threaded SHARED ${SRCS_THREADED} ${RC_THREADED_SRC} )

    target_compile_options( lcms2_threaded PRIVATE -DCMS_DLL_BUILD             )
    target_compile_options( lcms2_threaded PUBLIC  -DCMS_DLL                   )
    target_compile_options( lcms2_threaded PRIVATE ${opt_compiler_definitions} )
    target_compile_options( lcms2_threaded PUBLIC  ${opt_compiler_definitions} )

    target_link_libraries( lcms2_threaded lcms2 )
    target_include_directories( lcms2_threaded PUBLIC 
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../plugins/threaded/include> 
      $<INSTALL_INTERFACE:include> )

    set_target_properties(lcms2_threaded PROPERTIES PUBLIC_HEADER 
      "${CMAKE_CURRENT_LIST_DIR}/../../plugins/threaded/include/lcms2_threaded.h" )

    list( APPEND lcms_lib_targets lcms2_threaded )
  endif()
  if( BUILD_STATIC_LIBS )
    add_library( lcms2_threaded_static STATIC ${SRCS_THREADED} ${RC_THREADED_SRC} )

    target_compile_options( lcms2_threaded_static PRIVATE ${opt_compiler_definitions} )
    target_compile_options( lcms2_threaded_static PUBLIC  ${opt_compiler_definitions} )

    target_link_libraries( lcms2_threaded_static lcms2_static )
    target_include_directories( lcms2_threaded_static PUBLIC 
      $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/../../plugins/threaded/include> 
      $<INSTALL_INTERFACE:include> )

    set_target_properties( lcms2_threaded_static PROPERTIES PUBLIC_HEADER 
      "${CMAKE_CURRENT_LIST_DIR}/../../plugins/threaded/include/lcms2_threaded.h" )

    list( APPEND lcms_lib_targets lcms2_threaded_static )
  endif()
endif()
#==============================================================================#
# install library and plugins
#==============================================================================#
set( config_install_dir  "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"       )
set( include_install_dir "${CMAKE_INSTALL_INCLUDEDIR}/lcms2"                   )
set( generated_dir       "${CMAKE_CURRENT_BINARY_DIR}/generated"               )

set( version_config      "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake" )
set( project_config      "${generated_dir}/${PROJECT_NAME}Config.cmake"        )
set( targets_export_name "${PROJECT_NAME}Targets"                              )
set( namespace           "${PROJECT_NAME}::"                                   )

include( CMakePackageConfigHelpers )

# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)
configure_package_config_file(
    Config.cmake.in
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install( TARGETS ${lcms_lib_targets}
         EXPORT                    "${targets_export_name}"
         LIBRARY DESTINATION       "${CMAKE_INSTALL_LIBDIR}"
         ARCHIVE DESTINATION       "${CMAKE_INSTALL_LIBDIR}"
         RUNTIME DESTINATION       "${CMAKE_INSTALL_BINDIR}"
         INCLUDES DESTINATION      "${include_install_dir}"
         PUBLIC_HEADER DESTINATION "${include_install_dir}"
       )

install( FILES       "${project_config}" 
                     "${version_config}"
         DESTINATION "${config_install_dir}"
       )
install( EXPORT      "${targets_export_name}"
         NAMESPACE   "${namespace}"
         DESTINATION "${config_install_dir}"
       )
#==============================================================================#
# install library and plugin documentation
#==============================================================================#
if ( LCMS_INSTALL_DOCS )
  if ( CMAKE_INSTALL_DOCDIR )
    set( doc_install_dir 
         ${CMAKE_INSTALL_DOCDIR} )
  else()
    set( doc_install_dir 
         ${CMAKE_INSTALL_DATAROOTDIR}/doc/lcms2-${PROJECT_VERSION} )
  endif()
  INSTALL( DIRECTORY ../../doc/
           DESTINATION ${doc_install_dir}
         )
  INSTALL( FILES       ../../AUTHORS 
                       ../../ChangeLog 
                       ../../LICENSE 
           DESTINATION ${doc_install_dir} )
  if( LCMS_WITH_PLUGIN_FAST_FLOAT )
    INSTALL( DIRECTORY   ../../plugins/fast_float/doc/
             DESTINATION ${doc_install_dir}
           )
    INSTALL( FILES       ../../plugins/fast_float/COPYING.GPL3
             RENAME      lcms2_fast_float_plugin.license 
             DESTINATION ${doc_install_dir}
           )
  endif()
  if( LCMS_WITH_PLUGIN_THREADED )
    INSTALL( DIRECTORY   ../../plugins/threaded/doc/
             DESTINATION ${doc_install_dir}
           )
    INSTALL( FILES       ../../plugins/threaded/COPYING.GPL3 
             RENAME      lcms2_threaded_plugin.license
             DESTINATION ${doc_install_dir}
           )
  endif()
endif()
#==============================================================================#
if( BUILD_SHARED_LIBS )
  if( LCMS_BUILD_TOOLS )
    message( STATUS "Build tools linking against shared libraries")
  endif()
  set( lcms_target                   lcms2            )
  set( lcms_plugin_fast_float_target lcms2_fast_float )
  set( lcms_plugin_threaded_target   lcms2_threaded   )
else()
  if( LCMS_BUILD_TOOLS )
    message( STATUS "Build tools linking against static libraries")
  endif()
  set( lcms_target                   lcms2_static            )
  set( lcms_plugin_fast_float_target lcms2_fast_float_static )
  set( lcms_plugin_threaded_target   lcms2_threaded_static   )
endif()
#==============================================================================#
# build and install CLI tools
#==============================================================================#
set( TOOLS_TARGETS_BUILD ) # targets to install
set( TOOLS_MAN_FILES )     # man files to install
file( GLOB common_srcs ../../utils/common/*.c )
if( LCMS_BUILD_TOOLS )
  #
  # Tool linkicc
  #
  file( GLOB linkicc_srcs ../../utils/linkicc/*.c )
  add_executable( linkicc ${linkicc_srcs} ${common_srcs} )
  target_link_libraries(      linkicc ${lcms_target}             )
  target_include_directories( linkicc PRIVATE ../../utils/common )
  list( APPEND TOOLS_TARGETS_BUILD linkicc                       )
  list( APPEND TOOLS_MAN_FILES     ../../utils/linkicc/linkicc.1 )
  #
  # Tool psicc
  #
  file( GLOB psicc_srcs ../../utils/psicc/*.c )
  add_executable( psicc ${psicc_srcs} ${common_srcs} )
  target_link_libraries(      psicc ${lcms_target}             )
  target_include_directories( psicc PRIVATE ../../utils/common )
  list( APPEND TOOLS_TARGETS_BUILD psicc                       )
  list( APPEND TOOLS_MAN_FILES     ../../utils/psicc/psicc.1   )
  #
  # Tool transicc
  #
  file( GLOB transicc_srcs ../../utils/transicc/*.c )
  add_executable( transicc ${transicc_srcs} ${common_srcs} )
  target_link_libraries(      transicc ${lcms_target}              )
  target_include_directories( transicc PRIVATE ../../utils/common  )
  list( APPEND TOOLS_TARGETS_BUILD transicc                        )
  list( APPEND TOOLS_MAN_FILES     ../../utils/transicc/transicc.1 )
endif()
#
# JPEG tools
#
if( LCMS_BUILD_JPG_TOOLS )
  find_package( JPEG REQUIRED )
  file( GLOB jpgicc_srcs ../../utils/jpgicc/*.c )
  add_executable( jpgicc ${jpgicc_srcs} ${common_srcs} )
  target_link_libraries(      jpgicc PUBLIC ${lcms_target} JPEG::JPEG )
  target_include_directories( jpgicc PRIVATE ../../utils/common       )
  list( APPEND TOOLS_TARGETS_BUILD jpgicc                             )
  list( APPEND TOOLS_MAN_FILES     ../../utils/jpgicc/jpgicc.1        )
endif()
#
# TIFF tools
#
if( LCMS_BUILD_TIF_TOOLS )
  find_package( TIFF REQUIRED )
  # tificc
  add_executable( tificc ../../utils/tificc/tificc.c ${common_srcs} )
  target_link_libraries(      tificc PUBLIC ${lcms_target} TIFF::TIFF )
  target_include_directories( tificc PRIVATE ../../utils/common       )
  list( APPEND TOOLS_TARGETS_BUILD tificc                             )
  list( APPEND TOOLS_MAN_FILES     ../../utils/tificc/tificc.1        )
  # tiffdiff
  add_executable( tiffdiff ../../utils/tificc/tifdiff.c ${common_srcs}   )
  target_link_libraries(      tiffdiff PUBLIC  ${lcms_target} TIFF::TIFF )
  target_include_directories( tiffdiff PRIVATE ../../utils/common        )
  list( APPEND TOOLS_TARGETS_BUILD tiffdiff                              )
endif()
#
# install tools and man pages
#
if( LCMS_BUILD_TOOLS OR LCMS_BUILD_JPG_TOOLS OR LCMS_BUILD_TIF_TOOLS )
  INSTALL( TARGETS     ${TOOLS_TARGETS_BUILD}
           DESTINATION ${CMAKE_INSTALL_BINDIR} )
  if( LCMS_INSTALL_MANPAGES )
    INSTALL( FILES       ${TOOLS_DOC_FILES}
             DESTINATION ${CMAKE_INSTALL_MANDIR}/man1 )
  endif()
endif()
#==============================================================================#
# build testing
#==============================================================================#
if( BUILD_TESTING )
  include( CTest )
  set( test_src_dir ${PROJECT_SOURCE_DIR}/../../testbed )
  add_executable(             testcms ${test_src_dir}/testcms2.c 
                                      ${test_src_dir}/testplugin.c 
                                      ${test_src_dir}/zoo_icc.c     )
  target_link_libraries(      testcms PUBLIC  ${lcms_target} )
  target_include_directories( testcms PRIVATE ../../src      )
  add_test( NAME              testcms 
            COMMAND           testcms
            WORKING_DIRECTORY ${test_src_dir}
          )
  add_executable(             alpha_test ${test_src_dir}/alpha_test.c )
  target_link_libraries(      alpha_test PUBLIC  ${lcms_target} )
  target_include_directories( alpha_test PRIVATE ../../src      )
  add_test( NAME              alpha_test
            COMMAND           alpha_test 
            WORKING_DIRECTORY ${test_src_dir}
           )
  
  if( LCMS_WITH_PLUGIN_FAST_FLOAT )
    set( test_src_dir ${PROJECT_SOURCE_DIR}/../../plugins/fast_float/testbed )
    add_executable(             fast_float_test ${test_src_dir}/fast_float_testbed.c )
    target_link_libraries(      fast_float_test PUBLIC  ${lcms_target} 
                                                        ${lcms_plugin_fast_float_target} )
    target_include_directories( fast_float_test PRIVATE ../../plugins/fast_float/src )
    add_test( NAME              fast_float_test
              COMMAND           fast_float_test 
              WORKING_DIRECTORY ${test_src_dir}
            )
  endif()
  if( LCMS_WITH_PLUGIN_THREADED )
    set( test_src_dir ${PROJECT_SOURCE_DIR}/../../plugins/threaded/testbed )
    add_executable(             threaded_test 
                                ${test_src_dir}/threaded_testbed.c )
    target_link_libraries(      threaded_test PUBLIC  ${lcms_target} 
                                                      ${lcms_plugin_threaded_target} )
    target_include_directories( threaded_test PRIVATE ../../plugins/threaded/src )
    add_test( NAME              threaded_test
              COMMAND           threaded_test 
              WORKING_DIRECTORY ${test_src_dir}
            )
  endif()
endif()
