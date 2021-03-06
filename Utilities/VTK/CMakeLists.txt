PROJECT(vtkgdcm)
# We need VTK
# Technically we require VTK 5.0 and above
FIND_PACKAGE(VTK REQUIRED)
MARK_AS_ADVANCED(VTK_DIR)

# Define the src for the vtk-gdcm bridge
SET(vtkgdcm_SRCS
  vtkGDCMTesting.cxx
  vtkGDCMImageReader.cxx
  vtkGDCMImageWriter.cxx
  vtkGDCMMedicalImageProperties.cxx
  )

IF(GDCM_HAVE_PTHREAD_H AND CMAKE_USE_PTHREADS)
  SET(vtkgdcm_SRCS ${vtkgdcm_SRCS}
  vtkGDCMThreadedImageReader.cxx
  )
ENDIF(GDCM_HAVE_PTHREAD_H AND CMAKE_USE_PTHREADS)

INCLUDE(${VTK_USE_FILE})

INCLUDE(CheckCXXSourceCompiles)
SET(CMAKE_REQUIRED_INCLUDES ${VTK_INCLUDE_DIRS})
#SET(CMAKE_REQUIRED_LIBRARIES vtkVolumeRendering) # can't get it to work
CHECK_CXX_SOURCE_COMPILES(
  "\#include <vtkSmartVolumeMapper.h>\nint main() { vtkSmartVolumeMapper* p; return 0;}"
  VTK_HAS_SMARTVOLUMEMAPPER)
CHECK_CXX_SOURCE_COMPILES(
  "\#include <vtkImageResliceMapper.h>\nint main() { vtkImageResliceMapper* p; return 0;}"
  VTK_HAS_IMAGERESLICEMAPPER)

INCLUDE_DIRECTORIES(
  ${GDCM_BINARY_DIR}/Source/Common
  ${GDCM_SOURCE_DIR}/Source/Common
  ${GDCM_SOURCE_DIR}/Source/DataStructureAndEncodingDefinition
  ${GDCM_SOURCE_DIR}/Source/MediaStorageAndFileFormat
  ${GDCM_SOURCE_DIR}/Source/DataDictionary
  ${GDCM_SOURCE_DIR}/Utilities/VTK
  )

# FIXME: temp fix
#IF(UNIX)
#  LINK_DIRECTORIES(/usr/X11R6/lib)
#ENDIF(UNIX)

# List the kits from VTK that are needed by this project.
SET(vtkgdcm_LIBS
  vtkCommon
  vtkIO
  vtkImaging
)
IF(VTK_USE_RENDERING)
  SET(vtkgdcm_LIBS
    ${vtkgdcm_LIBS}
    vtkRendering
    )
ENDIF(VTK_USE_RENDERING)

# Use wrapping hints for this project.
#SET(VTK_WRAP_HINTS "${PROJECT_SOURCE_DIR}/hints")

SET(VTKGDCM_NAME vtkgdcm CACHE STRING "vtk-gdcm lib name")
MARK_AS_ADVANCED(VTKGDCM_NAME)

