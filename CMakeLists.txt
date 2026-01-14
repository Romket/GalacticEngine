#[[
    File modified from:
    https://github.com/pananton/cpp-lib-template/blob/main/CMakeLists.txt

    cpp-lib-template licensed under Apache-2.0
]]

cmake_minimum_required(VERSION 3.14)
project(galactic
    VERSION 0.0.1
    DESCRIPTION "Game engine library"
    LANGUAGES CXX)

#----------------------------------------------------------------------------------------------------------------------
# general settings and options
#----------------------------------------------------------------------------------------------------------------------

include(cmake/utils.cmake)
include(GNUInstallDirs)

string(COMPARE EQUAL "${CMAKE_SOURCE_DIR}" "${CMAKE_CURRENT_SOURCE_DIR}" is_top_level)

# GALACTIC_SHARED_LIBS option (undefined by default) can be used to force shared/static build
option(GALACTIC_BUILD_TESTS "Build Galactic Engine tests" OFF)
option(GALACTIC_BUILD_EXAMPLES "Build Galactic Engine examples" OFF)
option(GALACTIC_BUILD_DOCS "Build Galactic Engine documentation" OFF)
option(GALACTIC_INSTALL "Generate target for installing Galactic Engine" ${is_top_level})
set_if_undefined(GALACTIC_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/galacticengine" CACHE STRING
    "Install path for Galactic Engine package-related CMake files")

if(DEFINED GALACTIC_SHARED_LIBS)
    set(BUILD_SHARED_LIBS ${GALACTIC_SHARED_LIBS})
endif()

if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set_if_undefined(CMAKE_CXX_VISIBILITY_PRESET hidden)
set_if_undefined(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

add_library(galactic) # initialized below
add_library(galactic::galactic ALIAS galactic)

#----------------------------------------------------------------------------------------------------------------------
# Galactic Engine dependencies
#----------------------------------------------------------------------------------------------------------------------

# Search for your dependencies here

#----------------------------------------------------------------------------------------------------------------------
# Galactic Engine sources
#----------------------------------------------------------------------------------------------------------------------

include(GenerateExportHeader)
set(export_file_name "export_shared.h")

if(NOT BUILD_SHARED_LIBS)
    set(export_file_name "export_static.h")
endif()

generate_export_header(galactic EXPORT_FILE_NAME include/galactic/${export_file_name})

set(sources
#   include/galactic/export.h
    include/galactic/galactic.h
    src/galactic.cpp)
source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES ${sources})

#----------------------------------------------------------------------------------------------------------------------
# Galactic Engine target
#----------------------------------------------------------------------------------------------------------------------

include(CMakePackageConfigHelpers)

target_sources(galactic PRIVATE ${sources})
target_compile_definitions(galactic PUBLIC "$<$<NOT:$<BOOL:${BUILD_SHARED_LIBS}>>:GALACTIC_STATIC_DEFINE>")

target_include_directories(galactic
    PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>")

set_target_properties(galactic PROPERTIES
    SOVERSION ${PROJECT_VERSION_MAJOR}
    VERSION ${PROJECT_VERSION})

if(GALACTIC_INSTALL AND NOT CMAKE_SKIP_INSTALL_RULES)
    configure_package_config_file(cmake/galactic-config.cmake.in galactic-config.cmake
        INSTALL_DESTINATION "${GALACTIC_INSTALL_CMAKEDIR}")

    write_basic_package_version_file(galactic-config-version.cmake
        COMPATIBILITY SameMajorVersion)

    install(TARGETS galactic EXPORT galactic_export
        RUNTIME COMPONENT galactic
        LIBRARY COMPONENT galactic NAMELINK_COMPONENT galactic-dev
        ARCHIVE COMPONENT galactic-dev
        INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
    install(DIRECTORY include/
        TYPE INCLUDE
        COMPONENT galactic-dev)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/include/galactic/${export_file_name}"
        COMPONENT galactic-dev
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/galactic")

    set(targets_file "galactic-shared-targets.cmake")

    if(NOT BUILD_SHARED_LIBS)
        set(targets_file "galactic-static-targets.cmake")
    endif()

    install(EXPORT galactic_export
        COMPONENT galactic-dev
        FILE "${targets_file}"
        DESTINATION "${GALACTIC_INSTALL_CMAKEDIR}"
        NAMESPACE galactic::)

    install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/galactic-config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/galactic-config-version.cmake"
        COMPONENT galactic-dev
        DESTINATION "${GALACTIC_INSTALL_CMAKEDIR}")

    if(MSVC)
        set(pdb_file "")
        set(pdb_file_destination "")

        if(BUILD_SHARED_LIBS)
            set(pdb_file "$<TARGET_PDB_FILE:galactic>")
            set(pdb_file_destination "${CMAKE_INSTALL_BINDIR}")
        else()
            # TARGET_PDB_FILE does not work for pdb file generated for static library build, determining it manually
            set(pdb_file "$<TARGET_FILE_DIR:galactic>/$<TARGET_FILE_PREFIX:galactic>$<TARGET_FILE_BASE_NAME:galactic>.pdb")
            set(pdb_file_destination "${CMAKE_INSTALL_LIBDIR}")
        endif()

        install(FILES "${pdb_file}"
            COMPONENT galactic-dev
            CONFIGURATIONS Debug RelWithDebInfo
            DESTINATION "${pdb_file_destination}"
            OPTIONAL)
    endif()
endif()

#----------------------------------------------------------------------------------------------------------------------
# other targets
#----------------------------------------------------------------------------------------------------------------------

if(GALACTIC_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if(GALACTIC_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if(GALACTIC_BUILD_DOCS)
    find_package(Doxygen REQUIRED)
    doxygen_add_docs(docs include)
endif()
