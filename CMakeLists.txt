cmake_minimum_required(VERSION 3.0.0)

set(target_name crender)
project("${target_name}")


option(BUILD_CUSTOMER_RELEASE "Build for customer release" Off)
if( BUILD_CUSTOMER_RELEASE )
  message("Building customer release.")
endif()


# update submodules
find_package(Git REQUIRED)
function(add_submodule submodule)
  execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init ${submodule} WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}")
  add_subdirectory(${submodule} EXCLUDE_FROM_ALL)
endfunction()


# jinja2cpp
add_submodule(Jinja2Cpp)

# yaml-cpp
add_submodule(yaml-cpp)


# Not everyone considers GLOBing good form in a mature project.
file(GLOB_RECURSE src_cpp src/*.cpp src/*.cc)
file(GLOB_RECURSE src_h src/*.h)

# Add the executable here and use C++17.
add_executable(${target_name} ${src_cpp} ${src_h} )
set_property(TARGET ${target_name} PROPERTY CXX_STANDARD 17)

target_include_directories(${target_name} PRIVATE
  ${CMAKE_SOURCE_DIR}/src
  )

target_include_directories(${target_name} SYSTEM PRIVATE
  ${CMAKE_SOURCE_DIR}/Jinja2Cpp/include
  ${CMAKE_SOURCE_DIR}/yaml-cpp/include
  )

#target_link_directories(${target_name} PRIVATE  # maybe this should be public?
#  )

if( WIN32 )
    set(extra_libs "")
else()
    set(extra_libs pthread)
endif()

target_link_libraries(${target_name} PRIVATE # maybe this should be public?
  jinja2cpp
  yaml-cpp
  ${extra_libs}
)

if(BUILD_CUSTOMER_RELEASE)
  # Add -DCUSTOMER_RELEASE when BUILD_CUSTOMER_RELEASE is true.
  target_compile_definitions(${target_name} PUBLIC CUSTOMER_RELEASE)
endif()


if( APPLE OR UNIX OR MINGW )

  # Allow for a relase build
  if(BUILD_CUSTOMER_RELEASE)
    set(WERROR "")
  else()
    set(WERROR "-Werror")
  endif()

  target_compile_options(${target_name} PRIVATE
    -Wall
    -Wextra
    ${WERROR}
    -Wfloat-equal
    -Wshadow
#    -Wduplicated-branches
    -Wctor-dtor-privacy
    -Wnon-virtual-dtor
    -Wsuggest-override
    -Wredundant-decls
#    -Wlogical-op
    -Wold-style-cast
    )

elseif(MSVC)

  # Allow for a relase build
  if(BUILD_CUSTOMER_RELEASE)
    set(WERROR "")
  else()
    set(WERROR "/WX")
  endif()

  target_compile_options(${target_name} PRIVATE
    /W4
    ${WERROR}
    )

else()
  message(FATAL_ERROR "Can't set compile options")
endif()



# Install

# Main target
install(
  FILES
  $<TARGET_FILE:${target_name}>
  COMPONENT Runtime
  PERMISSIONS
    OWNER_READ OWNER_EXECUTE
    GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE
  DESTINATION .
  )

# Data directory - not everyone considers GLOBing good form in a mature project.
file(GLOB data_yaml data/*.yaml)
install(
  FILES
    ${data_yaml}
  COMPONENT Runtime
  DESTINATION crender-data
  )
