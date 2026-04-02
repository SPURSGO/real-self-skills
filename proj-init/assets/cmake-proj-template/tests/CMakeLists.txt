# tests/CMakeLists.txt
include(GoogleTest)

add_executable(targetxxx
)

target_include_directories(targetxxx PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}
)

target_link_libraries(targetxxx PRIVATE
  GTest::gtest_main
  GTest::gmock
)

gtest_discover_tests(targetxxx)
