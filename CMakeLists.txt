cmake_minimum_required (VERSION 3.29)

project (HelloTriangle)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")

# Enable module scanning always (needed for any C++20 modules, including import std;)
set(CMAKE_CXX_SCAN_FOR_MODULES ON)

# Add option to enable/disable Vulkan C++ module support
option(ENABLE_CPP20_MODULE "Enable C++ 20 module support for Vulkan imports" OFF)

find_package (glfw3 REQUIRED)
find_package (glm REQUIRED)
find_package (Vulkan 1.4.335 REQUIRED)      # Require Vulkan SDK version 1.4.335 or higher
#find_package (tinyobjloader REQUIRED)
#find_package (tinygltf REQUIRED)
#find_package (KTX REQUIRED)

# Precompile the C++ standard library module for Clang + libc++ (needed for import std;)
set(STD_MODULE_SOURCE "/usr/share/libc++/v1/std.cppm")
set(STD_MODULE_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/std.pcm")

add_custom_command(
  OUTPUT ${STD_MODULE_OUTPUT}
  COMMAND clang++ -stdlib=libc++ -std=gnu++20 -fno-implicit-module-maps
          --precompile -o ${STD_MODULE_OUTPUT} ${STD_MODULE_SOURCE}
  DEPENDS ${STD_MODULE_SOURCE}
  COMMENT "Precompiling C++ Standard Library module (std)"
)
add_custom_target(StdModule DEPENDS ${STD_MODULE_OUTPUT})

# set up Vulkan C++ module only if enabled
if(ENABLE_CPP20_MODULE)
    add_library(VulkanCppModule)
    add_library(Vulkan::cppm ALIAS VulkanCppModule)

    target_compile_definitions(VulkanCppModule
          PUBLIC VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
  )
    target_include_directories(VulkanCppModule
          PUBLIC
          "${Vulkan_INCLUDE_DIR}"
  )
    target_link_libraries(VulkanCppModule
          PUBLIC
          Vulkan::Vulkan
  )

    target_compile_options(VulkanCppModule PRIVATE
    -fno-implicit-module-maps
    -fmodule-file=std=${STD_MODULE_OUTPUT}
  )

    add_dependencies(VulkanCppModule StdModule)

    target_sources(VulkanCppModule
          PUBLIC
          FILE_SET cxx_modules TYPE CXX_MODULES
          BASE_DIRS
          "${Vulkan_INCLUDE_DIR}"
          FILES
          "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
  )

    target_sources(VulkanCppModule
          PRIVATE
          "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
  )

else()
    add_library(VulkanCppModule INTERFACE)
    add_library(Vulkan::cppm ALIAS VulkanCppModule)
    target_link_libraries(VulkanCppModule INTERFACE Vulkan::Vulkan)
    target_compile_definitions(VulkanCppModule
          INTERFACE VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
  )
endif()



add_executable (glslang::validator IMPORTED)


add_executable(HelloTriangle main.cpp)

target_compile_options(HelloTriangle PRIVATE
  -fno-implicit-module-maps
  -fmodule-file=std=${STD_MODULE_OUTPUT}
)
add_dependencies(HelloTriangle StdModule)

target_link_libraries(HelloTriangle
    Vulkan::cppm
    glfw
    glm::glm
)

find_path(STB_INCLUDE_DIR stb_image.h)
if(STB_INCLUDE_DIR)
    target_include_directories(HelloTriangle PRIVATE ${STB_INCLUDE_DIR})
endif()

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/.gitignore" "*\n")