# Create the instantiator for these classes.
# FIXME: Are instantiator really needed when only doing python wrapping ?
IF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
  SET(vtkgdcm_SRCS ${vtkgdcm_SRCS}
    ${CMAKE_CURRENT_SOURCE_DIR}/VTK4/vtkMedicalImageProperties.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/VTK4/vtkStringArray.cxx
  )
  # Setup vtkInstantiator registration for this library's classes.
  INCLUDE_DIRECTORIES(
    ${GDCM_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/VTK4
  )
  VTK_MAKE_INSTANTIATOR2(
    "${VTKGDCM_NAME}Instantiator"
    vtkgdcmInstantiator_SRCS
    "${vtkgdcm_SRCS}"
    EXPORT_MACRO "" #GDCM_EXPORT
    HEADER_LOCATION ${GDCM_BINARY_DIR}
  )
ELSE( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
  SET(vtkgdcm_SRCS ${vtkgdcm_SRCS}
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkImageMapToWindowLevelColors2.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkImageYBRToRGB.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkImageRGBToYBR.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkGDCMPolyDataReader.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkGDCMPolyDataWriter.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkRTStructSetProperties.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkLookupTable16.cxx
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkImageMapToColors16.cxx
    #${CMAKE_CURRENT_SOURCE_DIR}/vtkImagePlanarComponentsToComponents.cxx
  )
  IF(VTK_USE_RENDERING)
  SET(vtkgdcm_SRCS ${vtkgdcm_SRCS}
    ${CMAKE_CURRENT_SOURCE_DIR}/vtkImageColorViewer.cxx
    )
  ENDIF(VTK_USE_RENDERING)

  #IF(GDCM_HAVE_PTHREAD_H)
    SET(vtkgdcm_SRCS ${vtkgdcm_SRCS}
    vtkGDCMThreadedImageReader2.cxx
    )
  #ENDIF(GDCM_HAVE_PTHREAD_H)

  SET(VTK_USE_INSTANTIATOR_NEW 1)
  VTK_MAKE_INSTANTIATOR3(
    "${VTKGDCM_NAME}Instantiator"
    vtkgdcmInstantiator_SRCS
    "${vtkgdcm_SRCS}"
    "" #"VTK_${VTKGDCM_NAME}_EXPORT"
    ${PROJECT_BINARY_DIR}
    "" #"${VTKGDCM_NAME}Configure.h"
  )
ENDIF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )

#Hum... not sure why this is needed.
#IF(NOT VTK_BUILD_SHARED_LIBS AND GDCM_BUILD_SHARED_LIBS)
#  ADD_LIBRARY(vtkgdcm STATIC ${vtkgdcm_SRCS} ${vtkgdcmInstantiator_SRCS})
#ELSE(NOT VTK_BUILD_SHARED_LIBS AND GDCM_BUILD_SHARED_LIBS)
  ADD_LIBRARY(${VTKGDCM_NAME} ${vtkgdcm_SRCS} ${vtkgdcmInstantiator_SRCS})
#ENDIF(NOT VTK_BUILD_SHARED_LIBS AND GDCM_BUILD_SHARED_LIBS)

SET_TARGET_PROPERTIES(${VTKGDCM_NAME} PROPERTIES ${GDCM_LIBRARY_PROPERTIES})
IF(GDCM_USE_ACTIVIZ AND UNIX)
  # the following only works with Activiz VTK 5.4.2. The pure VTK lib are now shipped as separate libs.
  # so one can safely link to them, instead of having to build a VTK by hand, link vtkgdcm to it
  # and hope compiler configuration are compatible.
  # if you use Activiz/VTK 5.2.1 you are on your own (see gdcm 2.0.12 which supports Activiz 5.2.1)
  FIND_PACKAGE(ACTIVIZ REQUIRED)
  #message(${ACTIVIZ_KITWARE_VTK_LIBRARY})
  get_filename_component(bla ${ACTIVIZ_KITWARE_VTK_LIBRARY}
    PATH)

  #LINK_DIRECTORIES(${bla})
  #TARGET_LINK_LIBRARIES(${VTKGDCM_NAME} gdcmMSFF
  #  ${bla}/libvtkCommon.so
  #  #${bla}/libvtkIO.so
  #  ${bla}/libvtkImaging.so
  #  ${bla}/libvtkRendering.so
  #  )
  TARGET_LINK_LIBRARIES(${VTKGDCM_NAME} gdcmMSFF ${vtkgdcm_LIBS})
ELSE(GDCM_USE_ACTIVIZ AND UNIX)
  TARGET_LINK_LIBRARIES(${VTKGDCM_NAME} gdcmMSFF ${vtkgdcm_LIBS})
ENDIF(GDCM_USE_ACTIVIZ AND UNIX)

# prevent viral dep of vtkgdcm
SET_PROPERTY(TARGET ${VTKGDCM_NAME} PROPERTY LINK_INTERFACE_LIBRARIES "")
IF(NOT GDCM_INSTALL_NO_LIBRARIES)
  INSTALL(TARGETS ${VTKGDCM_NAME}
    EXPORT ${GDCM_TARGETS_NAME}
    RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT VTKLibraries
    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKLibraries #${NAMELINK_SKIP}
    ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKDevelopment
  )
#if(NAMELINK_ONLY)
#  INSTALL(TARGETS vtkgdcm
#    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Development ${NAMELINK_ONLY}
#  )
#  endif(NAMELINK_ONLY)
ENDIF(NOT GDCM_INSTALL_NO_LIBRARIES)

IF(NOT GDCM_INSTALL_NO_DEVELOPMENT)
  SET(header_files_glob "*.h" "*.txx")
  IF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
    SET(header_files_glob ${header_files_glob}
      "VTK4/*.h"
    )
  ENDIF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
  FILE(GLOB header_files ${header_files_glob})
  INSTALL(FILES ${header_files}
    DESTINATION ${GDCM_INSTALL_INCLUDE_DIR} COMPONENT VTKHeaders
  )
ENDIF(NOT GDCM_INSTALL_NO_DEVELOPMENT)

IF(GDCM_WRAP_PHP)
  IF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
    MESSAGE(FATAL_ERROR "you need a newer VTK version >= 5.0")
  ENDIF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )

