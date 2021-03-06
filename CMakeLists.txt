cmake_minimum_required(VERSION 3.5)
project(amazon_kinesis_producer)

set(THIRD_PARTY_LIB_DIR "${amazon_kinesis_producer_SOURCE_DIR}/third_party/lib")

if(CMAKE_COMPILER_IS_GNUCXX)
    set(ADDL_LINK_CONFIG "-static-libstdc++")
    add_compile_options("-fpermissive")
endif()

if(CMAKE_BUILD_TYPE MATCHES Debug)
    add_definitions(-DDEBUG)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    if (CMAKE_COMPILER_IS_GNUCXX)
        set(ADDL_LINK_CONFIG "${ADDL_LINK_CONFIG} -static-libasan")
    endif (CMAKE_COMPILER_IS_GNUCXX)
endif(CMAKE_BUILD_TYPE MATCHES Debug)

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mmacosx-version-min=10.9")
endif()

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXE_LINKER_FLAGS "-L${THIRD_PARTY_LIB_DIR} ${ADDL_LINK_CONFIG}")

set(SOURCE_FILES
        aws/auth/mutable_static_creds_provider.h
        aws/auth/mutable_static_creds_provider.cc
        aws/kinesis/core/aggregator.h
        aws/kinesis/core/attempt.h
        aws/kinesis/core/collector.h
        aws/kinesis/core/configuration.h
        aws/kinesis/core/ipc_manager.cc
        aws/kinesis/core/ipc_manager.h
        aws/kinesis/core/kinesis_producer.cc
        aws/kinesis/core/kinesis_producer.h
        aws/kinesis/core/kinesis_record.cc
        aws/kinesis/core/kinesis_record.h
        aws/kinesis/core/limiter.h
        aws/kinesis/core/pipeline.h
        aws/kinesis/core/put_records_context.h
        aws/kinesis/core/put_records_request.h
        aws/kinesis/core/reducer.h
        aws/kinesis/core/retrier.cc
        aws/kinesis/core/retrier.h
        aws/kinesis/core/serializable_container.h
        aws/kinesis/core/shard_map.cc
        aws/kinesis/core/shard_map.h
        aws/kinesis/core/user_record.cc
        aws/kinesis/core/user_record.h
        aws/kinesis/protobuf/config.pb.cc
        aws/kinesis/protobuf/config.pb.h
        aws/kinesis/protobuf/config.proto
        aws/kinesis/protobuf/messages.pb.cc
        aws/kinesis/protobuf/messages.pb.h
        aws/kinesis/protobuf/messages.proto
        aws/metrics/accumulator.h
        aws/metrics/metric.h
        aws/metrics/metrics_constants.cc
        aws/metrics/metrics_constants.h
        aws/metrics/metrics_finder.h
        aws/metrics/metrics_index.cc
        aws/metrics/metrics_index.h
        aws/metrics/metrics_manager.cc
        aws/metrics/metrics_manager.h
        aws/mutex.h
        aws/utils/concurrent_hash_map.h
        aws/utils/concurrent_linked_queue.h
        aws/utils/executor.h
        aws/utils/io_service_executor.h
        aws/utils/logging.cc
        aws/utils/logging.h
        aws/utils/spin_lock.cc
        aws/utils/spin_lock.h
        aws/utils/time_sensitive.h
        aws/utils/time_sensitive_queue.h
        aws/utils/token_bucket.h
        aws/utils/utils.cc
        aws/utils/utils.h
        aws/utils/processing_statistics_logger.cc
        aws/utils/processing_statistics_logger.h)

if(ENABLE_SEGFAULT_TRIGGER)
    list(APPEND LOGGING_TRIGGERS aws/utils/segfault_signal.cc)
    add_definitions(-DSEGFAULT_SIGNAL_ENABLED)
endif(ENABLE_SEGFAULT_TRIGGER)

