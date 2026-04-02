cmake_minimum_required(VERSION 3.20)

project(dav1d VERSION 1.5.3 LANGUAGES C)
set(DAV1D_ENABLE_NASM OFF)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

include(CheckCSourceCompiles)
include(CheckCCompilerFlag)
include(CheckIncludeFile)
include(CheckFunctionExists)
include(CheckSymbolExists)
include(CMakePushCheckState)
include(GNUInstallDirs)

set(DAV1D_API_VERSION_MAJOR 7)
set(DAV1D_API_VERSION_MINOR 0)
set(DAV1D_API_VERSION_PATCH 0)

option(DAV1D_ENABLE_ASM "Build asm files, if available" ON)
option(DAV1D_LOGGING "Enable logging" ON)
set(DAV1D_BITDEPTHS "8;16" CACHE STRING "Bitdepths to enable (8, 16)")
set(DAV1D_TRIM_DSP "if-release" CACHE STRING "Trim DSP functions (true, false, if-release)")

set(CONFIG_8BPC 0)
set(CONFIG_16BPC 0)
foreach(bd IN LISTS DAV1D_BITDEPTHS)
    if(bd STREQUAL "8")
        set(CONFIG_8BPC 1)
    elseif(bd STREQUAL "16")
        set(CONFIG_16BPC 1)
    endif()
endforeach()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64|ARM64|ARM64EC)")
    set(ARCH_ARM 0)
    set(ARCH_AARCH64 1)
    set(ARCH_X86 0)
    set(ARCH_X86_64 0)
    set(ARCH_X86_32 0)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm|ARM)")
    set(ARCH_ARM 1)
    set(ARCH_AARCH64 0)
    set(ARCH_X86 0)
    set(ARCH_X86_64 0)
    set(ARCH_X86_32 0)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64|amd64)")
    set(ARCH_ARM 0)
    set(ARCH_AARCH64 0)
    set(ARCH_X86 1)
    set(ARCH_X86_64 1)
    set(ARCH_X86_32 0)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(i[3-6]86|x86)")
    set(ARCH_ARM 0)
    set(ARCH_AARCH64 0)
    set(ARCH_X86 1)
    set(ARCH_X86_64 0)
    set(ARCH_X86_32 1)
else()
    set(ARCH_ARM 0)
    set(ARCH_AARCH64 0)
    set(ARCH_X86 0)
    set(ARCH_X86_64 0)
    set(ARCH_X86_32 0)
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "ppc64le")
    set(ARCH_PPC64LE 1)
else()
    set(ARCH_PPC64LE 0)
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^riscv64")
    set(ARCH_RISCV 1)
    set(ARCH_RV64 1)
    set(ARCH_RV32 0)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^riscv32")
    set(ARCH_RISCV 1)
    set(ARCH_RV64 0)
    set(ARCH_RV32 1)
else()
    set(ARCH_RISCV 0)
    set(ARCH_RV64 0)
    set(ARCH_RV32 0)
endif()

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^loongarch64")
    set(ARCH_LOONGARCH 1)
    set(ARCH_LOONGARCH64 1)
    set(ARCH_LOONGARCH32 0)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^loongarch32")
    set(ARCH_LOONGARCH 1)
    set(ARCH_LOONGARCH64 0)
    set(ARCH_LOONGARCH32 1)
else()
    set(ARCH_LOONGARCH 0)
    set(ARCH_LOONGARCH64 0)
    set(ARCH_LOONGARCH32 0)
endif()

set(HAVE_ASM 0)
if(DAV1D_ENABLE_ASM)
    if(ARCH_AARCH64 OR ARCH_ARM OR ARCH_PPC64LE OR ARCH_RISCV OR ARCH_LOONGARCH OR ARCH_X86)
        if(ARCH_X86_64)
            check_c_source_compiles("int main() { return sizeof(void*); }" _not_ilp32)
            if(_not_ilp32)
                set(HAVE_ASM 1)
            endif()
        else()
            set(HAVE_ASM 1)
        endif()
    endif()
endif()

set(TRIM_DSP_FUNCTIONS 0)
if(DAV1D_TRIM_DSP STREQUAL "true")
    set(TRIM_DSP_FUNCTIONS 1)
elseif(DAV1D_TRIM_DSP STREQUAL "if-release")
    string(TOUPPER "${CMAKE_BUILD_TYPE}" _build_type_upper)
    if(_build_type_upper STREQUAL "RELEASE" OR _build_type_upper STREQUAL "RELWITHDEBINFO" OR _build_type_upper STREQUAL "MINSIZEREL")
        set(TRIM_DSP_FUNCTIONS 1)
    endif()
endif()

set(CONFIG_LOG 0)
if(DAV1D_LOGGING)
    set(CONFIG_LOG 1)
endif()