FIND_PACKAGE(PHP5 REQUIRED)
INCLUDE_DIRECTORIES(
  ${PHP5_INCLUDE_PATH}
)

  FIND_PACKAGE(SWIG REQUIRED)
  MARK_AS_ADVANCED(SWIG_DIR SWIG_EXECUTABLE SWIG_VERSION)
  INCLUDE(${SWIG_USE_FILE})
  SET_SOURCE_FILES_PROPERTIES(vtkgdcm.i PROPERTIES CPLUSPLUS ON)
  #SET_SOURCE_FILES_PROPERTIES(vtkgdcm.i PROPERTIES COMPILE_FLAGS -DUSEACTIVIZ)
  #IF(GDCM_USE_ACTIVIZ)
  #SET(CMAKE_SWIG_FLAGS "-namespace vtkgdcm -dllimport vtkgdcmsharpglue -DUSEACTIVIZ")
  #ELSE(GDCM_USE_ACTIVIZ)
  #SET(CMAKE_SWIG_FLAGS "-namespace vtkgdcm -dllimport vtkgdcmsharpglue")
  #ENDIF(GDCM_USE_ACTIVIZ)
  #SEPARATE_ARGUMENTS(CMAKE_SWIG_FLAGS)

  SWIG_ADD_MODULE(php_vtkgdcm php vtkgdcm.i)
  SWIG_LINK_LIBRARIES(php_vtkgdcm vtkgdcm)
  IF(UNIX)
  SET_TARGET_PROPERTIES(${SWIG_MODULE_php_vtkgdcm_REAL_NAME} PROPERTIES OUTPUT_NAME "vtkgdcm")
  ENDIF(UNIX)
  SET_TARGET_PROPERTIES(${SWIG_MODULE_php_vtkgdcm_REAL_NAME} PROPERTIES PREFIX "")
  SET_TARGET_PROPERTIES(${SWIG_MODULE_php_vtkgdcm_REAL_NAME} PROPERTIES LINK_INTERFACE_LIBRARIES "")

IF(NOT GDCM_INSTALL_NO_LIBRARIES)
  INSTALL(TARGETS ${SWIG_MODULE_php_vtkgdcm_REAL_NAME}
      EXPORT ${GDCM_TARGETS_NAME}
    RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT Applications
    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Libraries #${NAMELINK_SKIP}
    ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Development
  )
#if(NAMELINK_ONLY)
#  INSTALL(TARGETS ${SWIG_MODULE_vtkgdcmsharpglue_REAL_NAME}
#    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Development ${NAMELINK_ONLY}
#  )
#  endif(NAMELINK_ONLY)

# See gdcm bug #3175803
add_custom_command(TARGET ${SWIG_MODULE_php_vtkgdcm_REAL_NAME}
  PRE_BUILD
  COMMAND sed -i -e 's/zend_error_noreturn/zend_error/g' "${swig_generated_file_fullname}"
  COMMENT "Patching zend_error_noreturn into zend_error"
  )

# Let's copy vtkgdcm.php into the bin dir:
ADD_CUSTOM_COMMAND(
  OUTPUT ${LIBRARY_OUTPUT_PATH}/vtkgdcm.php
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/vtkgdcm.php ${LIBRARY_OUTPUT_PATH}/vtkgdcm.php
  DEPENDS "${swig_generated_file_fullname}"
  COMMENT "copying vtkgdcm.php"
)
ADD_CUSTOM_TARGET(VTKGDCMPHP ALL
  DEPENDS ${LIBRARY_OUTPUT_PATH}/vtkgdcm.php
  COMMENT "building vtkgdcm.php"
)

ENDIF(NOT GDCM_INSTALL_NO_LIBRARIES)
ENDIF(GDCM_WRAP_PHP)

