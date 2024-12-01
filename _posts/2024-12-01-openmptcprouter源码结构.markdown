openmptcprouter数据流
=====================
openmptcprouter由本地的router和vps服务器组成，网络流量是走ss和glorytun通道到vps上面在出去吧。ss和glorytun被改造过，用multipath tcp连接的，所有能够聚合利用多个接口的带宽。


网页上的配置
============
https://github.com/Ysurac/openmptcprouter-feeds/blob/develop/luci-app-openmptcprouter/luasrc/controller/openmptcprouter.lua
配置向导里面设置server的 ip 和 username password openmptcprouter_vps_key


页面后台服务
============
https://github.com/Ysurac/openmptcprouter-feeds/blob/develop/openmptcprouter-api/files/usr/libexec/rpcd/openmptcprouter

```bash
function default_vpn(default_vpn)
	-- Get VPN set by default
	local vpn_port = ""
	local vpn_intf = ""
	if default_vpn:match("^glorytun.*") then
		vpn_port = 65001
		vpn_intf = "tun0"
		--ucic:set("network","omrvpn","proto","dhcp")
		ucic:set("network","omrvpn","proto","none")
		if default_vpn == "glorytun_udp" then
			ucic:set("glorytun","vpn","proto","udp")
			ucic:set("glorytun","vpn","localip","10.255.254.2")
			ucic:set("glorytun","vpn","remoteip","10.255.254.1")
			ucic:set("network","omr6in4","ipaddr","10.255.254.2")
			ucic:set("network","omr6in4","peeraddr","10.255.254.1")
		else
			ucic:set("glorytun","vpn","proto","tcp")
			ucic:set("glorytun","vpn","localip","10.255.255.2")
			ucic:set("glorytun","vpn","remoteip","10.255.255.1")
			ucic:set("network","omr6in4","ipaddr","10.255.255.2")
			ucic:set("network","omr6in4","peeraddr","10.255.255.1")
		end
```

和VPS服务器的交互
=================
VPS有一个json接口，用页面配置的用户名密码连接服务器，从服务器上拿到具体的ss和glorytun等 通道的配置用户名密码之类的
https://github.com/Ysurac/openmptcprouter-feeds/blob/develop/openmptcprouter/files/etc/init.d/openmptcprouter-vps

```text
https://$server:$serverport/token
https://$server:$serverport/config

_set_glorytun_vps() {
   [ -z "$vps_config" ] && vps_config=$(_get_json "config")

_set_ss_server_vps() {
```


网络链路监控
============
比如master 和backup服务器切换等
https://github.com/Ysurac/openmptcprouter-feeds/blob/develop/omr-tracker/files/bin/omr-tracker-ss    
https://github.com/Ysurac/openmptcprouter-feeds/tree/develop/omr-tracker/files/usr/share/omr    
https://github.com/Ysurac/openmptcprouter-feeds/blob/develop/omr-schedule/files/usr/share/omr/schedule.d/010-services#L10   



各个子服务
==========
https://github.com/Ysurac/openmptcprouter-feeds/tree/develop/mptcp   
https://github.com/Ysurac/openmptcprouter-feeds/tree/develop/glorytun   
https://github.com/Ysurac/openmptcprouter-feeds/tree/develop/shadowsocks-libev   


vps上运行的服务
===============
https://github.com/Ysurac/openmptcprouter-vps   
https://github.com/Ysurac/openmptcprouter-vps-admin/blob/develop/omr-admin.py   
json接口就是这个python脚本

