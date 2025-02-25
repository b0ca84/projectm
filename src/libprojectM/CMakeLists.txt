add_compile_definitions(
        DATADIR_PATH="${PROJECTM_DATADIR_PATH}"
        $<IF:$<PLATFORM_ID:Darwin>,GL_SILENCE_DEPRECATION,>
        $<IF:$<PLATFORM_ID:Windows>,NOMINMAX,>
        $<IF:$<PLATFORM_ID:Windows>,WIN32_LEAN_AND_MEAN,>
        $<IF:$<PLATFORM_ID:Windows>,STBI_NO_DDS,>
)

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    # dirent.h support
    set(MSVC_EXTRA_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/msvc")
    include_directories("${MSVC_EXTRA_INCLUDE_DIR}")
endif()

# List of targets used in export().
# Optional libraries will be added to the list as configured in the build options.
set(EXPORTED_TARGETS
        MilkdropPresetFactory
        Renderer
        hlslparser
        SOIL2
        )

add_subdirectory(MilkdropPresetFactory)
add_subdirectory(Renderer)

add_library(projectM_main OBJECT
        "${PROJECTM_EXPORT_HEADER}"
        Common.hpp
        HungarianMethod.hpp
        IdleTextures.hpp
        PCM.cpp
        PCM.hpp
        PipelineMerger.cpp
        PipelineMerger.hpp
        Preset.hpp
        PresetFactory.cpp
        PresetFactory.hpp
        PresetFactoryManager.cpp
        PresetFactoryManager.hpp
        ProjectM.cpp
        ProjectM.hpp
        ProjectMCWrapper.cpp
        ProjectMCWrapper.hpp
        RandomNumberGenerators.hpp
        TimeKeeper.cpp
        TimeKeeper.hpp
        fatal.h
        fftsg.cpp
        fftsg.h
        glError.h
        gltext.h
        projectM-opengl.h
        resource.h
        wipemalloc.cpp
        wipemalloc.h
        )

target_link_libraries(projectM_main
        PUBLIC
        MilkdropPresetFactory
        Renderer
        hlslparser
        SOIL2
        libprojectM::API
        )

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_link_libraries(projectM_main
            PUBLIC
            "-framework CoreFoundation"
            )
endif()

target_include_directories(projectM_main
        PRIVATE
        "${CMAKE_SOURCE_DIR}/src"
        "${CMAKE_CURRENT_SOURCE_DIR}"
        "${CMAKE_CURRENT_SOURCE_DIR}/Renderer"
        "${CMAKE_SOURCE_DIR}/vendor/hlslparser/src"
        "${CMAKE_CURRENT_SOURCE_DIR}/MilkdropPresetFactory"
        "${MSVC_EXTRA_INCLUDE_DIR}"
        )

# CMake cannot combine multiple static libraries using target_link_libraries.
# This syntax will pull in the compiled object files into the final library.
add_library(projectM
        ${PROJECTM_DUMMY_SOURCE_FILE} # CMake needs at least one "real" source file.
        $<TARGET_OBJECTS:MilkdropPresetFactory>
        $<TARGET_OBJECTS:Renderer>
        $<TARGET_OBJECTS:hlslparser>
        $<TARGET_OBJECTS:SOIL2>
        $<TARGET_OBJECTS:projectM_main>
        )

target_include_directories(projectM
        PUBLIC
        "$<INSTALL_INTERFACE:${PROJECTM_INCLUDE_DIR}>"
        )

target_link_libraries(projectM
        PUBLIC
        ${PROJECTM_OPENGL_LIBRARIES}
        libprojectM::API
        )

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_link_libraries(projectM
        PUBLIC
        "-framework CoreFoundation"
        )
endif()

if(ENABLE_LLVM)
    target_link_libraries(projectM
            PUBLIC
            LLVM::LLVM
            )
endif()

if(ENABLE_OPENMP)
    target_link_libraries(projectM
            PUBLIC
            OpenMP::OpenMP_CXX
            )
endif()

set_target_properties(projectM PROPERTIES
        VERSION "${PROJECTM_LIB_VERSION}"
        SOVERSION "${PROJECTM_SO_VERSION}"
        FOLDER libprojectM
        OUTPUT_NAME projectM-4
        )

if(BUILD_SHARED_LIBS)
    set_source_files_properties(ProjectMCWrapper.cpp PROPERTIES
            COMPILE_DEFINITIONS projectM_api_EXPORTS
            )

    target_link_libraries(projectM
            PUBLIC
            ${CMAKE_DL_LIBS}
            )
else()
    set_source_files_properties(ProjectMCWrapper.cpp PROPERTIES
            COMPILE_DEFINITIONS PROJECTM_STATIC_DEFINE
            )

    set_target_properties(projectM PROPERTIES
            OUTPUT_NAME $<IF:$<PLATFORM_ID:Windows>,libprojectM,projectM>
            FOLDER libprojectM
            )
endif()

add_library(libprojectM::projectM ALIAS projectM)

install(TARGETS projectM
        EXPORT libprojectMTargets
        LIBRARY DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Runtime
        RUNTIME DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Runtime
        ARCHIVE DESTINATION "${PROJECTM_LIB_DIR}" COMPONENT Devel
        PUBLIC_HEADER DESTINATION "${PROJECTM_INCLUDE_DIR}/libprojectM" COMPONENT Devel
        )

if(ENABLE_CXX_INTERFACE)
    if(ENABLE_THREADING)
        target_compile_definitions(projectM
                INTERFACE
                PROJECTM_USE_THREADS
                )
    endif()

    set_source_files_properties(ProjectM.cpp PCM.cpp PROPERTIES
            COMPILE_DEFINITIONS projectM_api_EXPORTS
            )

    install(FILES
            Common.hpp
            PCM.hpp
            ProjectM.hpp
            DESTINATION "${PROJECTM_INCLUDE_DIR}/projectM-4"
            COMPONENT Devel
            )
else()
    # Set PROJECTM_STATIC_EXPORT for C++ implementations to use project default visibility
    # and no dllimport/dllexport.
    set_source_files_properties(ProjectM.cpp PCM.cpp PROPERTIES
            COMPILE_DEFINITIONS PROJECTM_STATIC_DEFINE
            )
endif()


# CMake target exports

# For use from a local projectM build tree (without installing)
export(TARGETS
        projectM_api
        projectM
        NAMESPACE libprojectM::
        FILE projectM-exports.cmake
        )


# For use from an installed package (system install, vcpkg, homebrew etc.)
include(CMakePackageConfigHelpers)

write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY AnyNewerVersion
)