IF(GDCM_WRAP_CSHARP)
  IF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
    MESSAGE(FATAL_ERROR "you need a newer VTK version >= 5.0")
  ENDIF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
  IF(GDCM_USE_ACTIVIZ)
    FIND_PACKAGE(ACTIVIZ REQUIRED)
  ELSE(GDCM_USE_ACTIVIZ)
    MESSAGE("You are using the SWIG version of VTKGDCM. This is not compatible with Activiz")
  ENDIF(GDCM_USE_ACTIVIZ)
  FIND_PACKAGE(SWIG REQUIRED)
  MARK_AS_ADVANCED(SWIG_DIR SWIG_EXECUTABLE SWIG_VERSION)
  INCLUDE(${SWIG_USE_FILE})
  SET_SOURCE_FILES_PROPERTIES(vtkgdcm.i PROPERTIES CPLUSPLUS ON)
  #SET_SOURCE_FILES_PROPERTIES(vtkgdcm.i PROPERTIES COMPILE_FLAGS -DUSEACTIVIZ)
  IF(GDCM_USE_ACTIVIZ)
  SET(CMAKE_SWIG_FLAGS "-namespace vtkgdcm -dllimport vtkgdcmsharpglue -DUSEACTIVIZ")
  ELSE(GDCM_USE_ACTIVIZ)
  SET(CMAKE_SWIG_FLAGS "-namespace vtkgdcm -dllimport vtkgdcmsharpglue")
  ENDIF(GDCM_USE_ACTIVIZ)
  SEPARATE_ARGUMENTS(CMAKE_SWIG_FLAGS)
  SWIG_ADD_MODULE(vtkgdcmsharpglue csharp vtkgdcm.i)
  SWIG_LINK_LIBRARIES(vtkgdcmsharpglue vtkgdcm)
  # Stupid cmake-swig module is doing that for us, when not needed
  IF(UNIX)
    SET_TARGET_PROPERTIES(${SWIG_MODULE_vtkgdcmsharpglue_REAL_NAME} PROPERTIES PREFIX "lib")
  ENDIF(UNIX)

IF(NOT GDCM_INSTALL_NO_LIBRARIES)
  INSTALL(TARGETS ${SWIG_MODULE_vtkgdcmsharpglue_REAL_NAME}
      EXPORT ${GDCM_TARGETS_NAME}
    RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT Applications
    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Libraries #${NAMELINK_SKIP}
    ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Development
  )
#if(NAMELINK_ONLY)
#  INSTALL(TARGETS ${SWIG_MODULE_vtkgdcmsharpglue_REAL_NAME}
#    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT Development ${NAMELINK_ONLY}
#  )
#  endif(NAMELINK_ONLY)