```python
# Get VPS config
@app.get('/config', summary="Get full server configuration for current user")
async def config(userid: Optional[int] = Query(None), serial: Optional[str] = Query(None), current_user: User = Depends(get_current_user)):
    LOG.debug('Get config...')
    if not current_user.permissions == "admin":
        userid = current_user.userid
    if userid is None:
        userid = 0
    username = get_username_from_userid(userid)
    if not current_user.permissions == "admin" and serial is not None:
        if not check_username_serial(username, serial):
            return {'error': 'False serial number'}
    with open('/etc/openmptcprouter-vps-admin/omr-admin-config.json') as f:
        try:
            omr_config_data = json.load(f)
        except ValueError as e:
            with open('/etc/openmptcprouter-vps-admin/omr-admin-config.json') as f:
                try:
                    omr_config_data = json.load(f)
                except ValueError as e:
                    omr_config_data = {}
    LOG.debug('Get config... shadowsocks')
    proxy = 'shadowsocks'
    if 'proxy' in omr_config_data['users'][0][username]:
        proxy = omr_config_data['users'][0][username]['proxy']

    if os.path.isfile('/etc/shadowsocks-libev/manager.json'):
        with open('/etc/shadowsocks-libev/manager.json') as f:
            content = f.read()
        content = re.sub(",\s*}", "}", content) # pylint: disable=W1401
        try:
            data = json.loads(content)
        except ValueError as e:
            data = {'server_port': 65101, 'method': 'chacha20'}
    else:
        data = {'server_port': 65101, 'method': 'chacha20'}
    #shadowsocks_port = data["server_port"]
    shadowsocks_port = current_user.shadowsocks_port
    shadowsocks_key = ''
    if shadowsocks_port is not None:
        if 'port_key' in data:
            shadowsocks_key = data["port_key"][str(shadowsocks_port)]
        elif 'port_conf' in data:
            shadowsocks_key = data["port_conf"][str(shadowsocks_port)]["key"]
    shadowsocks_method = data["method"]
    if 'fast_open' in data:
        shadowsocks_fast_open = data["fast_open"]
    else:
        shadowsocks_fast_open = False
    if 'reuse_port' in data:
        shadowsocks_reuse_port = data["reuse_port"]
    else:
        shadowsocks_reuse_port = False
    if 'no_delay' in data:
        shadowsocks_no_delay = data["no_delay"]
    else:
        shadowsocks_no_delay = False
    if 'mptcp' in data:
        shadowsocks_mptcp = data["mptcp"]
    else:
        shadowsocks_mptcp = False
    if 'ebpf' in data:
        shadowsocks_ebpf = data["ebpf"]
    else:
        shadowsocks_ebpf = False
    if "plugin" in data:
        shadowsocks_obfs = True
        if 'v2ray' in data["plugin"]:
            shadowsocks_obfs_plugin = 'v2ray'
        else:
            shadowsocks_obfs_plugin = 'obfs'
        if 'tls' in data["plugin_opts"]:
            shadowsocks_obfs_type = 'tls'
        else:
            shadowsocks_obfs_type = 'http'
    else:
        shadowsocks_obfs = False
        shadowsocks_obfs_plugin = ''
        shadowsocks_obfs_type = ''
    shadowsocks_port = current_user.shadowsocks_port
    if not shadowsocks_port == None and proxy == 'shadowsocks':
        ss_traffic = get_bytes_ss(current_user.shadowsocks_port)
    else:
        ss_traffic = 0

    LOG.debug('Get config... glorytun')
    if os.path.isfile('/etc/glorytun-tcp/tun' + str(userid) +'.key'):
        glorytun_key = open('/etc/glorytun-tcp/tun' + str(userid) + '.key').readline().rstrip()
    else:
        glorytun_key = ''
    glorytun_port = '65001'
    glorytun_chacha = False
    glorytun_tcp_host_ip = ''
    glorytun_tcp_client_ip = ''
    glorytun_udp_host_ip = ''
    glorytun_udp_client_ip = ''
    if os.path.isfile('/etc/glorytun-tcp/tun' + str(userid)):
        with open('/etc/glorytun-tcp/tun' + str(userid), "r") as glorytun_file:
            for line in glorytun_file:
                if 'PORT=' in line:
                    glorytun_port = line.replace(line[:5], '').rstrip()
                if 'LOCALIP=' in line:
                    glorytun_tcp_host_ip = line.replace(line[:8], '').rstrip()
                if 'REMOTEIP=' in line:
                    glorytun_tcp_client_ip = line.replace(line[:9], '').rstrip()
                if 'chacha' in line:
                    glorytun_chacha = True
    if userid == 0 and glorytun_tcp_host_ip == '':
        if 'glorytun_tcp_type' in omr_config_data:
            if omr_config_data['glorytun_tcp_type'] == 'static':
                glorytun_tcp_host_ip = '10.255.255.1'
                glorytun_tcp_client_ip = '10.255.255.2'
            else:
                glorytun_tcp_host_ip = 'dhcp'
                glorytun_tcp_client_ip = 'dhcp'
        else:
            glorytun_tcp_host_ip = '10.255.255.1'
            glorytun_tcp_client_ip = '10.255.255.2'
    if os.path.isfile('/etc/glorytun-udp/tun' + str(userid)):
        with open('/etc/glorytun-udp/tun' + str(userid), "r") as glorytun_file:
            for line in glorytun_file:
                if 'LOCALIP=' in line:
                    glorytun_udp_host_ip = line.replace(line[:8], '').rstrip()
                if 'REMOTEIP=' in line:
                    glorytun_udp_client_ip = line.replace(line[:9], '').rstrip()

    if userid == 0 and glorytun_udp_host_ip == '':
        if 'glorytun_udp_type' in omr_config_data:
            if omr_config_data['glorytun_udp_type'] == 'static':
                glorytun_udp_host_ip = '10.255.254.1'
                glorytun_udp_client_ip = '10.255.254.2'
            else:
                glorytun_udp_host_ip = 'dhcp'
                glorytun_udp_client_ip = 'dhcp'
        else:
            glorytun_udp_host_ip = '10.255.254.1'
            glorytun_udp_client_ip = '10.255.254.2'
    available_vpn = ["glorytun_tcp", "glorytun_udp"]
```
