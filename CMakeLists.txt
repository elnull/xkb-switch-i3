cmake_minimum_required(VERSION 3.16)

project(XKB-SWITCH VERSION 2.0.1 LANGUAGES CXX)

# Enable modern CMake behavior
cmake_policy(SET CMP0074 NEW)

# Version string including i3 compatibility
set(I3_VERSION 5)
set(XKBSWITCH_VERSION "${PROJECT_VERSION}+i3-${I3_VERSION}")
add_definitions(-DXKBSWITCH_VERSION="${XKBSWITCH_VERSION}")

find_package(PkgConfig REQUIRED)
find_package(X11 REQUIRED)
pkg_check_modules(XKBFILE REQUIRED xkbfile)

add_subdirectory(i3ipc++)

# Option to build library for vim's libcall interface
option(BUILD_XKBSWITCH_LIB "Build a library compatible with vim's libcall interface" ON)

# Build shared library if option is enabled
if(BUILD_XKBSWITCH_LIB)
    add_library(xkbswitch SHARED src/XKbSwitchApi.cpp src/XKeyboard.cpp src/Utils.hpp)
    set_target_properties(xkbswitch PROPERTIES
        VERSION ${PROJECT_VERSION}
        SOVERSION ${PROJECT_VERSION_MAJOR}
        POSITION_INDEPENDENT_CODE ON
    )
    target_include_directories(xkbswitch PUBLIC ${X11_INCLUDE_DIR} ${XKBFILE_INCLUDE_DIRS})
    target_link_libraries(xkbswitch PUBLIC X11::Xkb ${XKBFILE_LIBRARIES})
endif()

# Build executable
add_executable(xkb-switch src/XKbSwitch.cpp src/XKeyboard.cpp src/Utils.hpp)
target_include_directories(xkb-switch PRIVATE ${X11_INCLUDE_DIR} ${XKBFILE_INCLUDE_DIRS})
target_link_libraries(xkb-switch PRIVATE X11::Xkb ${XKBFILE_LIBRARIES} i3ipc++)

# Installation rules
install(TARGETS xkb-switch xkbswitch
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
)

# Install header files
install(FILES src/XKeyboard.hpp src/Utils.hpp DESTINATION include)

# Man page compression settings
set(MAN_COMPRESSION "gzip" CACHE STRING "Manpages compression tool")
set(MANDIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Manpages installation path")

# Function to compress and install man pages
function(install_man man_filename man_type)
    # Check for available compression tool
    find_program(COMPRESS_EXECUTABLE NAMES ${MAN_COMPRESSION})
    if(NOT COMPRESS_EXECUTABLE)
        set(MAN_COMPRESSION "NO")
    endif()

    set(raw_man man/${man_filename}.${man_type})

    if(MAN_COMPRESSION AND NOT "${MAN_COMPRESSION}" STREQUAL "NO")
        if(COMPRESS_EXECUTABLE MATCHES "gzip")
            set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.gz)
        else()
            set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.${MAN_COMPRESSION})
        endif()

        add_custom_command(OUTPUT ${installed_man}
            COMMAND ${CMAKE_COMMAND} -E echo "Compressing ${raw_man}..."
            COMMAND ${COMPRESS_EXECUTABLE} -c ${raw_man} > ${installed_man}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Compressing man file ${raw_man} to ${installed_man}"
        )
    else()
        set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type})
        message(WARNING "There is no compression tool for man pages. Not compressed copy of man file will be used.")
        add_custom_command(OUTPUT ${installed_man}
            COMMAND ${CMAKE_COMMAND} -E copy ${raw_man} ${installed_man}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Copying man file from ${raw_man}."
        )
    endif()

    add_custom_target(man_${man_filename}_${man_type} ALL DEPENDS ${installed_man})
    install(FILES ${installed_man} DESTINATION ${MANDIR}/man${man_type})
endfunction()

# Install man page
install_man(xkb-switch 1)
