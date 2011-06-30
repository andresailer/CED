########################################################
# cmake file for building CED
# @author Jan Engels, Desy IT
CMAKE_MINIMUM_REQUIRED(VERSION 2.6 FATAL_ERROR)
########################################################


# project name
PROJECT( CED )

# project version
SET( ${PROJECT_NAME}_VERSION_MAJOR 1 )
SET( ${PROJECT_NAME}_VERSION_MINOR 2 )
SET( ${PROJECT_NAME}_VERSION_PATCH 0 )



### DEPENDENCIES ############################################################

FIND_PACKAGE( ILCUTIL COMPONENTS ILCSOFT_CMAKE_MODULES REQUIRED )

# GLUT and OpenGL are required by glced
FIND_PACKAGE( GLUT QUIET )
FIND_PACKAGE( OpenGL QUIET )

# load default settings from ILCSOFT_CMAKE_MODULES
INCLUDE( ilcsoft_default_settings )



### DOCUMENTATION ###########################################################

#OPTION( INSTALL_DOC "Set to OFF to skip build/install Documentation" ON )
#
#IF( INSTALL_DOC AND EXISTS "${PROJECT_SOURCE_DIR}/doc/CMakeLists.txt" )
#    ADD_SUBDIRECTORY( doc )
#ELSE()
#    MESSAGE( STATUS "INSTALL_DOC set to OFF" )
#    SET( INSTALL_DOC OFF )
#ENDIF()



### LIBRARY AND TOOLS #######################################################

# include directories
INCLUDE_DIRECTORIES( ./include )
INCLUDE_DIRECTORIES( ./src )
INSTALL_DIRECTORY( ./include DESTINATION . FILES_MATCHING PATTERN "*.h" )


# definitions to pass to the compiler
ADD_DEFINITIONS( "-Wall" ) # -ansi -pedantic



# --------- CED CLIENT LIBRARY ----------------------
AUX_SOURCE_DIRECTORY( ./src/client client_sources )
ADD_SHARED_LIBRARY( CED ${client_sources} )
INSTALL_SHARED_LIBRARY( CED DESTINATION lib )


# link CED client library to the math library
FIND_LIBRARY( c_math_lib NAMES m )
TARGET_LINK_LIBRARIES( CED ${c_math_lib} )
MARK_AS_ADVANCED( c_math_lib )



# --------- CED SERVER (GLCED) ----------------------
OPTION( CED_SERVER "Set to ON to build the glced server" ON )
MESSAGE( STATUS "CED_SERVER = ${CED_SERVER}" )

