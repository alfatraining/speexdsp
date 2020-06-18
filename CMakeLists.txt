cmake_minimum_required(VERSION 3.16)

# Extract version number from configure script
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac" CONFIG_FILE_CONTENT LIMIT 200)
string(REGEX MATCH "AC_INIT\\(\\[speexdsp\\],\\[([0-9]+)\\.([0-9]+)\\.([0-9]+)\\]" _ ${CONFIG_FILE_CONTENT})

set(SPEEX_VERSION_MAJOR ${CMAKE_MATCH_1})
set(SPEEX_VERSION_MINOR ${CMAKE_MATCH_2})
set(SPEEX_VERSION_PATCH ${CMAKE_MATCH_3})

project(libspeexdsp VERSION ${SPEEX_VERSION_MAJOR}.${SPEEX_VERSION_MINOR}.${SPEEX_VERSION_PATCH} LANGUAGES C)

set(CMAKE_DEBUG_POSTFIX "d")

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

include(CheckIncludeFile)
include(CheckSymbolExists)
include(CheckLibraryExists)
include(CMakeDependentOption)
include(FeatureSummary)
include(TestVarArrays)
include(CheckCCompilerFlag)
include(GNUInstallDirs)
include(TestBigEndian)

test_big_endian(WORDS_BIGENDIAN)