set(ENDIANNESS_BIG 0)
include(TestBigEndian)
test_big_endian(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
    set(ENDIANNESS_BIG 1)
endif()

set(DAV1D_TEST_ARGS "")

if(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "GNU" OR CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
    add_compile_definitions($<$<COMPILE_LANGUAGE:C>:_GNU_SOURCE>)
    set(DAV1D_TEST_ARGS "-D_GNU_SOURCE")
endif()

set(HAVE_CLOCK_GETTIME 0)
set(HAVE_POSIX_MEMALIGN 0)
set(HAVE_MEMALIGN 0)
set(HAVE_ALIGNED_ALLOC 0)
set(HAVE_DLSYM 0)
set(HAVE_SIGACTION 0)
set(DAV1D_THREAD_LIBS "")
set(DAV1D_DL_LIBS "")
set(DAV1D_RT_LIBS "")

if(WIN32)
    add_compile_definitions(
        $<$<COMPILE_LANGUAGE:C>:_WIN32_WINNT=0x0601>
        $<$<COMPILE_LANGUAGE:C>:UNICODE>
        $<$<COMPILE_LANGUAGE:C>:_UNICODE>
        $<$<COMPILE_LANGUAGE:C>:__USE_MINGW_ANSI_STDIO=1>
        $<$<COMPILE_LANGUAGE:C>:_CRT_DECLARE_NONSTDC_NAMES=1>
    )
    cmake_push_check_state(RESET)
    check_symbol_exists(fseeko "stdio.h" HAVE_FSEEKO)
    cmake_pop_check_state()
    if(HAVE_FSEEKO)
        add_compile_definitions($<$<COMPILE_LANGUAGE:C>:_FILE_OFFSET_BITS=64>)
    else()
        add_compile_definitions($<$<COMPILE_LANGUAGE:C>:fseeko=_fseeki64> $<$<COMPILE_LANGUAGE:C>:ftello=_ftelli64>)
    endif()
else()
    find_package(Threads REQUIRED)
    set(DAV1D_THREAD_LIBS Threads::Threads)

    cmake_push_check_state(RESET)
    set(CMAKE_REQUIRED_DEFINITIONS ${DAV1D_TEST_ARGS})
    check_symbol_exists(clock_gettime "time.h" _has_clock_gettime)
    cmake_pop_check_state()

    if(_has_clock_gettime)
        set(HAVE_CLOCK_GETTIME 1)
    elseif(NOT APPLE)
        find_library(LIBRT rt)
        if(LIBRT)
            cmake_push_check_state(RESET)
            set(CMAKE_REQUIRED_LIBRARIES ${LIBRT})
            check_symbol_exists(clock_gettime "time.h" _has_clock_gettime_rt)
            cmake_pop_check_state()
            if(_has_clock_gettime_rt)
                set(HAVE_CLOCK_GETTIME 1)
                set(DAV1D_RT_LIBS ${LIBRT})
            endif()
        endif()
    endif()

    cmake_push_check_state(RESET)
    set(CMAKE_REQUIRED_DEFINITIONS ${DAV1D_TEST_ARGS})
    check_symbol_exists(sigaction "signal.h" _has_sigaction)
    check_symbol_exists(posix_memalign "stdlib.h" _has_posix_memalign)
    check_symbol_exists(memalign "malloc.h" _has_memalign)
    check_symbol_exists(aligned_alloc "stdlib.h" _has_aligned_alloc)
    cmake_pop_check_state()

    if(_has_sigaction)
        set(HAVE_SIGACTION 1)
    endif()
    if(_has_posix_memalign)
        set(HAVE_POSIX_MEMALIGN 1)
    endif()
    if(_has_memalign)
        set(HAVE_MEMALIGN 1)
    endif()
    if(_has_aligned_alloc)
        set(HAVE_ALIGNED_ALLOC 1)
    endif()

    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        find_library(LIBDL dl)
        if(LIBDL)
            cmake_push_check_state(RESET)
            set(CMAKE_REQUIRED_LIBRARIES ${LIBDL})
            check_symbol_exists(dlsym "dlfcn.h" _has_dlsym)
            cmake_pop_check_state()
            if(_has_dlsym)
                set(HAVE_DLSYM 1)
                set(DAV1D_DL_LIBS ${LIBDL})
            endif()
        endif()
    endif()
endif()

check_include_file("stdatomic.h" HAVE_STDATOMIC_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H_RAW)
check_include_file("unistd.h" HAVE_UNISTD_H_RAW)
check_include_file("io.h" HAVE_IO_H_RAW)
check_include_file("pthread_np.h" HAVE_PTHREAD_NP_H_RAW)

set(HAVE_SYS_TYPES_H 0)
set(HAVE_UNISTD_H 0)
set(HAVE_IO_H 0)
set(HAVE_PTHREAD_NP_H 0)

if(HAVE_SYS_TYPES_H_RAW)
    set(HAVE_SYS_TYPES_H 1)
endif()
if(HAVE_UNISTD_H_RAW)
    set(HAVE_UNISTD_H 1)
endif()
if(HAVE_IO_H_RAW)
    set(HAVE_IO_H 1)
endif()
if(HAVE_PTHREAD_NP_H_RAW)
    set(HAVE_PTHREAD_NP_H 1)
endif()

set(HAVE_GETAUXVAL 0)
set(HAVE_ELF_AUX_INFO 0)
if(ARCH_AARCH64 OR ARCH_ARM OR ARCH_LOONGARCH OR ARCH_PPC64LE OR ARCH_RISCV)
    cmake_push_check_state(RESET)
    set(CMAKE_REQUIRED_DEFINITIONS ${DAV1D_TEST_ARGS})
    check_symbol_exists(getauxval "sys/auxv.h" _has_getauxval)
    check_symbol_exists(elf_aux_info "sys/auxv.h" _has_elf_aux_info)
    cmake_pop_check_state()
    if(_has_getauxval)
        set(HAVE_GETAUXVAL 1)
    endif()
    if(_has_elf_aux_info)
        set(HAVE_ELF_AUX_INFO 1)
    endif()
endif()

set(HAVE_PTHREAD_GETAFFINITY_NP 0)
set(HAVE_PTHREAD_SETAFFINITY_NP 0)
set(HAVE_PTHREAD_SETNAME_NP 0)
set(HAVE_PTHREAD_SET_NAME_NP 0)
if(NOT WIN32 AND TARGET Threads::Threads)
    cmake_push_check_state(RESET)
    set(CMAKE_REQUIRED_DEFINITIONS ${DAV1D_TEST_ARGS} "-DHAVE_PTHREAD_NP_H=${HAVE_PTHREAD_NP_H}")
    set(CMAKE_REQUIRED_LIBRARIES Threads::Threads)
    set(_pthread_prefix "#include <pthread.h>\n#if HAVE_PTHREAD_NP_H\n#include <pthread_np.h>\n#endif\n")

    check_c_source_compiles("${_pthread_prefix}\nint main(){ cpu_set_t s; pthread_getaffinity_np(pthread_self(), sizeof(s), &s); return 0; }" _has_pthread_getaffinity_np)
    check_c_source_compiles("${_pthread_prefix}\nint main(){ cpu_set_t s; pthread_setaffinity_np(pthread_self(), sizeof(s), &s); return 0; }" _has_pthread_setaffinity_np)
    check_c_source_compiles("${_pthread_prefix}\nint main(){ pthread_setname_np(pthread_self(), \"t\"); return 0; }" _has_pthread_setname_np)
    check_c_source_compiles("${_pthread_prefix}\nint main(){ pthread_set_name_np(pthread_self(), \"t\"); return 0; }" _has_pthread_set_name_np)
    cmake_pop_check_state()

    if(_has_pthread_getaffinity_np)
        set(HAVE_PTHREAD_GETAFFINITY_NP 1)
    endif()
    if(_has_pthread_setaffinity_np)
        set(HAVE_PTHREAD_SETAFFINITY_NP 1)
    endif()
    if(_has_pthread_setname_np)
        set(HAVE_PTHREAD_SETNAME_NP 1)
    endif()
    if(_has_pthread_set_name_np)
        set(HAVE_PTHREAD_SET_NAME_NP 1)
    endif()
endif()

set(HAVE_C11_GENERIC 0)
check_c_source_compiles("int main(){ int x = _Generic(0, default: 0); return x; }" _has_c11_generic)
if(_has_c11_generic)
    set(HAVE_C11_GENERIC 1)
endif()

set(PREFIX 0)
if(APPLE OR CMAKE_SYSTEM_NAME STREQUAL "OS/2")
    set(PREFIX 1)
elseif(WIN32 AND ARCH_X86_32)
    set(PREFIX 1)
endif()

set(HAVE_AS_FUNC 0)
set(HAVE_AS_ARCH_DIRECTIVE 0)

set(HAVE_DOTPROD 0)
set(HAVE_I8MM 0)
set(HAVE_SVE 0)
set(HAVE_SVE2 0)
set(HAVE_AS_ARCHEXT_DOTPROD_DIRECTIVE 0)
set(HAVE_AS_ARCHEXT_I8MM_DIRECTIVE 0)
set(HAVE_AS_ARCHEXT_SVE_DIRECTIVE 0)
set(HAVE_AS_ARCHEXT_SVE2_DIRECTIVE 0)

if(HAVE_ASM AND (ARCH_AARCH64 OR ARCH_ARM))
    check_c_source_compiles("__asm__ (\".func meson_test\\n\" \".endfunc\\n\"); int main(){return 0;}" _has_as_func)
    if(_has_as_func)
        set(HAVE_AS_FUNC 1)
    endif()

    if(ARCH_AARCH64)
        check_c_source_compiles("__asm__ (\".arch armv8-a\\n\"); int main(){return 0;}" _has_as_arch)
        if(_has_as_arch)
            set(HAVE_AS_ARCH_DIRECTIVE 1)
        endif()

        check_c_source_compiles("__asm__ (\"udot v0.4s, v0.16b, v0.16b\\n\"); int main(){return 0;}" _has_dotprod)
        if(_has_dotprod)
            set(HAVE_DOTPROD 1)
        endif()
        check_c_source_compiles("__asm__ (\"usdot v0.4s, v0.16b, v0.16b\\n\"); int main(){return 0;}" _has_i8mm)
        if(_has_i8mm)
            set(HAVE_I8MM 1)
        endif()
        check_c_source_compiles("__asm__ (\"whilelt p0.s, x0, x1\\n\"); int main(){return 0;}" _has_sve)
        if(_has_sve)
            set(HAVE_SVE 1)
        endif()
        check_c_source_compiles("__asm__ (\"sqrdmulh z0.s, z0.s, z0.s\\n\"); int main(){return 0;}" _has_sve2)
        if(_has_sve2)
            set(HAVE_SVE2 1)
        endif()

        check_c_source_compiles("__asm__ (\".arch_extension dotprod\\n\"); int main(){return 0;}" _has_archext_dotprod)
        if(_has_archext_dotprod)
            set(HAVE_AS_ARCHEXT_DOTPROD_DIRECTIVE 1)
        endif()
        check_c_source_compiles("__asm__ (\".arch_extension i8mm\\n\"); int main(){return 0;}" _has_archext_i8mm)
        if(_has_archext_i8mm)
            set(HAVE_AS_ARCHEXT_I8MM_DIRECTIVE 1)
        endif()
        check_c_source_compiles("__asm__ (\".arch_extension sve\\n\"); int main(){return 0;}" _has_archext_sve)
        if(_has_archext_sve)
            set(HAVE_AS_ARCHEXT_SVE_DIRECTIVE 1)
        endif()
        check_c_source_compiles("__asm__ (\".arch_extension sve2\\n\"); int main(){return 0;}" _has_archext_sve2)
        if(_has_archext_sve2)
            set(HAVE_AS_ARCHEXT_SVE2_DIRECTIVE 1)
        endif()
    endif()
endif()

check_c_compiler_flag("-fvisibility=hidden" HAS_VISIBILITY_HIDDEN)
if(HAS_VISIBILITY_HIDDEN)
    add_compile_options($<$<COMPILE_LANGUAGE:C>:-fvisibility=hidden>)
endif()

if(NOT MSVC)
    foreach(flag -Wundef -Werror=vla -Wno-maybe-uninitialized -Wno-missing-field-initializers
                 -Wno-unused-parameter -Wstrict-prototypes -Werror=missing-prototypes -Wshorten-64-to-32)
        string(MAKE_C_IDENTIFIER "HAS${flag}" flag_var)
        check_c_compiler_flag("${flag}" ${flag_var})
        if(${flag_var})
            add_compile_options($<$<COMPILE_LANGUAGE:C>:${flag}>)
        endif()
    endforeach()
    if(ARCH_X86_32)
        foreach(flag -msse2 -mfpmath=sse)
            string(MAKE_C_IDENTIFIER "HAS${flag}" flag_var)
            check_c_compiler_flag("${flag}" ${flag_var})
            if(${flag_var})
                add_compile_options($<$<COMPILE_LANGUAGE:C>:${flag}>)
            endif()
        endforeach()
    endif()
else()
    add_compile_options($<$<COMPILE_LANGUAGE:C>:/wd4028> $<$<COMPILE_LANGUAGE:C>:/wd4090> $<$<COMPILE_LANGUAGE:C>:/wd4996> $<$<COMPILE_LANGUAGE:C>:/wd5287>)
endif()

if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug" AND NOT CMAKE_BUILD_TYPE STREQUAL "")
    if(NOT MSVC)
        check_c_compiler_flag("-fomit-frame-pointer" HAS_OMIT_FRAME_POINTER)
        if(HAS_OMIT_FRAME_POINTER)
            add_compile_options($<$<COMPILE_LANGUAGE:C>:-fomit-frame-pointer>)
        endif()
        check_c_compiler_flag("-ffast-math" HAS_FAST_MATH)
        if(HAS_FAST_MATH)
            add_compile_options($<$<COMPILE_LANGUAGE:C>:-ffast-math>)
        endif()
    endif()
endif()

if((ARCH_AARCH64 OR ARCH_ARM) AND NOT MSVC)
    check_c_compiler_flag("-fno-align-functions" HAS_NO_ALIGN_FUNCTIONS)
    if(HAS_NO_ALIGN_FUNCTIONS)
        add_compile_options($<$<COMPILE_LANGUAGE:C>:-fno-align-functions>)
    endif()
endif()

set(STACK_ALIGNMENT 0)
if(ARCH_X86)
    if(ARCH_X86_64 OR CMAKE_SYSTEM_NAME STREQUAL "Linux" OR APPLE)
        set(STACK_ALIGNMENT 16)
    else()
        set(STACK_ALIGNMENT 4)
    endif()
endif()

set(PIC_VALUE "")
if(HAVE_ASM AND (ARCH_AARCH64 OR ARCH_ARM))
    check_c_source_compiles("
#if defined(PIC)
#error PIC already defined
#elif !(defined(__PIC__) || defined(__pic__))
#error no pic
#endif
int main(){return 0;}
" _has_pic)
    if(_has_pic)
        set(PIC_VALUE 3)
    endif()
endif()

# Configure config.h after all HAVE_ASM checks are complete
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
    @ONLY
)

if(HAVE_ASM AND ARCH_X86)
    enable_language(ASM_NASM)
    set(DAV1D_ENABLE_NASM ON)
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.asm.in
        ${CMAKE_CURRENT_BINARY_DIR}/config.asm
        @ONLY
    )
    set(CMAKE_ASM_NASM_FLAGS "${CMAKE_ASM_NASM_FLAGS} -I${CMAKE_CURRENT_SOURCE_DIR}/src/ -I${CMAKE_CURRENT_BINARY_DIR}/")
endif()

# ARM assembly support on Windows with MSVC
set(DAV1D_GASPP_PROCESSED_SOURCES "")
if(HAVE_ASM AND (ARCH_AARCH64 OR ARCH_ARM) AND WIN32 AND CMAKE_C_COMPILER_ID STREQUAL "MSVC")
    find_program(GASPP gas-preprocessor.pl)
    if(GASPP)
        set(DAV1D_USE_GASPP ON)
        set(DAV1D_GASPP_ARCH "${CMAKE_SYSTEM_PROCESSOR}")
        if(ARCH_AARCH64)
            set(DAV1D_ARMASM armasm64)
        else()
            set(DAV1D_ARMASM armasm)
        endif()
    else()
        message(WARNING "gas-preprocessor.pl not found. ARM assembly optimizations will be disabled.")
        set(HAVE_ASM 0)
    endif()
endif()

# For ARM on Windows with non-MSVC compilers (e.g., MinGW, Clang), enable assembly
if(HAVE_ASM AND (ARCH_AARCH64 OR ARCH_ARM) AND WIN32)
    if(NOT CMAKE_C_COMPILER_ID STREQUAL "MSVC")
        # Enable assembly for ARM with GCC/Clang on Windows
        # The .S files will be processed by the C preprocessor and then assembled
        set(DAV1D_USE_GASPP OFF)
    endif()
endif()

find_program(GIT_EXECUTABLE git)
if(GIT_EXECUTABLE)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --long --always
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE DAV1D_VCS_TAG
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
        RESULT_VARIABLE GIT_RESULT
    )
    if(NOT GIT_RESULT EQUAL 0)
        set(DAV1D_VCS_TAG "${PROJECT_VERSION}")
    endif()
