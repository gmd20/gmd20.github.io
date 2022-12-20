```text
ngx_http_parse_request_line
解析记录host
request->host_start
request->host_end



ngx_http_process_request_line

阿里的connect代理
 if (r->connect_host_start && r->connect_host_end) {
   r->connect_host  记录解析出来的coonect方法的host
   r->connect_port_n

   if (r->schema_end) {
     r->schema.len = r->schema_end - r->schema_start;
     r->schema.data = r->schema_start;
   }
   if (r->host_end) // http代理协议 url里面host
                if (ngx_http_set_virtual_server(r, &host) == NGX_ERROR) {
                    break;
                }
                r->headers_in.server = host;
   }

http header里面的HOST
ngx_http_process_host
  ngx_http_set_virtual_server
    rc = ngx_http_find_virtual_server(r->connection,
                                      hc->addr_conf->virtual_names, // 对应监听端口的对应的所有server
                                      host, r, &cscf);



处理http头里面的User-Agent记录记录浏览器类型
ngx_http_process_user_agent
    unsigned                          msie:1;
    unsigned                          msie6:1;
    unsigned                          opera:1;
    unsigned                          gecko:1;
    unsigned                          chrome:1;
    unsigned                          safari:1;
    unsigned                          konqueror:1;
} ngx_http_headers_in_t



ngx_http_core_find_config_phase
  ngx_http_core_find_location
    ngx_http_core_find_static_location
  找到ngx_http_core_loc_conf_t
```
