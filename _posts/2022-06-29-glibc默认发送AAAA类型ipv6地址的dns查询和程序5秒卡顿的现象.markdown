发现 nslookup  www.baidu.com  curl https://www.baidu.com  经常5秒才会返回，
但nslookup其实很快就返回了A类型的ipv4地址了。 自己抓包发现是， 默认会发送 A 和AAAA这两种类型的 dns查询，
AAAA 类型的响应慢或者无响应导致的，可能有些dns代理缓存对 AAAA类型支持不好导致，还挺常见的。

但AAAA类型查询用不上，但却禁止不了，linux下面禁用ipv6这些都没有用。
这个好像是glibc的默认行为，并没有设置可以禁止掉，  /etc/resolv.conf 的选项没有相关的设置, https://man7.org/linux/man-pages/man5/resolv.conf.5.html
提到的一些设置应该没有用的。
```text
              inet6  Sets RES_USE_INET6 in _res.options.  This has the
                     effect of trying an AAAA query before an A query
                     inside the gethostbyname(3) function, and of
                     mapping IPv4 responses in IPv6 "tunneled form" if
                     no AAAA records are found but an A record set
                     exists.  Since glibc 2.25, this option is
                     deprecated; applications should use getaddrinfo(3),
                     rather than gethostbyname(3).
              single-request (since glibc 2.10)
                     Sets RES_SNGLKUP in _res.options.  By default,
                     glibc performs IPv4 and IPv6 lookups in parallel
                     since version 2.9.  Some appliance DNS servers
                     cannot handle these queries properly and make the
                     requests time out.  This option disables the
                     behavior and makes glibc perform the IPv6 and IPv4
                     requests sequentially (at the cost of some slowdown
                     of the resolving process).

              single-request-reopen (since glibc 2.9)
                     Sets RES_SNGLKUPREOP in _res.options.  The resolver
                     uses the same socket for the A and AAAA requests.
                     Some hardware mistakenly sends back only one reply.
                     When that happens the client system will sit and
                     wait for the second reply.  Turning this option on
                     changes this behavior so that if two requests from
                     the same port are not handled caorrectly it will
                     close the socket and open a new one before sending
                     the second request.                     
``` 
比较新的curl可以通过
curl  --doh-url https://dns.alidns.com/dns-query   https://www.baidu.com 绕过dns的超时，不过并不是想要的结果吧。


curl -4 https://www.baidu.com  好像按照文档是可以避免 AAAA查询的，用 centos 7自带的版本测试确实没有发送AAAA记录出来.


看了一下curl的代码， --ipv4这个参数是在 tool_getparam.c 文件里面赋值给   config->ip_version 的。

7.61.1 版本里面 asyn-ares.c 的Curl_resolver_getaddrinfo 函数是有 根据 ip_version 来执行的不同的版本的。
```c
/*
 * Curl_resolver_getaddrinfo() - when using ares
 *
 * Returns name information about the given hostname and port number. If
 * successful, the 'hostent' is returned and the forth argument will point to
 * memory we need to free after use. That memory *MUST* be freed with
 * Curl_freeaddrinfo(), nothing else.
 */
Curl_addrinfo *Curl_resolver_getaddrinfo(struct connectdata *conn,
                                         const char *hostname,
                                         int port,
                                         int *waitp)
{
  char *bufp;
  struct Curl_easy *data = conn->data;
  struct in_addr in;
  int family = PF_INET;
#ifdef ENABLE_IPV6 /* CURLRES_IPV6 */
  struct in6_addr in6;
#endif /* CURLRES_IPV6 */

  *waitp = 0; /* default to synchronous response */

  /* First check if this is an IPv4 address string */
  if(Curl_inet_pton(AF_INET, hostname, &in) > 0) {
    /* This is a dotted IP address 123.123.123.123-style */
    return Curl_ip2addr(AF_INET, &in, hostname, port);
  }

#ifdef ENABLE_IPV6 /* CURLRES_IPV6 */
  /* Otherwise, check if this is an IPv6 address string */
  if(Curl_inet_pton (AF_INET6, hostname, &in6) > 0)
    /* This must be an IPv6 address literal.  */
    return Curl_ip2addr(AF_INET6, &in6, hostname, port);

  switch(conn->ip_version) {
  default:
#if ARES_VERSION >= 0x010601
    family = PF_UNSPEC; /* supported by c-ares since 1.6.1, so for older
                           c-ares versions this just falls through and defaults
                           to PF_INET */
    break;
#endif
  case CURL_IPRESOLVE_V4:
    family = PF_INET;
    break;
  case CURL_IPRESOLVE_V6:
    family = PF_INET6;
    break;
  }
#endif /* CURLRES_IPV6 */

  bufp = strdup(hostname);
  if(bufp) {
    struct ResolverResults *res = NULL;
    free(conn->async.hostname);
    conn->async.hostname = bufp;
    conn->async.port = port;
    conn->async.done = FALSE;   /* not done */
    conn->async.status = 0;     /* clear */
    conn->async.dns = NULL;     /* clear */
    res = calloc(sizeof(struct ResolverResults), 1);
    if(!res) {
      free(conn->async.hostname);
      conn->async.hostname = NULL;
      return NULL;
    }
    conn->async.os_specific = res;

    /* initial status - failed */
    res->last_status = ARES_ENOTFOUND;
#ifdef ENABLE_IPV6 /* CURLRES_IPV6 */
    if(family == PF_UNSPEC) {
      if(Curl_ipv6works()) {
        res->num_pending = 2;

        /* areschannel is already setup in the Curl_open() function */
        ares_gethostbyname((ares_channel)data->state.resolver, hostname,
                            PF_INET, query_completed_cb, conn);
        ares_gethostbyname((ares_channel)data->state.resolver, hostname,
                            PF_INET6, query_completed_cb, conn);
      }
      else {
        res->num_pending = 1;

        /* areschannel is already setup in the Curl_open() function */
        ares_gethostbyname((ares_channel)data->state.resolver, hostname,
                            PF_INET, query_completed_cb, conn);
      }
    }
    else
#endif /* CURLRES_IPV6 */
    {
      res->num_pending = 1;

      /* areschannel is already setup in the Curl_open() function */
      ares_gethostbyname((ares_channel)data->state.resolver, hostname, family,
                         query_completed_cb, conn);
    }

    *waitp = 1; /* expect asynchronous response */
  }
  return NULL; /* no struct yet */
}

```