else()
    set(DAV1D_VCS_TAG "${PROJECT_VERSION}")
endif()

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/vcs_version.h
    "#define DAV1D_VERSION \"${DAV1D_VCS_TAG}\"\n"
)

set(DAV1D_SOURCES
    src/cdf.c
    src/cpu.c
    src/ctx.c
    src/data.c
    src/decode.c
    src/dequant_tables.c
    src/getbits.c
    src/intra_edge.c
    src/itx_1d.c
    src/lf_mask.c
    src/lib.c
    src/log.c
    src/mem.c
    src/msac.c
    src/obu.c
    src/pal.c
    src/picture.c
    src/qm.c
    src/ref.c
    src/refmvs.c
    src/scan.c
    src/tables.c
    src/thread_task.c
    src/warpmv.c
    src/wedge.c
)

set(DAV1D_TMPL_SOURCES
    src/cdef_apply_tmpl.c
    src/cdef_tmpl.c
    src/fg_apply_tmpl.c
    src/filmgrain_tmpl.c
    src/ipred_prepare_tmpl.c
    src/ipred_tmpl.c
    src/itx_tmpl.c
    src/lf_apply_tmpl.c
    src/loopfilter_tmpl.c
    src/looprestoration_tmpl.c
    src/lr_apply_tmpl.c
    src/mc_tmpl.c
    src/recon_tmpl.c
)

