cmake_minimum_required (VERSION 2.8)
project(OpenSkyImager)

set(FORCE_GTK2 off CACHE BOOL "Force GTK2 Detection over GTK3")
set(FORCE_QHY_HEX off CACHE BOOL "Force QHY firmwares install/upgrade")
set(FORCE_QHY_ONLY on CACHE BOOL "Will only use QHY driver")
set(FORCE_NO_SBIG off CACHE BOOL "Set aside use of SBIG driver")
set(FORCE_NO_URVC on CACHE BOOL "Set aside use of URVC driver")
set(FORCE_NO_ATIK off CACHE BOOL "Set aside use of ATIK driver")
set(FORCE_SBIG_UPDATE off CACHE BOOL "Force SBIG driver update")
set(ENABLE_INDI_DRIVER off CACHE BOOL "Enable experimental INDI driver compilation (default: off)")
set(ENABLE_CMDLINE off CACHE BOOL "Enable command line frontend (default: off)")

add_definitions ("-Wall")

# For debug only 
# set(CMAKE_INSTALL_PREFIX /home/giampiero)

# http://francesco-cek.com/cmake-and-gtk-3-the-easy-way/
find_package(PkgConfig REQUIRED)
set(HAVE_GTK_FRONTEND off)

if(NOT FORCE_GTK2)
  pkg_check_modules(GTK3 gtk+-3.0)
endif(NOT FORCE_GTK2)

if(GTK3_FOUND)
  set(HAVE_GTK_FRONTEND on)
  include_directories(${GTK3_INCLUDE_DIRS})
  link_directories(${GTK3_LIBRARY_DIRS})
  add_definitions(${GTK3_CFLAGS_OTHER})
else(GTK3_FOUND)
  message("GTK3 libraries not found, checking for GTK2...")
  pkg_check_modules(GTK2 REQUIRED gtk+-2.0)
  if(GTK2_FOUND)
    set(HAVE_GTK_FRONTEND on)
    include_directories(${GTK2_INCLUDE_DIRS})
    link_directories(${GTK2_LIBRARY_DIRS})
    add_definitions(${GTK2_CFLAGS_OTHER})
  endif(GTK2_FOUND)
endif(GTK3_FOUND)

pkg_check_modules(GLIB REQUIRED glib-2.0)
include_directories(${GLIB_INCLUDE_DIRS})
link_directories(${GLIB_LIBRARY_DIRS})
add_definitions(${GLIB_CFLAGS_OTHER})

pkg_check_modules(CFITSIO REQUIRED cfitsio)
include_directories(${CFITSIO_INCLUDE_DIRS})
link_directories(${CFITSIO_LIBRARY_DIRS})
add_definitions(${CFITSIO_CFLAGS_OTHER})

#pkg_check_modules(LIBUSB REQUIRED libusb-1.0)
#include_directories(${LIBUSB_INCLUDE_DIRS})
#link_directories(${LIBUSB_LIBRARY_DIRS})
#add_definitions(${LIBUSB_CFLAGS_OTHER})

add_subdirectory(libusb-custom)
set(LIBUSB_LIBRARIES ${CMAKE_SOURCE_DIR}/libusb-custom/libusb/.libs/libusb-1.0.a -ludev)
include_directories(${CMAKE_SOURCE_DIR}/libusb-custom/libusb)

set(COMMON_LIBRARIES ${GLIB_LIBRARIES} ${CFITSIO_LIBRARIES} ${LIBUSB_LIBRARIES} pthread rt m)

if(NOT FORCE_QHY_ONLY)
	if(NOT FORCE_NO_ATIK)
		find_library(ATIK_LIB atikccd)
		if(ATIK_LIB)
			message("Found ATIK library: ${ATIK_LIB}")
			set(COMMON_LIBRARIES ${COMMON_LIBRARIES} ${ATIK_LIB})
			add_definitions(-DHAVE_ATIK=1)
		else(ATIK_LIB)
			message("ATIK library *not* found")
			message("Downloadable package at: http://www.cloudmakers.eu/atikdrivers/")
		endif(ATIK_LIB)
	else(NOT FORCE_NO_ATIK)
		message("Use of ATIK library, skipped as requested")
	endif(NOT FORCE_NO_ATIK)

	if(NOT FORCE_NO_URVC)
		add_subdirectory(urvc2)
		set(URVC2_LIB ${CMAKE_CURRENT_BINARY_DIR}/urvc2/liburvc2.a)
		include_directories(${CMAKE_SOURCE_DIR}/urvc2)
		set(COMMON_LIBRARIES ${COMMON_LIBRARIES} ${URVC2_LIB})
		add_definitions(-DHAVE_URVC=1)
	else(NOT FORCE_NO_URVC)
		message("Use of URVC library, skipped as requested")
	endif(NOT FORCE_NO_URVC)

	if(NOT FORCE_NO_SBIG)
		if(NOT FORCE_SBIG_UPDATE)
			find_library(SBIG_LIB sbigudrv)
			if(SBIG_LIB)
				message("Found SBIG library: ${SBIG_LIB}")
				set(COMMON_LIBRARIES ${COMMON_LIBRARIES} ${SBIG_LIB})
				add_definitions(-DHAVE_SBIG=1)
			endif(SBIG_LIB)
		endif(NOT FORCE_SBIG_UPDATE)

		if(NOT SBIG_LIB OR FORCE_SBIG_UPDATE)
			message("SBIG library will be installed: /usr/lib/libsbigudrv.so")
			set(COMMON_LIBRARIES ${COMMON_LIBRARIES} /usr/lib/libsbigudrv.so)
			add_definitions(-DHAVE_SBIG=1)
			add_subdirectory(sbig)
		endif(NOT SBIG_LIB OR FORCE_SBIG_UPDATE)
	else(NOT FORCE_NO_SBIG)
		message("Use of SBIG library, skipped as requested")
	endif(NOT FORCE_NO_SBIG)
else(NOT FORCE_QHY_ONLY)
	message("Will only use QHY driver, as requested")
endif(NOT FORCE_QHY_ONLY)

find_program(HAVE_QHY_HEX "fxload")
if (HAVE_QHY_HEX)
	message("Found fxload")
else(HAVE_QHY_HEX)
	message("Please install fxload in order to load QHY firmwares into camera")
endif(HAVE_QHY_HEX)

if(FORCE_QHY_HEX OR NOT EXISTS /etc/qhyccd)
	message("Will add QHY firmwares, please restart udev after program is installed")
	add_subdirectory(qhy)
endif(FORCE_QHY_HEX OR NOT EXISTS /etc/qhyccd)

add_subdirectory(files)
add_subdirectory(core)
include_directories(core)

if(HAVE_GTK_FRONTEND)
  add_subdirectory(gtk)
endif(HAVE_GTK_FRONTEND)

add_subdirectory(cpp)
include_directories(cpp)

if (ENABLE_INDI_DRIVER)
	pkg_check_modules(LIBINDI libindi)
	if(LIBINDI_FOUND)
		add_subdirectory(indi)
	endif(LIBINDI_FOUND)
endif(ENABLE_INDI_DRIVER)
if(ENABLE_CMDLINE)
  find_package(Boost 1.49 REQUIRED COMPONENTS thread chrono system program_options date_time filesystem)
  include_directories(${Boost_INCLUDE_DIRS})
  add_subdirectory(cmdline)
endif(ENABLE_CMDLINE)

# add uninstall target
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

