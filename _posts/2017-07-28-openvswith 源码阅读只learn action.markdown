learn action 是根据packet模式学习，再创建和添加新的flow到flow table的一个action

"learn action" 和"Resubmit action" 还有 "metadata registers" 、"stack操作"
是openvswitch 可编程flow table机制得以实现的关键吧。 有了这几样才能建立复杂的
“flow pipeline”



learn action的执行
==================
```text
./lib/learn.c:
./ofproto/ofproto-dpif-xlate.c:


upcall_xlate
  xlate_actions
    do_xlate_actions
      case OFPACT_LEARN:
        xlate_learn_action(ctx, ofpact_get_LEARN(a));
          learn_execute  // 这个函数会执行ovs的learn action的那些动作，设置寄存器等，应该在这里准备好了要新建的flow
          ofproto_flow_mod_learn  // 这个好像会把新建的流加到table里面去。

/* Composes 'fm' so that executing it will implement 'learn' given that the
 * packet being processed has 'flow' as its flow.
 *
 * Uses 'ofpacts' to store the flow mod's actions.  The caller must initialize
 * 'ofpacts' and retains ownership of it.  'fm->ofpacts' will point into the
 * 'ofpacts' buffer.
 *
 * The caller has to actually execute 'fm'. */
void
learn_execute(const struct ofpact_learn *learn, const struct flow *flow,
              struct ofputil_flow_mod *fm, struct ofpbuf *ofpacts)
    mf_read_subfield
    ofpact_put_reg_load
    ofpact_put_OUTPUT


learn action执行的时候的信息，应该是parse之后就解析出来的一些属性都保存在这里了吧
/* OFPACT_LEARN.
 *
 * Used for NXAST_LEARN. */
struct ofpact_learn {
    OFPACT_PADDED_MEMBERS(
        struct ofpact ofpact;

        uint16_t idle_timeout;     /* Idle time before discarding (seconds). */
        uint16_t hard_timeout;     /* Max time before discarding (seconds). */
        uint16_t priority;         /* Priority level of flow entry. */
        uint8_t table_id;          /* Table to insert flow entry. */
        enum nx_learn_flags flags; /* NX_LEARN_F_*. */
        ovs_be64 cookie;           /* Cookie for new flow. */
        uint16_t fin_idle_timeout; /* Idle timeout after FIN, if nonzero. */
        uint16_t fin_hard_timeout; /* Hard timeout after FIN, if nonzero. */
        /* If the number of flows on 'table_id' with 'cookie' exceeds this,
         * the action will not add a new flow. 0 indicates unlimited. */
        uint32_t limit;
        /* Used only if 'flags' has NX_LEARN_F_WRITE_RESULT.  If the execution
         * failed to install a new flow because 'limit' is exceeded,
         * result_dst will be set to 0, otherwise to 1. */
        struct mf_subfield result_dst;
    );

    struct ofpact_learn_spec specs[];
};



这个ofproto_flow_mod结构保存了要新加的flow是什么样子的flow吧，应该是复用了
mod-flows 这个action的结构了，文档也是这个写的
learn(argument[,argument]...)
       This action adds or modifies a flow in an OpenFlow table,
       similar  to  ovs-ofctl --strict mod-flows.

/* flow_mod with execution context. */
struct ofproto_flow_mod {
    /* Allocated by 'init' phase, may be freed after 'start' phase, as these
     * are not needed for 'revert' nor 'finish'. */
    struct rule *temp_rule;
    struct rule_criteria criteria;
    struct cls_conjunction *conjs;
    size_t n_conjs;

    /* Replicate needed fields from ofputil_flow_mod to not need it after the
     * flow has been created. */
    uint16_t command;
    bool modify_cookie;
    /* Fields derived from ofputil_flow_mod. */
    bool modify_may_add_flow;
    bool modify_keep_counts;
    enum nx_flow_update_event event;

    /* These are only used during commit execution.
     * ofproto_flow_mod_uninit() does NOT clean these up. */
    ovs_version_t version;              /* Version in which changes take
                                         * effect. */
    bool learn_adds_rule;               /* Learn execution adds a rule. */
    struct rule_collection old_rules;   /* Affected rules. */
    struct rule_collection new_rules;   /* Replacement rules. */
};



/* Executes the flow modification specified in 'fm'.  Returns 0 on success, or
 * an OFPERR_* OpenFlow error code on failure.
 *
 * This is a helper function for in-band control and fail-open. */
enum ofperr
ofproto_flow_mod(struct ofproto *ofproto, const struct ofputil_flow_mod *fm)
    OVS_EXCLUDED(ofproto_mutex)
{
    return handle_flow_mod__(ofproto, fm, NULL);
}

```



learn action的解析
==================
```text
ofpacts_parse
  ofpact_parse
    parse_LEARN
      learn_parse

ofpact_parse 用X macro额写法，解析所有的action的都在 ofp-actions.c
文件里面parse_XXX函数里面做的。


static char * OVS_WARN_UNUSED_RESULT
ofpact_parse(enum ofpact_type type, char *value,
             const struct ofputil_port_map *port_map, struct ofpbuf *ofpacts,
             enum ofputil_protocol *usable_protocols)
{
    switch (type) {
#define OFPACT(ENUM, STRUCT, MEMBER, NAME)                              \
        case OFPACT_##ENUM:                                             \
            return parse_##ENUM(value, port_map, ofpacts, usable_protocols);
        OFPACTS
#undef OFPACT
    default:
        OVS_NOT_REACHED();
    }
}



/* Returns NULL if successful, otherwise a malloc()'d string describing the
 * error.  The caller is responsible for freeing the returned string. */
static char * OVS_WARN_UNUSED_RESULT
learn_parse__(char *orig, char *arg, const struct ofputil_port_map *port_map,
              struct ofpbuf *ofpacts)




这部分代码看上去是解析openflow controller的返回结果，
看来controler里面也是可以返回这个learn action的。

```



ovs 内置的的mac learning
========================
```text

xlate_normal(struct xlate_ctx *ctx)
  /* Learn source MAC. */
  update_learning_table
    update_learning_table__
      mac_learning_update

   mac = mac_learning_lookup(ctx->xbridge->ml, flow->dl_dst, vlan);   // mac表的学习 vlan + mac
   mac_port = mac ? mac_entry_get_port(ctx->xbridge->ml, mac) : NULL;
   if (mac_port) {
     output_normal(ctx, mac_xbundle, &xvlan);
   } else {
     xlate_normal_flood(ctx, in_xbundle, &xvlan);
   }


xlate_mac_learning_update
  update_learning_table__
     mac_learning_update

```



最近没时间细看了，其实还不太清楚这个learn action。 从上面的代码知道learn_execute函数应该是 learn action的执行过程没错，这个是在用户空间执行的，
跟通常的直接在内核执行的aciton不一样，这种learn 每个网络包都要发到用户空间来学习一下，context switch会比较多影响性能？ 不过这个learn添加flow的
动作确实在用户空间来做比较好。现在对learn action是怎么编译成内核上方用户空间那个动作还不是很清楚，其实是还没怎么搞懂 openvswitch代码里面  xlate 
和 ofproto的代码，看上去ovs会把用户配置的流规则转成内部的二进制描述结构和openflow的标准消息的结构之类的吧。以后有时间再看一下。
