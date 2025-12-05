cmake_minimum_required(VERSION 3.13)
cmake_policy(SET CMP0015 NEW)


# Project Definition
project(odhcpd LANGUAGES C)
add_executable(${PROJECT_NAME})
target_sources(${PROJECT_NAME} PRIVATE
	src/odhcpd.c
	src/config.c
	src/dhcpv6.c
	src/dhcpv6-ia.c
	src/dhcpv6-pxe.c
	src/ndp.c
	src/netlink.c
	src/statefiles.c
	src/router.c
)


# Compiler Options
set_target_properties(${PROJECT_NAME} PROPERTIES C_STANDARD 11)
target_compile_definitions(${PROJECT_NAME} PRIVATE _GNU_SOURCE)
target_compile_options(${PROJECT_NAME} PRIVATE -g3)
target_compile_options(${PROJECT_NAME} PRIVATE -Os)
target_compile_options(${PROJECT_NAME} PRIVATE -Wall)
target_compile_options(${PROJECT_NAME} PRIVATE -Werror)
target_compile_options(${PROJECT_NAME} PRIVATE -Wextra)
target_compile_options(${PROJECT_NAME} PRIVATE -Werror=implicit-function-declaration)
target_compile_options(${PROJECT_NAME} PRIVATE -Wformat)
target_compile_options(${PROJECT_NAME} PRIVATE -Werror=format-security)
target_compile_options(${PROJECT_NAME} PRIVATE -Werror=format-nonliteral)
target_compile_options(${PROJECT_NAME} PRIVATE -Wimplicit-fallthrough=5)
target_compile_options(${PROJECT_NAME} PRIVATE -Wno-shadow=compatible-local)
target_compile_options(${PROJECT_NAME} PRIVATE -Wno-unused-parameter)
target_compile_options(${PROJECT_NAME} PRIVATE -Wmissing-declarations)
target_compile_options(${PROJECT_NAME} PRIVATE -Wshadow=local)


# Libraries
target_link_libraries(${PROJECT_NAME} PRIVATE resolv)

find_path(uci_include_dir uci.h)
target_include_directories(${PROJECT_NAME} PRIVATE ${uci_include_dir})
find_library(libuci uci)
target_link_libraries(${PROJECT_NAME} PRIVATE ${libuci})

find_path(ubox_include_dir uloop.h PATH_SUFFIXES libubox)
target_include_directories(${PROJECT_NAME} PRIVATE ${ubox_include_dir})
find_library(libubox ubox)
# libubox is also linked below if UBUS is enabled, but we need it here too
target_link_libraries(${PROJECT_NAME} PRIVATE ${libubox})

find_path(libnl-tiny_include_dir netlink-generic.h PATH_SUFFIXES libnl-tiny)
target_include_directories(${PROJECT_NAME} PRIVATE ${libnl-tiny_include_dir})
find_library(libnl nl-tiny)
target_link_libraries(${PROJECT_NAME} PRIVATE ${libnl})

find_path(json_include_dir json.h PATH_SUFFIXES json-c)
target_include_directories(${PROJECT_NAME} PRIVATE ${json_include_dir})
find_library(libjson json-c)
target_link_libraries(${PROJECT_NAME} PRIVATE ${libjson})


# Optional Features
if(${UBUS})
	target_compile_definitions(${PROJECT_NAME} PRIVATE WITH_UBUS)
	target_sources(${PROJECT_NAME} PRIVATE src/ubus.c)
	find_path(ubus_include_dir libubus.h)
	target_include_directories(${PROJECT_NAME} PRIVATE ${ubus_include_dir})
	find_library(libubus ubus)
	# Important: libubus depends on libubox symbols (like usock).
	# When static linking, order matters: dependent lib first, provider last.
	# libubus uses symbols from libubox. So libubus should come before libubox.
	# We already linked libubox above, but let's be explicit here to ensure correct order for this block.
	# However, target_link_libraries usually appends.
	# If we link libubus here, it will be appended.
	# We need libubus BEFORE libubox.
	# Since libubox was linked above, it might be after libubus if we are lucky or if CMake reorders.
	# But to be safe, let's link libubox AGAIN after libubus.
	target_link_libraries(${PROJECT_NAME} PRIVATE ${libubus} ${libubox})
endif(${UBUS})

if(${DHCPV4_SUPPORT})
	target_compile_definitions(${PROJECT_NAME} PRIVATE DHCPV4_SUPPORT)
	target_sources(${PROJECT_NAME} PRIVATE src/dhcpv4.c)
endif(${DHCPV4_SUPPORT})


# Installation
install(TARGETS ${PROJECT_NAME} DESTINATION sbin/)


# Packaging Information
set(CPACK_PACKAGE_VERSION "1")
set(CPACK_PACKAGE_CONTACT "Steven Barth <steven@midlink.org>")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PROJECT_NAME}")
set(CPACK_GENERATOR "DEB;RPM;STGZ")
set(CPACK_STRIP_FILES true)
set(CPACK_DEBIAN_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION})
set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}_${CPACK_DEBIAN_PACKAGE_VERSION}")
include(CPack)
