cmake_minimum_required(VERSION 3.21)

project(split_at_the_seam LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# OFF builds only the unit tests: no SFML is needed (used by CI).
option(CATCHER_BUILD_GAMES "Build the Catcher programs (requires SFML 3)" ON)

include(FetchContent)

set(CATCHER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/howto_split)

if(CATCHER_BUILD_GAMES)# ---------------------------------------------------------------------------
# SFML 3: use an installed copy if a usable one is found, otherwise download it.
#
# Searched, in order:
#   -DSFML_DIR=<sfml>/lib/cmake/SFML, -DSFML_ROOT=<sfml> or env SFML_ROOT,
#   -DCMAKE_PREFIX_PATH=<sfml> or env CMAKE_PREFIX_PATH,
#   -DSFML_SDK=<sfml> or env SFML_SDK (an SDK folder, or a folder holding SFML-3* SDKs),
#   C:\SFML-3*, C:\SFML-3*\SFML-3*, C:\SFML\SFML-3* and C:\SFML (Windows, newest first),
#   then CMake's usual places (PATH, Program Files, /usr, /usr/local, Homebrew).
#
# A candidate is only used if a small test program builds and links against it,
# so an SDK for the wrong compiler or architecture is skipped rather than
# breaking the build. Force the download with -DCATCHER_FETCH_SFML=ON.
# ---------------------------------------------------------------------------
option(CATCHER_FETCH_SFML "Always download and build SFML instead of searching for it" OFF)
set(SFML_SDK "" CACHE PATH "SFML 3 SDK folder, or a folder containing SFML-3* SDKs (env SFML_SDK also works)")

set(CATCHER_USE_INSTALLED OFF)

if(NOT CATCHER_FETCH_SFML)
    # Extra places to look, beyond what find_package searches by itself.
    set(_sfml_roots)
    foreach(_root IN ITEMS "${SFML_SDK}" "$ENV{SFML_SDK}")
        if(_root)
            file(TO_CMAKE_PATH "${_root}" _root)
            list(APPEND _sfml_roots "${_root}")
            file(GLOB _sub LIST_DIRECTORIES true "${_root}/SFML-3*")
            list(SORT _sub COMPARE NATURAL ORDER DESCENDING)
            list(APPEND _sfml_roots ${_sub})
        endif()
    endforeach()
    if(WIN32)
        # C:\SFML-3.x, its "Extract All" nested form C:\SFML-3.x\SFML-3.x,
        # C:\SFML\SFML-3.x, and an SDK renamed to plain C:\SFML.
        file(GLOB _sub LIST_DIRECTORIES true
            "C:/SFML-3*" "C:/SFML-3*/SFML-3*" "C:/SFML/SFML-3*")
        list(SORT _sub COMPARE NATURAL ORDER DESCENDING)
        list(APPEND _sfml_roots ${_sub} "C:/SFML")
    endif()
    set(_sfml_hints)
    foreach(_root IN LISTS _sfml_roots)
        if(IS_DIRECTORY "${_root}")
            list(APPEND _sfml_hints "${_root}")
        endif()
    endforeach()
    list(REMOVE_DUPLICATES _sfml_hints)

    # Re-check only when something that affects the result has changed.
    set(_sfml_sig "${_sfml_hints}|${SFML_DIR}|${SFML_ROOT}|$ENV{SFML_ROOT}|${CMAKE_PREFIX_PATH}|$ENV{CMAKE_PREFIX_PATH}|${CMAKE_CXX_COMPILER}|${CMAKE_GENERATOR_PLATFORM}|${SFML_STATIC_LIBRARIES}")

    if(NOT _sfml_sig STREQUAL CATCHER_SFML_CHECKED)
        message(STATUS "Looking for a usable SFML 3")
        string(REPLACE ";" "|" _hints_arg "${_sfml_hints}")
        string(REPLACE ";" "|" _prefix_arg "${CMAKE_PREFIX_PATH}")
        set(_check_flags
            "-DSFML_HINTS=${_hints_arg}"
            "-DSFML_PREFIX=${_prefix_arg}"
            "-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}"
        )
        foreach(_var IN ITEMS SFML_DIR SFML_ROOT SFML_STATIC_LIBRARIES CMAKE_TOOLCHAIN_FILE)
            if(${_var})
                list(APPEND _check_flags "-D${_var}=${${_var}}")
            endif()
        endforeach()

        set(_sfml_dir_file ${CMAKE_BINARY_DIR}/sfml_check/sfml_dir.txt)
        file(REMOVE ${_sfml_dir_file})
        try_compile(_sfml_ok
            ${CMAKE_BINARY_DIR}/sfml_check
            ${CMAKE_CURRENT_SOURCE_DIR}/cmake/sfml_check
            sfml_check
            CMAKE_FLAGS ${_check_flags}
        )

        set(_sfml_found_dir "")
        if(EXISTS ${_sfml_dir_file})
            file(READ ${_sfml_dir_file} _sfml_found_dir)
        endif()

        if(_sfml_ok AND _sfml_found_dir)
            set(SFML_DIR "${_sfml_found_dir}" CACHE PATH "Folder containing SFMLConfig.cmake" FORCE)
            set(CATCHER_SFML_USABLE ON CACHE INTERNAL "")
        else()
            set(CATCHER_SFML_USABLE OFF CACHE INTERNAL "")
            if(_sfml_found_dir)
                message(STATUS "SFML found at ${_sfml_found_dir} but it does not build with this "
                               "compiler/architecture (wrong SDK download?) - ignoring it")
            endif()
        endif()

        # Recompute: SFML_DIR may have just been set.
        set(_sfml_sig "${_sfml_hints}|${SFML_DIR}|${SFML_ROOT}|$ENV{SFML_ROOT}|${CMAKE_PREFIX_PATH}|$ENV{CMAKE_PREFIX_PATH}|${CMAKE_CXX_COMPILER}|${CMAKE_GENERATOR_PLATFORM}|${SFML_STATIC_LIBRARIES}")
        set(CATCHER_SFML_CHECKED "${_sfml_sig}" CACHE INTERNAL "")
    endif()

    if(CATCHER_SFML_USABLE)
        find_package(SFML 3 COMPONENTS Graphics Window System QUIET)
        set(CATCHER_USE_INSTALLED ${SFML_FOUND})
    endif()
endif()

if(CATCHER_USE_INSTALLED)
    message(STATUS "Using installed SFML ${SFML_VERSION} from ${SFML_DIR}")
else()
    set(SFML_FOUND FALSE)
    message(STATUS "No usable installed SFML 3 - fetching SFML 3.0.2")
    set(BUILD_SHARED_LIBS OFF CACHE BOOL "" FORCE)
    set(SFML_BUILD_AUDIO OFF CACHE BOOL "" FORCE)
    set(SFML_BUILD_NETWORK OFF CACHE BOOL "" FORCE)
    set(SFML_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
    set(SFML_BUILD_DOC OFF CACHE BOOL "" FORCE)
    FetchContent_Declare(
        SFML
        GIT_REPOSITORY https://github.com/SFML/SFML.git
        GIT_TAG        3.0.2
        GIT_SHALLOW    ON
    )
    FetchContent_MakeAvailable(SFML)
endif()


# ---------------------------------------------------------------------------
# Catcher: the worked example in howto_split.md, before and after the split
# ---------------------------------------------------------------------------
add_executable(catcher_before ${CATCHER_DIR}/before/catcher.cpp)
add_executable(catcher_after  ${CATCHER_DIR}/after/catcher.cpp ${CATCHER_DIR}/after/CatcherGame.h)
# One complete program per refactoring step (steps 1-6; step 7 is catcher_after)
add_executable(catcher_step1 ${CATCHER_DIR}/steps/step1_state.cpp)
add_executable(catcher_step2 ${CATCHER_DIR}/steps/step2_constants.cpp)
add_executable(catcher_step3 ${CATCHER_DIR}/steps/step3_input.cpp)
add_executable(catcher_step4 ${CATCHER_DIR}/steps/step4_inject.cpp)
add_executable(catcher_step5 ${CATCHER_DIR}/steps/step5_rect.cpp)
add_executable(catcher_step6 ${CATCHER_DIR}/steps/step6_update.cpp)

foreach(catcher catcher_before catcher_step1 catcher_step2 catcher_step3 catcher_step4 catcher_step5 catcher_step6 catcher_after)
    target_link_libraries(${catcher} PRIVATE SFML::Graphics SFML::Window SFML::System)
    set_target_properties(${catcher} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
    if(NOT catcher STREQUAL "catcher_before")
        # Build catcher_before first: it copies the SFML DLLs (below), and copying
        # them from several targets at once makes parallel builds fail.
        add_dependencies(${catcher} catcher_before)
    endif()
endforeach()

# An installed, shared SFML on Windows needs its DLLs beside the exes.
if(WIN32 AND SFML_FOUND AND NOT SFML_STATIC_LIBRARIES)
    add_custom_command(TARGET catcher_before POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_RUNTIME_DLLS:catcher_before> $<TARGET_FILE_DIR:catcher_before>
        COMMAND_EXPAND_LISTS
    )
endif()
endif() # CATCHER_BUILD_GAMES

# ---------------------------------------------------------------------------
# Unit tests (GoogleTest). CatcherTests does not link SFML.
# ---------------------------------------------------------------------------
enable_testing()

set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)   # match MSVC's default /MD runtime
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.15.2
    GIT_SHALLOW    ON
)
FetchContent_MakeAvailable(googletest)

add_executable(CatcherTests ${CATCHER_DIR}/after/CatcherGame_test.cpp)
target_include_directories(CatcherTests PRIVATE ${CATCHER_DIR}/after)
target_link_libraries(CatcherTests PRIVATE GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(CatcherTests TEST_PREFIX "Catcher.")