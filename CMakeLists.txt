cmake_minimum_required(VERSION 3.20)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

if(APPLE AND NOT CMAKE_OSX_DEPLOYMENT_TARGET)
    # 10.15 because of our use of supportsFamily()
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15" CACHE STRING "Minimum macOS deployment target")
endif()

project(plume)

if(APPLE)
    enable_language(OBJC OBJCXX)
endif()

string(COMPARE EQUAL ${CMAKE_SYSTEM_NAME} "Linux" IS_LINUX)

# Project options
include(CMakeDependentOption)
cmake_dependent_option(SDL_VULKAN_ENABLED "Enable SDL Vulkan integration" OFF IS_LINUX OFF)
cmake_dependent_option(D3D12_AGILITY_SDK_ENABLED "Enable D3D12 Agility SDK" OFF WIN32 OFF)
option(PLUME_PLATFORM_SWITCH "Enable Nintendo Switch/libnx Vulkan integration" OFF)
option(PLUME_SWITCH_OUNCEVK "Use the OunceVK static Vulkan library" OFF)
option(PLUME_BUILD_EXAMPLES "Build example applications" OFF)

if(PLUME_PLATFORM_SWITCH)
    set(PLUME_SWITCH_DEVKITPRO "/opt/devkitpro" CACHE PATH "devkitPro root for Switch builds")
    set(PLUME_SWITCH_NVK_ROOT "/opt/nvk-switch" CACHE PATH "Mesa/NVK Switch build or install root")
    set(PLUME_SWITCH_NVK_LIBRARY "${PLUME_SWITCH_NVK_ROOT}/lib/libvulkan.a"
        CACHE FILEPATH "Mesa/NVK Switch Vulkan static archive")
endif()

# Windows-specific definitions
if(WIN32)
    add_compile_definitions(NOMINMAX)
endif()

if(PLUME_SDL2_VULKAN_ENABLED AND PLUME_SDL3_VULKAN_ENABLED)
    message(FATAL_ERROR "Plume - PLUME_SDL2_VULKAN_ENABLED and PLUME_SDL3_VULKAN_ENABLED are mutually exclusive")
endif()

# Enable SDL Vulkan support
if(PLUME_SDL2_VULKAN_ENABLED)
    if (NOT TARGET SDL2::SDL2)
        find_package(SDL2 REQUIRED)
    else()
        set(SDL2_INCLUDE_DIRS "$<TARGET_PROPERTY:SDL2::SDL2,INTERFACE_INCLUDE_DIRECTORIES>")
    endif()

    message(STATUS "Plume - Building with SDL2_INCLUDE_DIRS: ${SDL2_INCLUDE_DIRS}")
elseif(PLUME_SDL3_VULKAN_ENABLED)
    if (NOT TARGET SDL3::Headers)
        find_package(SDL3 REQUIRED)
    endif()
endif()

# Enable D3D12 Agility SDK support
if(PLUME_D3D12_AGILITY_SDK_ENABLED)
    find_package(directx-headers CONFIG REQUIRED)
    find_package(directx12-agility CONFIG REQUIRED)
endif()

# Print status messages
message(STATUS "Plume - Building with backends: Vulkan=1 Metal=${APPLE} D3D12=${WIN32}")
message(STATUS "Plume - SDL Vulkan integration: ${SDL_VULKAN_ENABLED}")
message(STATUS "Plume - D3D12 Agility SDK: ${D3D12_AGILITY_SDK_ENABLED}")
message(STATUS "Plume - Switch/libnx integration: ${PLUME_PLATFORM_SWITCH}")
message(STATUS "Plume - Building examples: ${PLUME_BUILD_EXAMPLES}")

# Basic source files that are always included
set(PLUME_SOURCES
    plume_vulkan.cpp
    plume_vulkan.h
)

# Platform-specific files
if(APPLE)
    list(APPEND PLUME_SOURCES
        plume_metal.cpp
        plume_metal.h
        plume_apple.mm
        plume_apple.h
    )
elseif(WIN32)
    list(APPEND PLUME_SOURCES
        plume_d3d12.cpp
        plume_d3d12.h
        ${CMAKE_CURRENT_SOURCE_DIR}/contrib/D3D12MemoryAllocator/src/D3D12MemAlloc.cpp
    )
endif()