check_include_file(alloca.h HAVE_ALLOCA_H)
check_include_file(dlfcn.h HAVE_DLFCN_H)
check_include_file(getopt.h HAVE_GETOPT_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stdlib.h> header file HAVE_STDLIB_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(sys/audioio.h HAVE_SYS_AUDIOIO_H)
check_include_file(sys/soundcard.h HAVE_SYS_SOUNDCARD_H)
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(unistd.h HAVE_UNISTD_H)

if(HAVE_INTTYPES_H)
	set(INCLUDE_STDINT "#include <inttypes.h>")
else()
	if(HAVE_STDINT_H)
		set(INCLUDE_STDINT "#include <stdint.h>")
	else()
		if(HAVE_SYS_TYPES_H)
			set(INCLUDE_STDINT "#include <sys/types.h>")
		endif()
	endif()
endif()

set(SIZE16 int16_t)
set(USIZE16 uint16_t)
set(SIZE32 int32_t)
set(USIZE32 uint32_t)
set(SIZE64 int64_t)
configure_file(include/speex/speexdsp_config_types.h.in include/speex/speexdsp_config_types.h @ONLY)

test_vararrays(VAR_ARRAYS)
if(NOT VAR_ARRAYS)
	unset(VAR_ARRAYS)
	if(WIN32)
		set(USE_ALLOCA 1)
	else()
		check_symbol_exists(alloca "stdlib.h;alloca.h"  USE_ALLOCA)
	endif()
endif()

option(ENABLE_FLOATING_POINT "Compile as floating-point" ON)
option(ENABLE_FIXED_POINT "Compile as fixed-point" OFF)


if(ENABLE_FLOATING_POINT AND ENABLE_FIXED_POINT)
	message(FATAL_ERROR "Select one of ENABLE_FLOATING_POINT of ENABLE_FIXED_POINT.")
endif()
if((NOT ENABLE_FLOATING_POINT) AND (NOT ENABLE_FIXED_POINT))
	message(FATAL_ERROR "Select one of ENABLE_FLOATING_POINT of ENABLE_FIXED_POINT.")
endif()

if(ENABLE_FLOATING_POINT)
	set(FLOATING_POINT 1)
	set(FIXED_POINT 0)
	set(USE_KISS_FFT 0)
	set(USE_SMALLFT 1)
else()
	set(FLOATING_POINT 0)
	set(FIXED_POINT 1)
	set(USE_KISS_FFT 1)
	set(USE_SMALLFT 0)
endif()

cmake_dependent_option(ENABLE_FIXED_POINT_DEBUG "Debug fixed-point implementation" OFF "ENABLE_FIXED_POINT" OFF)
set(FIXED_POINT_DEBUG ${ENABLE_FIXED_POINT_DEBUG})

option(DISABLE_FLOAT_API "Disable all parts of the API that are using floats" OFF)
option(DISABLE_VBR "Disable VBR and VAD from the codec" OFF)
cmake_dependent_option(ENABLE_VORBIS_PSY "Enable the Vorbis psy model" OFF "NOT ENABLE_FIXED_POINT;VAR_ARRAYS" OFF)
set(VORBIS_PSYCHO ${ENABLE_VORBIS_PSY})

if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
    set(CPU_IS_X86 TRUE)
else()
    set(CPU_IS_X86 FALSE)
endif()
if(CPU_IS_X86)
	check_include_file(xmmintrin.h HAVE_XMMINTRIN_H)
endif()
cmake_dependent_option(ENABLE_SSE "Enable SSE support" ON "CPU_IS_X86;HAVE_XMMINTRIN_H;ENABLE_FLOATING_POINT" OFF)
if(ENABLE_SSE)
	set(USE_SSE 1)
	set(USE_SSE2 1)
	check_c_compiler_flag("/arch:SSE" ARCH_SSE_FLAG)
	if(ARCH_SSE_FLAG)
		set(SSE_FLAGS "/arch:SSE")
	else()
		check_c_compiler_flag("-msse" MSSE_FLAG)
		if(MSSE_FLAG)
			set(SSE_FLAGS "-msse")
		endif()
	endif()
endif()

if (CMAKE_SYSTEM_PROCESSOR MATCHES "^arm")
	set(CPU_IS_ARM 1)
	check_include_file(arm_neon.h USE_NEON)
endif ()
cmake_dependent_option(ENABLE_ARM4_ASM "Make use of ARM4 assembly optimizations" OFF "CPU_IS_ARM;ENABLE_FIXED_POINT" OFF)
set(ARM4_ASM ${ENABLE_ARM4_ASM})
cmake_dependent_option(ENABLE_ARM5E_ASM "Make use of ARM5E assembly optimizations" OFF "CPU_IS_ARM;ENABLE_FIXED_POINT" OFF)
set(ARM5E_ASM ${ENABLE_ARM5E_ASM})
cmake_dependent_option(ENABLE_BLACKFIN_ASM "Make use of Blackfin assembly optimizations" OFF "NOT CPU_IS_X86;ENABLE_FIXED_POINT" OFF)
set(BFIN_ASM ${ENABLE_BLACKFIN_ASM})
cmake_dependent_option(ENABLE_TI_C55X "Enable support for TI C55X DSP" OFF "NOT CPU_IS_X86" OFF)
set(TI_C55X ${ENABLE_TI_C55X})

if(NOT WIN32)
	check_library_exists(m cos "" HAVE_LIBM)
	if(HAVE_LIBM)
		#find_library(LIBM m)
		set(LIBM m)
	endif()
endif()

if(MSVC)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_CRT_NONSTDC_NO_DEPRECATE)
	option(MSVC_LINK_STATIC_RUNTIME "Use static linking of msvc runtime" OFF)

	# switch from dynamic to static linking
	if(MSVC_LINK_STATIC_RUNTIME)
		foreach(flag_var
			CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
			CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)

			if(${flag_var} MATCHES "/MD")
				string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
			endif(${flag_var} MATCHES "/MD")
		endforeach(flag_var)
	endif()

endif(MSVC)

check_c_compiler_flag("-fvisibility=hidden" HAS_VISIBILITY)
if(HAS_VISIBILITY)
	SET(EXPORT "__attribute__((visibility(\"default\")))")
endif()

if(HAVE_GETOPT_H)
	check_symbol_exists(getopt_long getopt.h HAVE_GETOPT_LONG)
endif()

option (ENABLE_PACKAGE_CONFIG "Generate and install package config file" ON)

set_package_properties(FFTW3 PROPERTIES TYPE OPTIONAL
	URL "http://www.fftw.org/"
	DESCRIPTION "fast Fourier transform library"
	PURPOSE "Enables use of FFTW3 for fast Fourier transforms")
