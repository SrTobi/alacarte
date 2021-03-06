project(alacarte-maps)

cmake_minimum_required( VERSION 2.8.8 )
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(SYSCONFDIR "${CMAKE_INSTALL_PREFIX}/etc")
if (CMAKE_INSTALL_PREFIX STREQUAL "/usr"
 OR CMAKE_INSTALL_PREFIX STREQUAL "/usr/local")
	set(SYSCONFDIR "/etc")
endif()

add_subdirectory(manpages)

#
# Depends on boost version from Debian Jessie
#
set(BOOST_MIN_VERSION_REQUIRED 1.55.0)
find_package(Boost ${BOOST_MIN_VERSION_REQUIRED} COMPONENTS
	system
	filesystem
	thread 
	# TODO unit_test_framework is only needed for unit tests, so move it to tests/CMakeLists.txt
	unit_test_framework
	program_options
	serialization
	regex
	iostreams
	log
	log_setup
	REQUIRED
)

if (NOT Boost_FOUND)
    message(FATAL_ERROR "Fatal error: Boost (version >= ${BOOST_MIN_VERSION_REQUIRED}) required.\n")
endif (NOT Boost_FOUND)

#find other packages
IF(WIN32)
	SET(ENV{FREETYPE_DIR} ${CMAKE_FIND_ROOT_PATH})
ELSE(WIN32)
    find_package(Inotify REQUIRED)
ENDIF(WIN32)

find_package(Cairo REQUIRED)
if(CAIRO_FOUND)
	message(STATUS "Cairo Version: ${CAIRO_VERSION}")
endif()
find_package(Freetype REQUIRED)

find_package(Threads REQUIRED)
find_package(Doxygen)

#
# Build options
#

# FIXME hack for old cairo and MinGW
if(CAIRO_VERSION_V LESS 1 OR CAIRO_VERSION_MAJOR LESS 12 OR CAIRO_VERSION_MINOR LESS 2 OR MINGW)
    OPTION(RENDER_LOCK "Render lock for old cairo and Windows" ON)
    if(MINGW)
        message (STATUS "MinGW detected: enabling render lock.")
    else(MINGW)
        message (STATUS "${CAIRO_VERSION} < 1.12.2: enabling render lock.")
    endif(MINGW)
else()
    OPTION(RENDER_LOCK "Render lock for old cairo and Windows" OFF)
endif()
if(RENDER_LOCK)
    set(RENDER_LOCK 1)
else()
    set(RENDER_LOCK 0)
endif()

# Set windows version for MinGW and Inotify for non-mingw targets
IF(MINGW)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_WIN32_WINNT=0x0501")
    add_definitions(-D_WIN32_WINNT=0x0501)
    set(SYSTEM_LIBRARIES ws2_32 wsock32)
ELSE(MINGW)
    include_directories(${INOTIFY_INCLUDE_DIRS})
    set(SYSTEM_LIBRARIES ${INOTIFY_LIBRARY})
ENDIF(MINGW)

link_directories(${Boost_LIBRARY_DIRS})
include_directories(include ${Boost_INCLUDE_DIRS} ${CAIRO_INCLUDE_DIR} ${FREETYPE_INCLUDE_DIRS})

add_definitions(-DStatistic_Activated)

if(MSVC)
   ADD_DEFINITIONS(-D_WIN32_WINNT=0x0501)
   ADD_DEFINITIONS(-DNOMINMAX)
   ADD_DEFINITIONS(-D_USE_MATH_DEFINES)
else()
    # don't enable -Werror with clang as it throws many warnings with boost.log 1.55
    if(CMAKE_COMPILER_IS_GNUCXX)
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
    endif()
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    SET(CMAKE_CXX_FLAGS_PROFILE "${CMAKE_CXX_FLAGS_DEBUG} -O0 -ftest-coverage -fprofile-arcs -fprofile-values -fvpt -pg")
    SET(CMAKE_CXX_FLAGS_SANITIZE "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=thread -fno-omit-frame-pointer")
    ADD_DEFINITIONS("-std=c++11")
    #ADD_DEFINITIONS("-fdiagnostics-color=auto")
    ADD_DEFINITIONS(-DBOOST_TEST_DYN_LINK)
    ADD_DEFINITIONS(-DBOOST_LOG_DYN_LINK)
