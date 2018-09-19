```text
很早之前就看到这篇文章
Measure Anything, Measure Everything
http://codeascraft.com/2011/02/15/measure-anything-measure-everything/

所以那时就玩了一会Graphite，用c++ 写了一个Graphite的前端，把数据推送给它。但那时觉得Statsd也是一个Graphite前端，又是node.js的，觉得安装起来又要多花点时间，就没去看。

最近我们组用到lua，我打算边学习lua边写一个简单的Graphite出来。这样就可以在lua脚本里面也把监控信息放到Graphite里面绘图了。今天摸索了一下。才发现其实就是要实现一个类似Statsd的功能啊。

Understanding StatsD and Graphite
http://blog.pkhamre.com/2012/07/24/understanding-statsd-and-graphite/

Practical Guide to StatsD/Graphite Monitoring
http://matt.aimonetti.net/posts/2013/06/26/practical-guide-to-graphite-monitoring/

Counting & Timing
http://code.flickr.net/2008/10/27/counting-timing/

参考第一篇文章的解释，很清楚了。我之前是看文章的时候不够仔细。我写的lua脚本其实也是为了计算counter和timer两种类型的数据而已，还有考虑到一些 90% 和平均值等，这些都在Statsd有实现了，人家考虑的比自己做的清楚多了。
Statsd其实也是一个200/300行代码的简单脚本而已，看上去直接把statsd的代码翻译一下成为lua就可以了。
如果不是说不想安装一个statsd的服务器的话，甚至是有现成的实现可以用的。
https://github.com/stvp/lua-statsd-client/blob/master/src/statsd.lua
如果用这个就数据先推送到statsd，再由statsd推送到graphite而已。
不过把statsd这个简单的脚本翻译成lua也不难吧。明天去公司试试看。

https://github.com/etsy/statsd/


2014-05-25 补充，最终完成了这个模块在这里
https://github.com/gmd20/lua-statsd
----------------------------------------------
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
    table.sort(m.timers)
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
  self.start_time = socket.gettime()
  return self.start_time
end

function metric:stoptimer (start_time)
  local t0 = start_time or self.start_time
  local t1 = socket.gettime()
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



http://gmd20.blog.163.com/blog/static/16843923201442845754283/  这里有一个在Windows优化版本，要比这个实现要快6倍左右。


lua-statsd
A Lua module to send statistics to Graphite， a clone of Statsd

说明
用Lua代码实现了类似Statsd的功能，可以记录counter和timer统计信息，然后通过UDP接口发送给Graphite。可用作程序的监控或者统计接口。在Graphite中可以图形查看统计情况。

使用例子
local statsd = require "statsd"
require "socket"
local s = statsd.metric:new {name= "testing_metric"}


s:starttimer()
socket.select(nil, nil, 1)
s:stoptimer()

s:starttimer()
socket.select(nil, nil, 0.01)
s:stoptimer()

s:starttimer()
socket.select(nil, nil, 0.876)
s:stoptimer()

socket.select(nil, nil, 10)

s:starttimer()
s:stoptimer()



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

类似或者相关项目
Statsd
lua-statsd-client
lua-statsd
查看直方图（histogram）
http://localhost:9000/render/?height=300&
width=740&from=-24h&title=Render time histogram&
vtitle=relative frequency in %&yMax=100&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_01,stats.timers.render_time.count),100),'2FFF00'),'0.01')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_05,stats.timers.render_time.count),100),'64DD0E'),'0.05')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_1,stats.timers.render_time.count),100),'9CDD0E'),'0.1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_5,stats.timers.render_time.count),100),'DDCC0E'),'0.5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_1,stats.timers.render_time.count),100),'DDB70E'),'1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_5,stats.timers.render_time.count),100),'FF6200'),'5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_10,stats.timers.render_time.count),100),'FF3C00'),'10')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_50,stats.timers.render_time.count),100),'FF1E00'),'50')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_inf,stats.timers.render_time.count),100),'FF0000'),'inf')&
lineMode=slope&areaMode=stacked&drawNullAsZero=false&hideLegend=false
http://localhost:9000/render/?height=300&
width=740&from=-24h&title=Render time histogram&
vtitle=relative frequency in %, leaving out first class&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_05,stats.timers.render_time.count),100),'64DD0E'),'0.05')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_1,stats.timers.render_time.count),100),'9CDD0E'),'0.1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_5,stats.timers.render_time.count),100),'DDCC0E'),'0.5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_1,stats.timers.render_time.count),100),'DDB70E'),'1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_5,stats.timers.render_time.count),100),'FF6200'),'5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_10,stats.timers.render_time.count),100),'FF3C00'),'10')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_50,stats.timers.render_time.count),100),'FF1E00'),'50')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_inf,stats.timers.render_time.count),100),'FF0000'),'inf')&
lineMode=slope&areaMode=stacked&drawNullAsZero=false&hideLegend=false
http://localhost:9000/render/?height=300&
width=740&from=-24h&title=Render time histogram&
vtitle=rel. freq with scale adjustment per band&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_01,stats.timers.render_time.count),0.01),'2FFF00'),'0.01')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_05,stats.timers.render_time.count),0.04),'64DD0E'),'0.05')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_1,stats.timers.render_time.count),0.05),'9CDD0E'),'0.1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_0_5,stats.timers.render_time.count),0.4),'DDCC0E'),'0.5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_1,stats.timers.render_time.count),0.5),'DDB70E'),'1')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_5,stats.timers.render_time.count),4),'FF6200'),'5')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_10,stats.timers.render_time.count),5),'FF3C00'),'10')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_50,stats.timers.render_time.count),40),'FF1E00'),'50')&
target=alias(color(scale(divideSeries(stats.timers.render_time.bin_inf,stats.timers.render_time.count),60),'FF0000'),'inf')&
lineMode=slope&areaMode=stacked&drawNullAsZero=false&hideLegend=false
可以使用上面这几个Graphite的render接口调用，视图稍微有点不同，参考http://dieter.plaetinck.be/histogram-statsd-graphing-over-time-with-graphite.html
```