set(DAV1D_ARCH_TMPL_SOURCES_DEFAULT "")
set(DAV1D_ARCH_TMPL_SOURCES_VSX "")
set(DAV1D_ARCH_TMPL_SOURCES_PWR9 "")
set(DAV1D_ASM_SOURCES "")

if(WIN32)
    list(APPEND DAV1D_SOURCES src/win32/thread.c)
endif()

if(HAVE_ASM)
    if(ARCH_AARCH64 OR ARCH_ARM)
        list(APPEND DAV1D_SOURCES src/arm/cpu.c)
        if(ARCH_AARCH64)
            list(APPEND DAV1D_ASM_SOURCES
                src/arm/64/itx.S
                src/arm/64/looprestoration_common.S
                src/arm/64/msac.S
                src/arm/64/refmvs.S
            )
            if(CONFIG_8BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/arm/64/cdef.S
                    src/arm/64/filmgrain.S
                    src/arm/64/ipred.S
                    src/arm/64/loopfilter.S
                    src/arm/64/looprestoration.S
                    src/arm/64/mc.S
                    src/arm/64/mc_dotprod.S
                )
            endif()
            if(CONFIG_16BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/arm/64/cdef16.S
                    src/arm/64/filmgrain16.S
                    src/arm/64/ipred16.S
                    src/arm/64/itx16.S
                    src/arm/64/loopfilter16.S
                    src/arm/64/looprestoration16.S
                    src/arm/64/mc16.S
                    src/arm/64/mc16_sve.S
                )
            endif()
        elseif(ARCH_ARM)
            list(APPEND DAV1D_ASM_SOURCES
                src/arm/32/itx.S
                src/arm/32/looprestoration_common.S
                src/arm/32/msac.S
                src/arm/32/refmvs.S
            )
            if(CONFIG_8BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/arm/32/cdef.S
                    src/arm/32/filmgrain.S
                    src/arm/32/ipred.S
                    src/arm/32/loopfilter.S
                    src/arm/32/looprestoration.S
                    src/arm/32/mc.S
                )
            endif()
            if(CONFIG_16BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/arm/32/cdef16.S
                    src/arm/32/filmgrain16.S
                    src/arm/32/ipred16.S
                    src/arm/32/itx16.S
                    src/arm/32/loopfilter16.S
                    src/arm/32/looprestoration16.S
                    src/arm/32/mc16.S
                )
            endif()
        endif()
    elseif(ARCH_X86)
        list(APPEND DAV1D_SOURCES src/x86/cpu.c)

        list(APPEND DAV1D_ASM_SOURCES
            src/x86/cpuid.asm
            src/x86/msac.asm
            src/x86/pal.asm
            src/x86/refmvs.asm
            src/x86/itx_avx512.asm
            src/x86/cdef_avx2.asm
            src/x86/itx_avx2.asm
            src/x86/cdef_sse.asm
            src/x86/itx_sse.asm
        )

        if(CONFIG_8BPC)
            list(APPEND DAV1D_ASM_SOURCES
                src/x86/cdef_avx512.asm
                src/x86/filmgrain_avx512.asm
                src/x86/ipred_avx512.asm
                src/x86/loopfilter_avx512.asm
                src/x86/looprestoration_avx512.asm
                src/x86/mc_avx512.asm
                src/x86/filmgrain_avx2.asm
                src/x86/ipred_avx2.asm
                src/x86/loopfilter_avx2.asm
                src/x86/looprestoration_avx2.asm
                src/x86/mc_avx2.asm
                src/x86/filmgrain_sse.asm
                src/x86/ipred_sse.asm
                src/x86/loopfilter_sse.asm
                src/x86/looprestoration_sse.asm
                src/x86/mc_sse.asm
            )
        endif()

        if(CONFIG_16BPC)
            list(APPEND DAV1D_ASM_SOURCES
                src/x86/cdef16_avx512.asm
                src/x86/filmgrain16_avx512.asm
                src/x86/ipred16_avx512.asm
                src/x86/itx16_avx512.asm
                src/x86/loopfilter16_avx512.asm
                src/x86/looprestoration16_avx512.asm
                src/x86/mc16_avx512.asm
                src/x86/cdef16_avx2.asm
                src/x86/filmgrain16_avx2.asm
                src/x86/ipred16_avx2.asm
                src/x86/itx16_avx2.asm
                src/x86/loopfilter16_avx2.asm
                src/x86/looprestoration16_avx2.asm
                src/x86/mc16_avx2.asm
                src/x86/cdef16_sse.asm
                src/x86/filmgrain16_sse.asm
                src/x86/ipred16_sse.asm
                src/x86/itx16_sse.asm
                src/x86/loopfilter16_sse.asm
                src/x86/looprestoration16_sse.asm
                src/x86/mc16_sse.asm
            )
        endif()
    elseif(ARCH_LOONGARCH)
        list(APPEND DAV1D_SOURCES src/loongarch/cpu.c)

        set(DAV1D_ARCH_TMPL_SOURCES_DEFAULT src/loongarch/looprestoration_inner.c)

        list(APPEND DAV1D_ASM_SOURCES
            src/loongarch/cdef.S
            src/loongarch/ipred.S
            src/loongarch/mc.S
            src/loongarch/loopfilter.S
            src/loongarch/looprestoration.S
            src/loongarch/msac.S
            src/loongarch/refmvs.S
            src/loongarch/itx.S
        )
    elseif(ARCH_PPC64LE)
        list(APPEND DAV1D_SOURCES src/ppc/cpu.c)

        set(DAV1D_ARCH_TMPL_SOURCES_VSX
            src/ppc/cdef_tmpl.c
            src/ppc/looprestoration_tmpl.c
        )
        set(DAV1D_ARCH_TMPL_SOURCES_PWR9
            src/ppc/itx_tmpl.c
            src/ppc/loopfilter_tmpl.c
            src/ppc/mc_tmpl.c
        )
    elseif(ARCH_RISCV)
        list(APPEND DAV1D_SOURCES src/riscv/cpu.c)
        if(ARCH_RV64)
            list(APPEND DAV1D_ASM_SOURCES
                src/riscv/64/cpu.S
                src/riscv/64/pal.S
            )
            if(CONFIG_8BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/riscv/64/cdef.S
                    src/riscv/64/ipred.S
                    src/riscv/64/itx.S
                    src/riscv/64/mc.S
                )
            endif()
            if(CONFIG_16BPC)
                list(APPEND DAV1D_ASM_SOURCES
                    src/riscv/64/cdef16.S
                    src/riscv/64/ipred16.S
                    src/riscv/64/mc16.S
                )
            endif()
        endif()
    endif()