configure_package_config_file(projectM4Config.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4Config.cmake"
        INSTALL_DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
        PATH_VARS PROJECTM_BIN_DIR PROJECTM_INCLUDE_DIR
        )

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4ConfigVersion.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/libprojectM/projectM4Config.cmake"
        DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
        COMPONENT Devel
        )

if(NOT ENABLE_EMSCRIPTEN AND ENABLE_GLES)
    install(FILES
            "${CMAKE_SOURCE_DIR}/cmake/gles/FindOpenGL.cmake"
            DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
            COMPONENT Devel
            )
endif()

if(ENABLE_LLVM)
    install(FILES
            "${CMAKE_SOURCE_DIR}/cmake/gles/FindLLVM.cmake"
            DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
            COMPONENT Devel
            )
endif()


install(EXPORT libprojectMTargets
        FILE projectM4Targets.cmake
        NAMESPACE libprojectM::
        DESTINATION "${PROJECTM_LIB_DIR}/cmake/projectM4"
        COMPONENT Devel
        )

# pkg-config export, only supported on UNIX systems.
if(UNIX)
    include(GeneratePkgConfigFiles)

    if(ENABLE_LLVM)
        set(PKGCONFIG_LIBS "${PKGCONFIG_LIBS} -L${LLVM_LIB_DIR} ${LLVM_LIBRARIES}")
    endif()

    if(ENABLE_OPENMP)
        set(PKGCONFIG_FLAGS "${PKGCONFIG_FLAGS} ${OpenMP_CXX_FLAGS}")
        foreach(_library ${OpenMP_CXX_LIBRARIES})
            set(PKGCONFIG_LIBS "${PKGCONFIG_LIBS} -l${_library}")
        endforeach()
    endif()

    set(PKGCONFIG_PACKAGE_NAME "projectM-4")
    set(PKGCONFIG_PACKAGE_DESCRIPTION "projectM Music Visualizer")
    set(PKGCONFIG_PACKAGE_REQUIREMENTS_ALL "opengl")

    generate_pkg_config_files(projectM projectM-4)

endif()