但 7.84.0 版本的代码已经改成忽略 命令行参数里面的ip_version输入的值了，直接根据 Curl_ipv6works（）函数的结果确定是否 发送AAAA查询了，
看注释大概是系统支持ipv6就会总是发送AAAA请求了。
```c
/*
 * Curl_resolver_getaddrinfo() - when using ares
 *
 * Returns name information about the given hostname and port number. If
 * successful, the 'hostent' is returned and the forth argument will point to
 * memory we need to free after use. That memory *MUST* be freed with
 * Curl_freeaddrinfo(), nothing else.
 */
struct Curl_addrinfo *Curl_resolver_getaddrinfo(struct Curl_easy *data,
                                                const char *hostname,
                                                int port,
                                                int *waitp)
{
  char *bufp;

  *waitp = 0; /* default to synchronous response */

  bufp = strdup(hostname);
  if(bufp) {
    struct thread_data *res = NULL;
    free(data->state.async.hostname);
    data->state.async.hostname = bufp;
    data->state.async.port = port;
    data->state.async.done = FALSE;   /* not done */
    data->state.async.status = 0;     /* clear */
    data->state.async.dns = NULL;     /* clear */
    res = calloc(sizeof(struct thread_data), 1);
    if(!res) {
      free(data->state.async.hostname);
      data->state.async.hostname = NULL;
      return NULL;
    }
    data->state.async.tdata = res;

    /* initial status - failed */
    res->last_status = ARES_ENOTFOUND;

#ifdef HAVE_CARES_GETADDRINFO
    {
      struct ares_addrinfo_hints hints;
      char service[12];
      int pf = PF_INET;
      memset(&hints, 0, sizeof(hints));
#ifdef CURLRES_IPV6
      if(Curl_ipv6works(data))
        /* The stack seems to be IPv6-enabled */
        pf = PF_UNSPEC;
#endif /* CURLRES_IPV6 */
      hints.ai_family = pf;
      hints.ai_socktype = (data->conn->transport == TRNSPRT_TCP)?
        SOCK_STREAM : SOCK_DGRAM;
      msnprintf(service, sizeof(service), "%d", port);
      res->num_pending = 1;
      ares_getaddrinfo((ares_channel)data->state.async.resolver, hostname,
                       service, &hints, addrinfo_cb, data);
    }
#else

#ifdef HAVE_CARES_IPV6
    if(Curl_ipv6works(data)) {
      /* The stack seems to be IPv6-enabled */
      res->num_pending = 2;

      /* areschannel is already setup in the Curl_open() function */
      ares_gethostbyname((ares_channel)data->state.async.resolver, hostname,
                          PF_INET, query_completed_cb, data);
      ares_gethostbyname((ares_channel)data->state.async.resolver, hostname,
                          PF_INET6, query_completed_cb, data);
    }
    else
#endif
    {
      res->num_pending = 1;

      /* areschannel is already setup in the Curl_open() function */
      ares_gethostbyname((ares_channel)data->state.async.resolver,
                         hostname, PF_INET,
                         query_completed_cb, data);
    }
#endif
    *waitp = 1; /* expect asynchronous response */
  }
  return NULL; /* no struct yet */
}
```