endif()

set(DAV1D_INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include/dav1d
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}
)

set(DAV1D_STDATOMIC_INCLUDE "")
if(NOT HAVE_STDATOMIC_H)
    if(MSVC)
        set(DAV1D_STDATOMIC_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/include/compat/msvc)
    else()
        check_c_source_compiles("int main() { int v = 0; return __atomic_fetch_add(&v, 1, __ATOMIC_SEQ_CST); }" HAS_GCC_ATOMICS)
        if(HAS_GCC_ATOMICS)
            set(DAV1D_STDATOMIC_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR}/include/compat/gcc)
        else()
            message(FATAL_ERROR "Atomics not supported")
        endif()
    endif()
endif()

set(DAV1D_BITDEPTH_LIBS "")
foreach(bd IN LISTS DAV1D_BITDEPTHS)
    set(bd_target dav1d_bitdepth_${bd})
    add_library(${bd_target} OBJECT ${DAV1D_TMPL_SOURCES})
    target_compile_definitions(${bd_target} PRIVATE BITDEPTH=${bd})
    target_include_directories(${bd_target} PRIVATE ${DAV1D_INCLUDE_DIRS})
    if(DAV1D_STDATOMIC_INCLUDE)
        target_include_directories(${bd_target} PRIVATE ${DAV1D_STDATOMIC_INCLUDE})
    endif()
    list(APPEND DAV1D_BITDEPTH_LIBS $<TARGET_OBJECTS:${bd_target}>)

    if(DAV1D_ARCH_TMPL_SOURCES_DEFAULT)
        set(bd_arch_target dav1d_arch_bitdepth_${bd}_default)
        add_library(${bd_arch_target} OBJECT ${DAV1D_ARCH_TMPL_SOURCES_DEFAULT})
        target_compile_definitions(${bd_arch_target} PRIVATE BITDEPTH=${bd})
        target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_INCLUDE_DIRS})
        if(DAV1D_STDATOMIC_INCLUDE)
            target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_STDATOMIC_INCLUDE})
        endif()
        list(APPEND DAV1D_BITDEPTH_LIBS $<TARGET_OBJECTS:${bd_arch_target}>)
    endif()

    if(DAV1D_ARCH_TMPL_SOURCES_VSX)
        set(bd_arch_target dav1d_arch_bitdepth_${bd}_vsx)
        add_library(${bd_arch_target} OBJECT ${DAV1D_ARCH_TMPL_SOURCES_VSX})
        target_compile_definitions(${bd_arch_target} PRIVATE BITDEPTH=${bd})
        target_compile_options(${bd_arch_target} PRIVATE -maltivec -mvsx)
        target_compile_definitions(${bd_arch_target} PRIVATE DAV1D_VSX)
        target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_INCLUDE_DIRS})
        if(DAV1D_STDATOMIC_INCLUDE)
            target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_STDATOMIC_INCLUDE})
        endif()
        list(APPEND DAV1D_BITDEPTH_LIBS $<TARGET_OBJECTS:${bd_arch_target}>)
    endif()

    if(DAV1D_ARCH_TMPL_SOURCES_PWR9)
        set(bd_arch_target dav1d_arch_bitdepth_${bd}_pwr9)
        add_library(${bd_arch_target} OBJECT ${DAV1D_ARCH_TMPL_SOURCES_PWR9})
        target_compile_definitions(${bd_arch_target} PRIVATE BITDEPTH=${bd})
        target_compile_options(${bd_arch_target} PRIVATE -mcpu=power9)
        target_compile_definitions(${bd_arch_target} PRIVATE DAV1D_PWR9)
        target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_INCLUDE_DIRS})
        if(DAV1D_STDATOMIC_INCLUDE)
            target_include_directories(${bd_arch_target} PRIVATE ${DAV1D_STDATOMIC_INCLUDE})
        endif()
        list(APPEND DAV1D_BITDEPTH_LIBS $<TARGET_OBJECTS:${bd_arch_target}>)
    endif()
