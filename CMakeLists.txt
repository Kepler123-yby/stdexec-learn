cmake_minimum_required(VERSION 3.22)

project(stdexec_learn)

# 设置 C++ 标准。若上层没有指定，则默认使用 C++17。
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 20)
endif()

# GCC 和 Clang 常用警告选项。
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

add_subdirectory(stdexec)

# target_link_libraries(stdexec_learn PRIVATE stdexec)