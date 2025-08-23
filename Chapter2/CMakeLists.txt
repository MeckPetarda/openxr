cmake_minimum_required(VERSION 3.28.3)
set(PROJECT_NAME OpenXRTutorialChapter2)
project("${PROJECT_NAME}")

# Additional Directories for find_package() to search within.
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../cmake")

# For FetchContent_Declare() and FetchContent_MakeAvailable()
include(FetchContent)

# openxr_loader - From github.com/KhronosGroup
set(BUILD_API_LAYERS
        ON
        CACHE INTERNAL "Use OpenXR layers"
)
set(BUILD_TESTS
        OFF
        CACHE INTERNAL "Build tests"
)
FetchContent_Declare(
    OpenXR
    EXCLUDE_FROM_ALL
    DOWNLOAD_EXTRACT_TIMESTAMP
    URL_HASH MD5=f52248ef83da9134bec2b2d8e0970677
    URL https://github.com/KhronosGroup/OpenXR-SDK-Source/archive/refs/tags/release-1.1.49.tar.gz
    SOURCE_DIR
    openxr
)

FetchContent_MakeAvailable(OpenXR)

# Files
set(SOURCES
        "main.cpp"
        "../Common/GraphicsAPI.cpp"
        "../Common/GraphicsAPI_Vulkan.cpp"
        "../Common/OpenXRDebugUtils.cpp")
set(HEADERS
        "../Common/DebugOutput.h"
        "../Common/GraphicsAPI.h"
        "../Common/GraphicsAPI_Vulkan.h"
        "../Common/HelperFunctions.h"
        "../Common/OpenXRDebugUtils.h"
        "../Common/OpenXRHelper.h")

set(GLSL_SHADERS "../Shaders/VertexShader.glsl" "../Shaders/PixelShader.glsl")

add_library(${PROJECT_NAME} SHARED ${SOURCES} ${HEADERS})
add_dependencies(${PROJECT_NAME} XrApiLayer_core_validation)
target_include_directories(
    ${PROJECT_NAME}
    PRIVATE
        # In this repo
        ../Common/
        # From OpenXR repo
        ../openxr/src/common
        ../openxr/external/include
)

# export ANativeActivity_onCreate for java to call.
set_property(
    TARGET ${PROJECT_NAME}
    APPEND_STRING
    PROPERTY LINK_FLAGS " -u ANativeActivity_onCreate"
)

# Native app glue - Method 1: Using find_library (recommended for newer NDK)
find_library(native_app_glue_lib native_app_glue)
if(native_app_glue_lib)
    target_link_libraries(${PROJECT_NAME} android ${native_app_glue_lib} openxr_loader)
else()
    # Method 2: Manual inclusion for older NDK versions or if find_library fails
    if(ANDROID_NDK)
        set(NATIVE_APP_GLUE_DIR "${ANDROID_NDK}/sources/android/native_app_glue")
        if(EXISTS "${NATIVE_APP_GLUE_DIR}")
            target_sources(${PROJECT_NAME} PRIVATE
                "${NATIVE_APP_GLUE_DIR}/android_native_app_glue.c"
            )
            target_include_directories(${PROJECT_NAME} PRIVATE
                "${NATIVE_APP_GLUE_DIR}"
            )
        endif()
    endif()
    target_link_libraries(${PROJECT_NAME} android log openxr_loader)
endif()

target_compile_options(${PROJECT_NAME} PRIVATE -Wno-cast-calling-convention)

# VulkanNDK
find_library(vulkan-lib vulkan)
if(vulkan-lib)
    target_include_directories(
        ${PROJECT_NAME}
        PUBLIC ${ANDROID_NDK}/sources/third_party/vulkan/src/include
    )
    target_link_libraries(${PROJECT_NAME} ${vulkan-lib})
    target_compile_definitions(
        ${PROJECT_NAME} PUBLIC XR_TUTORIAL_USE_VULKAN
    )
endif()

# Vulkan GLSL
set(SHADER_DEST "${CMAKE_CURRENT_SOURCE_DIR}/app/src/main/assets/shaders")
include(glsl_shader)
set_source_files_properties(
    ../Shaders/VertexShader.glsl PROPERTIES ShaderType "vert"
)
set_source_files_properties(
    ../Shaders/PixelShader.glsl PROPERTIES ShaderType "frag"
)

foreach(FILE ${GLSL_SHADERS})
    get_filename_component(FILE_WE ${FILE} NAME_WE)
    get_source_file_property(shadertype ${FILE} ShaderType)
    glsl_spv_shader(
        INPUT
        "${CMAKE_CURRENT_SOURCE_DIR}/${FILE}"
        OUTPUT
        "${SHADER_DEST}/${FILE_WE}.spv"
        STAGE
        ${shadertype}
        ENTRY_POINT
        main
        TARGET_ENV
        vulkan1.0
    )
    # Make our project depend on these files
    target_sources(${PROJECT_NAME} PRIVATE "${SHADER_DEST}/${FILE_WE}.spv")
endforeach()
