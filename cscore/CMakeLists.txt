project(cscore)

include(SubDirList)
include(CompileWarnings)
include(AddTest)
include(LinkMacOSGUI)

find_package( OpenCV REQUIRED )

file(GLOB
    cscore_native_src src/main/native/cpp/*.cpp)
file(GLOB cscore_linux_src src/main/native/linux/*.cpp)
file(GLOB cscore_osx_src src/main/native/osx/*.cpp)
file(GLOB cscore_osx_objc_src src/main/native/objcpp/*.mm)
file(GLOB cscore_windows_src src/main/native/windows/*.cpp)

add_library(cscore ${cscore_native_src})
set_target_properties(cscore PROPERTIES DEBUG_POSTFIX "d")

if(NOT MSVC)
    if (APPLE)
        target_sources(cscore PRIVATE ${cscore_osx_src} ${cscore_osx_objc_src})
        target_compile_options(cscore PRIVATE "-fobjc-arc")
        set_target_properties(cscore PROPERTIES LINK_FLAGS "-framework CoreFoundation -framework AVFoundation -framework Foundation -framework CoreMedia -framework CoreVideo")
    else()
        target_sources(cscore PRIVATE ${cscore_linux_src})
    endif()
else()
    target_sources(cscore PRIVATE ${cscore_windows_src})
    target_compile_definitions(cscore PUBLIC -DNOMINMAX)
    target_compile_definitions(cscore PRIVATE -D_CRT_SECURE_NO_WARNINGS)
endif()

target_include_directories(cscore PUBLIC
                $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/main/native/include>
                            $<INSTALL_INTERFACE:${include_dest}/cscore>)
target_include_directories(cscore PRIVATE src/main/native/cpp)
wpilib_target_warnings(cscore)
target_link_libraries(cscore PUBLIC wpinet wpiutil ${OpenCV_LIBS})

set_property(TARGET cscore PROPERTY FOLDER "libraries")

install(TARGETS cscore EXPORT cscore DESTINATION "${main_lib_dest}")
install(DIRECTORY src/main/native/include/ DESTINATION "${include_dest}/cscore")

if (WITH_FLAT_INSTALL)
    set (cscore_config_dir ${wpilib_dest})
else()
    set (cscore_config_dir share/cscore)
endif()

configure_file(cscore-config.cmake.in ${WPILIB_BINARY_DIR}/cscore-config.cmake )
install(FILES ${WPILIB_BINARY_DIR}/cscore-config.cmake DESTINATION ${cscore_config_dir})
install(EXPORT cscore DESTINATION ${cscore_config_dir})

SUBDIR_LIST(cscore_examples "${CMAKE_CURRENT_SOURCE_DIR}/examples")
foreach(example ${cscore_examples})
    file(GLOB cscore_example_src examples/${example}/*.cpp)
    unset(add_libs)
    if(${example} STREQUAL "usbviewer")
        if(TARGET wpigui)
            set(add_libs wpigui)
        else()
            unset(cscore_example_src)
        endif()
    endif()
    if(cscore_example_src)
        add_executable(cscore_${example} ${cscore_example_src})
        wpilib_target_warnings(cscore_${example})

        if (${example} STREQUAL "usbviewer")
            wpilib_link_macos_gui(cscore_${example})
        endif()

        target_link_libraries(cscore_${example} cscore ${add_libs})
        set_property(TARGET cscore_${example} PROPERTY FOLDER "examples")
    endif()
endforeach()

# Java bindings
if (WITH_JAVA)
    find_package(Java REQUIRED)
    find_package(JNI REQUIRED)
    include(UseJava)
    set(CMAKE_JAVA_COMPILE_FLAGS "-encoding" "UTF8" "-Xlint:unchecked")

    #find java files, copy them locally

    if("${OPENCV_JAVA_INSTALL_DIR}" STREQUAL "")
        set(OPENCV_JAVA_INSTALL_DIR ${OpenCV_INSTALL_PATH}/share/OpenCV/java/)
    endif()

    find_file(OPENCV_JAR_FILE NAMES opencv-${OpenCV_VERSION_MAJOR}${OpenCV_VERSION_MINOR}${OpenCV_VERSION_PATCH}.jar PATHS ${OPENCV_JAVA_INSTALL_DIR} ${OpenCV_INSTALL_PATH}/bin ${OpenCV_INSTALL_PATH}/share/java NO_DEFAULT_PATH)
    find_file(OPENCV_JNI_FILE NAMES libopencv_java${OpenCV_VERSION_MAJOR}${OpenCV_VERSION_MINOR}${OpenCV_VERSION_PATCH}.so
                                    libopencv_java${OpenCV_VERSION_MAJOR}${OpenCV_VERSION_MINOR}${OpenCV_VERSION_PATCH}.dylib
                                    opencv_java${OpenCV_VERSION_MAJOR}${OpenCV_VERSION_MINOR}${OpenCV_VERSION_PATCH}.dll
                                    PATHS ${OPENCV_JAVA_INSTALL_DIR} ${OpenCV_INSTALL_PATH}/bin ${OpenCV_INSTALL_PATH}/bin/Release ${OpenCV_INSTALL_PATH}/bin/Debug ${OpenCV_INSTALL_PATH}/lib ${OpenCV_INSTALL_PATH}/lib/jni NO_DEFAULT_PATH)

    file(GLOB
        cscore_jni_src src/main/native/cpp/jni/CameraServerJNI.cpp)

    file(GLOB_RECURSE JAVA_SOURCES src/main/java/*.java)
    set(CMAKE_JNI_TARGET true)

    if(${CMAKE_VERSION} VERSION_LESS "3.11.0")
        set(CMAKE_JAVA_COMPILE_FLAGS "-h" "${CMAKE_CURRENT_BINARY_DIR}/jniheaders")
        add_jar(cscore_jar ${JAVA_SOURCES} INCLUDE_JARS wpiutil_jar ${OPENCV_JAR_FILE} OUTPUT_NAME cscore)
    else()
        add_jar(cscore_jar ${JAVA_SOURCES} INCLUDE_JARS wpiutil_jar ${OPENCV_JAR_FILE} OUTPUT_NAME cscore GENERATE_NATIVE_HEADERS cscore_jni_headers)
    endif()

    get_property(CSCORE_JAR_FILE TARGET cscore_jar PROPERTY JAR_FILE)
    install(FILES ${CSCORE_JAR_FILE} DESTINATION "${java_lib_dest}")
    install(FILES ${OPENCV_JAR_FILE} DESTINATION "${java_lib_dest}")

    if (MSVC)
        install(FILES ${OPENCV_JNI_FILE} DESTINATION "${jni_lib_dest}")

        foreach(cvFile ${OpenCV_LIBS})
            find_file(${cvFile}Loc NAMES ${cvFile}${OpenCV_VERSION_MAJOR}${OpenCV_VERSION_MINOR}${OpenCV_VERSION_PATCH}.dll
                              PATHS ${OPENCV_JAVA_INSTALL_DIR} ${OpenCV_INSTALL_PATH}/bin ${OpenCV_INSTALL_PATH}/bin/Release ${OpenCV_INSTALL_PATH}/bin/Debug ${OpenCV_INSTALL_PATH}/lib NO_DEFAULT_PATH)
            install(FILES ${${cvFile}Loc} DESTINATION "${jni_lib_dest}")
        endforeach()
    endif()

    set_property(TARGET cscore_jar PROPERTY FOLDER "java")

    add_library(cscorejni ${cscore_jni_src})
    wpilib_target_warnings(cscorejni)
    target_link_libraries(cscorejni PUBLIC cscore wpiutil ${OpenCV_LIBS})

    set_property(TARGET cscorejni PROPERTY FOLDER "libraries")

    if(${CMAKE_VERSION} VERSION_LESS "3.11.0")
        target_include_directories(cscorejni PRIVATE ${JNI_INCLUDE_DIRS})
        target_include_directories(cscorejni PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/jniheaders")
    else()
        target_link_libraries(cscorejni PRIVATE cscore_jni_headers)
    endif()
    add_dependencies(cscorejni cscore_jar)

    if (MSVC)
        install(TARGETS cscorejni RUNTIME DESTINATION "${jni_lib_dest}" COMPONENT Runtime)
    endif()

    install(TARGETS cscorejni EXPORT cscorejni DESTINATION "${main_lib_dest}")

endif()

if (WITH_TESTS)
    wpilib_add_test(cscore src/test/native/cpp)
    target_link_libraries(cscore_test cscore gmock)
endif()
