cmake_minimum_required(VERSION 3.15)
project(pspg VERSION 5.8.12 LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(WANT_CURSES ncursesw CACHE STRING "Which curses library to use (ncursesw or pdcurses)")
set_property(CACHE WANT_CURSES PROPERTY STRINGS ncursesw pdcurses)

set(WANT_SHLIB ALL CACHE STRING "Whether to link against dynamic libraries (NO, PQ, ALL)")

include(CheckIncludeFile)
# no sizeable benefits
# include(CheckIPOSupported)
# check_ipo_supported(RESULT lto_supported)

if("$ENV{MSYSTEM}" MATCHES "^CLANG" AND "$ENV{CC}" STREQUAL "")
    set(CMAKE_C_COMPILER "clang" CACHE STRING "" FORCE)
endif()
if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wno-deprecated-declarations)
endif()

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    # PDCurses
    find_package(unofficial-pdcurses CONFIG REQUIRED)
    set(CURSES_LIBRARIES unofficial::pdcurses::pdcurses)
    add_definitions(-DPDCURSES -DPDC_WIDE -DPDC_FORCE_UTF8 -DHAVE_CURSES_H)

    # getopt
    find_package(getopt CONFIG REQUIRED)
    if(TARGET getopt::getopt_shared)
        set(GETOPT_LIBRARIES getopt::getopt_shared)
    else()
        set(GETOPT_LIBRARIES getopt::getopt_static)
    endif()

    if(NOT "vcpkg-pg" IN_LIST VCPKG_MANIFEST_FEATURES)
        message(STATUS "vcpkg-pg feature not requested. Using local system PostgreSQL.")
        # Extra location to check besides $env{ProgramFiles}/NN.M
        set(PostgreSQL_ROOT "C:/PostgreSQL/18.1") 
    endif()

    # readline
    find_package(unofficial-readline-win32 CONFIG REQUIRED)
else(MSVC)
    set(PREFIX "$ENV{MINGW_PREFIX}")
    if(NOT PREFIX)
        set(PREFIX "/usr")
    endif()
    find_package(PkgConfig REQUIRED)
    find_library(CURSES_LIBRARIES NAMES "${WANT_CURSES}_wincon" "${WANT_CURSES}" REQUIRED)
	check_include_file("${WANT_CURSES}/curses.h" HAVE_CURSES_H)
	# TODO configure_file(config.h.in config.h) with #cmakedefine HAVE_CURSES_H
    add_definitions(-DHAVE_NCURSESW -DHAVE_CURSES_H=${HAVE_CURSES_H})
    include_directories(SYSTEM "${PREFIX}/include/${WANT_CURSES}")
	if(WANT_CURSES STREQUAL "pdcurses")
        add_definitions(-DPDCURSES -DPDC_WIDE -DPDC_FORCE_UTF8)
    else()
        check_include_file("ncursesw/panel.h" HAVE_NCURSESW_PANEL_H)
        add_definitions(-DHAVE_NCURSESW_PANEL_H=${HAVE_NCURSESW_PANEL_H})
        list(APPEND CURSES_LIBRARIES -lpanelw)
    endif()
    pkg_search_module(libpq IMPORTED_TARGET libpq REQUIRED)
    pkg_search_module(readline IMPORTED_TARGET readline)
endif(MSVC)

# PostgreSQL
find_package(PostgreSQL REQUIRED)

# Source files
set(PSPG_SOURCES
    src/args.c
    src/bscommands.c
    src/commands.c
    src/config.c
    src/export.c
    src/infra.c
    src/inputs.c
    src/linebuffer.c
    src/pgclient.c
    src/pretty-csv.c
    src/print.c
    src/pspg.c
    src/readline.c
    src/sort.c
    src/string.c
    src/table.c
    src/theme_loader.c
    src/themes.c
    src/unicode.c
    $<$<PLATFORM_ID:Windows>:src/platform/windows.c>
    src/st_menu.c
    src/st_menu_styles.c
    src/menu.c
)

add_definitions(-DCOMPILE_MENU)

# Create executable
add_executable(pspg ${PSPG_SOURCES})

target_compile_options(pspg PRIVATE $<IF:$<C_COMPILER_ID:MSVC>,/W3 /we4013,-Wall -Wextra>)

# Include directories
target_include_directories(pspg PRIVATE 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/src/platform
    $<$<PLATFORM_ID:Windows>:${CMAKE_CURRENT_SOURCE_DIR}/src/platform/compat/windows>
    # ${PostgreSQL_INCLUDE_DIRS}
)