if(PLUME_PLATFORM_SWITCH)
    set(PLUME_LSFG_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../lsfg-vk")
    file(GLOB PLUME_LSFG_SOURCES CONFIGURE_DEPENDS
        "${PLUME_LSFG_ROOT}/lsfg-vk-common/src/helpers/*.cpp"
        "${PLUME_LSFG_ROOT}/lsfg-vk-common/src/vulkan/*.cpp"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/src/*.cpp"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/src/extraction/*.cpp"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/src/helpers/*.cpp"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/src/shaderchains/*.cpp"
    )
    list(APPEND PLUME_SOURCES
        "${PLUME_LSFG_ROOT}/bridge/plume_lsfg_bridge.cpp"
        "${PLUME_LSFG_ROOT}/bridge/plume_lsfg_bridge.h"
        ${PLUME_LSFG_SOURCES}
    )
endif()

# Create target
add_library(plume STATIC ${PLUME_SOURCES})

# Include directories
target_include_directories(plume PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/volk
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/Vulkan-Headers/include
    ${CMAKE_CURRENT_SOURCE_DIR}/contrib/VulkanMemoryAllocator/include
)

if(WIN32)
    target_include_directories(plume PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/contrib/D3D12MemoryAllocator/include
    )
endif()

if(PLUME_SDL2_VULKAN_ENABLED)
    target_compile_definitions(plume PUBLIC PLUME_SDL2_VULKAN_ENABLED)
    target_include_directories(plume PUBLIC ${SDL2_INCLUDE_DIRS})
elseif(PLUME_SDL3_VULKAN_ENABLED)
    target_compile_definitions(plume PUBLIC PLUME_SDL3_VULKAN_ENABLED)
    target_link_libraries(plume PUBLIC PUBLIC SDL3::Headers)
endif()

if(PLUME_D3D12_AGILITY_SDK_ENABLED)
    target_compile_definitions(plume PUBLIC PLUME_D3D12_AGILITY_SDK_ENABLED)
    target_compile_definitions(plume PRIVATE D3D12MA_USING_DIRECTX_HEADERS)
    target_link_libraries(plume PRIVATE Microsoft::DirectX-Headers Microsoft::DirectX-Guids Microsoft::DirectX12-Agility)
endif()

if(PLUME_PLATFORM_SWITCH)
    if(NOT EXISTS "${PLUME_SWITCH_NVK_LIBRARY}")
        message(FATAL_ERROR "PLUME_SWITCH_NVK_LIBRARY does not exist: ${PLUME_SWITCH_NVK_LIBRARY}")
    endif()
    target_compile_definitions(plume PUBLIC
        __SWITCH__
        NX
        VK_USE_PLATFORM_VI_NN
    )
    if(PLUME_SWITCH_OUNCEVK)
        target_compile_definitions(plume PUBLIC PLUME_SWITCH_OUNCEVK)
    endif()
    target_include_directories(plume PUBLIC
        "${PLUME_SWITCH_NVK_ROOT}/include"
        "${PLUME_SWITCH_DEVKITPRO}/libnx/include"
        "${PLUME_SWITCH_DEVKITPRO}/portlibs/switch/include"
    )
    target_include_directories(plume PRIVATE
        "${PLUME_LSFG_ROOT}/bridge"
        "${PLUME_LSFG_ROOT}/lsfg-vk-common/include"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/include"
        "${PLUME_LSFG_ROOT}/lsfg-vk-backend/src"
    )
    target_compile_features(plume PUBLIC cxx_std_20)
    set(PLUME_SWITCH_NVK_EXTRA_LIBRARIES "")
    foreach(PLUME_SWITCH_LIBRARY_NAME IN ITEMS expat drm_nouveau)
        foreach(PLUME_SWITCH_LIBRARY_DIR IN ITEMS "${PLUME_SWITCH_NVK_ROOT}/lib" "${PLUME_SWITCH_DEVKITPRO}/portlibs/switch/lib")
            if(EXISTS "${PLUME_SWITCH_LIBRARY_DIR}/lib${PLUME_SWITCH_LIBRARY_NAME}.a")
                list(APPEND PLUME_SWITCH_NVK_EXTRA_LIBRARIES "${PLUME_SWITCH_LIBRARY_DIR}/lib${PLUME_SWITCH_LIBRARY_NAME}.a")
                break()
            endif()
        endforeach()
    endforeach()
    target_link_libraries(plume PUBLIC
        "${PLUME_SWITCH_NVK_LIBRARY}"
        $<$<NOT:$<BOOL:${PLUME_SWITCH_OUNCEVK}>>:-Wl,-u,vk_icdGetInstanceProcAddr>
        $<$<NOT:$<BOOL:${PLUME_SWITCH_OUNCEVK}>>:-Wl,-u,vk_icdNegotiateLoaderICDInterfaceVersion>
        "-Wl,--start-group"
        ${PLUME_SWITCH_NVK_EXTRA_LIBRARIES}
        zstd
        z
        nx
        "-Wl,--end-group"
    )
endif()

# Platform-specific includes
if(APPLE)
    if(PLUME_APPLE_RETINA_ENABLED)
        target_compile_definitions(plume PRIVATE PLUME_APPLE_RETINA_ENABLED)
    endif()

    target_include_directories(plume PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/contrib/metal-cpp
    )
endif()

# Add examples if requested
if(PLUME_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
