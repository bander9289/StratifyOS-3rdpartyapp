cmake_minimum_required (VERSION 3.6)

# This will set the default RAM used by the application
set(SOS_APP_RAM_SIZE 4096)

set(THIRDPARTY_ROOT root)
if (NOT THIRDPARTY_TARGET)
	set(THIRDPARTY_TARGET exec_util) #FIXME: this should not be hard-coded, only passed in
endif()

# 3rd party directory may return many targets and libraries, just put the needed info into SOS_APP_*
set(SOURCES_PREFIX ${CMAKE_SOURCE_DIR}/${THIRDPARTY_ROOT})
add_subdirectory(${THIRDPARTY_ROOT})

# get sources and dependency sources for sos to build all at once
get_target_property(MYSOURCES ${THIRDPARTY_TARGET} SOURCES)
list(APPEND SOS_APP_SOURCELIST ${MYSOURCES})
get_target_property(${THIRDPARTY_TARGET}_LIBS ${THIRDPARTY_TARGET} LINK_LIBRARIES)
function(get_all_sources mylibs)
	message(STATUS "mylibs: " ${mylibs})
	foreach(l ${mylibs})
	message(STATUS "l: " ${l})
		get_target_property(TARGET_IS_IMPORTED ${l} IMPORTED)
		message(STATUS "target: " ${l} " imported: " ${TARGET_IS_IMPORTED})
		if (NOT TARGET_IS_IMPORTED)
			get_target_property(MYSOURCES ${l} SOURCES)
			if (MYSOURCES)
				list(APPEND SOS_APP_SOURCELIST ${MYSOURCES})
			endif()
			get_target_property(${l}_LIBS ${l} LINK_LIBRARIES)
			if (${l}_LIBS)
				get_all_sources("${${l}_LIBS}")
				if (SOS_APP_SOURCELIST_NEW)
					list(APPEND SOS_APP_SOURCELIST ${SOS_APP_SOURCELIST_NEW})
				endif()
				if (EXTERNAL_APP_LIBRARIES_NEW)
					list(APPEND EXTERNAL_APP_LIBRARIES ${EXTERNAL_APP_LIBRARIES_NEW})
				endif()
			endif()
		else()
			list(APPEND EXTERNAL_APP_LIBRARIES ${l}) # added as external dependency if no source files
		endif()
	endforeach()

	# propagate to caller but don't destroy caller's data
	if (SOS_APP_SOURCELIST)
		set(SOS_APP_SOURCELIST_NEW ${SOS_APP_SOURCELIST} PARENT_SCOPE)
	endif()
	if (EXTERNAL_APP_LIBRARIES)
		set(EXTERNAL_APP_LIBRARIES_NEW ${EXTERNAL_APP_LIBRARIES} PARENT_SCOPE) 
	endif()
endfunction()
if (${THIRDPARTY_TARGET}_LIBS)
	get_all_sources("${${THIRDPARTY_TARGET}_LIBS}")
	if (SOS_APP_SOURCELIST_NEW)
		list(APPEND SOS_APP_SOURCELIST ${SOS_APP_SOURCELIST_NEW})
	endif()
	if (EXTERNAL_APP_LIBRARIES_NEW)
		list(APPEND EXTERNAL_APP_LIBRARIES ${EXTERNAL_APP_LIBRARIES_NEW})
	endif()
endif()

# get global and target-specific include directories
get_target_property(MYINCLUDEDIRS ${THIRDPARTY_TARGET} INCLUDE_DIRECTORIES)
if (MYINCLUDEDIRS)
	# TODO: eliminate duplicates with 'list(FIND ...' or other method
	list(APPEND SOS_APP_INCLUDE_DIRECTORIES ${MYINCLUDEDIRS})
endif()
get_property(MYINCLUDEDIRS DIRECTORY ${SOURCES_PREFIX} PROPERTY INCLUDE_DIRECTORIES)
if (MYINCLUDEDIRS)
	list(APPEND SOS_APP_INCLUDE_DIRECTORIES ${MYINCLUDEDIRS})
endif()

# get global and target-specific preprocessor definitions
get_target_property(MYDEFINES ${THIRDPARTY_TARGET} COMPILE_DEFINITIONS)
if (MYDEFINES)
	list(APPEND SOS_APP_DEFINITIONS ${MYDEFINES})