endforeach()

# Process ARM assembly files through gas-preprocessor and armasm on Windows/MSVC
if(DAV1D_USE_GASPP AND CMAKE_C_COMPILER_ID STREQUAL "MSVC")
    set(DAV1D_ASM_OBJECTS "")
    foreach(asm_source IN LISTS DAV1D_ASM_SOURCES)
        get_filename_component(source_name ${asm_source} NAME)
        set(output_obj "${CMAKE_CURRENT_BINARY_DIR}/${source_name}.obj")
        
        add_custom_command(
            OUTPUT ${output_obj}
            COMMAND ${GASPP}
                -as-type armasm
                -arch ${DAV1D_GASPP_ARCH}
                --
                ${DAV1D_ARMASM}
                -nologo
                -I${CMAKE_CURRENT_SOURCE_DIR}/src
                -I${CMAKE_CURRENT_BINARY_DIR}
                ${asm_source}
                -c
                -o ${output_obj}
            DEPENDS ${asm_source}
            COMMENT "Processing ARM assembly ${source_name}"
        )
        list(APPEND DAV1D_ASM_OBJECTS ${output_obj})
    endforeach()
    set(DAV1D_FINAL_ASM_SOURCES ${DAV1D_ASM_OBJECTS})
else()
    # For non-MSVC compilers or when gas-preprocessor is not used, use sources directly
    set(DAV1D_FINAL_ASM_SOURCES ${DAV1D_ASM_SOURCES})
