```text
    

cmake的标准模块支持protobuf的，参考官方文档对于FindProtobuf的说明
http://www.cmake.org/cmake/help/v2.8.12/cmake.html#module:FindProtobuf

webrtc项目里面一个例子
if (${enable_protobuf})
	set(PROTOBUF_SRC_ROOT_FOLDER "${webrtc_root}/third_party/protobuf-2.5.0")
	#include(FindProtobuf)
	find_package(Protobuf REQUIRED)
	if (!${PROTOBUF_FOUND})
		message (STATUS "找不到protobuf，自己下载源码编译，并修改commoo.cmake文件里面 set\(PROTOBUF_SRC_ROOT_FOLDER 为相应源码目录。")
	endif()
	message (STATUS "PROTOBUF_INCLUDE_DIRS = ${PROTOBUF_INCLUDE_DIRS}")
	message (STATUS "PROTOBUF_LIBRARIES  = ${PROTOBUF_LIBRARIES}")
endif()


if (${enable_protobuf})
	PROTOBUF_GENERATE_CPP(PROTO_SRCS PROTO_HDRS "debug.proto")
	add_library(audioproc_debug_proto STATIC ${PROTO_SRCS} ${PROTO_HDRS})
	# PROTOBUF_GENERATE_CPP默认生成的文件放在build目录下，
	# 复制出来便于其他项目文件引用
	add_custom_command(TARGET audioproc_debug_proto POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E copy ${PROTO_HDRS} "${CMAKE_CURRENT_SOURCE_DIR}/debug.pb.h")
	target_link_libraries(audioproc_debug_proto ${PROTOBUF_LIBRARIES})
	target_include_directories(audioproc_debug_proto
		PUBLIC "${PROTOBUF_INCLUDE_DIRS}")
	target_link_libraries(audio_processing audioproc_debug_proto)
	target_compile_definitions(audio_processing PRIVATE "WEBRTC_AUDIOPROC_DEBUG_DUMP")
endif()
PROTOBUF_SRC_ROOT_FOLDER 这个变量先制定自己的protocol的源码目录，Linux安装的官方package可能不需要这么做。
详细自己查看cmake安装目录下的 FindProtobuf 文件的源码。
PROTOBUF_GENERATE_CPP 为proto文件生成调用protobuf编译器自动生成源码的规则，并返回所有的源文件和头文件到第一个第二个参数。用的时候把返回的这两个list 传给add_library 或者其他创建target的命令的源码参数就可以了。

${PROTOBUF_LIBRARIES} ${PROTOBUF_INCLUDE_DIRS} 这两个变量分别是 protobuf需要链接和包含的目录。像上面这样使用就可以了。


```
