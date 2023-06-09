# MIT License
#
# Copyright (c) 2022 University of California
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

add_executable(demo_app app_common.cc app_common.h demo_app.cc demo_app.h)
target_link_libraries(demo_app PRIVATE ${PROJECT_NAME} project_options
                                       project_warnings)

set(DEPENDENCIES_CONFIGURED cxxopts fmt magic_enum)
foreach(DEPENDENCY ${DEPENDENCIES_CONFIGURED})
  find_package(${DEPENDENCY} REQUIRED CONFIG)
endforeach()
target_link_system_libraries(demo_app PRIVATE cxxopts::cxxopts fmt::fmt
                             magic_enum::magic_enum)
target_compile_definitions(demo_app PRIVATE FMT_ENFORCE_COMPILE_STRING)

find_package(Arrow "${arrow_minimum_required_version}" REQUIRED)
target_link_system_libraries(demo_app PRIVATE Arrow::arrow Arrow::parquet)
target_compile_definitions(demo_app PRIVATE ARROW_NO_DEPRECATED_API)

if(NOT Arrow_IS_BUILT AND Arrow_VERSION VERSION_LESS 10.0.0)
  find_package(CURL REQUIRED CONFIG)
  # The system-installed Arrow library misses the CURL dependency. Fixed in
  # https://github.com/apache/arrow/commit/7a0ba80702ef63e63e346d2a9ca3137d8baca8bb
  target_link_system_libraries(Arrow::arrow INTERFACE CURL::libcurl)
endif()