add_feature_info(ENABLE_ALLOCA USE_ALLOCA "Make use of alloca function.")
add_feature_info(ENABLE_VAR_ARRAYS VAR_ARRAYS "Make use of variable-length arrays.")
add_feature_info(ENABLE_FLOATING_POINT ENABLE_FLOATING_POINT "compile as floating-point.")
add_feature_info(ENABLE_FIXED_POINT ENABLE_FIXED_POINT "compile as fixed-point.")
add_feature_info(ENABLE_FIXED_POINT_DEBUG ENABLE_FIXED_POINT_DEBUG "debug fixed-point implementation.")
add_feature_info(ENABLE_SSE ENABLE_SSE "enable SSE support.")
add_feature_info(ENABLE_ARM4_ASM ENABLE_ARM4_ASM "make use of ARM4 assembly optimizations.")
add_feature_info(ENABLE_ARM5E_ASM ENABLE_ARM5E_ASM "make use of ARM5E assembly optimizations.")
add_feature_info(ENABLE_BLACKFIN_ASM ENABLE_BLACKFIN_ASM "make use of Blackfin assembly optimizations.")
add_feature_info(ENABLE_TI_C55X ENABLE_TI_C55X "enable support for TI C55X DSP.")
add_feature_info(DISABLE_FLOAT_API DISABLE_FLOAT_API "disable all parts of the API that are using floats.")
add_feature_info(DISABLE_VBR DISABLE_VBR "disable VBR and VAD from the codec.")
add_feature_info(ENABLE_VORBIS_PSY ENABLE_VORBIS_PSY "enable the Vorbis psy model.")
add_feature_info(ENABLE_SPEEXDSP USE_SPEEXDSP "enable speexenc preprocessing options.")
add_feature_info(BUILD_BINARIES BUILD_BINARIES "build the encoder and decoder programs, not only the library.")
add_feature_info(ENABLE_PACKAGE_CONFIG ENABLE_PACKAGE_CONFIG "generate and install package config file")
add_feature_info(USE_GPL_FFTW3 USE_GPL_FFTW3 "Use FFTW3 library for fast Fourier transforms")
feature_summary(WHAT ALL)

configure_file(config.h.cmake config.h)

set(speex_PUBLIC_HEADERS
	include/speex/speex_buffer.h
	include/speex/speexdsp_types.h
	include/speex/speex_echo.h
	include/speex/speex_jitter.h
	include/speex/speex_preprocess.h
	include/speex/speex_resampler.h
	${CMAKE_CURRENT_BINARY_DIR}/include/speex/speexdsp_config_types.h)

set(speex_SOURCES
	libspeexdsp/arch.h
	libspeexdsp/filterbank.c
	libspeexdsp/fixed_arm5e.h
	libspeexdsp/fixed_generic.h
	libspeexdsp/mdf.c
	libspeexdsp/preprocess.c
	libspeexdsp/resample_neon.h
	libspeexdsp/bfin.h
    libspeexdsp/fftwrap.c
	libspeexdsp/filterbank.h
	libspeexdsp/fixed_bfin.h
	libspeexdsp/jitter.c
	libspeexdsp/misc_bfin.h
	libspeexdsp/pseudofloat.h
	libspeexdsp/resample_sse.h
	libspeexdsp/smallft.h
	libspeexdsp/buffer.c
	libspeexdsp/fftwrap.h
	libspeexdsp/fixed_arm4.h
	libspeexdsp/fixed_debug.h
	libspeexdsp/math_approx.h
	libspeexdsp/os_support.h
	libspeexdsp/resample.c
	libspeexdsp/scal.c
	libspeexdsp/stack_alloc.h)

add_library(libspeexdsp ${speex_PUBLIC_HEADERS} ${speex_SOURCES})
target_compile_definitions(libspeexdsp
	PRIVATE -DHAVE_CONFIG_H)
