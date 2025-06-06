# include some defines automatically made by qpm
include(qpm_defines.cmake)

cmake_minimum_required(VERSION 3.22)
project(${COMPILE_ID})

# c++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED 20)
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)

# define that stores the actual source directory
set(SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)
set(INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

# compile options used
add_link_options(-Wl,--exclude-libs,ALL)
add_compile_options(-frtti -fPIE -fPIC -fexceptions -fvisibility=hidden)
add_compile_options(-Wall -Werror -Wextra -Wno-invalid-offsetof -O3 -Wno-vla-cxx-extension)
# Extra -Wno's that were not there originally to make it work
add_compile_options(-Wno-unused-variable -Wno-unused-function)
# compile definitions used
add_compile_definitions(VERSION=\"${MOD_VERSION}\")
add_compile_definitions(MOD_ID=\"${MOD_ID}\")
# TODO: Implement more robustly
add_compile_definitions(VERSION_LONG=1)
add_compile_definitions(NEED_UNSAFE_CSHARP)
add_compile_definitions(UNITY_2021)
add_compile_definitions(NO_VERBOSE_LOGS)
# For performing debug logging of very hard to diagnose issues
if (NOT DEFINED LOCAL_TEST)
    add_compile_definitions(CT_USE_GCDESCRIPTOR_DEBUG)
endif()

string(LENGTH "${CMAKE_CURRENT_LIST_DIR}/" FOLDER_LENGTH)
add_compile_definitions("PAPER_ROOT_FOLDER_LENGTH=${FOLDER_LENGTH}")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (DEFINED LOCAL_TEST)
    MESSAGE(STATUS "Building lib with Test defines")
    add_compile_definitions(LOCAL_TEST)
    add_compile_definitions(TEST_DELEGATE)
elseif(DEFINED LOCAL_TEST_COROUTINE)
    MESSAGE(STATUS "Building lib with Coroutine test defines")
    add_compile_definitions(LOCAL_TEST_COROUTINE)
endif()

# recursively get all src files
RECURSE_FILES(cpp_file_list ${SOURCE_DIR}/*.cpp)
RECURSE_FILES(c_file_list ${SOURCE_DIR}/*.c)
RECURSE_FILES(inline_hook_c ${EXTERN_DIR}/includes/beatsaber-hook/shared/inline-hook/*.c)
RECURSE_FILES(inline_hook_cpp ${EXTERN_DIR}/includes/beatsaber-hook/shared/inline-hook/*.cpp)

# add all src files to compile
add_library(
    ${COMPILE_ID}
    SHARED
    ${cpp_file_list}
    ${c_file_list}
    ${inline_hook_c}
    ${inline_hook_cpp}
)

# add src dir as include dir
target_include_directories(${COMPILE_ID} PRIVATE ${SOURCE_DIR})
# add include dir as include dir
target_include_directories(${COMPILE_ID} PRIVATE ${INCLUDE_DIR})
# add shared dir as include dir
target_include_directories(${COMPILE_ID} PUBLIC ${SHARED_DIR})

target_link_libraries(${COMPILE_ID} PRIVATE -llog)
# add extern stuff like libs and other includes
include(extern.cmake)

add_custom_command(TARGET ${COMPILE_ID} POST_BUILD
            COMMAND ${CMAKE_STRIP} -g -S -d --strip-all
            "lib${COMPILE_ID}.so" -o "stripped_lib${COMPILE_ID}.so"
            COMMENT "Strip debug symbols done on final binary.")

add_custom_command(TARGET ${COMPILE_ID} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E make_directory debug
            COMMENT "Make directory for debug symbols"
            )

add_custom_command(TARGET ${COMPILE_ID} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E rename lib${COMPILE_ID}.so debug/lib${COMPILE_ID}.so
            COMMENT "Rename the lib to debug_ since it has debug symbols"
            )

add_custom_command(TARGET ${COMPILE_ID} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E rename stripped_lib${COMPILE_ID}.so lib${COMPILE_ID}.so
            COMMENT "Rename the stripped lib to regular"
            )
