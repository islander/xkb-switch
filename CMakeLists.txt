CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

PROJECT( XKB-SWITCH )
SET(MAJOR_VERSION 1)
SET(MINOR_VERSION 7)
SET(RELEASE_VERSION 0)
SET(XKBSWITCH_VERSION ${MAJOR_VERSION}.${MINOR_VERSION}.${RELEASE_VERSION})
ADD_DEFINITIONS(-DXKBSWITCH_VERSION="${XKBSWITCH_VERSION}")

SET(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules")

# Check presence of development libraries required for build
FIND_PACKAGE(X11 REQUIRED)
if(NOT X11_FOUND)
    message(FATAL_ERROR "Not found development files of 'libx11' required for build. (Install libx11-dev or libx11-devel package.) CMake will exit.")
elseif(NOT X11_Xkbfile_FOUND)
    message(FATAL_ERROR "Not found development files of 'libxkbfile' required for build. (Install libxkbfile-dev or libxkbfile-devel package.) CMake will exit.")
endif()
INCLUDE_DIRECTORIES(${X11_INCLUDE_DIR})
LINK_DIRECTORIES(${X11_LIBRARY_DIR})

# Compile and link program
OPTION(BUILD_XKBSWITCH_LIB
    "Build a library compatible with vim's libcall interface" ON)
if(BUILD_XKBSWITCH_LIB)
    SET(xkblib xkbswitch)
    ADD_LIBRARY(${xkblib} SHARED src/XKbSwitchApi.cpp src/XKeyboard.cpp)
    SET_TARGET_PROPERTIES(${xkblib} PROPERTIES VERSION ${XKBSWITCH_VERSION} SOVERSION ${MAJOR_VERSION})
    TARGET_LINK_LIBRARIES(${xkblib} X11 xkbfile)
    ADD_EXECUTABLE(xkb-switch src/XKbSwitch.cpp)
    TARGET_LINK_LIBRARIES(xkb-switch ${xkblib})
else()
    ADD_EXECUTABLE(xkb-switch src/XKbSwitch.cpp src/XKeyboard.cpp)
    TARGET_LINK_LIBRARIES(xkb-switch X11 xkbfile)
endif()

# Install program
INSTALL(TARGETS xkb-switch ${xkblib}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib OPTIONAL
)

SET(MAN_COMPRESSION "xz" CACHE STRING "Manpages compression tool")
SET(MANDIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE STRING "Manpages installation path")

# Function to compress and install man page
# Gets file name and type number
function(install_man man_filename man_type)
    # check what compression tool is available
    FIND_PROGRAM(COMPRESS_EXECUTABLE NAMES ${MAN_COMPRESSION})
    if(NOT COMPRESS_EXECUTABLE)
        FIND_PROGRAM(COMPRESS_EXECUTABLE NAMES gzip)
        if(COMPRESS_EXECUTABLE)
            SET(MAN_COMPRESSION gz)
        else()
            SET(MAN_COMPRESSION NO)
        endif()
    endif()
    # set input an output file names
    SET(raw_man man/${man_filename}.${man_type})
    SET(compressed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.${MAN_COMPRESSION})
    # compress if there is the compression tool
    if(MAN_COMPRESSION)
        ADD_CUSTOM_COMMAND(OUTPUT ${compressed_man}
            COMMAND cat ${raw_man} | ${COMPRESS_EXECUTABLE} > ${compressed_man}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Compressing man file ${raw_man} to ${compressed_man}"
        )
    # elsewise just copy
    else()
        MESSAGE(WARNING "There is no compression tool for man pages (xz or gzip). Not compressed copy of man file will be used.")
        SET(compressed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type})
        ADD_CUSTOM_COMMAND(OUTPUT ${compressed_man}
            COMMAND cp ${raw_man} ${CMAKE_BINARY_DIR}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Copying man file from ${raw_man}."
        )
    endif()
    # add actions
    ADD_CUSTOM_TARGET(man_${man_filename}_${man_type} ALL DEPENDS ${compressed_man})
    INSTALL(FILES ${compressed_man}
        DESTINATION ${MANDIR}/man${man_type}
    )
endfunction()

# Compress and install man page
install_man(xkb-switch 1)

###############
#  Packaging  #
###############

if(UNIX AND NOT APPLE AND NOT CYGWIN)
    SET(CPACK_GENERATOR "DEB")
    STRING(TOLOWER "${CMAKE_PROJECT_NAME}" CPACK_PACKAGE_NAME)
    SET(CPACK_PACKAGE_VERSION_MAJOR ${MAJOR_VERSION})
    SET(CPACK_PACKAGE_VERSION_MINOR ${MINOR_VERSION})
    SET(CPACK_PACKAGE_VERSION_PATCH ${RELEASE_VERSION})
    SET(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
    SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "C++ program that allows to query and change the XKB layout state.")
    SET(CPACK_PACKAGE_CONTACT "islander <zombie32@gmail.com>")
    SET(CPACK_PACKAGE_VENDOR "https://github.com/grwlf/xkb-switch")
    SET(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}")

    SET(PPA_VERSION "ppa1")
    SET(CPACK_DEBIAN_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION}-${PPA_VERSION}")
    SET(CPACK_DEBIAN_PACKAGE_MAINTAINER "${CPACK_PACKAGE_CONTACT}")
    SET(CPACK_DEBIAN_PACKAGE_SECTION "misc")
    SET(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")

    execute_process(COMMAND dpkg --print-architecture
        OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE _dpkg_arch_failed)

    execute_process(COMMAND lsb_release -is
        OUTPUT_VARIABLE LSB_DISTRIBUTOR_ID OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE _lsb_release_failed)

    execute_process(COMMAND lsb_release -cs
        OUTPUT_VARIABLE CPACK_DEBIAN_DISTRIBUTION_CODENAME OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE _lsb_release_failed)

    SET(DPUT_HOST "ppa:zombie32/xkb-switch")

    LIST(APPEND CPACK_DEBIAN_BUILD_DEPENDS "libx11-dev" "libxkbfile-dev")
    SET(CPACK_DEBIAN_PACKAGE_NAME "${CPACK_PACKAGE_NAME}")
    STRING(TOLOWER "${LSB_DISTRIBUTOR_ID}" CPACK_DEBIAN_DISTRIBUTION_NAME)
    SET(CPACK_DEBIAN_DISTRIBUTION_RELEASES ${CPACK_DEBIAN_DISTRIBUTION_CODENAME})

    INCLUDE(CPack)
    INCLUDE(DebSourcePPA)
endif()
