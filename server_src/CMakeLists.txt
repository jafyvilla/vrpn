cmake_minimum_required(VERSION 2.6)

#-----------------------------------------------------------------------------
# XXX Things to make better.
#
# Move applications into subprojects in visual studio
# XXX tracker_to_poser
# XXX daq_server
# Fix the ones that are commented out so that they compile and run?

if(NOT VRPN_BUILD_SERVER_LIBRARY)
	message(FATAL_ERROR
		"Cannot compile servers without the full (server) library.  Enable it and try again.")
endif()

set(VRPN_SERVER_LIBRARY vrpnserver)
set(VRPN_GPSNMEA_LIBRARY gpsnmea)

if(NOT WIN32)
	set(VRPN_ATMEL_LIBRARY vrpn_atmel)
endif()

#-----------------------------------------------------------------------------
# Phantom library (if configured)
find_package(OpenGL)
if(VRPN_USE_PHANTOM_SERVER AND OPENGL_FOUND)
	set(PHANTOM_SOURCES
		ghostEffects/InstantBuzzEffect.cpp
		buzzForceField.C
		constraint.C
		forcefield.C
		plane.C
		texture_plane.C
		trimesh.C
		vrpn_Phantom.C)

	set(PHANTOM_HEADERS
		ghostEffects/InstantBuzzEffect.h
		buzzForceField.h
		constraint.h
		forcefield.h
		ghost.h
		plane.h
		texture_plane.h
		trimesh.h
		vrpn_Phantom.h)
	include_directories(${VRPN_SOURCE_DIR}/server_src/ghostEffects)
	if(VRPN_USE_GHOST AND GHOST_STL_INCLUDE_DIR)
		include_directories(${GHOST_STL_INCLUDE_DIR})
	endif()
	add_library(vrpn_phantom ${PHANTOM_SOURCES} ${PHANTOM_HEADERS})
	target_link_libraries(vrpn_phantom vrpnserver ${OPENGL_LIBRARIES})

	set_property(TARGET
		vrpn_phantom
		PROPERTY
		PUBLIC_HEADER
		${PHANTOM_HEADERS})
	set_target_properties(vrpn_phantom PROPERTIES FOLDER Library)

	install(TARGETS
		vrpn_phantom
		ARCHIVE
		DESTINATION
		lib
		COMPONENT serversdk
		PUBLIC_HEADER
		DESTINATION
		include
		COMPONENT serversdk)
	set(VRPNPHANTOMLIB vrpn_phantom)
endif()

#------------------------------------------------------------------
# DirectShow video server - only for pre-VS10 due to header removal
if(VRPN_BUILD_DIRECTSHOW_VIDEO_SERVER AND DIRECTSHOW_FOUND)
	add_subdirectory(directshow_video_server)
endif()

add_subdirectory(timecode_generator_server)

#-----------------------------------------------------------------------------
# Include directories needed
include_directories(${VRPN_SOURCE_DIR}/server_src
	${VRPN_SOURCE_DIR}/server_src/timecode_generator_server)

#-----------------------------------------------------------------------------
# Applications that need more than one file to run.
add_executable(vrpn_server
	vrpn.cfg
	vrpn.C)
target_link_libraries(vrpn_server
	${VRPN_SERVER_LIBRARY}
	vrpn_timecode_generator
	${VRPN_ATMEL_LIBRARY}
	${VRPN_GPSNMEA_LIBRARY}
	${VRPNPHANTOMLIB})
set_target_properties(vrpn_server PROPERTIES SOLUTION_FOLDER servers)
set_property(TARGET
	vrpn_server
	PROPERTY
	PROJECT_LABEL
	"vrpn_server")
set_target_properties(vrpn_server PROPERTIES FOLDER "Main Server")
install(TARGETS vrpn_server
	RUNTIME DESTINATION bin
	COMPONENT mainserver)

#-----------------------------------------------------------------------------
# Install the vrpn.cfg file needed by the server.
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/vrpn.cfg" "${CMAKE_CURRENT_BINARY_DIR}/vrpn.cfg" @ONLY)
if(WIN32)
	install(FILES "${CMAKE_CURRENT_BINARY_DIR}/vrpn.cfg"
		DESTINATION bin
		COMPONENT mainserver)
else()
	install(FILES "${CMAKE_CURRENT_BINARY_DIR}/vrpn.cfg"
		DESTINATION etc
		COMPONENT mainserver)
endif()

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/vrpn.cfg"
	DESTINATION share/vrpn-${BRIEF_VERSION}
	RENAME vrpn.cfg.sample
	COMPONENT mainserver)

#-----------------------------------------------------------------------------
# Declaration of the simple test applications that don't have extra files

# XXX Are these files compiled somewhere?
#NIUtil.cpp
#NIUtil.h
#clock_server.C
#daq_server.C
#directXFF.C
#make_vrpn_libs_build.cpp
#nidaqClient.cpp
#nidaqServer.cpp



set(SRV_TEST_SOURCES
	client_and_server.C
	#forward.C
	#last_of_sequence.C
	#sample_analog.C
	#sample_server.C
	#testSharedObject.C
	test_analogfly.C
	test_auxiliary_logger.C
	test_freespace.C
	test_logging.C
	#test_mutex.C
	test_peerMutex.C
	test_radamec_spi.C
	test_rumble.C
	test_vrpn.C
	testimager_server.cpp
	#text.C
)
if(VRPN_USE_HID)
	list(APPEND SRV_TEST_SOURCES vrpn_HID_device_watcher.cpp)
endif()

set(SRV_SERVER_SOURCES
	wiimote_head_tracker.C
)


###
# Tests
###
if(BUILD_TESTING)
	foreach(SOURCE ${SRV_TEST_SOURCES})
		get_filename_component(APP ${SOURCE} NAME_WE)
		add_executable(${APP} ${SOURCE})
		target_link_libraries(${APP} ${VRPN_SERVER_LIBRARY})
		set_target_properties(${APP} PROPERTIES FOLDER Tests)
		set_target_properties(${APP} PROPERTIES HAS_CXX yes)
		install(TARGETS ${APP}
			RUNTIME DESTINATION bin COMPONENT tests)
	endforeach()
	add_test(test_vrpn test_vrpn)
endif()

###
# Servers
###
foreach(SOURCE ${SRV_SERVER_SOURCES})
	get_filename_component(APP ${SOURCE} NAME_WE)
	add_executable(${APP} ${SOURCE})
	target_link_libraries(${APP} ${VRPN_SERVER_LIBRARY})
	set_target_properties(${APP} PROPERTIES FOLDER Servers)
	set_target_properties(${APP} PROPERTIES HAS_CXX yes)
	install(TARGETS ${APP} RUNTIME DESTINATION bin COMPONENT servers)
endforeach()

###
# Install extra files
###

# Wiimote jconf files for fishtank VR
set(VRPN_WIIMOTE_JCONFS
	vrpn.wiimoteheadtracker.fishtankvr.1280x1024.jconf
	vrpn.wiimoteheadtracker.fishtankvr.1024x768.jconf
	vrpn.wiimoteheadtracker.fishtankvr.800x600.jconf)

install(FILES
	${VRPN_WIIMOTE_JCONFS}
	DESTINATION
	share/vrpn-${BRIEF_VERSION}/vrjuggler-2.2-config
	COMPONENT servers)

