project(VoxelEngineSrc)

file(GLOB_RECURSE headers ${CMAKE_CURRENT_SOURCE_DIR}/*.hpp)
file(GLOB_RECURSE sources ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
list(REMOVE_ITEM sources ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp)

add_library(VoxelEngineSrc STATIC ${sources} ${headers})

find_package(OpenGL REQUIRED)
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    # Use CONFIG so vcpkg's GLEW is found (GLEW::GLEW); FindGLEW.cmake would need GLEW_INCLUDE_DIRS/GLEW_LIBRARIES
    find_package(GLEW CONFIG REQUIRED)
    find_package(OpenAL CONFIG REQUIRED)
else()
    find_package(GLEW REQUIRED)
endif()
find_package(glm REQUIRED)
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    # OpenAL already found above
else()
    find_package(OpenAL REQUIRED)
endif()
find_package(ZLIB REQUIRED)
find_package(PNG REQUIRED)
find_package(CURL REQUIRED)
find_package(glfw3 REQUIRED)
if(NOT APPLE)
    find_package(EnTT REQUIRED)
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    find_package(vorbis REQUIRED)
    # Prefer manifest-mode path (VCPKG_INSTALLED_DIR); fallback to classic VCPKG_ROOT
    if(DEFINED VCPKG_INSTALLED_DIR)
        set(_luajit_base "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}")
    else()
        set(_luajit_base "$ENV{VCPKG_ROOT}/packages/luajit_${VCPKG_TARGET_TRIPLET}")
    endif()
    if(VCPKG_TARGET_TRIPLET MATCHES "static")
        add_library(luajit STATIC IMPORTED)
        set_target_properties(
            luajit
            PROPERTIES
                IMPORTED_LOCATION
                "${_luajit_base}/lib/libluajit-5.1.a"
                INTERFACE_INCLUDE_DIRECTORIES
                "${_luajit_base}/include;${_luajit_base}/include/luajit"
        )
    else()
        add_library(luajit SHARED IMPORTED)
        set_target_properties(
            luajit
            PROPERTIES
                IMPORTED_LOCATION
                "${_luajit_base}/bin/lua51.dll"
                IMPORTED_IMPLIB
                "${_luajit_base}/lib/lua51.lib"
                INTERFACE_INCLUDE_DIRECTORIES
                "${_luajit_base}/include/luajit"
        )
    endif()

    add_library(luajit::luajit ALIAS luajit)
else()
    find_package(PkgConfig)

    pkg_check_modules(luajit REQUIRED IMPORTED_TARGET luajit)
    pkg_check_modules(vorbis REQUIRED IMPORTED_TARGET vorbis)
    pkg_check_modules(vorbisfile REQUIRED IMPORTED_TARGET vorbisfile)
    add_library(Vorbis::vorbis ALIAS PkgConfig::vorbis)
    add_library(Vorbis::vorbisfile ALIAS PkgConfig::vorbisfile)
    add_library(luajit::luajit ALIAS PkgConfig::luajit)
endif()

target_include_directories(VoxelEngineSrc PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

target_link_libraries(
    VoxelEngineSrc
    PRIVATE glfw
            OpenGL::GL
            GLEW::GLEW
            ZLIB::ZLIB
            PNG::PNG
            CURL::libcurl
            OpenAL::OpenAL
            Vorbis::vorbis
            Vorbis::vorbisfile
            luajit::luajit
    PUBLIC glm::glm # Need public for src/delegates.hpp, which including to
                    # main.cpp
)

target_compile_options(
    VoxelEngineSrc
    PUBLIC $<$<CXX_COMPILER_ID:MSVC>:
           /utf-8
           /MP
           /D_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR
           /W4
           /wd4244 # conversion from 'a' to 'T', possible loss of data
           /wd4267 # conversion from 'size_t' to 'int', possible loss of data
           /wd4245 # conversion from 'int' to 'const size_t', signed/unsigned
                   # mismatch
           /wd4100 # unreferenced formal parameter
           /wd4457 # declaration of 'var' hides function parameter 
           /wd4458 # declaration of 'var' hides class member
           /wd4459 # declaration of 'var' hides global declaration 
           /wd4101 # 'var': unreferenced local variable
           /wd4388 # 'token' : signed/unsigned mismatch
           /wd4018 # '>': signed/unsigned mismatch
           >
           $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:
           -Wall
           -Wextra
           # additional warnings
           -Wformat-nonliteral
           #-Wsign-conversion
           -Wcast-align
           -Wpointer-arith
           -Wundef
           -Wwrite-strings
           -Wno-unused-parameter
           -Wno-sign-compare
           -Wno-unknown-pragmas
           >
           $<$<CXX_COMPILER_ID:GNU>:
           -Wduplicated-branches
           -Wduplicated-cond
           >)

target_link_options(
    VoxelEngineSrc
    PUBLIC
    $<$<CXX_COMPILER_ID:GNU>:
    -no-pie
    >
    # Need for static compilation on Windows with clang TODO: Make single build
    # on Windows to avoid dependence on combinations of platforms and compilers
    # and make it independent
    $<$<PLATFORM_ID:Windows>:$<$<CXX_COMPILER_ID:Clang>:-static>>)
