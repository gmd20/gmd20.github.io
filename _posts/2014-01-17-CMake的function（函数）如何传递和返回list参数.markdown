作为参数传进去的时候，可以只写list的名字，而不是list的值
比如下面脚本要把systerm_wrappers_sources这个list传过去，那么只想下面这样写就可以了
extract_platform_specific_source(systerm_wrappers_sources)
不要写extract_platform_specific_source(${systerm_wrappers_sources})
然后里面函数根据这个字符串参数的值（内容为systerm_wrappers_sources），可以解析两次${${source_list}}就得到systerm_wrappers_sources这个list的值了。

然后返回值可以通过set(${source_list} ${new_source_list} PARENT_SCOPE) 这样的方式直接赋值给父级的函数变量。参考下面的这个systerm_wrappers_sources 这个list的传递和返回值赋值修改。

另外有网上找到的LIST_SPACES_APPEND_ONCE函数，应该可以把list转换成空格分开的字符串来传递的，也可以方便使用的吧。 

```text
include_directories("../interface" spreadsortlib)

set (systerm_wrappers_sources
	../interface/aligned_malloc.h
	../interface/atomic32.h
	../interface/clock.h
	../interface/compile_assert.h
	../interface/condition_variable_wrapper.h
	../interface/cpu_info.h
	../interface/cpu_features_wrapper.h
	../interface/critical_section_wrapper.h
	../interface/data_log.h
	../interface/data_log_c.h
	../interface/data_log_impl.h
	../interface/event_tracer.h
	../interface/event_wrapper.h
	../interface/file_wrapper.h
	../interface/fix_interlocked_exchange_pointer_win.h
	../interface/logging.h
	../interface/ref_count.h
	../interface/rw_lock_wrapper.h
	../interface/scoped_ptr.h
	../interface/scoped_refptr.h
	../interface/sleep.h
	../interface/sort.h
	../interface/static_instance.h
	../interface/stringize_macros.h
	../interface/thread_annotations.h
	../interface/thread_wrapper.h
	../interface/tick_util.h
	../interface/trace.h
	../interface/trace_event.h
	aligned_malloc.cc
	atomic32_mac.cc
	atomic32_posix.cc
	atomic32_win.cc
	clock.cc
	condition_variable.cc
	condition_variable_posix.cc
	condition_variable_posix.h
	condition_variable_event_win.cc
	condition_variable_event_win.h
	condition_variable_native_win.cc
	condition_variable_native_win.h
	cpu_info.cc
	cpu_features.cc
	critical_section.cc
	critical_section_posix.cc
	critical_section_posix.h
	critical_section_win.cc
	critical_section_win.h
	data_log.cc
	data_log_c.cc
	data_log_no_op.cc
	event.cc
	event_posix.cc
	event_posix.h
	event_tracer.cc
	event_win.cc
	event_win.h
	file_impl.cc
	file_impl.h
	logging.cc
	rw_lock.cc
	rw_lock_generic.cc
	rw_lock_generic.h
	rw_lock_posix.cc
	rw_lock_posix.h
	rw_lock_win.cc
	rw_lock_win.h
	set_thread_name_win.h
	sleep.cc
	sort.cc
	tick_util.cc
	thread.cc
	thread_posix.cc
	thread_posix.h
	thread_win.cc
	thread_win.h
	trace_impl.cc
	trace_impl.h
	trace_posix.cc
	trace_posix.h
	trace_win.cc
	trace_win.h)


if (${CMAKE_SYSTEM_NAME} MATCHES  "Android")
 list(APPEND systerm_wrappers_sources "../interface/logcat_trace_context.h")
 list(APPEND systerm_wrappers_sources "logcat_trace_context.cc")
endif()

extract_platform_specific_source(systerm_wrappers_sources)
#message(STATUS "systerm_wrappers_sources =${systerm_wrappers_sources}")

add_library(system_wrappers STATIC ${systerm_wrappers_sources})

if(MSVC)
	target_link_libraries(system_wrappers winmm.lib)
elseif(UNIX)
	add_definitions(-DWEBRTC_THREAD_RR)
	target_link_libraries(system_wrappers rt)
endif()

```


```text

###################################################
#  list functions from  curl/CMake/Utilities.cmake
###################################################


# File containing various utilities

# Converts a CMake list to a string containing elements separated by spaces
function(TO_LIST_SPACES _LIST_NAME OUTPUT_VAR)
  set(NEW_LIST_SPACE)
  foreach(ITEM ${${_LIST_NAME}})
    set(NEW_LIST_SPACE "${NEW_LIST_SPACE} ${ITEM}")
  endforeach()
  string(STRIP ${NEW_LIST_SPACE} NEW_LIST_SPACE)
  set(${OUTPUT_VAR} "${NEW_LIST_SPACE}" PARENT_SCOPE)
endfunction()

# Appends a lis of item to a string which is a space-separated list, if they don't already exist.
function(LIST_SPACES_APPEND_ONCE LIST_NAME)
  string(REPLACE " " ";" _LIST ${${LIST_NAME}})
  list(APPEND _LIST ${ARGN})
  list(REMOVE_DUPLICATES _LIST)
  to_list_spaces(_LIST NEW_LIST_SPACE)
  set(${LIST_NAME} "${NEW_LIST_SPACE}" PARENT_SCOPE)
endfunction()

# Convinience function that does the same as LIST(FIND ...) but with a TRUE/FALSE return value.
# Ex: IN_STR_LIST(MY_LIST "Searched item" WAS_FOUND)
function(IN_STR_LIST LIST_NAME ITEM_SEARCHED RETVAL)
  list(FIND ${LIST_NAME} ${ITEM_SEARCHED} FIND_POS)
  if(${FIND_POS} EQUAL -1)
    set(${RETVAL} FALSE PARENT_SCOPE)
  else()
    set(${RETVAL} TRUE PARENT_SCOPE)
  endif()
endfunction()




################################################
# functions to extract platform specific source 
################################################

function (extract_windows_source source_list out_arg)
	set (new_source_list)
	foreach (source_name ${${source_list}})
		if (NOT source_name MATCHES "(_mac\\.h$|_mac\\.cc$|_posix\\.cc$|_posix\\.h$)")
			#message(STATUS "source_name =${source_name}")
			list(APPEND new_source_list ${source_name})
		endif()
	endforeach()
	set(${out_arg} ${new_source_list} PARENT_SCOPE)
endfunction()

function (extract_linux_source source_list out_arg)
	set (new_source_list)
	foreach (source_name ${${source_list}})
		if (NOT source_name MATCHES "_win\\.h$|_win\\.cc$|_mac\\.cc$|_mac\\.h$")
			list(APPEND new_source_list ${source_name})
		endif()
	endforeach()
	set(${out_arg} ${new_source_list} PARENT_SCOPE)
endfunction()

function (extract_platform_specific_source source_list)
	if (MSVC)
		extract_windows_source(${source_list} new_source_list)
	elseif(UNIX)
		extract_linux_source(${source_list} new_source_list)
	endif()
	set(${source_list} ${new_source_list} PARENT_SCOPE)
endfunction()
```
