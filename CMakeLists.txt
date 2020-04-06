# Asio can be a header-only library (here we configured to be header only, 
# without any generated binaries).
# See https://dominikberner.ch/cmake-interface-lib/ for the basics on 
# configuring a header-only library.

cmake_minimum_required(VERSION 3.12)

include(GNUInstallDirs)

project("Asio"
    VERSION      1.14.0
    DESCRIPTION  "Asio is a cross-platform C++ library for network and low-level I/O programming."
    HOMEPAGE_URL "http://think-async.com/Asio/")

add_library(${PROJECT_NAME} INTERFACE)

# Name alias so targets can always be used with namespace
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# BUILD_INTERFACE:   include path if being used to build
# INSTALL_INTERFACE: include path if being used as an installation (relative to install prefix)
target_include_directories(${PROJECT_NAME} INTERFACE 
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/asio/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

target_compile_features(${PROJECT_NAME} INTERFACE cxx_std_17)

# Specifying rules for installing targets from a project
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}_exported_targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

# So that Boost is not required.
# Note that with C++11 and later, ASIO_STANDALONE be defined automatically;
# however we define it anyway just to be sure. 
target_compile_definitions(${PROJECT_NAME} INTERFACE 
    ASIO_STANDALONE)

# FIXME: currently VS produces deprecation warning, which is wrong, see
# https://github.com/chriskohlhoff/asio/issues/290#issuecomment-371867040
# remove this when it is fixed
if(WIN32)
    target_compile_definitions(${PROJECT_NAME} INTERFACE 
        _SILENCE_CXX17_ALLOCATOR_VOID_DEPRECATION_WARNING)
endif()

# Automatically define the value of _WIN32_WINNT for Windows; this is required
# by Asio. 
# Reference: https://stackoverflow.com/questions/9742003/platform-detection-in-cmake
if(WIN32)
    macro(get_WIN32_WINNT result)
        if(CMAKE_SYSTEM_VERSION)
            set(version ${CMAKE_SYSTEM_VERSION})
            string(REGEX MATCH "^([0-9]+).([0-9])" version ${version})
            string(REGEX MATCH "^([0-9]+)" majorVersion ${version})

            # Check for Windows 10 specifically as we will need to convert to hex 'A'
            if ("${majorVersion}" MATCHES "10")
                set(majorVersion "A")
                string(REGEX REPLACE "^([0-9]+)" ${majorVersion} version ${version})
            endif()

            # Remove all remaining '.' characters
            string(REPLACE "." "" version ${version})

            # Prepend each digit with a zero
            string(REGEX REPLACE "([0-9A-Z])" "0\\1" version ${version})

            # Prepend "0x" to signify hexadecimal
            set(${result} "0x${version}")
        endif()
    endmacro()

    get_WIN32_WINNT(win32winntVersion)
    target_compile_definitions(${PROJECT_NAME} INTERFACE 
        _WIN32_WINNT=${win32winntVersion})
endif()

# Note that using "-config" and "-config-version" suffixes are mandatory 
# for Cmake to find them automatically (target file is included manually by
# the config file and is not required to have the "-targets" suffix, however
# it's good practice to follow the same convention)

include(CMakePackageConfigHelpers)

write_basic_package_version_file(${PROJECT_NAME}-config-version.cmake
    VERSION       ${PROJECT_VERSION}
    COMPATIBILITY ExactVersion)

configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}-config.cmake.in
    ${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/cmake/)

# Create/Install the export file
install(EXPORT ${PROJECT_NAME}_exported_targets
    FILE        ${PROJECT_NAME}-targets.cmake
    NAMESPACE   ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/cmake/)

# Basically copying cmake files to the desired location
install(FILES
    ${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
    ${PROJECT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
    DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/cmake/)

# Basically copying headers to the desired location
install(DIRECTORY ${PROJECT_SOURCE_DIR}/asio/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    PATTERN "asio/*"
    PATTERN ".gitignore" EXCLUDE
    PATTERN "Makefile.am" EXCLUDE)