endif()
get_property(MYDEFINES DIRECTORY ${SOURCES_PREFIX} PROPERTY COMPILE_DEFINITIONS)
if (MYDEFINES)
	list(APPEND SOS_APP_DEFINITIONS ${MYDEFINES})
endif()
list(APPEND SOS_APP_DEFINITIONS _GLIBCXX__PTHREADS)

# get compiler flags
get_target_property(MYFLAGS ${THIRDPARTY_TARGET} COMPILE_OPTIONS)
if (MYFLAGS)
	list(APPEND SOS_APP_COMPILE_OPTIONS ${MYFLAGS})
endif()

if( ${CMAKE_HOST_SYSTEM_NAME} STREQUAL "Darwin" )
	set(SOS_TOOLCHAIN_CMAKE_PATH /Applications/StratifyLabs-SDK/Tools/gcc/arm-none-eabi/cmake)
elseif( ${CMAKE_HOST_SYSTEM_NAME} STREQUAL "Windows" )
	set(SOS_TOOLCHAIN_CMAKE_PATH C:/StratifyLabs-SDK/Tools/gcc/arm-none-eabi/cmake)
else()
	message(FATAL_ERROR "Cross compiling is not yet supportd on: " ${CMAKE_HOST_SYSTEM_NAME})
endif()

include(${SOS_TOOLCHAIN_CMAKE_PATH}/sos-resolve-build-config.cmake)

if( ${SOS_BUILD_CONFIG} STREQUAL arm )
	message(STATUS "Load SOS Lib Toolchain for arm: " ${SOS_TOOLCHAIN_CMAKE_PATH})
	set(CMAKE_TOOLCHAIN_FILE ${SOS_TOOLCHAIN_CMAKE_PATH}/sos-app-toolchain.cmake)
elseif( ${SOS_BUILD_CONFIG} STREQUAL link )
	set(CMAKE_TOOLCHAIN_FILE ${SOS_TOOLCHAIN_CMAKE_PATH}/link-toolchain.cmake)
endif()

get_filename_component(SOS_APP_NAME ${CMAKE_SOURCE_DIR} NAME)
project(${SOS_APP_NAME} CXX C)

message(STATUS "Building App sources: " ${SOS_APP_SOURCELIST})
message(STATUS "Using App include directories: " ${SOS_APP_INCLUDE_DIRECTORIES})
message(STATUS "Using App defines: " ${SOS_APP_DEFINITIONS})
message(STATUS "Using App flags: " ${SOS_APP_COMPILE_OPTIONS})


if( ${SOS_BUILD_CONFIG} STREQUAL arm )
	#include(toolchain.cmake)
	include(${SOS_TOOLCHAIN_CMAKE_PATH}/sos-app-std.cmake)
elseif( ${SOS_BUILD_CONFIG} STREQUAL link )
	set(SOS_APP_ARCH link)
	set(SOS_APP_OPTION "")
	set(SOS_APP_TYPE release)
	include(${SOS_TOOLCHAIN_CMAKE_PATH}/sos-app.cmake)
endif()

# additional library paths and dependencies
target_link_directories(${BUILD_TARGET}
	PUBLIC
		${TOOLCHAIN_LIB_DIR}/../../lib/gcc/${TOOLCHAIN_HOST}/${CMAKE_CXX_COMPILER_VERSION}/${SOS_BUILD_INSTALL_DIR}/${SOS_BUILD_FLOAT_DIR}
		${SOS_ARCH_LIBRARY_PATH}
		)
list(APPEND EXTERNAL_APP_LIBRARIES gcc)

message(STATUS "Imported App libraries: " ${EXTERNAL_APP_LIBRARIES})
message(STATUS "CMake library path: " ${CMAKE_LIBRARY_PATH})
message(STATUS "Toolchain library path: " ${TOOLCHAIN_LIB_DIR})
message(STATUS "SOS ARCH library path: " ${SOS_ARCH_LIBRARY_PATH})
message(STATUS "Suffix 1: " ${SOS_BUILD_INSTALL_DIR})
message(STATUS "Suffix 2: " ${SOS_BUILD_FLOAT_DIR})
message(STATUS "Build target: " ${BUILD_TARGET})

foreach(l ${EXTERNAL_APP_LIBRARIES})
	target_link_libraries(${BUILD_TARGET} -l${l})
endforeach()
