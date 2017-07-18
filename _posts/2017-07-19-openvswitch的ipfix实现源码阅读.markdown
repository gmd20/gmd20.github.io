Open vSwitch Architectural Overview
===================================
[Porting Open vSwitch to New Software or Hardware]( http://docs.openvswitch.org/en/latest/topics/porting/)
参考这个文档的的介绍理解ovs-vswitchd ofproto netdev dpif几个层次关系。

ovs源码目录里面
 
datapath 目录应该是底层内核接口代码
ofproto  应该是用户空间的openflow协议的实现


ofproto-provider.h 是接口定义
=============================
struct ofproto_class {
  packet_xlate（） 把啮合网络包translate成openflow包
  packet_xlate_revert（） 反方向从 openflow处理完发回 内核
  set_ipfix()  所有openflow 的provider实现都要支持这个 iffix设置。
  get_ipfix_stats 
}


ipfix在用户空间实现的
=====================
ofproto-dpif.c dpif是对ovs项目的一个ofproto_class接口的一个内置实现
这个provider实现就支持了 set_ipfix 接口

dpif_ipfix_run
  dpif_ipfix_cache_expire   // 应该是定时发送ipfix包了
    ipfix_send_data_msg
      ipfix_send_msg

从dpif_ipfix_run函数代码可以看到ovs除了支持基于bridge全局的ipfix，还支持基于某个flow的单独的ipfix。 只对某些匹配的流才做ipfix监控可能对性能影响更小了。
这个基于流的细粒度ipfix监控应该是通过sample action来实现的吧


用户空间处理内核的translate请求
===============================
ofproto-dpif-upcall.c 
process_upcall()  //这个应该是收到upcall已经在用户空间了，
  switch (classify_upcall(upcall->type, userdata)) {
  case MISS_UPCALL:   // 内核里面没有找到flow对应的action的话，先发到用户空间来构建（translate）action。
    upcall_xlate(udpif, upcall, odp_actions, wc);
  case IPFIX_UPCALL:    // 这个是全局的
    dpif_ipfix_bridge_sample()    
  case FLOW_SAMPLE_UPCALL:   // 基于流的采样
    dpif_ipfix_flow_sample()  // 这个应该就是对网络包做分析统计了
       ipfix_cache_update
          hmap_insert              // ovs自己实现的一个hmap 哈希表吧，里面缓存所有的流
          ipfix_update_stats


用户空间构建（translate）flow的action
=====================================
/* If flow IPFIX is enabled, make sure IPFIX flow sample action
 * at egress point of tunnel port is just in front of corresponding
 * output action. If bridge IPFIX is enabled, this appends an IPFIX
 * sample action to 'ctx->odp_actions'. */
static void
compose_ipfix_action(struct xlate_ctx *ctx, odp_port_t output_odp_port)



/* Appends a "sample" action for sFlow or IPFIX to 'ctx->odp_actions'.  The
 * 'probability' is the number of packets out of UINT32_MAX to sample.  The
 * 'cookie' (of length 'cookie_size' bytes) is passed back in the callback for
 * each sampled packet.  'tunnel_out_port', if not ODPP_NONE, is added as the
 * OVS_USERSPACE_ATTR_EGRESS_TUN_PORT attribute.  If 'include_actions', an
 * OVS_USERSPACE_ATTR_ACTIONS attribute is added.  If 'emit_set_tunnel',
 * sample(sampling_port=1) would translate into datapath sample action
 * set(tunnel(...)), sample(...) and it is used for sampling egress tunnel
 * information.
 */
static size_t
compose_sample_action(struct xlate_ctx *ctx,
                      const uint32_t probability,
                      const union user_action_cookie *cookie,
                      const size_t cookie_size,
                      const odp_port_t tunnel_out_port,
                      bool include_actions)


/* Translates the flow, actions, or rule in 'xin' into datapath actions in
 * 'xout'.
 * The caller must take responsibility for eventually freeing 'xout', with
 * xlate_out_uninit().
 * Returns 'XLATE_OK' if translation was successful.  In case of an error an
 * empty set of actions will be returned in 'xin->odp_actions' (if non-NULL),
 * so that most callers may ignore the return value and transparently install a
 * drop flow when the translation fails. */
enum xlate_error
xlate_actions(struct xlate_in *xin, struct xlate_out *xout)



xlate_actions()  // 根据用户配置的流表，为flow构建action处理动作，
do_xlate_actions()
  xlate_sample_action()  // ovs里面有一个sample action应该就是对应这个了吧
xlate_output_action()    // output action输出网络包到某个端口之前也调用ipfix action。

output_normal()->compose_output_action() -> compose_output_action__()
    ->compose_ipfix_action()->compose_sample_action()->odp_put_userspace_action()


xlate_send_packet()
  ofproto_dpif_execute_actions__()
    xlate_actions()
    dpif_execute()


ofproto-dpif-trace.c

从上面和这个trace的实现知道，ovs碰到一个新的flow，先根据配置信息，进行translate（dpif-xlate.c)操作，构建这个flow对应的action。
xlate构建的时候在output这个action之前会插入ipfix的action。如果flow的action构建成功，也就是知道怎么处理这个flow的包了，最后应该
把构建好的action发回内核，让内核缓存起来吧，后面在处理这个流应该就不需要再发送到用户空间来处理了，避免内核和用户空间的context switch的开销吧。
从代码来看如果ovs的ipfix设置起作用了，ovs应该会在合适的地方插入ipfix的action的。 ovs的包处理其实也很简单，收到一个网络包就找flow流，然后根据流
执行flow已定义的action。




内核用updcall通知用户空间构建flow对应的action
===========================================
vport.c 
datapath.c

ovs_vport_receive()  // 内核收到网络包
 ovs_dp_process_packet()
     flow = ovs_flow_tbl_lookup_stats(&dp->table, key, skb_get_hash(skb),
					 &n_mask_hit);
     if (unlikely(!flow)) {
        upcall.cmd = OVS_PACKET_CMD_MISS;
        ovs_dp_upcall() 
          queue_userspace_packet()    // 没有找到flow对应的actin，就发到用户空间去translate 构建action
          queue_gso_packets()
     } else {
       ovs_execute_actions()
     }



actions.c
  do_execute_actions     // 执行flow对应的action
    case OVS_ACTION_ATTR_USERSPACE:    // 执行action的时候，如果遇到预先构建好的用户空间的action，比如ipfix的aciton，就发到用户空间处理。
      output_userspace()
         upcall.cmd = OVS_PACKET_CMD_ACTION;
         ovs_dp_upcall()

看看Linux的实现， 确实从port上面收到网络包之后，在ovs_dp_process_packet 处理的时候先是查找缓存的flow的action，
如果找到就执行aciton了，没找到就通过upcall传到用户空间vswitchd进程处理。










