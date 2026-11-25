# The Flutter tooling requires that developers have CMake 3.10 or later
# installed. You should not increase this version, as doing so will cause
# the plugin to fail to compile for some customers of the plugin.
cmake_minimum_required(VERSION 3.10)
if (CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
  cmake_policy(SET CMP0135 NEW)
endif()

project(recovery_library VERSION 0.0.1 LANGUAGES C)

if(NOT DEFINED SRC_DIR)
  set(SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
endif()

set(COMMON_FILES
    ${SRC_DIR}/crapto1.c
    ${SRC_DIR}/crypto1.c
    ${SRC_DIR}/bucketsort.c
    ${SRC_DIR}/mfkey.c
    ${SRC_DIR}/parity.c
    ${SRC_DIR}/hardnested.c
    ${SRC_DIR}/pm3/commonutil.c
    ${SRC_DIR}/pm3/ui.c
    ${SRC_DIR}/pm3/util_posix.c
    ${SRC_DIR}/pm3/util.c
    ${SRC_DIR}/hardnested/tables.c
    ${SRC_DIR}/hardnested/hardnested_bruteforce.c
    ${SRC_DIR}/hardnested/hardnested_bitarray_core.c
    ${SRC_DIR}/hardnested/hardnested_bf_core.c
)

include_directories(${SRC_DIR}/)
add_subdirectory(minlzlib)

add_library(recovery SHARED
  "recovery.c"
  "crapto1.c"
  "crypto1.c"
  "bucketsort.c"
  "mfkey.c"
  "parity.c"
  "hardnested.c"
  "pm3/commonutil.c"
  "pm3/ui.c"
  "pm3/util_posix.c"
  "pm3/util.c"
  "hardnested/tables.c"
  "hardnested/hardnested_bruteforce.c"
  "hardnested/hardnested_bitarray_core.c"
  "hardnested/hardnested_bf_core.c"
)

# Apply optimization to the recovery target itself, including Debug Flutter
# builds. Avoid fast-math: hardnested uses floating-point work estimates.
if(MSVC)
  target_compile_options(recovery PRIVATE "$<$<NOT:$<CONFIG:Debug>>:/O2>")
else()
  target_compile_options(recovery PRIVATE -O3)
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
    # These internal calls are never interposed by the app. Let GCC specialize
    # rollback/forward operations with constant encryption flags.
    target_compile_options(recovery PRIVATE -fno-semantic-interposition)
  endif()
  if(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR ANDROID)
    target_compile_definitions(recovery PRIVATE _GNU_SOURCE)
  endif()
endif()

set_target_properties(recovery PROPERTIES
  C_STANDARD 11
  C_STANDARD_REQUIRED YES
  PUBLIC_HEADER recovery.h
  OUTPUT_NAME "recovery"
)

target_compile_definitions(recovery PUBLIC DART_SHARED_LIB)

if (WIN32)
  # Workaround Flutter bug for Windows
  set_target_properties(recovery PROPERTIES
  C_STANDARD 11
  C_STANDARD_REQUIRED YES
    WINDOWS_EXPORT_ALL_SYMBOLS ON
  )

  add_custom_command(
    TARGET recovery POST_BUILD
    COMMAND ../../../src/post_build.bat $<CONFIG> $<TARGET_FILE_DIR:recovery>
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
  )

  # NOTE: We intentionally do NOT use find_package()'s OVERRIDE_FIND_PACKAGE
  # redirect here. That mechanism is version-sensitive across CMake releases
  # (it has been observed to silently fail to redirect on some CMake 4.x
  # installs, falling through to a normal find_package() search that then
  # errors with "Could not find a package configuration file ..." since
  # pthreads4w is never actually installed on the system). Populating the
  # dependency directly with FetchContent_MakeAvailable() and relying on the
  # pthreads4w::pthreadVC3 ALIAS target that pthread-win32's own
  # CMakeLists.txt defines is equivalent for our purposes and works
  # consistently from CMake 3.14 through at least CMake 4.x.
  include(FetchContent)
  FetchContent_Declare(
          pthreads4w
          GIT_REPOSITORY "https://github.com/GerHobbelt/pthread-win32"
          EXCLUDE_FROM_ALL
  )

  # pthread-win32's own CMakeLists.txt defines install() rules (it uses
  # GNUInstallDirs/CMakePackageConfigHelpers to build its own install
  # package). Those rules get pulled into our build's global "install"
  # target via add_subdirectory(), and fail there (the flutter tool runs
  # `cmake --install` on Windows as part of `flutter run`/`flutter build`),
  # since pthread-win32 was never meant to be installed as part of this
  # project. CMAKE_SKIP_INSTALL_RULES (CMake >= 3.27) disables install()
  # rule generation for this directory and below, so it only suppresses
  # pthread-win32's rules, not ours.
  if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.27")
    set(CMAKE_SKIP_INSTALL_RULES ON)
  endif()
  FetchContent_MakeAvailable(pthreads4w)
  if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.27")
    set(CMAKE_SKIP_INSTALL_RULES OFF)
  endif()
endif (WIN32)

if(ANDROID)
  list(APPEND XZ_CMAKE_ARGS
    -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
    -DANDROID_ABI=${ANDROID_ABI}
    -DANDROID_PLATFORM=${ANDROID_PLATFORM}
    -DANDROID_NDK=${ANDROID_NDK}
    -DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}
    -DCMAKE_SYSTEM_VERSION=${CMAKE_SYSTEM_VERSION}
  )
elseif(UNIX AND NOT APPLE)
  list(APPEND XZ_CMAKE_ARGS
    -DCMAKE_C_FLAGS="-fPIC"
    -DCMAKE_POSITION_INDEPENDENT_CODE=ON
  )
endif()

if (WIN32)
  target_link_libraries(recovery PUBLIC pthreads4w::pthreadVC3 minlzlib)
else ()
  find_package(Threads REQUIRED)
  target_link_libraries(recovery PUBLIC minlzlib Threads::Threads)
  if(NOT APPLE)
    target_link_libraries(recovery PRIVATE m)
  endif()
endif(WIN32)
