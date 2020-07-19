project(darling-nghttp2 VERSION 1.41.90)

set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})
include(Version)

set(PACKAGE_VERSION     "${PROJECT_VERSION}")
HexVersion(PACKAGE_VERSION_NUM ${PROJECT_VERSION_MAJOR} ${PROJECT_VERSION_MINOR} ${PROJECT_VERSION_PATCH})

configure_file(lib/includes/nghttp2/nghttp2ver.h.in lib/includes/nghttp2/nghttp2ver.h @ONLY)

set(DYLIB_CURRENT_VERSION "${PROJECT_VERSION}")

add_darling_library(nghttp2
	lib/nghttp2_buf.c
	lib/nghttp2_callbacks.c
	lib/nghttp2_debug.c
	lib/nghttp2_frame.c
	lib/nghttp2_hd_huffman_data.c
	lib/nghttp2_hd_huffman.c
	lib/nghttp2_hd.c
	lib/nghttp2_helper.c
	lib/nghttp2_http.c
	lib/nghttp2_map.c
	lib/nghttp2_mem.c
	lib/nghttp2_npn.c
	lib/nghttp2_option.c
	lib/nghttp2_outbound_item.c
	lib/nghttp2_pq.c
	lib/nghttp2_priority_spec.c
	lib/nghttp2_queue.c
	lib/nghttp2_rcbuf.c
	lib/nghttp2_session.c
	lib/nghttp2_stream.c
	lib/nghttp2_submit.c
	lib/nghttp2_version.c
)
make_fat(nghttp2)

set_target_properties(nghttp2 PROPERTIES
	OUTPUT_NAME "apple_nghttp2"
)

target_compile_definitions(nghttp2 PRIVATE
	BUILDING_NGHTTP2
)

target_include_directories(nghttp2 PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}/lib/includes"
  "${CMAKE_CURRENT_BINARY_DIR}/lib/includes"
)

target_link_libraries(nghttp2 PRIVATE
	system
)

install(TARGETS nghttp2 DESTINATION libexec/darling/usr/lib)