target_include_directories(libspeexdsp
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
	PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include/speex>
	PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
if(HAVE_LIBM)
	if(BUILD_SHARED_LIBS)
		target_link_libraries(libspeexdsp PRIVATE ${LIBM})
	else()
		target_link_libraries(libspeexdsp PUBLIC ${LIBM})
	endif()
endif()
target_compile_options(libspeexdsp PRIVATE ${SSE_FLAGS})
if(WIN32 AND BUILD_SHARED_LIBS)
	target_sources(libspeexdsp PRIVATE win32/libspeexdsp.def)
	set_target_properties (libspeexdsp PROPERTIES OUTPUT_NAME "libspeexdsp")
endif()
set_target_properties (libspeexdsp PROPERTIES
		PUBLIC_HEADER "${speex_PUBLIC_HEADERS}"
		SOVERSION 1.5.1
		VERSION 1)
if(EXPORT AND BUILD_SHARED_LIBS)
	set_target_properties (libspeexdsp PROPERTIES
			C_VISIBILITY_PRESET hidden)
endif()
if(USE_SMALLFT)
	target_sources(libspeexdsp PRIVATE
		libspeexdsp/smallft.c)
elseif(USE_KISS_FFT)
	target_sources(libspeexdsp PRIVATE
		libspeexdsp/_kiss_fft_guts.h
		libspeexdsp/kiss_fftr.h
		libspeexdsp/kiss_fft.h
		libspeexdsp/kiss_fft.c
		libspeexdsp/kiss_fftr.c)
endif()
if(VORBIS_PSYCHO)
	target_sources(libspeexdsp PRIVATE
		libspeex/vorbis_psy.h)
endif()

# Programs

if(BUILD_BINARIES)

# Tests (no automatic checking supported)

add_executable(testdenoise libspeexdsp/testdenoise.c)
target_include_directories(testdenoise
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(testdenoise
	PRIVATE ${LIBM}
	PUBLIC libspeexdsp)
target_compile_definitions(testdenoise
	PRIVATE -DHAVE_CONFIG_H)

add_executable(testresample2 libspeexdsp/testresample2.c)
target_include_directories(testresample2
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(testresample2
	PRIVATE ${LIBM}
	PUBLIC libspeexdsp)
target_compile_definitions(testresample2
	PRIVATE -DHAVE_CONFIG_H)

add_executable(testecho libspeexdsp/testecho.c)
target_include_directories(testecho
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(testecho
	PRIVATE ${LIBM}
	PUBLIC libspeexdsp)
target_compile_definitions(testecho
	PRIVATE -DHAVE_CONFIG_H)

add_executable(testresample libspeexdsp/testresample.c)
target_include_directories(testresample
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(testresample
	PRIVATE ${LIBM}
	PUBLIC libspeexdsp)
target_compile_definitions(testresample
	PRIVATE -DHAVE_CONFIG_H)

add_executable(testjitter libspeexdsp/testjitter.c)
target_include_directories(testjitter
	PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_link_libraries(testjitter
	PRIVATE ${LIBM}
	PUBLIC libspeexdsp)
target_compile_definitions(testjitter
	PRIVATE -DHAVE_CONFIG_H)

endif()

# Installation

install(TARGETS libspeexdsp
	EXPORT libspeexdsp-targets
	RUNTIME	DESTINATION			${CMAKE_INSTALL_BINDIR}
	ARCHIVE DESTINATION			${CMAKE_INSTALL_LIBDIR}
	LIBRARY DESTINATION			${CMAKE_INSTALL_LIBDIR}
	PUBLIC_HEADER DESTINATION	${CMAKE_INSTALL_INCLUDEDIR}/libspeexdsp)

#install(FILES doc/manual.pdf DESTINATION ${CMAKE_INSTALL_DOCDIR})
#install(FILES src/speexenc.1 src/speexdec.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1)
#install(FILES speex.m4 DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/aclocal)

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix $\{prefix\})
set(libdir $\{exec_prefix\}/${CMAKE_INSTALL_LIBDIR})
set(includedir $\{prefix\}/${CMAKE_INSTALL_INCLUDEDIR})
set(SPEEX_VERSION ${PROJECT_VERSION})
if(HAVE_LIBM)
	set(LIBM -lm)
endif()
configure_file(speexdsp.pc.in speexdsp.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/speexdsp.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

set(PACKAGE ${PROJECT_NAME})
set(VERSION ${PROJECT_VERSION})
configure_file(SpeexDSP.spec.in SpeexDSP.spec @ONLY)

if (ENABLE_PACKAGE_CONFIG)
	include (CMakePackageConfigHelpers)
	write_basic_package_version_file (libspeexdsp-config-version.cmake COMPATIBILITY SameMajorVersion)
	set(CMAKE_INSTALL_PACKAGEDIR ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})
	install(EXPORT libspeexdsp-targets NAMESPACE SpeexDSP:: DESTINATION ${CMAKE_INSTALL_PACKAGEDIR})
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libspeexdsp-config-version.cmake DESTINATION ${CMAKE_INSTALL_PACKAGEDIR})
	install(FILES cmake/libspeexdsp-config.cmake DESTINATION ${CMAKE_INSTALL_PACKAGEDIR})
endif()

