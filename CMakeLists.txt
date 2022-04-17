cmake_minimum_required(VERSION 3.2)
project(toy)

set(IS_FALSE_PATTERN "^[Oo][Ff][Ff]$|^0$|^[Ff][Aa][Ll][Ss][Ee]$|^[Nn][Oo]$|^[Nn][Oo][Tt][Ff][Oo][Uu][Nn][Dd]$|.*-[Nn][Oo][Tt][Ff][Oo][Uu][Nn][Dd]$|^$")
set(IS_TRUE_PATTERN "^[Oo][Nn]$|^[1-9][0-9]*$|^[Tt][Rr][Uu][Ee]$|^[Yy][Ee][Ss]$|^[Yy]$")
set(CMAKE_CXX_STANDARD 14)

macro(find_llvm use_llvm)
    if(${use_llvm} MATCHES ${IS_FALSE_PATTERN})
        return()
    endif()
    set(LLVM_CONFIG ${use_llvm})
    if(${ARGC} EQUAL 2)
        set(llvm_version_required ${ARGV1})
    endif()

    if(${LLVM_CONFIG} MATCHES ${IS_TRUE_PATTERN})
        find_package(LLVM ${llvm_version_required} REQUIRED CONFIG)
        llvm_map_components_to_libnames(LLVM_LIBS "all")
        if (NOT LLVM_LIBS)
            message(STATUS "Not found - LLVM_LIBS")
            message(STATUS "Fall back to using llvm-config")
            set(LLVM_CONFIG "${LLVM_TOOLS_BINARY_DIR}/llvm-config")
        endif()
    endif()

    if(LLVM_LIBS) # check if defined, not if it is true
        list (FIND LLVM_LIBS "LLVM" _llvm_dynlib_index)
        if (${_llvm_dynlib_index} GREATER -1)
            set(LLVM_LIBS LLVM)
            message(STATUS "Link with dynamic LLVM library")
        else()
            list(REMOVE_ITEM LLVM_LIBS LTO)
            message(STATUS "Link with static LLVM libraries")
        endif()
    else()
        # use llvm config
        message(STATUS "Use llvm-config=" ${LLVM_CONFIG})
        separate_arguments(LLVM_CONFIG)
        execute_process(COMMAND ${LLVM_CONFIG} --libfiles
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_libfiles_space
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --libfiles")
        endif()
        execute_process(COMMAND ${LLVM_CONFIG} --system-libs
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_system_libs
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --system-libs")
        endif()
        execute_process(COMMAND ${LLVM_CONFIG} --cxxflags
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_cxxflags_space
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --cxxflags")
        endif()
        execute_process(COMMAND ${LLVM_CONFIG} --version
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_version
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --version")
        endif()
        execute_process(COMMAND ${LLVM_CONFIG} --prefix
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_prefix
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --prefix")
        endif()
        execute_process(COMMAND ${LLVM_CONFIG} --libdir
                RESULT_VARIABLE __llvm_exit_code
                OUTPUT_VARIABLE __llvm_libdir
                OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(NOT "${__llvm_exit_code}" STREQUAL "0")
            message(FATAL_ERROR "Fatal error executing: ${use_llvm} --libdir")
        endif()
        # map prefix => $
        # to handle the case when the prefix contains space.
        string(REPLACE ${__llvm_prefix} "$" __llvm_cxxflags ${__llvm_cxxflags_space})
        string(REPLACE ${__llvm_prefix} "$" __llvm_libfiles ${__llvm_libfiles_space})
        # llvm version
        string(REGEX REPLACE "^([^.]+).([^.])+.[^.]+.*$" "\\1\\2" TVM_LLVM_VERSION ${__llvm_version})
        string(STRIP ${TVM_LLVM_VERSION} TVM_LLVM_VERSION)
        # definitions
        string(REGEX MATCHALL "(^| )-D[A-Za-z0-9_]*" __llvm_defs ${__llvm_cxxflags})
        set(LLVM_DEFINITIONS "")
        foreach(__flag IN ITEMS ${__llvm_defs})
            string(STRIP "${__flag}" __llvm_def)
            list(APPEND LLVM_DEFINITIONS "${__llvm_def}")
        endforeach()
        # include dir
        string(REGEX MATCHALL "(^| )-I[^ ]*" __llvm_include_flags ${__llvm_cxxflags})
        set(LLVM_INCLUDE_DIRS "")
        foreach(__flag IN ITEMS ${__llvm_include_flags})
            string(REGEX REPLACE "(^| )-I" "" __dir "${__flag}")
            # map $ => prefix
            string(REPLACE "$" ${__llvm_prefix} __dir_with_prefix "${__dir}")
            list(APPEND LLVM_INCLUDE_DIRS "${__dir_with_prefix}")
        endforeach()
        # libfiles
        set(LLVM_LIBS "")
        separate_arguments(__llvm_libfiles)
        foreach(__flag IN ITEMS ${__llvm_libfiles})
            # map $ => prefix
            string(REPLACE "$" ${__llvm_prefix} __lib_with_prefix "${__flag}")
            list(APPEND LLVM_LIBS "${__lib_with_prefix}")
        endforeach()
        separate_arguments(__llvm_system_libs)
        foreach(__flag IN ITEMS ${__llvm_system_libs})
            # If the library file ends in .lib try to
            # also search the llvm_libdir
            if(__flag MATCHES ".lib$")
                if(EXISTS "${__llvm_libdir}/${__flag}")
                    set(__flag "${__llvm_libdir}/${__flag}")
                endif()
            endif()
            list(APPEND LLVM_LIBS "${__flag}")
        endforeach()
    endif()
    message(STATUS "Found LLVM_INCLUDE_DIRS=" "${LLVM_INCLUDE_DIRS}")
    message(STATUS "Found LLVM_DEFINITIONS=" "${LLVM_DEFINITIONS}")
    message(STATUS "Found LLVM_LIBS=" "${LLVM_LIBS}")
endmacro(find_llvm)

add_definitions(-DDMLC_USE_FOPEN64=0 -DNDEBUG=1)
set(USE_LLVM ON)

if(NOT ${USE_LLVM} MATCHES ${IS_FALSE_PATTERN})
    find_llvm(${USE_LLVM})
    include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
    add_definitions(${LLVM_DEFINITIONS})
    message(STATUS "Build with LLVM " ${LLVM_PACKAGE_VERSION})

    list(APPEND COMPILER_SRCS ${COMPILER_LLVM_SRCS})
    if(NOT MSVC)
        set_source_files_properties(${COMPILER_LLVM_SRCS}
                PROPERTIES COMPILE_DEFINITIONS "DMLC_ENABLE_RTTI=0")
        set_source_files_properties(${COMPILER_LLVM_SRCS}
                PROPERTIES COMPILE_FLAGS "-fno-rtti")
    endif()
endif()

include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})
message(LLVM_DEFINITIONS="${LLVM_DEFINITIONS}")
add_executable(toy toy.cpp)

target_link_libraries(toy ${LLVM_LIBS})


