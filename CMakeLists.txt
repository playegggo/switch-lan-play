cmake_minimum_required(VERSION 3.0)
option(STATIC_LINKING "Static linking" OFF)
project (SwitchLanPlay)
execute_process(COMMAND git describe --tags
                WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                OUTPUT_VARIABLE LANPLAY_VERSION
                OUTPUT_STRIP_TRAILING_WHITESPACE)

if (LANPLAY_VERSION STREQUAL "")
    set(LANPLAY_VERSION v0.0.7)
    MESSAGE("git describe --tags output is empty, set version to ${LANPLAY_VERSION}")
endif()

add_compile_options(
    "-Wall"
    "-Wno-unused-function"
    "-DLANPLAY_VERSION=\"${LANPLAY_VERSION}\""
)

include(TestBigEndian)
test_big_endian(BIG_ENDIAN)
if (BIG_ENDIAN)
    add_definitions(-DLANPLAY_BIG_ENDIAN)
else ()
    add_definitions(-DLANPLAY_LITTLE_ENDIAN)
endif ()

if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_definitions(-DLANPLAY_LINUX)
elseif(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    add_definitions(-DLANPLAY_FREEBSD)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    add_definitions(-DLANPLAY_DARWIN)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    add_definitions(-DLANPLAY_ANDROID)
endif()

set(SOURCE_FILES
    base/debug.c
    base/llog.c

    src/main.c
    src/packet.c
    src/arp.c
    src/helper.c
    src/lan-client.c
    src/gateway.c
    src/proxy.c
    src/ipv4/ipv4.c
    src/ipv4/tcp.c
    src/ipv4/udp.c
    src/ipv4/icmp.c

    src/uv_lwip.c
)
add_executable(lan-play ${SOURCE_FILES})

include_directories(
    lan-play
    ${CMAKE_CURRENT_SOURCE_DIR}
    lwip/custom
    lwip/src/include
    src
)
add_subdirectory(lwip)
target_link_libraries(lan-play lwip_lib)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
if (WIN32)
    include(ExternalProject)
    ExternalProject_Add(
        winpcap
        PREFIX ${CMAKE_BINARY_DIR}/winpcap
        URL "https://www.winpcap.org/install/bin/WpdPack_4_1_2.zip"
        URL_MD5 "bae2236af062b0900ad1416b2c4878b9"
        CONFIGURE_COMMAND "" BUILD_COMMAND "" INSTALL_DIR "" INSTALL_COMMAND ""
        # LOG_DOWNLOAD ON
    )
    ExternalProject_Get_Property(winpcap SOURCE_DIR)
    set(PCAP_INCLUDE_DIR ${SOURCE_DIR}/Include)
    set(PCAP_LIBRARY ${SOURCE_DIR}/Lib/x64/wpcap.lib ${SOURCE_DIR}/Lib/wpcap.lib)
    message("PCAP_ROOT_DIR = ${PCAP_ROOT_DIR} INCLUDE = ${PCAP_INCLUDE_DIR}")
    add_dependencies(lan-play winpcap)
else()
    find_package(PCAP)
endif()
include_directories(${PCAP_INCLUDE_DIR})
find_package(Libuv)
if (LIBUV_FOUND)
    include_directories(${LIBUV_INCLUDE_DIR})
endif()

if (STATIC_LINKING)
    target_link_libraries(lan-play ${PCAP_LIBRARY} ${LIBUV_LIBRARIES} -static)
else()
    target_link_libraries(lan-play ${PCAP_LIBRARY} ${LIBUV_LIBRARIES})
endif()
if (WIN32)
    target_link_libraries(lan-play ws2_32.lib iphlpapi.lib libwinpthread.a psapi userenv.lib)
endif()
if (!ANDROID)
    target_link_libraries(lan-play pthread -static)
endif()

INSTALL(TARGETS lan-play RUNTIME DESTINATION bin)