set(TESTS_SOURCE
    aws/utils/test/concurrent_hash_map_test.cc
    aws/utils/test/concurrent_linked_queue_test.cc
    aws/utils/test/spin_lock_test.cc
    aws/utils/test/token_bucket_test.cc
    aws/kinesis/core/test/aggregator_test.cc
    aws/kinesis/core/test/ipc_manager_test.cc
    aws/kinesis/core/test/kinesis_record_test.cc
    aws/kinesis/core/test/limiter_test.cc
    aws/kinesis/core/test/put_records_request_test.cc
    aws/kinesis/core/test/reducer_test.cc
    aws/kinesis/core/test/retrier_test.cc
    aws/kinesis/core/test/shard_map_test.cc
    aws/kinesis/core/test/test_utils.cc
    aws/kinesis/core/test/test_utils.h
    aws/kinesis/core/test/user_record_test.cc
    aws/metrics/test/accumulator_test.cc
    aws/metrics/test/metric_test.cc
    aws/metrics/test/metrics_manager_test.cc
    aws/auth/test/mutable_static_creds_provider_test.cc
)

set(THIRD_PARTY_LIBS third_party/lib)
set(THIRD_PARTY_INCLUDE third_party/include)




include_directories(${CMAKE_CURRENT_SOURCE_DIR})

if(UNIX AND NOT APPLE)
    set(LINUX TRUE)
endif()

if(LINUX)
    set(ADDITIONAL_LIBS idn)
endif()

# Ensure that we can find the necessary libraries for Linux builds
include(CheckLibraryExists)
if(LINUX)
    #
    # Ensure that librt is abailable
    #
    check_library_exists(rt clock_gettime "" HAVE_CLOCK_GETTIME )
    if(NOT HAVE_CLOCK_GETTIME)
        message(FATAL_ERROR "clock_gettime not found")
    endif(NOT HAVE_CLOCK_GETTIME)
    set(LIBRT_LIBRARIES rt)

    #
    # Ensure that we have to the dynamic loader
    #
    check_library_exists(dl dlopen "" HAVE_DL_OPEN)
    if (NOT HAVE_DL_OPEN)
        message(FATAL_ERROR "dl_open not found")
    endif(NOT HAVE_DL_OPEN)
    set(LIBDL_LIBRARIES dl)

    #
    # Find libuuid which is another dependency of the AWS C++ SDK
    # TODO: Normalize this with the rest of the library checks
    #
    find_library(UUID_LIBRARIES NAMES libuuid.a uuid)
    if("${UUID_INCLUDE_DIR}" STREQUAL "UUID_INCLUDE_DIR-NOTFOUND" OR "${UUID_LIBRARIES}" STREQUAL "UUID_LIBRARIES-NOTFOUND")
        message(FATAL_ERROR "Could not find uuid components")
    else()
        message(STATUS "  Uuid library: ${UUID_LIBRARIES}")
    endif()

    add_library(libUUID STATIC IMPORTED)

endif(LINUX)



set(LOGGING_TRIGGERS
    aws/utils/signal_handler.h
    aws/utils/signal_handler.cc
    aws/utils/segfault_signal.h
    aws/utils/segfault_signal.cc
    aws/utils/backtrace/backtrace.h
    aws/utils/writer_methods.h
    aws/utils/writer_methods.cc
    aws/utils/backtrace/gcc_backtrace.cc
    aws/utils/backtrace/bsd_backtrace.cc
    aws/utils/backtrace/null_backtrace.cc
    )

include(CheckFunctionExists)
check_library_exists(backtrace backtrace_state "backtrace.h" HAVE_LIB_BACKTRACE)
check_function_exists(backtrace HAVE_BACKTRACE)
if(HAVE_LIB_BACKTRACE OR FORCE_LIB_BACKTRACE)
    message(">>>> Using LibBacktrace")
    add_library(libBacktrace STATIC IMPORTED)
    set_property(TARGET libBacktrace PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libbacktrace.a)
    set(LIBBACKTRACE_LIBRARIES backtrace)

    add_definitions(-DLIB_BACKTRACE)
elseif(HAVE_BACKTRACE)
    message(">>>> Using Backtrace")
    add_definitions(-DBSD_BACKTRACE)
else()
    message(">>>> No Backtrace Support")
    add_definitions(-DNULL_BACKTRACE)
endif()




