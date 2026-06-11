cmake_minimum_required (VERSION 3.29)

project (HelloTriangle)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/CMake")

# Add option to enable/disable C++ 20 module
option(ENABLE_CPP20_MODULE "Enable C++ 20 module support for Vulkan" ON)

# Enable C++ module dependency scanning only if C++ 20 module is enabled
if(ENABLE_CPP20_MODULE)
  set(CMAKE_CXX_SCAN_FOR_MODULES ON)
endif()

find_package (glfw3 REQUIRED)
find_package (glm REQUIRED)
find_package (Vulkan 1.4.335 REQUIRED)      # Require Vulkan SDK version 1.4.335 or higher
#find_package (tinyobjloader REQUIRED)
#find_package (tinygltf REQUIRED)
#find_package (KTX REQUIRED)

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

  set_target_properties(VulkanCppModule PROPERTIES CXX_STANDARD 20)

  # Precompile the C++ standard library module for Clang + libc++
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

  # Clang needs to be told where to find the precompiled std module
  # and to avoid the legacy module.modulemap conflict
  target_compile_options(VulkanCppModule PRIVATE
    -fno-implicit-module-maps
    -fmodule-file=std=${STD_MODULE_OUTPUT}
  )

  add_dependencies(VulkanCppModule StdModule)

  # Add MSVC-specific compiler options for proper C++ module support
  if(MSVC)
    target_compile_options(VulkanCppModule PRIVATE
      /std:c++latest      # Use latest C++ standard for better module support
      /permissive-        # Standards conformance mode
      /Zc:__cplusplus     # Enable correct __cplusplus macro
      /EHsc               # Enable C++ exception handling
      /Zc:preprocessor    # Use conforming preprocessor
      /translateInclude   # Automatically translate #include to import for standard library
    )
  endif()

  target_sources(VulkanCppModule
          PUBLIC
          FILE_SET cxx_modules TYPE CXX_MODULES
          BASE_DIRS
          "${Vulkan_INCLUDE_DIR}"
          FILES
          "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
  )


  # Add the vulkan.cppm file directly as a source file
  target_sources(VulkanCppModule
          PRIVATE
          "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
  )

else()
  # Create a dummy interface library when C++ 20 module is disabled
  add_library(VulkanCppModule INTERFACE)
  add_library(Vulkan::cppm ALIAS VulkanCppModule)
  target_link_libraries(VulkanCppModule INTERFACE Vulkan::Vulkan)
  target_compile_definitions(VulkanCppModule
          INTERFACE VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
  )
endif()



add_executable (glslang::validator IMPORTED)
find_program (GLSLANG_VALIDATOR "glslangValidator" HINTS $ENV{VULKAN_SDK}/bin REQUIRED)
set_property (TARGET glslang::validator PROPERTY IMPORTED_LOCATION "${GLSLANG_VALIDATOR}")
find_program(SLANGC_EXECUTABLE slangc HINTS $ENV{VULKAN_SDK}/bin REQUIRED)

function (add_shaders_target TARGET)
  cmake_parse_arguments ("SHADER" "" "CHAPTER_NAME" "SOURCES" ${ARGN})
  set (SHADERS_DIR ${SHADER_CHAPTER_NAME}/shaders)
  add_custom_command (
    OUTPUT ${SHADERS_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADERS_DIR}
    )
  add_custom_command (
    OUTPUT ${SHADERS_DIR}/frag.spv ${SHADERS_DIR}/vert.spv
    COMMAND glslang::validator
    ARGS --target-env vulkan1.0 ${SHADER_SOURCES} --quiet
    WORKING_DIRECTORY ${SHADERS_DIR}
    DEPENDS ${SHADERS_DIR} ${SHADER_SOURCES}
    COMMENT "Compiling Shaders"
    VERBATIM
    )
  add_custom_target (${TARGET} DEPENDS ${SHADERS_DIR}/frag.spv ${SHADERS_DIR}/vert.spv)
endfunction ()

function (add_slang_shader_target TARGET)
  cmake_parse_arguments ("SHADER" "" "CHAPTER_NAME" "SOURCES" ${ARGN})
  set (SHADERS_DIR ${SHADER_CHAPTER_NAME}/shaders)
  file(GLOB HAS_COMPUTE ${CHAPTER_SHADER}.comp)
  set (ENTRY_POINTS -entry vertMain -entry fragMain)
  if(HAS_COMPUTE)
    list(APPEND ENTRY_POINTS -entry compMain)
  endif()
  add_custom_command (
          OUTPUT ${SHADERS_DIR}
          COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADERS_DIR}
  )
  add_custom_command (
          OUTPUT  ${SHADERS_DIR}/slang.spv
          COMMAND ${SLANGC_EXECUTABLE} ${SHADER_SOURCES} -target spirv -profile spirv_1_4+spvRayQueryKHR -emit-spirv-directly -fvk-use-entrypoint-name ${ENTRY_POINTS} -o slang.spv
          WORKING_DIRECTORY ${SHADERS_DIR}
          DEPENDS ${SHADERS_DIR} ${SHADER_SOURCES}
          COMMENT "Compiling Slang Shaders"
          VERBATIM
  )
  add_custom_target (${TARGET} DEPENDS ${SHADERS_DIR}/slang.spv)
endfunction()

add_executable(HelloTriangle main.cpp)

if(ENABLE_CPP20_MODULE)
  target_compile_options(HelloTriangle PRIVATE
    -fno-implicit-module-maps
    -fmodule-file=std=${STD_MODULE_OUTPUT}
  )
endif()

target_link_libraries(HelloTriangle
    Vulkan::Vulkan
    glfw
    glm::glm
)

find_path(STB_INCLUDE_DIR stb_image.h)
if(STB_INCLUDE_DIR)
  target_include_directories(HelloTriangle PRIVATE ${STB_INCLUDE_DIR})
endif()

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/.gitignore" "*\n")