endif()

MARK_AS_ADVANCED(CMAKE_CXX_FLAGS_PROFILE)
SET( CMAKE_BUILD_TYPE "${CMAKE_BUILD_TYPE}" CACHE STRING
    "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel Profile Sanitize."
    FORCE )

set(DEBUG_BUILD 0)
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(DEBUG_BUILD 1)
endif ()
if (CMAKE_BUILD_TYPE STREQUAL "Profile")
    set(DEBUG_BUILD 1)
endif ()
if (CMAKE_BUILD_TYPE STREQUAL "Sanitize")
    set(DEBUG_BUILD 1)
endif ()
if (DEBUG_BUILD)
	message (STATUS "Debug code enabled (DEBUG_BUILD macro set)")
else()
	message (STATUS "Debug code disabled (DEBUG_BUILD macro not set)")
endif()

message(STATUS "Compile Flags:")
message(STATUS "-----------------")
message(STATUS "| Default : ${CMAKE_CXX_FLAGS}")
message(STATUS "| Debug   : ${CMAKE_CXX_FLAGS_DEBUG}")
message(STATUS "| Profile : ${CMAKE_CXX_FLAGS_PROFILE}")
message(STATUS "| Release : ${CMAKE_CXX_FLAGS_RELEASE}")
message(STATUS "| Sanitize: ${CMAKE_CXX_FLAGS_SANITIZE}")
message(STATUS "-----------------")


#
# Sources
#
file(GLOB_RECURSE server_sources                src/server/*.cpp)
file(GLOB_RECURSE importer_sources              src/importer/*.cpp)
file(GLOB_RECURSE alacarte_sources              src/general/*.cpp src/utils/*.cpp)


#
# Targets
#
configure_file("data/config/alacarte-maps.conf.in" "${CMAKE_CURRENT_BINARY_DIR}/data/config/alacarte-maps.conf")
configure_file("include/config.hpp.in" "${CMAKE_CURRENT_BINARY_DIR}/include/config.hpp")
include_directories("${CMAKE_CURRENT_BINARY_DIR}/include")

add_library(alacarte-obj OBJECT ${alacarte_sources})
add_library(server-obj OBJECT ${server_sources})

add_executable(alacarte-maps-server      src/alacarte_server.cpp     $<TARGET_OBJECTS:server-obj> $<TARGET_OBJECTS:alacarte-obj>)
add_executable(alacarte-maps-importer    src/alacarte_importer.cpp   ${importer_sources} $<TARGET_OBJECTS:alacarte-obj>)

target_link_libraries(alacarte-maps-server       ${Boost_LIBRARIES} ${CAIRO_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${SYSTEM_LIBRARIES})
target_link_libraries(alacarte-maps-importer     ${Boost_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${SYSTEM_LIBRARIES})


option(BUILD_TESTING "Build unit tests" OFF)
if(BUILD_TESTING)
	message(STATUS "Building unit tests enabled")
	enable_testing()
	add_subdirectory(tests)
else()
	message(STATUS "Building unit tests disabled")
endif()


#
# Add cpplint target
#
add_custom_target(lint
	COMMAND cpplint --filter=-whitespace --extensions=c,h,cpp,hpp ${server_sources} ${importer_source} ${alacarte_sources}
	COMMENT "Check if there are style violations" VERBATIM
)


#
# Generate documentation
# add a target to generate API documentation with Doxygen
#
if(DOXYGEN_FOUND)
	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
	add_custom_target(doc ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating API documentation with Doxygen" VERBATIM
    )
endif(DOXYGEN_FOUND)


#
# Installation
#
install(TARGETS alacarte-maps-server RUNTIME DESTINATION bin)
install(TARGETS alacarte-maps-importer RUNTIME DESTINATION bin)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/data/config/alacarte-maps.conf DESTINATION ${SYSCONFDIR})
install(FILES data/default.png DESTINATION share/alacarte-maps)
install(DIRECTORY data/mapcss/ DESTINATION share/alacarte-maps/styles)
install(DIRECTORY data/img DESTINATION share/alacarte-maps)
install(DIRECTORY data/icons DESTINATION share/alacarte-maps)