ENDIF(NOT GDCM_INSTALL_NO_LIBRARIES)
  CONFIGURE_FILE(
    ${CMAKE_CURRENT_SOURCE_DIR}/AssemblyInfo.cs.in
    ${CMAKE_CURRENT_BINARY_DIR}/AssemblyInfo.cs
    @ONLY)
  IF(GDCM_USE_ACTIVIZ)
  ADD_CUSTOM_COMMAND(
    OUTPUT ${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll
    COMMAND ${CMAKE_CSHARP_COMPILER} ARGS "/r:${ACTIVIZ_KITWARE_VTK_LIBRARY}" "/r:${ACTIVIZ_KITWARE_MUMMY_RUNTIME_LIBRARY}" "/t:library" "/out:${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll" "*.cs"
    #COMMAND ${CMAKE_CSHARP_COMPILER} ARGS "/t:library" "/out:${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll" "*.cs"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS "${swig_generated_file_fullname}"
      ${CMAKE_CURRENT_BINARY_DIR}/AssemblyInfo.cs
    COMMENT "csc *.cs"
  )
  ELSE(GDCM_USE_ACTIVIZ)
  ADD_CUSTOM_COMMAND(
    OUTPUT ${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll
    #COMMAND ${CMAKE_CSHARP_COMPILER} ARGS "/r:${ACTIVIZ_KITWARE_VTK_LIBRARY}" "/r:${ACTIVIZ_KITWARE_MUMMY_RUNTIME_LIBRARY}" "/t:library" "/out:${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll" "*.cs"
    COMMAND ${CMAKE_CSHARP_COMPILER} ARGS "/t:library" "/out:${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll" "*.cs"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    DEPENDS "${swig_generated_file_fullname}"
      ${CMAKE_CURRENT_BINARY_DIR}/AssemblyInfo.cs
    COMMENT "csc *.cs"
  )

  ENDIF(GDCM_USE_ACTIVIZ)

  #ADD_CUSTOM_COMMAND(
  #  OUTPUT  ${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll.config
  #  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/vtkgdcm-sharp.dll.config ${GDCM_LIBRARY_DIR}
  #  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/vtkgdcm-sharp.dll.config
  #  COMMENT "Copying vtkgdcm-sharp.dll.config"
  #)

ADD_CUSTOM_TARGET(VTKGDCMCSharp ALL
  DEPENDS
    ${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll
    #${GDCM_LIBRARY_DIR}/vtkgdcm-sharp.dll.config
    #${GDCM_EXECUTABLE_DIR}/HelloWorld.exe
  COMMENT "building vtkgdcm-sharp.dll"
)

  # because vtkgdcm-sharp.dll is constructed with custom commands, it need the INSTALL(FILES signature:
SET(GDCM_LIBRARY_DIR2 ${LIBRARY_OUTPUT_PATH}/\${BUILD_TYPE})
  INSTALL(FILES
    ${GDCM_LIBRARY_DIR2}/vtkgdcm-sharp.dll
    #${GDCM_LIBRARY_DIR2}/vtkgdcm-sharp.dll.config
    DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKCSharpModule
  )

ENDIF(GDCM_WRAP_CSHARP)

IF(GDCM_WRAP_JAVA)
  IF(VTK_WRAP_JAVA)
    FIND_PACKAGE(Java 1.5 REQUIRED) # javac, jar
    FIND_PACKAGE(JNI REQUIRED)
    INCLUDE_DIRECTORIES(${JNI_INCLUDE_DIRS})
    SET(VTK_WRAP_JAVA3_INIT_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
    # Lars Matth�us patch (package vtk => imply vtk subdir )
    SET(VTK_JAVA_HOME ${CMAKE_CURRENT_BINARY_DIR}/java/vtk)
    # This is *required* don't ask
    MAKE_DIRECTORY(${VTK_JAVA_HOME})
    INCLUDE(${VTK_CMAKE_DIR}/vtkWrapJava.cmake)
    INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})

    VTK_WRAP_JAVA3(${VTKGDCM_NAME}Java vtkgdcmJAVA_SRCS "${vtkgdcm_SRCS}")
    # libvtk-java is a pain to handle...
    if(EXISTS ${VTK_JAVA_JAR})
      # http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=533193
      set(GDCM_VTK_JAVA_JAR ${VTK_JAVA_JAR})
    endif(EXISTS ${VTK_JAVA_JAR})
    if(EXISTS /usr/lib/jni/libvtkCommonJava.so)
      # http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=533198
      message("Implicitely adding debian package layout...")
      link_directories("/usr/lib/jni")
    endif()
    MARK_AS_ADVANCED(GDCM_VTK_JAVA_JAR)
    if(EXISTS ${GDCM_VTK_JAVA_JAR})
    else(EXISTS ${GDCM_VTK_JAVA_JAR})
      message(FATAL_ERROR "Could not find vtk.jar file, VTK_JAVA_JAR is wrong: ${VTK_JAVA_JAR}, please set proper GDCM_VTK_JAVA_JAR: ${GDCM_VTK_JAVA_JAR} replacement var")
    endif(EXISTS ${GDCM_VTK_JAVA_JAR})
    ADD_LIBRARY(${VTKGDCM_NAME}Java SHARED ${vtkgdcmJAVA_SRCS})
    # special jnilib extension:
    IF(APPLE)
      SET_TARGET_PROPERTIES(${VTKGDCM_NAME}Java PROPERTIES SUFFIX ".jnilib")
    ENDIF(APPLE)
    TARGET_LINK_LIBRARIES(${VTKGDCM_NAME}Java ${VTKGDCM_NAME})
    FOREACH(c ${vtkgdcm_LIBS})
      TARGET_LINK_LIBRARIES(${VTKGDCM_NAME}Java ${c}Java)
    ENDFOREACH(c)
    # Create the jar file:
    # I am pretty sure this *.java thingy will bite me one day, when someone will try
    # to recompile from an existing build tree with invalid generated *.java file...
    ADD_CUSTOM_COMMAND(
      OUTPUT ${LIBRARY_OUTPUT_PATH}/vtkgdcm.jar
      #COMMAND ${Java_JAVAC_EXECUTABLE} ARGS -cp ${GDCM_VTK_JAVA_JAR} "vtk/*.java"
      # No such thing as -cp for javac only java is listed:
      # http://java.sun.com/j2se/1.5.0/docs/tooldocs/windows/classpath.html
      COMMAND ${Java_JAVAC_EXECUTABLE} ARGS -classpath ${GDCM_VTK_JAVA_JAR} "vtk/*.java"
      COMMAND ${Java_JAR_EXECUTABLE} ARGS -cvfm ${LIBRARY_OUTPUT_PATH}/${PROJECT_NAME}.jar ${CMAKE_CURRENT_SOURCE_DIR}/manifest.txt vtk/*.class
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/java
      DEPENDS ${VTKGDCM_NAME}Java
      #${VTK_JAVA_DEPENDENCIES}
      #vtkgdcmJavaJavaClasses
      COMMENT   "javac *.java -> jar; jar cvf -> vtkgdcm.jar"
    )
    # Target to execute custom command:
    ADD_CUSTOM_TARGET(VTKGDCMJavaJar ALL
      DEPENDS ${LIBRARY_OUTPUT_PATH}/vtkgdcm.jar
      COMMENT "building vtkgdcm.jar"
    )
    ADD_DEPENDENCIES(VTKGDCMJavaJar vtkgdcmJavaJavaClasses)

    # Install rules:
    IF(NOT GDCM_INSTALL_NO_LIBRARIES)
      INSTALL(TARGETS ${VTKGDCM_NAME}Java
      EXPORT ${GDCM_TARGETS_NAME}
        RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT VTKJavaModule
        LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKJavaModule #${NAMELINK_SKIP}
        ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKJavaModule
      )
  #Development
  #if(NAMELINK_ONLY)
  #  INSTALL(TARGETS ${VTKGDCM_NAME}Java
  #    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKJavaModule ${NAMELINK_ONLY}
  #  )
  #endif(NAMELINK_ONLY)

    # because vtkgdcm.jar is constructed with custom commands, it need the INSTALL(FILES signature:
    INSTALL(FILES ${LIBRARY_OUTPUT_PATH}/vtkgdcm.jar
      DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKJavaModule
      )
    ENDIF(NOT GDCM_INSTALL_NO_LIBRARIES)

  ELSE(VTK_WRAP_JAVA)
    MESSAGE("GDCM_WRAP_JAVA canot be build without VTK_WRAP_JAVA")
  ENDIF(VTK_WRAP_JAVA)
ENDIF(GDCM_WRAP_JAVA)

IF(GDCM_WRAP_PYTHON)
  IF(VTK_WRAP_PYTHON)
      SET(DEXTENSION "")
    IF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
        VTK_WRAP_PYTHON2(${VTKGDCM_NAME}Python vtkgdcmPYTHON_SRCS ${vtkgdcm_SRCS})
    ELSE( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
    INCLUDE(${VTK_CMAKE_DIR}/vtkWrapPython.cmake)
        INCLUDE_DIRECTORIES(
          ${PYTHON_INCLUDE_DIR}
          ${CMAKE_CURRENT_SOURCE_DIR}
        )
        VTK_WRAP_PYTHON3(${VTKGDCM_NAME}Python vtkgdcmPYTHON_SRCS "${vtkgdcm_SRCS}")
      SET(DEXTENSION "D")
    ENDIF( "${VTK_MAJOR_VERSION}.${VTK_MINOR_VERSION}" LESS 4.5 )
    ADD_LIBRARY(${VTKGDCM_NAME}PythonD ${vtkgdcmPYTHON_SRCS})
    # this is a library set the version:
    # do not set the version on the Python module:
    #SET_TARGET_PROPERTIES(${VTKGDCM_NAME}PythonD PROPERTIES ${GDCM_LIBRARY_PROPERTIES})
    ADD_LIBRARY(${VTKGDCM_NAME}Python MODULE ${VTKGDCM_NAME}PythonInit.cxx)
    # do not set the version on the PythonD module:
    SET_TARGET_PROPERTIES(${VTKGDCM_NAME}Python PROPERTIES ${GDCM_LIBRARY_PROPERTIES})
    TARGET_LINK_LIBRARIES(${VTKGDCM_NAME}PythonD ${VTKGDCM_NAME})
    FOREACH(c ${vtkgdcm_LIBS})
      TARGET_LINK_LIBRARIES(${VTKGDCM_NAME}PythonD ${c}Python${DEXTENSION})
    ENDFOREACH(c)
    TARGET_LINK_LIBRARIES(${VTKGDCM_NAME}Python ${VTKGDCM_NAME}PythonD)
    # Python extension modules on Windows must have the extension ".pyd"
    # instead of ".dll" as of Python 2.5.  Older python versions do support
    # this suffix.
    IF(WIN32 AND NOT CYGWIN)
      SET_TARGET_PROPERTIES(${VTKGDCM_NAME}Python PROPERTIES SUFFIX ".pyd")
    ENDIF(WIN32 AND NOT CYGWIN)

    ADD_CUSTOM_COMMAND(
      TARGET    ${VTKGDCM_NAME}Python
      POST_BUILD
      COMMAND   ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/vtkgdcm.py ${LIBRARY_OUTPUT_PATH}/${CMAKE_CFG_INTDIR}
      DEPENDS   "${CMAKE_CURRENT_SOURCE_DIR}/vtkgdcm.py"
      COMMENT   "Copy vtkgdcm.py into ${LIBRARY_OUTPUT_PATH}"
    )
    IF(NOT GDCM_INSTALL_NO_LIBRARIES)
        INSTALL(TARGETS ${VTKGDCM_NAME}Python
      EXPORT ${GDCM_TARGETS_NAME}
          RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT                        VTKPythonModule
          LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR}/${python_site_package} COMPONENT VTKPythonModule #${NAMELINK_SKIP}
          ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR}/${python_site_package} COMPONENT VTKPythonModule
        )
        INSTALL(TARGETS ${VTKGDCM_NAME}PythonD
      EXPORT ${GDCM_TARGETS_NAME}
          RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT VTKPythonModule
          LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKPythonModule #${NAMELINK_ONLY}
          ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT VTKPythonModule
        )
      #  if(NAMELINK_ONLY)
      #      INSTALL(TARGETS ${VTKGDCM_NAME}PythonD ${VTKGDCM_NAME}Python
      #        LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR}/${python_site_package} COMPONENT VTKDevelopment ${NAMELINK_ONLY}
      #        )
      #  endif(NAMELINK_ONLY)
    # the python file is not a dev file, but part of the gdcm module...
      INSTALL(FILES ${CMAKE_CURRENT_SOURCE_DIR}/vtkgdcm.py
        DESTINATION ${GDCM_INSTALL_LIB_DIR}/${python_site_package} COMPONENT VTKPythonModule
      )
    ENDIF(NOT GDCM_INSTALL_NO_LIBRARIES)

  ELSE(VTK_WRAP_PYTHON)
    MESSAGE("GDCM_WRAP_PYTHON canot be build without VTK_WRAP_PYTHON")
  ENDIF(VTK_WRAP_PYTHON)
ENDIF(GDCM_WRAP_PYTHON)

IF(BUILD_TESTING)
  SUBDIRS(Testing)
ENDIF(BUILD_TESTING)

IF(BUILD_APPLICATIONS)
  SUBDIRS(Applications)
ENDIF(BUILD_APPLICATIONS)

IF(BUILD_EXAMPLES)
  SUBDIRS(Examples)
ENDIF(BUILD_EXAMPLES)

IF(GDCM_USE_PARAVIEW)
  # http://www.cmake.org/Wiki/Plugin_HowTo#Adding_a_Reader
  FIND_PACKAGE(ParaView REQUIRED)
  INCLUDE(${PARAVIEW_USE_FILE})
  ADD_PARAVIEW_PLUGIN(GDCMImageReader "1.0"
    SERVER_MANAGER_SOURCES vtkGDCMImageReader.cxx
    SERVER_MANAGER_XML GDCMImageReader.xml
    GUI_RESOURCE_FILES GDCMImageGUI.xml)
  TARGET_LINK_LIBRARIES(GDCMImageReader ${VTKGDCM_NAME})
  INSTALL(TARGETS GDCMImageReader
    EXPORT ${GDCM_TARGETS_NAME}
    RUNTIME DESTINATION ${GDCM_INSTALL_BIN_DIR} COMPONENT ParaViewModule
    LIBRARY DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT ParaViewModule
    ARCHIVE DESTINATION ${GDCM_INSTALL_LIB_DIR} COMPONENT ParaViewModule
    )

ENDIF(GDCM_USE_PARAVIEW)
