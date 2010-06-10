####################################
# cmake file for building CED
# @author Jan Engels, DESY
####################################


########################################################
# CMake compatibility issues: don't modify this, please!
CMAKE_MINIMUM_REQUIRED( VERSION 2.4.6 )
#SET( CMAKE_BACKWARDS_COMPATIBILITY 2.4.6 )
MARK_AS_ADVANCED(CMAKE_BACKWARDS_COMPATIBILITY)
# allow more human readable "if then else" constructs
SET( CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS TRUE )
########################################################

# project name
PROJECT( CED C )

# project options
OPTION( BUILD_SHARED_LIBS "Set to OFF to build static libraries" ON )
OPTION( BUILD_32BIT_COMPATIBLE "Set to ON to build in 32 bit compatibility mode" ON )
OPTION( CED_SERVER "Set to ON to build the glced server" OFF )

# project version
SET( CED_MAJOR_VERSION 0 )
SET( CED_MINOR_VERSION 6 )
SET( CED_PATCH_LEVEL 0 )

# set default install prefix to project root directory
IF( CMAKE_INSTALL_PREFIX STREQUAL "/usr/local" )
    SET( CMAKE_INSTALL_PREFIX "${PROJECT_SOURCE_DIR}" )
ENDIF()

#---------------- 32/64 bit issues ---------------------------------------
#INCLUDE( CheckTypeSize )
#CHECK_TYPE_SIZE( "void*" SIZE_OF_VOID_P )

IF( CMAKE_SIZEOF_VOID_P EQUAL 4 )
    MESSAGE( STATUS "32 bit architecture detected" )
ENDIF()

