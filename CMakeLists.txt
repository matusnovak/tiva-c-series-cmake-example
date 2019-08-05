cmake_minimum_required(VERSION 3.14)
project(example)

# Specify build type
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Select build type")
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")

# Path to the TivaWare C SDK folder
option(TIVA_WARE_C_PATH "Path to TivaWare C Series SDK folder")
if(NOT TIVA_WARE_C_PATH)
  message(FATAL_ERROR "You must specify TIVA_WARE_C_PATH variable!")
endif()

# Project source files
file(GLOB_RECURSE SOURCES ${CMAKE_CURRENT_LIST_DIR}/src/*.cpp ${CMAKE_CURRENT_LIST_DIR}/src/*.c)
file(GLOB_RECURSE HEADERS ${CMAKE_CURRENT_LIST_DIR}/src/*.h)
set(LINKER_SCRIPT ${CMAKE_CURRENT_LIST_DIR}/blinky.ld)

# Add the project
add_executable(${PROJECT_NAME} ${SOURCES} ${HEADERS})

# Include the TivaWare C SDK
target_include_directories(${PROJECT_NAME} PUBLIC ${TIVA_WARE_C_PATH})

# Platform specific flags
target_compile_options(${PROJECT_NAME} PRIVATE 
  -mthumb -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 
  -mfloat-abi=hard -ffunction-sections -fdata-sections
  -Wall -pedantic -MD -fno-exceptions -fno-rtti
)

# Platform specific definitions
target_compile_definitions(${PROJECT_NAME} PRIVATE
  -DPART_TM4C1294NCPDT
)

# Find the compiler specific libraries
get_target_property(COMPILE_FLAGS ${PROJECT_NAME} COMPILE_OPTIONS)
execute_process(COMMAND ${CMAKE_C_COMPILER} ${COMPILE_FLAGS} -print-libgcc-file-name OUTPUT_VARIABLE LIBGCC_PATH)
execute_process(COMMAND ${CMAKE_C_COMPILER} ${COMPILE_FLAGS} -print-file-name=libc.a OUTPUT_VARIABLE LIBC_PATH)
execute_process(COMMAND ${CMAKE_C_COMPILER} ${COMPILE_FLAGS} -print-file-name=libm.a OUTPUT_VARIABLE LIBM_PATH)
execute_process(COMMAND ${CMAKE_C_COMPILER} ${COMPILE_FLAGS} -print-file-name=libstdc++.a OUTPUT_VARIABLE LIBSTDCPP_PATH)
string(STRIP "${LIBGCC_PATH}" LIBGCC_PATH)
string(STRIP "${LIBSTDCPP_PATH}" LIBSTDCPP_PATH)
string(STRIP "${LIBC_PATH}" LIBC_PATH)
string(STRIP "${LIBM_PATH}" LIBM_PATH)

# Platform specific link definitions
target_link_options(${PROJECT_NAME} PRIVATE
  -T ${LINKER_SCRIPT} --entry ResetISR --gc-sections
)

# Include other libraries
target_link_libraries(${PROJECT_NAME} PUBLIC
  ${TIVA_WARE_C_PATH}/driverlib/gcc/libdriver.a
  ${LIBM_PATH}
  ${LIBC_PATH}
  ${LIBGCC_PATH}
  ${LIBSTDCPP_PATH}
)

# Objcopy to create a binary file
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD COMMAND 
  ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}>.axf 
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.bin
)

# Enable C++14 and C11
set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 14)
set_property(TARGET ${PROJECT_NAME} PROPERTY C_STANDARD 11)
