project(thinkfan)
include(GNUInstallDirs)

cmake_minimum_required(VERSION 2.6)

# Generate absolute paths or something
cmake_policy(SET CMP0015 NEW)

find_package(PkgConfig)
pkg_check_modules(SYSTEMD "systemd")
pkg_check_modules(OPENRC "openrc")

pkg_check_modules(YAML_CPP "yaml-cpp")

if(YAML_CPP_FOUND AND YAML_CPP_VERSION VERSION_LESS "0.5.3")
    message(WARNING "yaml-cpp version ${YAML_CPP_VERSION} is very old, buggy and lacks some features. Thinkfan will not always be able to point out the location of errors in the YAML config.")
    add_definitions(-DHAVE_OLD_YAMLCPP)
endif()

pkg_check_modules(ATASMART "libatasmart")

if(SYSTEMD_FOUND)
    set(PID_FILE "/run/thinkfan.pid")
else(SYSTEMD_FOUND)
    set(PID_FILE "/var/run/thinkfan.pid")
endif(SYSTEMD_FOUND)


add_definitions(-DPID_FILE=\"${PID_FILE}\")

#
# Defaults to OFF because libatasmart seems to be horribly inefficient
#
option(USE_ATASMART "Enable reading temperatures from HDDs via S.M.A.R.T" OFF)

#
# Defaults to ON because it seems reasonably fast. The libnvidia-ml.so is
# loaded at runtime, so we don't add a compile-time dependency on the
# proprietary nVidia driver.
#
option(USE_NVML "Get temperatures directly from nVidia GPUs via their proprietary NVML API" ON)

#
# The shiny new YAML config parser. Depends on yaml-cpp.
#
option(USE_YAML "Enable the new YAML-based config format" ON)


option(DISABLE_BUGGER "Disable bug detection, i.e. dont't catch segfaults and unhandled exceptions" OFF)
option(DISABLE_SYSLOG "Disable logging to syslog, always log to stdout" OFF)
option(DISABLE_EXCEPTION_CATCHING "Terminate with SIGABRT on all exceptions, causing a core dump on every error" OFF)


set(SRC_FILES src/thinkfan.cpp src/config.cpp src/drivers.cpp
    src/message.cpp src/parser.cpp src/error.cpp)

if(USE_YAML)
    if(NOT YAML_CPP_FOUND)
	message(FATAL_ERROR "USE_YAML enabled but yaml-cpp not found. Please install yaml-cpp[-devel]!")
    endif()
    set(SRC_FILES ${SRC_FILES} src/yamlconfig.cpp)
endif(USE_YAML)


#
# Set default build type
#
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Options are: Debug Release RelWithDebInfo MinSizeRel."
	FORCE)
endif(NOT CMAKE_BUILD_TYPE)

add_compile_options(-Wall -std=c++1y)
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0 -g3 -DDEBUG")


add_executable(thinkfan ${SRC_FILES})


if(USE_ATASMART)
    if(NOT ATASMART_FOUND)
	message(FATAL_ERROR "USE_ATASMART enabled but libatasmart not found. Please install libatasmart[-devel]!")
    else()
	add_definitions(-DUSE_ATASMART)
	target_link_libraries(thinkfan atasmart)
    endif()
endif(USE_ATASMART)

if(USE_NVML)
    include_directories(AFTER "include")
    add_definitions(-DUSE_NVML)
    target_link_libraries(thinkfan dl)
endif(USE_NVML)

if(USE_YAML)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUSE_YAML")
	#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I/home/vmatare/devel/yaml-cpp/include -L/home/vmatare/devel/build-yaml-cpp-Desktop-Default")
	target_link_libraries(thinkfan yaml-cpp)
endif(USE_YAML)

if(DISABLE_BUGGER)
    add_definitions(-DDISABLE_BUGGER)
endif(DISABLE_BUGGER)
if(DISABLE_SYSLOG)
    add_definitions(-DDISABLE_SYSLOG)
endif(DISABLE_SYSLOG)
if(DISABLE_EXCEPTION_CATCHING)
    add_definitions(-DDISABLE_EXCEPTION_CATCHING)
endif(DISABLE_EXCEPTION_CATCHING)


install(TARGETS thinkfan DESTINATION "${CMAKE_INSTALL_SBINDIR}")
install(FILES COPYING README examples/thinkfan.conf.complex
    examples/thinkfan.conf.simple examples/thinkfan.conf.yaml DESTINATION "${CMAKE_INSTALL_DOCDIR}")
install(FILES src/thinkfan.1 DESTINATION "${CMAKE_INSTALL_MANDIR}/man1")
install(FILES src/thinkfan.conf.5 DESTINATION "${CMAKE_INSTALL_MANDIR}/man5")

if(SYSTEMD_FOUND)
    configure_file(rcscripts/systemd/thinkfan.service.cmake
	rcscripts/systemd/thinkfan.service)
    install(FILES
	rcscripts/systemd/thinkfan-wakeup.service
	"${CMAKE_BINARY_DIR}/rcscripts/systemd/thinkfan.service"
	DESTINATION "${CMAKE_INSTALL_LIBDIR}/systemd/system")
    if(NOT EXISTS "/etc/systemd/system/thinkfan.service.d/override.conf")
	install(FILES
	    rcscripts/systemd/override.conf
	    DESTINATION "/etc/systemd/system/thinkfan.service.d")
    else()
	install(FILES
	    rcscripts/systemd/override.conf
	    DESTINATION "/etc/systemd/system/thinkfan.service.d"
	    RENAME "default.conf")
    endif()
endif(SYSTEMD_FOUND)

if(OPENRC_FOUND)
    configure_file(rcscripts/openrc/thinkfan.cmake
	rcscripts/openrc/thinkfan)
    install(FILES
	"${CMAKE_BINARY_DIR}/rcscripts/openrc/thinkfan"
	PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
	DESTINATION "/etc/init.d")
endif(OPENRC_FOUND)