IF( CED_SERVER )

    IF( NOT GLUT_FOUND OR NOT OPENGL_FOUND )

        #MESSAGE( FATAL_ERROR "could not find packages glut + opengl required to build glced" )

        MESSAGE( "" )
        MESSAGE( "GLUT was not found on your system !" )
        MESSAGE( "" )

        IF( "${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" LESS 2.8 )
        MESSAGE( "

            Your cmake version is less than 2.8. To install freeglut
            automatically you need cmake >= 2.8.

            However if you have root privileges it is recommended to install
            an operating system supported package, e.g.:
              # sudo apt-get install freeglut3  # debian/ubuntu
              # yum install freeglut-devel      # scientific linux /redhat

            You need to reconfigure CED afterwards.
            
            If you do not have cmake >= 2.8 neither root privileges, you can
            get the freeglut source code from http://freeglut.sourceforge.net
            and install freeglut manually.
            
            In order to tell cmake how to find glut you need to set two special
            variables before calling cmake to locate libraries / header
            files in custom places, e.g.:
            export CMAKE_LIBRARY_PATH=$HOME/myglut/lib
            export CMAKE_INCLUDE_PATH=$HOME/myglut/include
            "
        )

 
            MESSAGE( FATAL_ERROR " CMake version < 2.8 - cannot configure and install GLUT (freeglut) from source" )
        ELSE()
            include( ExternalProject )

            SET( GLUT_EXTERNAL TRUE )

            IF( NOT DEFINED GLUT_VERSION)
                SET( GLUT_VERSION "2.6.0" )
            ENDIF()

            #SET( GLUT_DOWNLOAD_MIRROR "http://sourceforge.net/projects/freeglut/files/freeglut" )
            SET( GLUT_DOWNLOAD_MIRROR "http://ignum.dl.sourceforge.net/project/freeglut/freeglut" )
            SET( GLUT_DOWNLOAD_URL ${GLUT_DOWNLOAD_MIRROR}/freeglut-${GLUT_VERSION}.tar.gz )
            #SET( GLUT_INSTALL_PREFIX ${PROJECT_BINARY_DIR} )
            SET( GLUT_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX} )
            SET( GLUT_CONFIGURE_COMMAND ./configure --prefix=${GLUT_INSTALL_PREFIX} )

            MESSAGE( STATUS "GLUT (freeglut) will be configured and built from source" )
            MESSAGE( STATUS "GLUT_VERSION: ${GLUT_VERSION} (chage with cmake -DGLUT_VERSION=2.6.0)" )
            MESSAGE( STATUS "GLUT_DOWNLOAD_MIRROR: ${GLUT_DOWNLOAD_MIRROR}" )
            MESSAGE( STATUS "GLUT_DOWNLOAD_URL: ${GLUT_DOWNLOAD_URL}" )
            MESSAGE( STATUS "GLUT_INSTALL_PREFIX: ${GLUT_INSTALL_PREFIX}" )
            MESSAGE( STATUS "GLUT_CONFIGURE_COMMAND ${GLUT_CONFIGURE_COMMAND}" )

            ExternalProject_Add( freeglut
                URL ${GLUT_DOWNLOAD_URL}
                CONFIGURE_COMMAND ${GLUT_CONFIGURE_COMMAND}
                BUILD_IN_SOURCE 1
                INSTALL_COMMAND make install
            )
            # FIXME use FIND_XXX
            SET( GLUT_INCLUDE_DIR ${GLUT_INSTALL_PREFIX}/include )
            SET( GLUT_INCLUDE_DIRS ${GLUT_INCLUDE_DIR} )
            SET( GLUT_glut_LIBRARY ${GLUT_INSTALL_PREFIX}/lib/libglut.so )
            SET( GLUT_LIBRARIES ${GLUT_glut_LIBRARY} )
            SET( GLUT_FOUND TRUE )
        ENDIF()
    ENDIF()

    INCLUDE_DIRECTORIES( ${GLUT_INCLUDE_DIR} )

    # copy paths from OPENGL_LIBRARIES to rpath list of glced to avoid system
    # opengl libraries used in the linking process of glced to be overwriten
    # by LD_LIBRARY_PATH, e.g.: LD_LIBRARY_PATH set by geant4 env_init_script
    # points to old mesa libraries in afs
    FOREACH( _opengl_lib ${OPENGL_LIBRARIES} )
        GET_FILENAME_COMPONENT( _opengl_lib_path ${_opengl_lib} PATH )
        LIST( APPEND _opengl_linker_paths ${_opengl_lib_path} )
    ENDFOREACH()

    # remove duplicate paths
    IF( _opengl_linker_paths )
        LIST( REMOVE_DUPLICATES _opengl_linker_paths )
    ENDIF()

    #MESSAGE( "_opengl_linker_paths: ${_opengl_linker_paths}" )

    # this would affect all targets
    #LIST( APPEND CMAKE_INSTALL_RPATH "${_opengl_linker_paths}" )
    #IF( APPLE )
    #    LIST( APPEND CMAKE_INSTALL_NAME_DIR "${_opengl_linker_paths}" )
    #ENDIF()

    AUX_SOURCE_DIRECTORY( ./src/server server_sources )
    ADD_EXECUTABLE( glced ${server_sources} )

    SET_TARGET_PROPERTIES( glced PROPERTIES
        INSTALL_RPATH "${CMAKE_INSTALL_RPATH};${_opengl_linker_paths}"
        INSTALL_NAME_DIR "${CMAKE_INSTALL_NAME_DIR};${_opengl_linker_paths}"
    )

    TARGET_LINK_LIBRARIES( glced CED 
        ${GLUT_glut_LIBRARY}
        ${OPENGL_LIBRARIES}
    )

    INSTALL( TARGETS glced DESTINATION bin )

ENDIF( CED_SERVER )


# ------------------ CED TESTS ------------------
FOREACH( _testname test_ced test_ced_mhits )

    IF( BUILD_TESTING )
        ADD_EXECUTABLE( ${_testname} ./src/tests/${_testname}.cc )
    ELSE()
        ADD_EXECUTABLE( ${_testname} EXCLUDE_FROM_ALL ./src/tests/${_testname}.cc )
    ENDIF()

    TARGET_LINK_LIBRARIES( ${_testname} CED )

    INSTALL( TARGETS ${_testname} DESTINATION bin )

ENDFOREACH()



# display some variables and write them to cache
DISPLAY_STD_VARIABLES()


# generate and install following configuration files
GENERATE_PACKAGE_CONFIGURATION_FILES( CEDConfig.cmake CEDConfigVersion.cmake CEDLibDeps.cmake )

