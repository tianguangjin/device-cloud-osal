#
# Copyright (C) 2017-2018 Wind River Systems, Inc. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software  distributed
# under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES
# OR CONDITIONS OF ANY KIND, either express or implied.
#

set( CMAKE_MODULE_PATH           "${CMAKE_SOURCE_DIR}/build-sys/cmake" "${CMAKE_MODULE_PATH}" )
cmake_minimum_required( VERSION 2.8.8 )
project( "osal" )

### Project Version ###
include( VersionSupport )
string( REGEX REPLACE "20([0-9][0-9])" "\\1" PACKAGE_VERSION "${PROJECT_COMMIT_DATE}" )
string( REPLACE "-" "." PACKAGE_VERSION "${PACKAGE_VERSION}" )
version_split( PACKAGE_VERSION "${PACKAGE_VERSION}" )

### Definitions ###
set( PACKAGE_NAME                      "Operating System Abstraction Library" )
set( PACKAGE_DESCRIPTION_SUMMARY       "Operating System Abstraction Library" )
set( PACKAGE_VENDOR                    "Wind River Systems" )

set( PROJECT_VERSION                   "${PACKAGE_VERSION}" )
set( PROJECT_VENDOR                    "${PACKAGE_VENDOR}" )

### General Build Config ###
include( BuildSupport )
include( DefineCompilerFlags )

set( TARGET                            "${PACKAGE_NAME_SHORT}" )
set( TARGET_STATIC_SUFFIX              "-static" )
set( TARGET_SHARED_SUFFIX              "-shared" )

### Output Directories ###
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY    "${CMAKE_BINARY_DIR}/out" )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY    "${CMAKE_BINARY_DIR}/out" )
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY    "${CMAKE_BINARY_DIR}/out" )
set( CMAKE_INCLUDE_OUTPUT_DIRECTORY    "${CMAKE_BINARY_DIR}/out" )

### Operating System Specific Options ###
if( APPLE )
	set( CMAKE_MACOSX_RPATH ON )
endif( APPLE )

### Build Library ###
# Similiar to the cmake "option" command, except that it ensures that the
# given variable has a value defined and is not defined to "blank"
# Arguments:
#   VAR_NAME - name of the variable
#   VAR_DESCRIPTION - description of the variable
#   ... - initial value for th eoption (ON or OFF)
function( OPTION_ENSURE_SET VAR_NAME VAR_DESCRIPTION )
	if ( DEFINED ${VAR_NAME} AND "x${${VAR_NAME}}" STREQUAL "x" )
		unset( ${VAR_NAME} CACHE )
	endif ( DEFINED ${VAR_NAME} AND "x${${VAR_NAME}}" STREQUAL "x" )
	option( ${ARGV} )
endfunction( OPTION_ENSURE_SET )
option_ensure_set( OSAL_THREAD_SUPPORT "enable multi-thread support" ON )
option_ensure_set( OSAL_WRAP "provide wrappers for simple functions, this is useful for mocking and unit testing" OFF )

# Definitions for build
#######################
if ( OSAL_WRAP )
	add_definitions( "-DOSAL_WRAP=1" ) # true (use defines)
endif( OSAL_WRAP )

if ( OSAL_THREAD_SUPPORT )
	find_package( Threads )
	if ( THREADS_FOUND )
		add_definitions( "-DOSAL_THREAD_SUPPORT=1" ) # true (thread support)
	endif()
endif()

add_subdirectory( "src" )

### Doxygen ###
find_package( Doxygen )

if( DOXYGEN_FOUND )
	set( DOXYFILE_SOURCE_DIR "build-sys/doxygen" )
	string( TIMESTAMP DATE_YEAR "%Y" UTC )
	string( TIMESTAMP DATE_MONTH "%m" UTC )
	string( TIMESTAMP DATE_DAY "%d" UTC )
	string( TIMESTAMP DATE_TIME "%H:%M" UTC )
	configure_file( "${DOXYFILE_SOURCE_DIR}/Doxyfile.in" "${CMAKE_BINARY_DIR}/Doxyfile" @ONLY )
	configure_file( "${DOXYFILE_SOURCE_DIR}/footer.html.in" "${CMAKE_BINARY_DIR}/footer.html" @ONLY )

	add_custom_target( doc
		COMMAND "${CMAKE_COMMAND}" -E echo "Generating documentation..."
		COMMAND "${CMAKE_COMMAND}" -E make_directory "${CMAKE_BINARY_DIR}/doxygen"
		COMMAND "${DOXYGEN_EXECUTABLE}" "${CMAKE_BINARY_DIR}/Doxyfile"
		COMMAND "${CMAKE_COMMAND}" -E echo_append "Complete"
		DEPENDS "${CMAKE_BINARY_DIR}/Doxyfile"
		WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
		COMMENT "Generating Documentation" VERBATIM
	)
endif( DOXYGEN_FOUND )

### CPack Config ###
set( CPACK_PACKAGE_VERSION ${PACKAGE_VERSION} )
set( CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_DESCRIPTION_SUMMARY}" )
set( CPACK_PACKAGE_CONTACT "support@windriver.com" )

if ( WIN32 )
	set( CPACK_GENERATOR "ZIP" )
else()
	set( CPACK_GENERATOR "DEB" )
	find_program( RPMBUILD_EXECUTABLE rpmbuild
		DOC "path to rpmbuild executable" )
	mark_as_advanced( RPMBUILD_EXECUTABLE )
	if ( RPMBUILD_EXECUTABLE )
		set( GENERATOR_DEFAULT "RPM" )
	endif ( RPMBUILD_EXECUTABLE )
	set( GENERATOR_LIST "RPM" "DEB" )
	set( CPACK_DEBIAN_PACKAGE_DEPENDS "sudo, libc6" )
endif()

include( CPack )
include( TestSupport )

