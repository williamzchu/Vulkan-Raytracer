cmake_minimum_required(VERSION 3.20)
project(VulkanRaytracer)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find all packages from vcpkg.json
find_package(Vulkan REQUIRED)
find_package(glfw3 CONFIG REQUIRED)
find_package(glm CONFIG REQUIRED)
find_package(tinyobjloader CONFIG REQUIRED)
find_package(Stb REQUIRED)
find_path(TINYGLTF_INCLUDE_DIR tiny_gltf.h REQUIRED)
find_package(nlohmann_json CONFIG REQUIRED)
find_package(ktx CONFIG REQUIRED)

# Find Slang shader compiler
find_program(SLANG_COMPILER slangc 
    HINTS $ENV{VULKAN_SDK}/bin
          "C:/VulkanSDK/*/Bin"
    REQUIRED
)

# Your main executable
add_executable(VulkanRaytracer
    src/main.cpp
)

target_compile_definitions(VulkanRaytracer PRIVATE
    VULKAN_HPP_NO_STRUCT_CONSTRUCTORS
    VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1
)

# Link all libraries
target_link_libraries(VulkanRaytracer PRIVATE
    Vulkan::Vulkan
    glfw
    glm::glm
    tinyobjloader::tinyobjloader
    nlohmann_json::nlohmann_json
    KTX::ktx
)

# Include STB and TinyGLTF headers
target_include_directories(VulkanRaytracer PRIVATE
    ${Stb_INCLUDE_DIRS}
    ${TINYGLTF_INCLUDE_DIR}
)

# Set output directory
set_target_properties(VulkanRaytracer PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)

# Shader compilation function for Slang shaders
function(add_shaders TARGET)
    set(SHADER_DIR ${CMAKE_CURRENT_SOURCE_DIR}/shaders)
    set(SHADER_OUTPUT_DIR ${CMAKE_BINARY_DIR}/bin/shaders)
    
    file(MAKE_DIRECTORY ${SHADER_OUTPUT_DIR})
    
    # Compile Slang shaders
    file(GLOB SLANG_SHADERS ${SHADER_DIR}/*.slang)
    
    foreach(SHADER ${SLANG_SHADERS})
        get_filename_component(SHADER_NAME ${SHADER} NAME_WE)
        set(SHADER_OUTPUT ${SHADER_OUTPUT_DIR}/${SHADER_NAME}.spv)
        
        add_custom_command(
            OUTPUT ${SHADER_OUTPUT}
            COMMAND ${SLANG_COMPILER} ${SHADER} 
                    -target spirv 
                    -profile spirv_1_4
                    -emit-spirv-directly
                    -fvk-use-entrypoint-name
                    -o ${SHADER_OUTPUT}
            DEPENDS ${SHADER}
            COMMENT "Compiling Slang shader: ${SHADER_NAME}"
            VERBATIM
        )
        
        list(APPEND SHADER_OUTPUTS ${SHADER_OUTPUT})
    endforeach()
    
    add_custom_target(${TARGET}_Shaders ALL DEPENDS ${SHADER_OUTPUTS})
    add_dependencies(${TARGET} ${TARGET}_Shaders)
endfunction()

# Compile shaders
add_shaders(VulkanRaytracer)

# Copy assets to build directory (optional)
file(GLOB TEXTURES ${CMAKE_CURRENT_SOURCE_DIR}/assets/textures/*)
file(GLOB MODELS ${CMAKE_CURRENT_SOURCE_DIR}/assets/models/*)

if(TEXTURES)
    file(COPY ${TEXTURES} DESTINATION ${CMAKE_BINARY_DIR}/bin/textures)
endif()

if(MODELS)
    file(COPY ${MODELS} DESTINATION ${CMAKE_BINARY_DIR}/bin/models)
endif()

# Print configuration summary
message(STATUS "VulkanRaytracer Configuration:")
message(STATUS "  Vulkan SDK: ${Vulkan_LIBRARY}")
message(STATUS "  Slang Compiler: ${SLANG_COMPILER}")
message(STATUS "  Output Directory: ${CMAKE_BINARY_DIR}/bin")

# Windows-specific: Set working directory for Visual Studio
if(WIN32 AND MSVC)
    set_target_properties(VulkanRaytracer PROPERTIES
        VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    )
endif()