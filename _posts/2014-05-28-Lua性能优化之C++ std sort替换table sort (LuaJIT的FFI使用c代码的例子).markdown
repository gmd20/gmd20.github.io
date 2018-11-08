```lua
-- require('mobdebug').start("192.168.56.1")

local os     = require "os"
local math   = require "math"
local string = require "string"
local socket = require "socket"

--[[
Example:
local statsd = require "statsd"
local s = statsd.metric:new{name = "haha"}
s:starttimer()
socket.select(nil, nil, 3)
s:stoptimer()
--]]


local GRAPHITE_IP                  = "192.168.30.169"
local GRAPHITE_PORT                = 2013
local DEFAULT_FLUSH_INTERVALS      = 10         -- flush the metrics to graphite every n seconds
local MAX_COUNTER                  = 4096 * 8   -- flush the metrics to graphite once the counter is larger than this
local DEFAULT_PERCENTAGE_THRESHOLD = {90}       -- percentage threshold to compute
local DEFAULT_SAMPLE_RATE          = 1          -- a real number in the range [0, 1], data sampling rate
local DEFAULT_HISTOGRAM_BINS       = {4,8,16,32,64,128,256,512,1024,8192}

local graphite_udp = socket.udp()
graphite_udp:setpeername(GRAPHITE_IP, GRAPHITE_PORT)
math.randomseed(os.time())



local ffi = require("ffi")
local clib = ffi.load("quick_sort.dll")
ffi.cdef[[
void quick_sort(int * buf, int size);
double gettime();
]]
local int_buffer = ffi.new("int[?]", MAX_COUNTER)
function quick_sort(array, count)
  local i
  for i = 1, count do
    int_buffer[i-1] = array[i]
  end
  clib.quick_sort(int_buffer, count)
  for i = 1, count do
    array[i] = int_buffer[i-1]
  end
end

function fast_gettime()
  local now = clib.gettime()
  return now
end




--[[
--http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-plaintext-protocol
Graphite metrics should use the following format:

metricname value timestamp

metricname is a period-delimited path, such as servers.mario.memory.free The periods will turn each path component into a sub-tree. The graphite project website has some metric naming advice.

value is an integer or floating point number.

timestamp  is a UNIX timestamp, which is the number of seconds since Jan 1st 1970 (always UTC, never local time) .

You can send multiple metric values at the same time by putting them on separate lines in the same message:
--]]
function send_graphite_udp_packet(buffer)
  -- print (table.concat(buffer))
  ---[[
  if graphite_udp ~= nil then
    graphite_udp:send(table.concat(buffer))
  end
  ---]]
end


function flush_metric(metric_t, current_time)
  local m   = metric_t
  local now = current_time or socket.gettime()  -- m.time()
  if m == nil then
    return
  end
  if now - m.last_flush_time < m.flush_intervals and m.counter < MAX_COUNTER then
    return
  end
  if m.sample_rate ~= 1 and m.sample_rate <= math.random() then
    -- ignore this sampling
    m.last_flush_time = now
    if m.reset_after_flush == true then
      m:reset()
    end
    return
  end

  local timestamp = " " .. math.floor(now) .. "\n"
  local buffer = {}

  if #(m.timers) > 0 then
    -- timer --
    -- table.sort(m.timers)
    quick_sort(m.timers, m.counter)
    -- print (table.concat(m.timers, " "))

    local count  = m.counter
    local values = m.timers
    local min    = values[1]
    local max    = values[count]

    local cumulativeValues = {min}
    local cumulSumSquaresValues = {min * min}
    local i = 0
    for i = 2, count do
      cumulativeValues[i] = values[i] + cumulativeValues[i-1]
      cumulSumSquaresValues[i] = (values[i] * values[i]) + cumulSumSquaresValues[i-1]
    end

    local sum = min
    local sumSquares = min * min
    local mean = min
    local thresholdBoundary = max

    local pct_key
    local pct = 0
    for pct_key, pct in ipairs(m.pctThreshold) do
      local numInThreshold = count

      if count > 1 then
        numInThreshold = math.ceil((math.abs(pct) / 100) * count)
        if numInThreshold == 0 then
          goto continue
        end

        if pct > 0 then
          thresholdBoundary = values[numInThreshold]
          sum = cumulativeValues[numInThreshold]
          sumSquares = cumulSumSquaresValues[numInThreshold]
        else
          thresholdBoundary = values[count - numInThreshold + 1]
          sum = cumulativeValues[count] - cumulativeValues[count - numInThreshold]
          sumSquares = cumulSumSquaresValues[count] - cumulSumSquaresValues[count - numInThreshold]
        end
        mean = sum / numInThreshold
      end

      local clean_pct = pct .. " "
      clean_pct = string.gsub(clean_pct, '[.]', '_')
      clean_pct = string.gsub(clean_pct, '-', 'top')

      buffer[#buffer + 1] = "stats." .. m.name .. ".count_"       .. clean_pct .. numInThreshold    .. timestamp
      buffer[#buffer + 1] = "stats." .. m.name .. ".mean_"        .. clean_pct .. mean              .. timestamp
      if pct > 0 then
      buffer[#buffer + 1] = "stats." .. m.name .. ".upper_"       .. clean_pct .. thresholdBoundary .. timestamp
      else
      buffer[#buffer + 1] = "stats." .. m.name .. ".lower_"       .. clean_pct .. thresholdBoundary .. timestamp
      end
      buffer[#buffer + 1] = "stats." .. m.name .. ".sum_squares_" .. clean_pct .. sumSquares        .. timestamp

      ::continue::
    end

    sum = cumulativeValues[count]
    sumSquares = cumulSumSquaresValues[count]
    mean = sum / count

    local sumOfDiffs = 0
    for i = 1, count do
      sumOfDiffs = sumOfDiffs + (values[i] - mean) * (values[i] - mean)
    end

    local mid = math.floor(count/2)
    local median = 0
    if count % 2 == 0 then
      median = (values[mid] + values[mid+1])/2
    else
      median = values[mid+1]
    end

    local stddev = math.sqrt(sumOfDiffs / count)


    buffer[#buffer + 1] = "stats." .. m.name .. ".std "         .. stddev                          .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".upper "       .. max                             .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".lower "       .. min                             .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".count "       .. count                           .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".count_ps "    .. count/(now - m.last_flush_time) .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".sum "         .. sum                             .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".sum_squares " .. sumSquares                      .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".mean "        .. mean                            .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".median "      .. median                          .. timestamp


    --histogram--
    if #(m.histogram_bins) > 0 then
      local bins_count = #(m.histogram_bins)
      local bins = m.histogram_bins
      local bin_i = 1
      i = 1
      for bin_i =1, bins_count do
        local freq  = 0
        while i <= count and values[i] <= bins[bin_i] do
          freq = freq +1
          i = i + 1
        end

        local metric_name = bins[bin_i] .. " "
        metric_name = string.gsub(metric_name, "[.]", "_")
        metric_name = "stats." .. m.name .. ".histogram.bin_" .. metric_name
        buffer[#buffer + 1] = metric_name .. freq .. timestamp

        if bin_i == bins_count then
          -- the last bin
          freq = count - i + 1
          buffer[#buffer + 1] = "stats." .. m.name .. ".histogram.bin_inf " .. freq .. timestamp
          break
        end
      end
    end

  elseif m.counter > 0 then
    -- counter --
    buffer[#buffer + 1] = "stats." .. m.name .. ".count "    .. m.counter                           .. timestamp
    buffer[#buffer + 1] = "stats." .. m.name .. ".count_ps " .. m.counter/(now - m.last_flush_time) .. timestamp
  else
    m.last_flush_time = now
    return
  end

  send_graphite_udp_packet(buffer)

  m.last_flush_time = now
  if m.reset_after_flush == true then
    m:reset()
  end
end

----------------------------------------------

metric = {
  name              = "unknown",
  start_time        = 0,
  flush_intervals   = DEFAULT_FLUSH_INTERVALS,
  last_flush_time   = 0,
  reset_after_flush = true,
  ----------------
  counter           = 0,
  timers            = {},
  pctThreshold      = DEFAULT_PERCENTAGE_THRESHOLD,
  sample_rate       = DEFAULT_SAMPLE_RATE,
  histogram_bins    = DEFAULT_HISTOGRAM_BINS
}

function metric:new (o)
  local o = o or {}
  setmetatable(o, self)
  self.__index = self
  return o
end

-- Clear the counter
function metric:reset()
  self.start_time  = 0
  self.counter     = 0
  self.timers      = {}
end

function metric:increment (value)
  local v = value or 1
  self.counter = self.counter + v
  flush_metric(self)
end

function metric:decrement (value)
  local v = value or 1
  self.counter = self.counter - v
  flush_metric(self)
end

function metric:starttimer ()
  -- self.start_time = socket.gettime()
  self.start_time = fast_gettime()
  return self.start_time
end

function metric:stoptimer (start_time)
  local t0 = start_time or self.start_time
  -- local t1 = socket.gettime()
  local t1 = fast_gettime()

  local duration = math.floor((t1-t0)*1000)
  -- print(self.name ..  " used time: ".. duration .."ms")

  self.counter = self.counter + 1
  self.timers[self.counter] = duration

  flush_metric(self, t1)
end



return {
  flush_metric = flush_metric,
  metric = metric
}
```

