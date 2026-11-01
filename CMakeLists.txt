# sfiii-cps3-port: Street Fighter III New Generation (games/sfn, the decomp) on PC and 3DS
#
#   PC (MSYS2 mingw32):
#     cmake -G Ninja -B build -DCMAKE_C_COMPILER=gcc
#     ninja -C build
#
#   3DS (devkitARM):
#     cmake -G Ninja -B build-3ds -DCMAKE_TOOLCHAIN_FILE=port/cmake/3ds-native.cmake
#     ninja -C build-3ds
cmake_minimum_required(VERSION 3.20)
project(sfiii_cps3_port C)

set(GAME ${CMAKE_SOURCE_DIR}/games/sfn)
set(PORT ${CMAKE_SOURCE_DIR}/port)

if(NOT CMAKE_SIZEOF_VOID_P EQUAL 4)
    message(FATAL_ERROR "the game stores addresses in 32 bits: build for a 32-bit target (MSYS2 mingw32)")
endif()

find_package(Python3 COMPONENTS Interpreter REQUIRED)

# the game's data files, laid out the way the original compiler laid them out (port/tools/host_data.py)
file(GLOB GAME_DATA ${GAME}/data/*.c)
set(HOST_DATA_DIR ${CMAKE_BINARY_DIR}/host_data)
set(HOST_DATA)
foreach(f ${GAME_DATA})
    get_filename_component(n ${f} NAME)
    list(APPEND HOST_DATA ${HOST_DATA_DIR}/${n})
endforeach()
add_custom_command(OUTPUT ${HOST_DATA}
    COMMAND Python3::Interpreter ${PORT}/tools/host_data.py ${HOST_DATA_DIR} ${GAME_DATA}
    DEPENDS ${GAME_DATA} ${PORT}/tools/host_data.py
    COMMENT "Laying out the game data for the host")

file(GLOB GAME_SOURCES ${GAME}/src/*.c)
# 19 routines of the sound driver are built from assembler upstream (src/*_x1.src, *_x2.src, sound_voice_2.src);
# their reconstructed C is in src/nonmatching, under NONMATCHING
set(GAME_NONMATCHING ${GAME}/src/nonmatching/textsound_2.c ${GAME}/src/nonmatching/sound_voice.c
    ${GAME}/src/nonmatching/sound_voice_2.c)
set_source_files_properties(${GAME_NONMATCHING} PROPERTIES COMPILE_DEFINITIONS NONMATCHING=1)
list(APPEND GAME_SOURCES ${GAME_NONMATCHING})
add_library(sfn_game STATIC ${GAME_SOURCES} ${HOST_DATA})
target_include_directories(sfn_game PUBLIC ${GAME}/include ${PORT}/include)
target_compile_definitions(sfn_game PUBLIC HOST_PORT=1 HOST_LITTLE_ENDIAN=1)
# compile the game code the way the arcade compiler treated it: K&R, signed char, wrapping, type punning
target_compile_options(sfn_game PRIVATE
    -std=gnu17 -fpermissive -fwrapv -fno-strict-aliasing -fsigned-char -w
    -include ${PORT}/include/port_prefix.h
    # keep data in definition order (checked after linking by port/tools/check_layout.py)
    -fno-toplevel-reorder)
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86|i686|AMD64|i386")
    target_compile_options(sfn_game PRIVATE -malign-data=abi)
endif()
# Optimised (-O2), except the files with functions that can return without a value: those stay at -O0, so the
# value left in the return register is the one the earlier builds gave (checked frame by frame and by sound).
target_compile_options(sfn_game PRIVATE -O2)
set(GAME_O0_FILES
    CHARMOVE.c Ck_Pass.c Com_Pl.c Com_Sub.c EFF10.c EFF16.c EFF18.c EFF26.c EFF27.c EFF41.c EFF67.c EFF71.c EFF75.c
    EFF76.c EFFB1.c EFFC1.c EFFC4.c EFFC6.c EFFD0.c EFFD1.c EFFD5.c EFFD6.c EFFD7.c EFFE2.c EFFE4.c EFFECT.c
    EFFF2_code.c EFFG5.c PLS02.c PLS03.c VITAL.c aboutspr.c bg000.c cmd_main_2.c continue.c eeprom.c eff05.c eff06.c
    eff65.c eff87.c eff88.c eff94.c effd2.c effe9.c n_input.c sc_sub_2.c sc_trans.c scrn_ctrl.c se_3.c
    nonmatching/textsound_2.c)
foreach(f ${GAME_O0_FILES})
    set_source_files_properties(${GAME}/src/${f} PROPERTIES COMPILE_OPTIONS -O0)
endforeach()

# the board: memory, kernel, video, sound, EEPROM
set(CPS3_SOURCES
    ${PORT}/cps3/cps3_hw.c
    ${PORT}/cps3/cps3_kernel.c
    ${PORT}/cps3/cps3_video.c
    ${PORT}/cps3/cps3_video_dma.c
    ${PORT}/cps3/cps3_eeprom.c
    ${PORT}/cps3/cps3_sound.c
    ${PORT}/cps3/simmram.c
    ${PORT}/cps3/sh2_sub.c)

if(NINTENDO_3DS)
    add_executable(sfn
        ${CPS3_SOURCES}
        ${PORT}/sys/coro_arm.c
        ${PORT}/sys/divzero_arm.c
        ${PORT}/ctr/main_3ds.c
        ${PORT}/ctr/ctr_video.c
        ${PORT}/ctr/ctr_prefetch.c
        ${PORT}/ctr/ctr_sound.c
        ${PORT}/ctr/ctr_controls.c
        ${PORT}/cps3/cps3_romset.c
        ${PORT}/ctr/ctr_depth.c)
    target_link_libraries(sfn PRIVATE sfn_game citro3d ctru z m)
    # ctr_depth.c reads the game's state through the game's headers
    set_source_files_properties(${PORT}/ctr/ctr_depth.c PROPERTIES
        COMPILE_OPTIONS "-include;${PORT}/include/port_prefix.h;-Wno-error;-w")
    # division by zero returns 0, as on the arcade (divzero_arm.c)
    foreach(fn __aeabi_idiv __aeabi_uidiv __aeabi_idivmod __aeabi_uidivmod)
        target_link_options(sfn PRIVATE "-Wl,--wrap=${fn}")
    endforeach()
    set(SYM_PREFIX "")
else()
    find_package(SDL2 REQUIRED)
    add_executable(sfn WIN32
        ${CPS3_SOURCES}
        ${PORT}/cps3/cps3_romset.c
        ${PORT}/sys/coro_win32.c
        ${PORT}/sys/divzero_win32.c
        ${PORT}/sdl/main_sdl.c)
    # zlib linked in, so the game needs only SDL2.dll
    target_link_libraries(sfn PRIVATE sfn_game SDL2::SDL2main SDL2::SDL2 -l:libz.a)
    set(SYM_PREFIX "_")     # i686 C symbols have a leading underscore
endif()
target_include_directories(sfn PRIVATE ${PORT}/sys)
target_compile_options(sfn PRIVATE -O2 -std=gnu17 -fwrapv -fno-strict-aliasing -fsigned-char -Wall -Wno-unused-variable)

add_custom_command(TARGET sfn POST_BUILD
    COMMAND Python3::Interpreter ${PORT}/tools/check_layout.py $<TARGET_FILE:sfn> ${CMAKE_NM}
    COMMENT "Checking the game's data layout")

if(WIN32)
    # a fixed load address, so runs are repeatable
    target_link_options(sfn PRIVATE "-Wl,--disable-dynamicbase")
endif()

if(NINTENDO_3DS)
    ctr_create_3dsx(sfn "SF III New Generation" "CPS3 port" ${PORT}/ctr/cia/icon.png)
    ctr_create_cia(sfn "SF III New Generation" "Street Fighter III New Generation (CPS3 port)" ${PORT}/ctr/cia)

    # PICA200 shaders (vertex, then geometry), assembled and embedded as a C header (<name>_shbin)
    function(ctr_shader target pica)
        get_filename_component(name ${pica} NAME_WE)
        set(shbin ${CMAKE_CURRENT_BINARY_DIR}/${name}.shbin)
        set(header ${CMAKE_CURRENT_BINARY_DIR}/shaders/${name}_shbin.h)
        add_custom_command(OUTPUT ${header}
            COMMAND ${CTR_PICASSO} -o ${shbin} ${pica} ${ARGN}
            COMMAND Python3::Interpreter ${PORT}/tools/bin2c.py ${shbin} ${header} ${name}_shbin
            DEPENDS ${pica} ${ARGN} ${PORT}/tools/bin2c.py
            COMMENT "Assembling ${name} shaders")
        target_sources(${target} PRIVATE ${header})
        target_include_directories(${target} PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/shaders)
    endfunction()

    ctr_shader(sfn ${PORT}/ctr/ctr_video.v.pica ${PORT}/ctr/ctr_video.g.pica)
endif()