endif()

add_library(dav1d
    ${DAV1D_SOURCES}
    ${DAV1D_FINAL_ASM_SOURCES}
    ${DAV1D_BITDEPTH_LIBS}
)

target_include_directories(dav1d
    PRIVATE
        ${DAV1D_INCLUDE_DIRS}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(DAV1D_STDATOMIC_INCLUDE)
    target_include_directories(dav1d PRIVATE ${DAV1D_STDATOMIC_INCLUDE})
endif()

target_link_libraries(dav1d
    PRIVATE
        ${DAV1D_THREAD_LIBS}
        ${DAV1D_DL_LIBS}
        ${DAV1D_RT_LIBS}
)

find_library(LIBM m)
if(LIBM)
    target_link_libraries(dav1d PRIVATE ${LIBM})
endif()

set_target_properties(dav1d PROPERTIES
    VERSION ${DAV1D_API_VERSION_MAJOR}.${DAV1D_API_VERSION_MINOR}.${DAV1D_API_VERSION_PATCH}
    SOVERSION ${DAV1D_API_VERSION_MAJOR}
    C_VISIBILITY_PRESET hidden
    POSITION_INDEPENDENT_CODE ON
)

if(WIN32 AND BUILD_SHARED_LIBS)
    target_compile_definitions(dav1d PRIVATE DAV1D_BUILDING_DLL)
endif()

if(WIN32 AND ARCH_X86_64 AND CMAKE_C_COMPILER_ID STREQUAL "GNU")
    target_compile_options(dav1d PRIVATE $<$<COMPILE_LANGUAGE:C>:-mcmodel=small>)
endif()

# ARM64EC support on Windows
if(WIN32 AND ARCH_AARCH64 AND CMAKE_C_COMPILER_ID STREQUAL "MSVC")
    # ARM64EC uses different calling convention, ensure proper code generation
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "ARM64EC")
        target_compile_options(dav1d PRIVATE /arm64EC)
    endif()
    # Enable NEON intrinsics support
    target_compile_options(dav1d PRIVATE /arch:ARMv8.1)
endif()

add_library(dav1d::dav1d ALIAS dav1d)

install(TARGETS dav1d
    EXPORT dav1dTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(FILES
    include/dav1d/common.h
    include/dav1d/data.h
    include/dav1d/dav1d.h
    include/dav1d/headers.h
    include/dav1d/picture.h
    include/dav1d/version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/dav1d
)

install(EXPORT dav1dTargets
    FILE dav1dTargets.cmake
    NAMESPACE dav1d::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dav1d
)

include(CMakePackageConfigHelpers)

configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/dav1dConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/dav1dConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dav1d
)

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/dav1dConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/dav1dConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/dav1dConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/dav1d
)
