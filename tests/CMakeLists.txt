add_executable(orderbook_tests OrderBookTests.cpp)
target_link_libraries(orderbook_tests PRIVATE orderbook GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(orderbook_tests)