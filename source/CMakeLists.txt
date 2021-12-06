cmake_minimum_required(VERSION 3.12)

project(Kyty)

if (NOT (WIN32 AND (MINGW OR MSVC)))
	message(FATAL_ERROR "only mingw and msvc supported")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

include(utils.cmake)

	remove_generator_macros(CXX __cplusplus)
	remove_generator_macros(CXX __clang_version__)
	remove_generator_macros(C __clang_version__)
	remove_generator_macros(CXX __VERSION__)
	remove_generator_macros(C __VERSION__)
	
	add_generator_macros(CXX __cplusplus "201703L")

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

include(TestBigEndian)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	set(KYTY_BITNESS 64)
else()
	set(KYTY_BITNESS 32)
endif()

test_big_endian(BIG_ENDIAN)
if(BIG_ENDIAN)
	set(KYTY_ENDIAN KYTY_ENDIAN_BIG)
else()
	set(KYTY_ENDIAN KYTY_ENDIAN_LITTLE)
endif()

if(CMAKE_BUILD_TYPE MATCHES Debug)
	set(KYTY_BUILD KYTY_BUILD_DEBUG)
else()
	set(KYTY_BUILD KYTY_BUILD_RELEASE)
endif()

set(KYTY_PLATFORM KYTY_PLATFORM_WINDOWS)

if(MINGW)
	if(CMAKE_CXX_COMPILER_ID MATCHES "(C|c?)lang")
		set(CLANG 1)
		set(KYTY_COMPILER CLANG)
		set(KYTY_LINKER LLD)
	else()
		set(KYTY_COMPILER MINGW)
		set(KYTY_LINKER LD)
	endif()	
else()
	if(CMAKE_CXX_COMPILER_ID MATCHES "(C|c?)lang")
		set(CLANG 1)
		set(KYTY_COMPILER CLANG)
		set(KYTY_LINKER LLD_LINK)
	else()
		set(KYTY_COMPILER MSVC)
		set(KYTY_LINKER LINK)
	endif()
endif()

set(KYTY_PROJECT_NAME "Emulator" CACHE STRING "Project name")

string(TOUPPER ${KYTY_PROJECT_NAME} KYTY_PROJECT)
string(TOLOWER ${KYTY_COMPILER} KYTY_COMPILER_ID)
string(TOLOWER ${KYTY_LINKER} KYTY_LINKER_ID)

if (CLANG AND (KYTY_LINKER STREQUAL LLD))
	set(KYTY_LD_OPTIONS "-fuse-ld=lld")
endif()

if (KYTY_LINKER STREQUAL LD)
	set(KYTY_LD_OPTIONS "-Wl,--image-base=0x100000000000")
endif()

project(Kyty${KYTY_PROJECT_NAME}${CMAKE_BUILD_TYPE}${KYTY_COMPILER} VERSION 0.0.2)

include(src_script.cmake)

include_directories(	
	3rdparty/lua/include
	3rdparty/rijndael/source
	3rdparty/miniz
	3rdparty/magic_enum/include
	${PROJECT_BINARY_DIR}
	include
    )

set(KYTY_VERSION "${PROJECT_VERSION}")

configure_file(
  ${PROJECT_SOURCE_DIR}/cmake_config.h.in
  ${PROJECT_BINARY_DIR}/cmake_config.h
  )
  
add_custom_target( KytyGitVersion
    COMMAND ${CMAKE_COMMAND} 
	-D INPUT_FILE=${CMAKE_CURRENT_SOURCE_DIR}/KytyGitVersion.h.in
    -D OUTPUT_FILE=${CMAKE_CURRENT_BINARY_DIR}/KytyGitVersion.h
	-P ${CMAKE_CURRENT_SOURCE_DIR}/generate_version.cmake
	COMMENT "Generate KytyGitVersion.h"
)

list(APPEND KYTY_IWYU 
	emulator_obj
	#core_obj
	#math_obj
	#scripts_obj
	#sys_obj
)

list(APPEND KYTY_CLANG_TYDY 
	emulator_obj
	#core_obj
	#core
	#math_obj
	#scripts_obj
	#sys_obj
)

set(KYTY_LIB_HEADERS ${PROJECT_SOURCE_DIR}/include)
  
config_compiler_and_linker()  
  
add_subdirectory(3rdparty)
add_subdirectory(emulator)
add_subdirectory(lib)

add_executable(fc_script ${KYTY_SCRIPT_SRC}) 

target_link_libraries(fc_script core)
target_link_libraries(fc_script sys)
target_link_libraries(fc_script math)
target_link_libraries(fc_script scripts)
target_link_libraries(fc_script lua)
target_link_libraries(fc_script opengl32)
target_link_libraries(fc_script iphlpapi)
target_link_libraries(fc_script sdl2)
target_link_libraries(fc_script setupapi)
target_link_libraries(fc_script emulator)
target_link_libraries(fc_script rijndael)
target_link_libraries(fc_script lzma)
target_link_libraries(fc_script sqlite)
target_link_libraries(fc_script zstd)
target_link_libraries(fc_script easy_profiler)
target_link_libraries(fc_script ws2_32)
target_link_libraries(fc_script psapi)
if (CLANG AND NOT MSVC)
target_link_libraries(fc_script pthread)
endif()
if (CLANG AND MSVC)
target_link_libraries(fc_script winpthread)
endif()

set(KYTY_SCRIPT_BIN "../../__bin/fc_script")

if(MSVC)
	set_target_properties(fc_script PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$(SolutionDir)$(Configuration)/")
	set_target_properties(fc_script
		PROPERTIES
		LINK_FLAGS "/DYNAMICBASE:NO /MAP:fc_script_msvc_${KYTY_BITNESS}.map"
	)
	add_custom_command(TARGET fc_script POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy_if_different fc_script_msvc_${KYTY_BITNESS}.map $<TARGET_FILE_DIR:fc_script>/fc_script_msvc_${KYTY_LINKER_ID}_${KYTY_BITNESS}.map)
	add_custom_command(TARGET fc_script POST_BUILD
		COMMAND ${KYTY_SCRIPT_BIN} ${PROJECT_SOURCE_DIR}/map_to_csv.lua $<TARGET_FILE_DIR:fc_script>/fc_script msvc ${KYTY_BITNESS} ${KYTY_LINKER_ID})
endif()

if(MINGW)
	set_target_properties(fc_script PROPERTIES LINK_FLAGS "${KYTY_LD_OPTIONS} -Wl,-Map=fc_script_${KYTY_COMPILER_ID}_${KYTY_LINKER_ID}_${KYTY_BITNESS}.map")
	add_custom_command(TARGET fc_script POST_BUILD COMMAND ${KYTY_SCRIPT_BIN} ${PROJECT_SOURCE_DIR}/map_to_csv.lua fc_script ${KYTY_COMPILER_ID} ${KYTY_BITNESS} ${KYTY_LINKER_ID})
endif()

add_dependencies(fc_script KytyGitVersion)  

		
if(MSVC)
    set(CMAKE_USE_RELATIVE_PATHS ON)
	
endif()


