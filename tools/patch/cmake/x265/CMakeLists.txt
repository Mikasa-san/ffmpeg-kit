python3 - <<'PY'
from pathlib import Path

p = Path("src/x265/source/CMakeLists.txt")
if not p.exists():
    p = Path("src/x265/CMakeLists.txt")
if not p.exists():
    p = Path("CMakeLists.txt")

s = p.read_text()

s = s.replace(
"""project (x265)
cmake_minimum_required (VERSION 2.8.8) # OBJECT libraries require 2.8.8""",
"""cmake_minimum_required(VERSION 4.3.2)
project(x265 LANGUAGES C CXX)"""
)

s = s.replace(
"""if(POLICY CMP0025)
    cmake_policy(SET CMP0025 OLD) # report Apple's Clang as just Clang
endif()""",
"""if(POLICY CMP0025)
    cmake_policy(SET CMP0025 NEW)
endif()"""
)

s = s.replace(
"""if(POLICY CMP0054)
    cmake_policy(SET CMP0054 OLD) # Only interpret if() arguments as variables or keywords when unquoted
endif()""",
"""if(POLICY CMP0054)
    cmake_policy(SET CMP0054 NEW)
endif()"""
)

insert_after = "project(x265 LANGUAGES C CXX)\n"
ios_block = r'''
# FFmpegKit Apple/iOS compatibility for modern CMake.
# Build only static libx265.a for FFmpeg linking.
if(APPLE)
    set(ENABLE_SHARED OFF CACHE BOOL "Build shared library" FORCE)
    set(ENABLE_CLI OFF CACHE BOOL "Build standalone CLI application" FORCE)
    set(ENABLE_ASSEMBLY OFF CACHE BOOL "Enable assembly primitives" FORCE)
    set(ENABLE_LIBNUMA OFF CACHE BOOL "Enable libnuma usage" FORCE)
    set(ENABLE_PIC ON CACHE BOOL "Enable Position Independent Code" FORCE)

    add_compile_options(-fPIC)

    set(CMAKE_MACOSX_RPATH ON)
    set(CMAKE_XCODE_ATTRIBUTE_ONLY_ACTIVE_ARCH NO)
    set(CMAKE_XCODE_ATTRIBUTE_ENABLE_BITCODE NO)
    set(CMAKE_XCODE_ATTRIBUTE_CODE_SIGNING_ALLOWED NO)
endif()

'''
if ios_block.strip() not in s:
    s = s.replace(insert_after, insert_after + ios_block)

s = s.replace(
"""if(ARM OR CROSS_COMPILE_ARM)
    option(ENABLE_ASSEMBLY "Enable use of assembly coded primitives" ON)""",
"""if(APPLE AND ARM)
    option(ENABLE_ASSEMBLY "Enable use of assembly coded primitives" OFF)
elseif(ARM OR CROSS_COMPILE_ARM)
    option(ENABLE_ASSEMBLY "Enable use of assembly coded primitives" ON)"""
)

s = s.replace(
"""option(ENABLE_SHARED "Build shared library" ON)""",
"""if(APPLE)
    option(ENABLE_SHARED "Build shared library" OFF)
else()
    option(ENABLE_SHARED "Build shared library" ON)
endif()"""
)

s = s.replace(
"""set(ENABLE_CLI ON CACHE BOOL "Build standalone CLI application")""",
"""if(APPLE)
    set(ENABLE_CLI OFF CACHE BOOL "Build standalone CLI application" FORCE)
else()
    set(ENABLE_CLI ON CACHE BOOL "Build standalone CLI application")
endif()"""
)

p.write_text(s)
print(f"Updated {p}")
PY