set(STATIC_LIBS
        boost_regex
        boost_thread
        boost_log
        boost_system
        boost_random
        boost_filesystem
        boost_chrono)

find_package(Threads)

add_library(LibCrypto STATIC IMPORTED)
set_property(TARGET LibCrypto PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libcrypto.a)
set_property(TARGET LibCrypto PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES ${LIBDL_LIBRARIES})

add_library(LibSsl STATIC IMPORTED)
set_property(TARGET LibSsl PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libssl.a)
set_property(TARGET LibSsl PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES LibCrypto)

add_library(LibZ STATIC IMPORTED)
set_property(TARGET LibZ PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libz.a)

add_library(LibCurl STATIC IMPORTED)
set_property(TARGET LibCurl PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libcurl.a)
set_property(TARGET LibCurl PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES ${CMAKE_THREAD_LIBS_INIT} ${LIBRT_LIBRARIES} ${LIBDL_LIBRARIES} LibSsl LibZ)

add_library(LibProto STATIC IMPORTED)
set_property(TARGET LibProto PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libprotobuf.a)
set_property(TARGET LibProto PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES LibZ)

add_library(LibAwsCore STATIC IMPORTED)
set_property(TARGET LibAwsCore PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libaws-cpp-sdk-core.a)
set_property(TARGET LibAwsCore PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES ${LIBRT_LIBRARIES} ${LIBIDN_LIBRARIES} LibSsl LibCurl)

add_library(LibAwsKinesis STATIC IMPORTED)
set_property(TARGET LibAwsKinesis PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libaws-cpp-sdk-kinesis.a)
set_property(TARGET LibAwsKinesis PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES LibAwsCore)

add_library(LibAwsMonitoring STATIC IMPORTED)
set_property(TARGET LibAwsMonitoring PROPERTY IMPORTED_LOCATION ${THIRD_PARTY_LIB_DIR}/libaws-cpp-sdk-monitoring.a)
set_property(TARGET LibAwsMonitoring PROPERTY IMPORTED_LINK_INTERFACE_LIBRARIES LibAwsCore)


add_executable(kinesis_producer ${SOURCE_FILES} ${LOGGING_TRIGGERS} aws/kinesis/main.cc)
target_include_directories(kinesis_producer SYSTEM PUBLIC ${THIRD_PARTY_INCLUDE})
target_link_libraries(kinesis_producer ${CMAKE_THREAD_LIBS_INIT} LibProto LibAwsCore LibAwsKinesis LibAwsMonitoring ${LIBBACKTRACE_LIBRARIES} ${STATIC_LIBS} ${UUID_LIBRARIES})

add_executable(test_driver ${SOURCE_FILES} ${LOGGING_TRIGGERS} aws/kinesis/test_driver.cc)
target_include_directories(test_driver SYSTEM PUBLIC ${THIRD_PARTY_INCLUDE})
target_link_libraries(test_driver ${CMAKE_THREAD_LIBS_INIT} LibProto LibAwsCore LibAwsKinesis LibAwsMonitoring ${LIBBACKTRACE_LIBRARIES} ${STATIC_LIBS} ${UUID_LIBRARIES})

add_executable(tests ${SOURCE_FILES} ${TESTS_SOURCE} ${LOGGING_TRIGGERS} aws/kinesis/test/test.cc)
target_include_directories(tests SYSTEM PUBLIC ${THIRD_PARTY_INCLUDE})
target_link_libraries(tests ${CMAKE_THREAD_LIBS_INIT} LibProto LibAwsCore LibAwsKinesis LibAwsMonitoring ${LIBBACKTRACE_LIBRARIES} ${STATIC_LIBS} boost_unit_test_framework ${UUID_LIBRARIES})

add_library(kpl STATIC ${SOURCE_FILES} ${LOGGING_TRIGGERS})
target_include_directories(kpl SYSTEM PUBLIC ${THIRD_PARTY_INCLUDE})
target_link_libraries(kpl ${CMAKE_THREAD_LIBS_INIT} LibProto LibAwsCore LibAwsKinesis LibAwsMonitoring ${LIBBACKTRACE_LIBRARIES} ${STATIC_LIBS} ${UUID_LIBRARIES})
