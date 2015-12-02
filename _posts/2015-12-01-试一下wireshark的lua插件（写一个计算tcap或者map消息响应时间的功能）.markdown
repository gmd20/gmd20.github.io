其他计算任何request/reply的协议的响应时间的wireshark插件都可以用类似代码实现吧。比如计算什么自定义rpc的响应时间等等。
完成的最终脚本 tcap_response_time.lua

```lua
--  Know issues:
--  This offical wireshark "tcap stat feature" can not identify the correct
--  tcap session (the tcap message matching is wrong, it seems the hash
--  function in packet_tcap.c has some problems.)
--
--  This script is base on tcap.otid and tcap.dtid only, the tcap message
--  matching may be wrong.,  if two tcap dailog between many DPCs use the
--  same transation_id.
--  The workaround is mannually split the capure files using filter like
--  "gsm_old.localValue == 45" or sccp.digits == "gt number", until each
--  capture file contains only one direction of 1 to 1 tcap messages,
--  that is to say no duplicate transation_id in one capture file.
--

print "wireshark tcap response time lua plugin from gmd20"
local original_m3ua_dissector
local tcap_requests_time_table = {}

-- declare some Fields to be read
-- local frame_time_f = Field.new("frame.time")
-- local frame_len_f = Field.new("frame.len")
local frame_number_f = Field.new("frame.number")
local frame_epochtime_f = Field.new("frame.time_epoch")
local tcap_otid_f = Field.new("tcap.otid")
local tcap_dtid_f = Field.new("tcap.dtid")
-- declare our (pseudo) protocol
local tcap_time_proto = Proto("tcap_rsp_time","TCAP response time")
-- create the fields for our "protocol"
-- local req_time_F = ProtoField.string("tcap_rsp_time.req_time","request time")
-- local rsp_time_F = ProtoField.string("tcap_rsp_time.time","response time")
local req_frame_number_F = ProtoField.string("tcap_rsp_time.req_frame_number","request frame number")
local req_time_F = ProtoField.double("tcap_rsp_time.req_time","request time")
local rsp_time_F = ProtoField.double("tcap_rsp_time.time","response time")
-- add the field to the protocol
tcap_time_proto.fields = {req_frame_number_F,req_time_F,rsp_time_F}

-- create a function to "postdissect" each frame
function tcap_time_proto.dissector(buffer,pinfo,tree)
  -- we've replaced the original http dissector in the dissector table,
  -- but we still want the original to run, especially because we need to read its data
  original_m3ua_dissector:call(buffer, pinfo, tree)

  -- obtain the current values the protocol fields
  local otid = tcap_otid_f()
  local dtid = tcap_dtid_f()
  local epochtime = tonumber(tostring(frame_epochtime_f()))
  if otid then
    local otid_s = tostring(otid)
    tcap_requests_time_table[otid_s] = epochtime
    -- local subtree = tree:add(tcap_time_proto,"TCAP response time")
    -- subtree:add(req_time_F, tostring(otid))
    -- subtree:add(rsp_time_F, tostring(epochtime))
  elseif dtid then
    local dtid_s = tostring(dtid)
    if tcap_requests_time_table[dtid_s] ~= nil then
      local req_time = tcap_requests_time_table[dtid_s];
      local duration = epochtime - req_time
      if duration >= 0 and duration < 10 then
        local subtree = tree:add(tcap_time_proto,"TCAP response time")
        local frame_number = frame_number_f()
        subtree:add(req_frame_number_F, tostring(frame_number))
        subtree:add(req_time_F,req_time)
        -- subtree:add(rsp_time_F,duration)
        subtree:add(rsp_time_F,duration * 1000) -- wireshark's "io graph"'s auto scale doesn't work
      end
    end
  end
end

-- register our protocol as a postdissector.
-- our dissector funtion get called on every packet
-- register_postdissector(tcap_time_proto)

-- replace original m3ua dissector,
-- so our dissector function get called on every tcap message
local sctp_payload_dissector_table = DissectorTable.get("sctp.ppi")
original_m3ua_dissector = sctp_payload_dissector_table:get_dissector(3) -- save the original dissector so we can still get to it
sctp_payload_dissector_table:add(3, tcap_time_proto)                    -- and take its place in the dissector
```


把这个文件，放到 C:\Program Files\Wireshark\plugins\1.12.8  目录里面去，然后
修改C:\Program Files\Wireshark\init.lua,  检查disable_lua 等变量设置，确保lua插件功能已经启用。

wireshark 启动的时候就会自动加载我们这个脚本。  在wireshark的 about 窗体上面可以查看 这个插件目录在哪里
试一下wireshark的lua插件（写一个计算tcap或者map消息响应时间的功能） - widebright - widebright的个人空间
 

wireshark官方文档的lua插件的例子是个很好的参考。
https://wiki.wireshark.org/Lua
https://wiki.wireshark.org/Lua/Dissectors

lua插件执行的效果：
-----------------
会多出来几个节点
 



简单介绍一下lua代码：
-------------------

``lua
local tcap_otid_f = Field.new("tcap.otid")
local tcap_dtid_f = Field.new("tcap.dtid")
```
这种是读取其他已经解析出来的属性， 直接可以在 wireshark里面的filter输入框里面输入使用的属性。

```lua
local tcap_time_proto = Proto("tcap_rsp_time","TCAP response time")
local req_time_F = ProtoField.double("tcap_rsp_time.req_time","request time")
```
这种是自己的协议要增加的属性，整个就是一个树形结构。看参考wireshark的文档。


这样的注册方式，应该是每个包会调用一次我们dissector 函数。
```lua
register_postdissector(tcap_time_proto)
```


这种替换m3ua dissector的chain  dissector方式，是每个tcap的消息都被调用到一次。 因为一个网络包里面有可能有好几个tcap消息，所以最后采用后面这种方式。我们希望针对每个tcap消息进行处理。
```lua
  -- we've replaced the original http dissector in the dissector table,
  -- but we still want the original to run, especially because we need to read its data
  original_m3ua_dissector:call(buffer, pinfo, tree)

-- replace original m3ua dissector,
-- so our dissector function get called on every tcap message
local sctp_payload_dissector_table = DissectorTable.get("sctp.ppi")
original_m3ua_dissector = sctp_payload_dissector_table:get_dissector(3) -- save the original dissector so we can still get to it
sctp_payload_dissector_table:add(3, tcap_time_proto)                    -- and take its place in the dissector
```


dissector  具体要替换哪一个合适，要看DissectorTable 里面哪个Dissector 被使用来解析网络包。
wireshark 菜单 internal  ->  DissectorTable  可以查看到当前的DissectorTable  是什么样的。
这个是我们替换之后的，



wiershark 菜单  tools ->  lua - >  evaluate 会出来lua窗口，可以直接输入lua代码执行调试。比如检查DissectorTable这些
是不是对的，或者 在lua插件代码里面 print 或者 通过设置输出一些调试属性来调试代码都可以。

这个tcap的lua插件有个问题，就是只比较transaction id的，所有如果一个抓包文件里面有好多不同节点后者方向的tcap连接的话，可能结构连接的transaction id 刚好有冲突，消息的匹配就有问题了。所以使用之前先自己过滤一下，把一个方向的tcap消息全部过滤出来，保存到单个文件，这样transaction id不会有冲突，就可以正常使用。

新加的  tcap_rsp_time.time  可以作为wireshark的过滤条件和io graph里面的统计时间使用。和http.time类似的。