# Link libraries
target_link_libraries(pspg PRIVATE
    ${CURSES_LIBRARIES}
    ${GETOPT_LIBRARIES}
    $<$<NOT:$<PLATFORM_ID:Windows>>:m>
    $<$<PLATFORM_ID:Windows>:ws2_32>
    $<$<AND:$<PLATFORM_ID:Windows>,$<STREQUAL:${WANT_CURSES},pdcurses>>:Winmm>
    $<$<AND:$<PLATFORM_ID:Windows>,$<STREQUAL:${CMAKE_C_COMPILER_FRONTEND_VARIANT},GNU>>:winpthread>
)

if(TARGET unofficial::readline-win32::readline)
    target_compile_definitions(pspg PRIVATE HAVE_LIBREADLINE HAVE_READLINE_READLINE_H HAVE_READLINE_HISTORY HAVE_READLINE_HISTORY_H)
    target_link_libraries(pspg PRIVATE unofficial::readline-win32::readline)
endif()

if(readline_FOUND)
    target_compile_definitions(pspg PRIVATE HAVE_LIBREADLINE HAVE_READLINE_READLINE_H HAVE_READLINE_HISTORY HAVE_READLINE_HISTORY_H)
    target_link_libraries(pspg PRIVATE PkgConfig::readline)
endif()


if(PostgreSQL_FOUND)
    target_compile_definitions(pspg PRIVATE HAVE_POSTGRESQL)
endif()


# mostly static stuff for non-MSVC toolchains
if(CMAKE_C_COMPILER_FRONTEND_VARIANT STREQUAL "GNU")
    if(NOT WANT_SHLIB STREQUAL "ALL") # either PG-only DLL or everything static
        if(WANT_CURSES STREQUAL "ncursesw")
            target_compile_definitions(pspg PRIVATE NCURSES_STATIC)
        endif()

        if (CMAKE_C_COMPILER_ID MATCHES "Clang")
            if(PKGCONFIG_readline_FOUND)
                file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/termcap_stub.c" 
                    "char *BC = 0; char *PC = 0; char *UP = 0;\n"
                )
                target_sources(pspg PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/termcap_stub.c")
            endif()

            if(lto_supported)
                set_property(TARGET pspg PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
                target_compile_options(pspg PRIVATE "-flto=full")
                target_link_options(pspg PRIVATE "-flto=full")
            endif()
        elseif(PKGCONFIG_readline_FOUND)
            message(STATUS "Using termcap stub to satisfy readline's -ltermcap dependency")
            target_link_libraries(pspg PRIVATE -ltermcap)
        endif()

        target_compile_options(pspg PRIVATE -Os -ffunction-sections -fdata-sections)
        target_link_options(pspg BEFORE PRIVATE -static -s -Wl,--gc-sections)
    endif()

    if(WANT_SHLIB STREQUAL "NO")
        list(FIND libpq_STATIC_LIBRARIES "pgport" pgport_index)
        if(pgport_index GREATER -1)
            # We need to inject -lmingwex before -lpgport to pick getopt related symbols from mingwex
            # instead of pgport, which has its own getopt_long implementation but is missing other symbols.
            # Also https://www.postgresql.org/message-id/2656597.1728582542%40sss.pgh.pa.us
            list(INSERT libpq_STATIC_LIBRARIES ${pgport_index} pgcommon_shlib mingwex)
        endif()
        list(APPEND libpq_STATIC_LIBRARIES iconv)
        target_link_libraries(pspg PRIVATE ${libpq_STATIC_LIBRARIES})
    else()
        target_link_libraries(pspg PRIVATE PkgConfig::libpq)
    endif()
elseif(MSVC)
    if(lto_supported AND CMAKE_C_FLAGS MATCHES "MSVC_STATIC_POSTGRESQL")
        set_property(TARGET pspg PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
        # target_compile_options(pspg PRIVATE "/GL")
        # target_link_options(pspg PRIVATE "/LTCG")
    endif()

    if(PostgreSQL_FOUND)
        target_link_libraries(pspg PRIVATE PostgreSQL::PostgreSQL)
    endif()
endif(CMAKE_C_COMPILER_FRONTEND_VARIANT STREQUAL "GNU")

# Install
install(TARGETS pspg DESTINATION bin)
