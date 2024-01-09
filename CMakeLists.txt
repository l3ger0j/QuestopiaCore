cmake_minimum_required (VERSION 3.16)
project ("QuestopiaCore")

if(WIN32)
	if(NOT MSVC OR MSVC_TOOLSET_VERSION LESS 110)
		message(FATAL_ERROR "Only Visual Studio 2012 and later is supported for Windows targets")
	endif()
elseif(APPLE OR NOT UNIX)
	message(FATAL_ERROR "Only Windows and Linux targets are supported")
endif()

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)

set(CMAKE_INCLUDE_CURRENT_DIR True)

if(CMAKE_COMPILER_IS_GNUCC)
	set(PROJECT_COMPILER_FLAGS "-Wall")
	set(PROJECT_LINKER_FLAGS "-no-pie")
	set(CMAKE_CXX_FLAGS "-Wall -Wextra")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -D_DEBUG -DDEBUG")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -D_DEBUG -DDEBUG")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3 -DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -DNDEBUG")
endif()
if(MSVC)
	set(PROJECT_COMPILER_FLAGS "/W4")
	set(PROJECT_LINKER_FLAGS "")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /D_DEBUG /DDEBUG")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /D_DEBUG /DDEBUG")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /O2 /DNDEBUG")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /O2 /DNDEBUG")
endif()

include(CMakePackageConfigHelpers)
include(GenerateExportHeader)
include(GNUInstallDirs)

set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)

option(USE_INSTALLED_ONIGURUMA "Use system-wide oniguruma library instead of bundling it" OFF)
if(USE_INSTALLED_ONIGURUMA)
	find_package(oniguruma REQUIRED)
else()
	# Download and unpack oniguruma at configure time
	configure_file(CMakeLists-oniguruma.txt.in "${CMAKE_BINARY_DIR}/oniguruma-download/CMakeLists.txt")
	execute_process(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
			WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/oniguruma-download"
	)
	execute_process(COMMAND "${CMAKE_COMMAND}" --build .
			WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/oniguruma-download"
	)

	# Disable oniguruma features we don't need
	set(INSTALL_DOCUMENTATION OFF)
	set(INSTALL_EXAMPLES OFF)
	set(BUILD_TEST OFF)

	# Prevent oniguruma from overriding our compiler/linker options
	# when building with Visual Studio
	set(oniguruma_force_shared_crt ON CACHE BOOL "" FORCE)

	# Configure the CRNL handling
	add_compile_definitions(USE_CRNL_AS_LINE_TERMINATOR)

	# Add oniguruma directly to our build
	set(CMAKE_POLICY_DEFAULT_CMP0063 NEW)
	set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)
	add_subdirectory("${CMAKE_BINARY_DIR}/oniguruma-src"
					 "${CMAKE_BINARY_DIR}/oniguruma-build"
	)
endif()

set(QTPC_SOURCES
	src/bindings/default/qsp_default.h
	src/bindings/default/default_callbacks.c
	src/bindings/default/default_control.c
	src/bindings/bindings_config.h
	src/bindings/qsp.h
	src/actions.c
	src/callbacks.c
	src/codetools.c
	src/coding.c
	src/common.c
	src/errors.c
	src/game.c
	src/locations.c
	src/mathops.c
	src/memwatch.c
	src/menu.c
	src/objects.c
	src/playlist.c
	src/statements.c
	src/text.c
	src/time.c
	src/towlower.c
	src/towupper.c
	src/variables.c
	src/variant.c
	src/actions.h
	src/callbacks.h
	src/codetools.h
	src/coding.h
	src/common.h
	src/declarations.h
	src/errors.h
	src/game.h
	src/locations.h
	src/mathops.h
	src/memwatch.h
	src/menu.h
	src/objects.h
	src/playlist.h
	src/statements.h
	src/text.h
	src/time.h
	src/variables.h
	src/variant.h
)
add_library(qtpcore SHARED ${QTPC_SOURCES})
target_compile_definitions(qtpcore PRIVATE _UNICODE EXPORT)
if(WIN32)
	target_compile_definitions(qtpcore PRIVATE _CRT_SECURE_NO_WARNINGS _WIN)
endif()
if(USE_INSTALLED_ONIGURUMA)
	target_link_libraries(qtpcore PRIVATE oniguruma::onig)
else()
	target_link_libraries(qtpcore PRIVATE onig)
	install(TARGETS onig)
endif()

install(FILES
		${CMAKE_SOURCE_DIR}/cmake/Modules/Findoniguruma.cmake
	DESTINATION
		"${CMAKE_INSTALL_DATAROOTDIR}/cmake/Modules"
)