# *******************************************
# ************* CMake Content ***************
# *******************************************
# This CMake setups the following libraries
# 
# - gegelati

set(GEGELATI_VERSION "2.0.0")
set(GEGELATI_MSVC_ARCHIVE "gegelatilib-msvc-${GEGELATI_VERSION}.zip")
set(GEGELATI_MSVC_URL "https://github.com/gegelati/gegelati/releases/download/v${GEGELATI_VERSION}/${GEGELATI_MSVC_ARCHIVE}")

# *******************************************
# *********** GEGELATI LIBRARY **************
# *******************************************

if(WIN32)

    # find the Gegelati-x.y.z folder in the lib directory.
    file(GLOB GEGELATI_ROOT_DIR "${LIBS_DIR}/gegelatilib-[\\.|0-9]*")
    set(ENV{GEGELATI_DIR} ${GEGELATI_ROOT_DIR})

    # if Gegelati lib folder doesn't exist
    if(GEGELATI_ROOT_DIR MATCHES "$^")

        if(MSVC)
            set(GEGELATI_ARCHIVE ${GEGELATI_MSVC_ARCHIVE})
            set(GEGELATI_URL ${GEGELATI_MSVC_URL})

            message("Downloading GEGELATI library from ${GEGELATI_URL}")
            file(DOWNLOAD ${GEGELATI_URL} "${LIBS_DIR}/${GEGELATI_ARCHIVE}")

            message("Extracting GEGELATI library")
            file(ARCHIVE_EXTRACT
                INPUT "${LIBS_DIR}/${GEGELATI_ARCHIVE}"
                DESTINATION ${LIBS_DIR})
            file(REMOVE ${LIBS_DIR}/${GEGELATI_ARCHIVE})

            file(GLOB GEGELATI_ROOT_DIR "${LIBS_DIR}/gegelatilib-[\\.|0-9]*")
            set(ENV{GEGELATI_DIR} ${GEGELATI_ROOT_DIR})
        endif()
    endif()
endif()

find_package(GEGELATI QUIET)

if(NOT GEGELATI_FOUND)
    message("-- Building GEGELATI")
    # Setup git clone
    configure_file("${LIBS_DIR}/CMakeLists.txt.in" ${CMAKE_CURRENT_BINARY_DIR}/gegelati-download/CMakeLists.txt)
    execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
        RESULT_VARIABLE result
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/gegelati-download)
    if(result)
        message(FATAL_ERROR "CMake step for gegelati failed: ${result}")
    endif()
    # Do git clone
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
        RESULT_VARIABLE result
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/gegelati-download)
    if(result)
        message(FATAL_ERROR "Build step for gegelati failed: ${result}")
    endif()
    # Configure gegelati cmake
    execute_process(COMMAND ${CMAKE_COMMAND} ../gegelati-src -G "${CMAKE_GENERATOR}" -DSKIP_DOXYGEN_BUILD=ON -DBUILD_TESTING=OFF
        RESULT_VARIABLE result
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/gegelati-build)
    if(result)
        message(FATAL_ERROR "Build step for gegelati failed: ${result}")
    endif()

    # Build gegelati 
    if(WIN32)
        execute_process(COMMAND ${CMAKE_COMMAND} --build . --target install -j
            RESULT_VARIABLE result
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/gegelati-build)
        if(result)
            message(FATAL_ERROR "Build step for gegelati failed: ${result}")
        endif()
        # Copy to lib folder
        file(GLOB SRC_GEGELATI_ROOT_DIR "${CMAKE_CURRENT_BINARY_DIR}/gegelati-build/gegelatilib-[\\.|0-9]*")
        get_filename_component(GEGELATI_DIR_NAME "${SRC_GEGELATI_ROOT_DIR}" NAME)
        file(COPY "${SRC_GEGELATI_ROOT_DIR}/" DESTINATION "${LIBS_DIR}/${GEGELATI_DIR_NAME}")
    else()
        execute_process(COMMAND ${CMAKE_COMMAND} --build . --target install DESTDIR=${CMAKE_CURRENT_BINARY_DIR}/gegelati-build/gegelatilib-${GEGELATI_VERSION} -j
            RESULT_VARIABLE result
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/gegelati-build)
        if(result)
            message(FATAL_ERROR "Build step for gegelati failed: ${result}")
        endif()
        # Copy to lib folder
        file(GLOB SRC_GEGELATI_ROOT_DIR "${CMAKE_CURRENT_BINARY_DIR}/gegelati-build/gegelatilib-${GEGELATI_VERSION}/usr/local")
        get_filename_component(GEGELATI_DIR_NAME "${SRC_GEGELATI_ROOT_DIR}" NAME)
        file(COPY "${SRC_GEGELATI_ROOT_DIR}/" DESTINATION "${LIBS_DIR}/gegelatilib-${GEGELATI_VERSION}")
    endif()


    file(GLOB GEGELATI_ROOT_DIR "${LIBS_DIR}/gegelatilib-[\\.|0-9]*")
    set(ENV{GEGELATI_DIR} ${GEGELATI_ROOT_DIR})
endif()

find_package(GEGELATI REQUIRED)
set(GEGELATI_INCLUDE_DIRS ${GEGELATI_INCLUDE_DIRS})

if(NOT GEGELATI_FOUND)
    MESSAGE(FATAL_ERROR "GEGELATI not found !")
endif()

if(WIN32)
    file(GLOB
        GEGELATI_DLL
        ${GEGELATI_ROOT_DIR}/bin/*.dll
        ${GEGELATI_ROOT_DIR}/lib/*.dll
    )

    MESSAGE("Copy GEGELATI DLLs into ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
    file(COPY ${GEGELATI_DLL} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
	if(${CMAKE_GENERATOR} MATCHES "Visual Studio.*")
		file(COPY ${GEGELATI_DLL} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/Debug)
		file(COPY ${GEGELATI_DLL} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/Release)
	endif()
endif()
