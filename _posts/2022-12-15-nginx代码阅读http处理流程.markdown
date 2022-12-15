```text
配置文件command的处理函数
ngx_http_core_server    // server命令
ngx_http_core_location  // locaotion命令


加载配置
ngx_http_block
  ngx_http_init_phases
  module->postconfiguration
  ngx_http_variables_init_vars
  ngx_http_init_phase_handlers



ngx_http_subrequest
  ngx_http_core_run_phases

ngx_http_add_listening
ngx_http_init_connection
ngx_http_process_request_line
  ngx_http_parse_request_line
  ngx_http_process_request_uri
  ngx_http_set_virtual_server
  ngx_http_process_request_headers
  ngx_http_process_request
    ngx_http_handler
      ngx_http_core_run_phases
        ngx_http_core_find_config_phase
          ngx_http_core_find_location
        ngx_http_core_generic_phase
        ngx_http_core_access_phase
  ngx_http_run_posted_requests



扩展模块
ngx_module.h
struct ngx_module_s {

模块handler的注册
static ngx_int_t
ngx_http_access_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);  
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_access_handler;

    return NGX_OK;
}

模块handler函数注册例子
static ngx_int_t
ngx_http_proxy_connect_init(ngx_conf_t *cf)
{
    ngx_http_core_main_conf_t  *cmcf;
    ngx_http_handler_pt        *h;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_POST_READ_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_proxy_connect_post_read_handler;

    return NGX_OK;
}


调用各个phase的handler，包含ngx_http_core_find_config_phase和各个modules里面注册的那些吧
void
ngx_http_core_run_phases(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_phase_handler_t   *ph;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

    ph = cmcf->phase_engine.handlers;

    while (ph[r->phase_handler].checker) {

        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

        if (rc == NGX_OK) {
            return;
        }
    }
}



请求处理的哥哥phase
typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,

    NGX_HTTP_SERVER_REWRITE_PHASE,

    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,
    NGX_HTTP_POST_REWRITE_PHASE,

    NGX_HTTP_PREACCESS_PHASE,

    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,

    NGX_HTTP_PRECONTENT_PHASE,

    NGX_HTTP_CONTENT_PHASE,

    NGX_HTTP_LOG_PHASE
} ngx_http_phases;

```