quick_sort.cpp
```cpp
#include <algorithm>

#include "windows.h"
#include "stdint.h"

extern "C"  {
  __declspec(dllexport) void quick_sort(int * buf, int size)
  {
    std::sort(buf, buf + size);

  }

  __declspec(dllexport) double gettime() {
    uint64_t  intervals;
    FILETIME  ft;

    GetSystemTimeAsFileTime(&ft);

    /*
     * A file time is a 64-bit value that represents the number
     * of 100-nanosecond intervals that have elapsed since
     * January 1, 1601 12:00 A.M. UTC.
     *
     * Between January 1, 1970 (Epoch) and January 1, 1601 there were
     * 134744 days,
     * 11644473600 seconds or
     * 11644473600,000,000,0 100-nanosecond intervals.
     *
     * See also MSKB Q167296.
     */

    intervals = ((uint64_t)ft.dwHighDateTime << 32) | ft.dwLowDateTime;
    intervals -= 116444736000000000;
		
	struct timeval tp;
    tp.tv_sec = (long)(intervals / 10000000);
    tp.tv_usec = (long)((intervals % 10000000) / 10);
	double now = tp.tv_sec + ((double)tp.tv_usec )/ 1000000;
	return now;
  }
}
```

hello.lua
```lua
local i = 1
local t0 = socket.gettime()

for i =1, 10000000 do
  s:starttimer()
  s:stoptimer()
end

local t1 = socket.gettime()
require "math"
local duration = math.floor((t1-t0)*1000)
print(" used time: ".. duration .."ms")


```

run_luajit.bat
```bat
set ZBS=C:\ZeroBraneStudioEduPack-0.60-win32
set LUA_PATH=./?.lua;%ZBS%/lualibs/?/?.lua;%ZBS%/lualibs/?.lua
set LUA_CPATH=./?.dll;%ZBS%/bin/?.dll;%ZBS%/bin/clibs/?.dll
luajit.exe hello.lua
pause
```
用c代码实现排序和时间的获取之后，性能提升了6倍左右。上面的循环耗时从5500毫秒左右降低到880毫秒作用。