IF( CMAKE_SIZEOF_VOID_P EQUAL 8 )
    MESSAGE( STATUS "64 bit architecture detected" )

    IF( BUILD_32BIT_COMPATIBLE )
        IF( COMMAND SET_PROPERTY )
            SET_PROPERTY(GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS 0)
            SET( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32" )
        ELSE()
            MESSAGE( FATAL_ERROR "to build in 32 bit compatibility mode you need cmake >= 2.6" )
        ENDIF()
        MESSAGE( STATUS "Build in 32 bit compatibility mode" )
    ELSE()
        MESSAGE( STATUS "Build in native 64 bit mode" )
    ENDIF()
ENDIF()

#-------------------------------------------------------------------------


# library *nix style versioning
SET( CED_SOVERSION "${CED_MAJOR_VERSION}.${CED_MINOR_VERSION}" )
SET( CED_VERSION "${CED_SOVERSION}.${CED_PATCH_LEVEL}" )

# add install path to the rpath list
SET( CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib" )
MARK_AS_ADVANCED( CMAKE_INSTALL_RPATH )

# add install path to the rpath list (apple)
IF( APPLE )
    SET( CMAKE_INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib" )
    MARK_AS_ADVANCED( CMAKE_INSTALL_NAME_DIR )
ENDIF()

# append link pathes to rpath list
SET( CMAKE_INSTALL_RPATH_USE_LINK_PATH 1 )
MARK_AS_ADVANCED( CMAKE_INSTALL_RPATH_USE_LINK_PATH )

# cmake output directories
SET( EXECUTABLE_OUTPUT_PATH "${PROJECT_BINARY_DIR}/bin" CACHE PATH
    "EXECUTABLE_OUTPUT_PATH" FORCE )
SET( LIBRARY_OUTPUT_PATH "${PROJECT_BINARY_DIR}/lib" CACHE PATH
    "LIBRARY_OUTPUT_PATH" FORCE )
MARK_AS_ADVANCED( EXECUTABLE_OUTPUT_PATH LIBRARY_OUTPUT_PATH )

# create uninstall configuration file
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/cmake_uninstall.cmake.in"
                "${PROJECT_BINARY_DIR}/cmake_uninstall.cmake"
                IMMEDIATE @ONLY )

# create uninstall target
ADD_CUSTOM_TARGET( uninstall "${CMAKE_COMMAND}" -P "${PROJECT_BINARY_DIR}/cmake_uninstall.cmake" )

# create configuration file from .in file
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/CEDConfig.cmake.in"
                "${PROJECT_BINARY_DIR}/CEDConfig.cmake" @ONLY )

# install configuration file
INSTALL( FILES "${PROJECT_BINARY_DIR}/CEDConfig.cmake" DESTINATION . )

# install header files
INSTALL( FILES  "${PROJECT_SOURCE_DIR}/ced_cli.h" DESTINATION include )

###################################################################################

# include directories
INCLUDE_DIRECTORIES( . )

# add definitions
ADD_DEFINITIONS( "-Wall" )


# -------------- CLIENT LIBRARY -------------------
ADD_LIBRARY( lib_CED ced_cli.c ced.c glut_socks.c )

ADD_CUSTOM_TARGET( lib DEPENDS lib_CED )

SET_TARGET_PROPERTIES( lib_CED PROPERTIES
    # create *nix style library versions + symbolic links
    VERSION ${CED_VERSION}
    SOVERSION ${CED_SOVERSION}
    # allow creating static and shared libs without conflicts
    CLEAN_DIRECT_OUTPUT 1
    # avoid conflicts between library and binary target names
    OUTPUT_NAME CED )

# install library
INSTALL( TARGETS lib_CED DESTINATION lib PERMISSIONS
        OWNER_READ OWNER_WRITE OWNER_EXECUTE
        GROUP_READ GROUP_EXECUTE
        WORLD_READ WORLD_EXECUTE )


# ------------------ GLCED --------------------------
IF( CED_SERVER )

    FIND_PACKAGE( GLUT )

    #SET( GLUT_FOUND FALSE )
    IF( NOT GLUT_FOUND )
        
        MESSAGE( WARNING "
            GLUT was not found on your system !

            If you have root privileges it is recommended to install an
            operating system specific package, e.g.:
              # sudo apt-get install freeglut3  # debian/ubuntu
              # yum install freeglut-devel      # scientific linux /redhat
            reconfigure CED afterwards.
            
            However if you do not have root privileges you can let cmake
            install freeglut for you (cmake version has to be >= 2.8).
            
            If none of the above options are possible, you can still get
            the freeglut source code from http://freeglut.sourceforge.net
            and install it yourself.
            
            If cmake has problems finding glut, you can set two special
            variables before calling cmake to locate libraries / header
            files in custom places, e.g.:
            export CMAKE_LIBRARY_PATH=$HOME/myglut/lib
            export CMAKE_INCLUDE_PATH=$HOME/myglut/include
            "
        )

        IF( "${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" LESS 2.8 )
            MESSAGE( WARNING " CMake version < 2.8 - cannot configure and install GLUT (freeglut) from source" )
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
            SET( GLUT_glut_LIBRARY ${GLUT_INSTALL_PREFIX}/lib/libglut.so )
            SET( GLUT_FOUND TRUE )
        ENDIF()
    ENDIF()


    FIND_PACKAGE( OpenGL )

    IF( NOT OPENGL_GLU_FOUND )
        MESSAGE( WARNING " OpenGL GLU library not found on your system" )
    ENDIF()


    IF( GLUT_FOUND AND OPENGL_FOUND )
        INCLUDE_DIRECTORIES( ${GLUT_INCLUDE_DIR} )
        ADD_EXECUTABLE( glced glced.c ced_srv.c )
        IF( GLUT_EXTERNAL )
            ADD_DEPENDENCIES( glced freeglut )
        ENDIF()
        # mac needs more than the OPENGL_glu_LIBRARY
        #TARGET_LINK_LIBRARIES( glced lib_CED ${GLUT_glut_LIBRARY} ${OPENGL_glu_LIBRARY} )
        TARGET_LINK_LIBRARIES( glced lib_CED ${GLUT_glut_LIBRARY} ${OPENGL_LIBRARIES} )
        INSTALL( TARGETS glced DESTINATION bin )
    ELSE()
        MESSAGE( ERROR " GLUT or OpenGL GLU component missing. cannot build glced. CED_SERVER set to OFF" )
        SET( CED_SERVER OFF )
    ENDIF()

ENDIF( CED_SERVER )



# ------------------ CED TESTS ------------------
# all test files should start with prefix test_
# e.g. test_ced.c test_millionhits.c
FILE( GLOB test_srcs ./test_*.c )

FOREACH( test_src_filename ${test_srcs} )

    # get filename without extension, i.e.:
    # test_ced.c -> test_ced
    GET_FILENAME_COMPONENT( test_target ${test_src_filename} NAME_WE )

    ADD_EXECUTABLE( ${test_target} ${test_src_filename} )
    TARGET_LINK_LIBRARIES( ${test_target} lib_CED )

    INSTALL( TARGETS ${test_target} DESTINATION bin )

ENDFOREACH()


###################################################################################

# display status message for important variables
MESSAGE( STATUS )
MESSAGE( STATUS "-------------------------------------------------------------------------------" )
MESSAGE( STATUS "BUILD_SHARED_LIBS = ${BUILD_SHARED_LIBS}" )
MESSAGE( STATUS "CMAKE_INSTALL_PREFIX = ${CMAKE_INSTALL_PREFIX}" )
IF( CMAKE_SIZEOF_VOID_P EQUAL 8 )
    MESSAGE( STATUS "BUILD_32BIT_COMPATIBLE = ${BUILD_32BIT_COMPATIBLE}" )
ENDIF()
MESSAGE( STATUS "CED_SERVER = ${CED_SERVER}" )
MESSAGE( STATUS "Change a value with: cmake -D<Variable>=<Value>" )
MESSAGE( STATUS "-------------------------------------------------------------------------------" )
MESSAGE( STATUS )

# force some variables that could be defined in the command line
# to be written to cache
SET( BUILD_SHARED_LIBS "${BUILD_SHARED_LIBS}" CACHE BOOL
    "Set to OFF to build static libraries" FORCE )
SET( CMAKE_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}" CACHE PATH
    "Where to install CED" FORCE )
SET( CED_SERVER "${CED_SERVER}" CACHE BOOL
    "Set to ON to build the glced server" FORCE )

# export build settings
INCLUDE( CMakeExportBuildSettings )
CMAKE_EXPORT_BUILD_SETTINGS( "CEDBuildSettings.cmake" )
INSTALL( FILES "${PROJECT_BINARY_DIR}/CEDBuildSettings.cmake" DESTINATION lib/cmake )

# export library dependencies (keep this as the last line in the file)
EXPORT_LIBRARY_DEPENDENCIES( "CEDLibDeps.cmake" )
INSTALL( FILES "${PROJECT_BINARY_DIR}/CEDLibDeps.cmake" DESTINATION lib/cmake )

