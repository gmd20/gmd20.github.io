http://www.infoq.com/cn/Dockers/presentations/

容器技术栈，开源项目
====================
```
资源调度 Mesos Swarm Kubernetes
服务编排 Compose Kubernetes marathon
监控  cAdvisor Sysdig Datadog
访问管理 LDAP AD  Github  SAML
镜像，发布 DockerHub  Quay.io  Artifactory
分布式计划任务 chronos
容器引擎 docker Rkt Triton VIC
安全  Notary Vault
服务注册和发现，分布式数据库Etcd consul+registrator MongoDB
存储  Ceph ，Gluster ， OpenStack swift， glance 
网络  VXLAN IPSEC HAProxy
容器系统  Red Hat， Ubuntu， CoreOS， RancherOS
资源 AWS VMware OpenStack
```



0. 一个云计算人对容器技术现在以及未来的冷思考
==============================================
http://www.infoq.com/cn/presentations/thinking-on-the-container-technology?utm_source=presentations_about_cloud-computing&utm_medium=link&utm_campaign=cloud-computing



1.  知乎万级规模容器平台架构和实战
===================================

http://www.infoq.com/cn/presentations/platform-architecture-and-combatof-zhihu-container-platform

知乎采用Mesos+Docker容器方式构建弹性计算平台，配合Consul和Haproxy进行服务发现及负载均衡，并结合持续集成系统提升业务的整体开发效率。目前，知乎绝大部分业务都已经平稳的运行在该平台之上。
```
知乎使用 mesos （http://mesos.apache.org/） 管理容器。
没有容器 配置一个haproxy作为入口，haproxy可以提供详细统计、流量的熔断保护、

docker镜像分层优化，依赖包不需要更新
docker stop命令， 会发送SIGTERM通知容器进程退出。
服务发现用consul（https://www.consul.io/） service注册 基于Key-value
docker exec 模拟ssh 登录
docker register的分布式优化和负载均衡
```

2. 新浪微博混合云DCP平台的设计难点与业务上云实践
================================================
http://www.infoq.com/cn/presentations/sina-weibo-hybrid-cloud-dcp-platform

随着公有云的逐渐成熟，Container技术的兴起，一时间各大型企业纷纷开始升级内部运维系统，提供自动化能力的同时，更注重弹性调度。新浪微博14年就开始尝试应用Docker，并逐步Docker化Web类业务。截止目前，已构建好一套基于私有云+公有云的弹性调度平台，内称混合云DCP，上线以来G95%的Web类业务全部迁入此平台。本次分享包括但不限于以下内容： 1. 混合云DCP平台的架构设计与难点解析 2. 业务Docker化中遇到的问题及解决方案 3. 降低成本，解析基于公有云的弹性方案 4. 业务迁移至DCP及上云实践 

```
热点峰值应对--自动化扩容
 公有云： 主要aliyun ，还有用到其他云 AWS ， VPC 的 VM池。
 用Compose（https://docs.docker.com/compose/） 和Swarm （https://github.com/docker/swarmkit）管理docker
  服务发现consul  nginx  motan SLB DNS  
  

 微博内部网络和 阿里云 之间必须使用 专线 ，平常1G专线，春节用10G专线。redis等数据同步用到
2015年10上线，4000+ 容器
```

3. 京东618大促，15万个Docker实例背后的技术挑战
===============================================
http://www.infoq.com/cn/presentations/jingdong-618-docker-instance-behind-technical-challenges

在2016年的京东618大促中，基于Docker和OpenStack的弹性云项目担当重任，在应用层面，京东所有应用100%通过容器技术来发布和管理应用集群，同时还有5600个容器实例支撑DB集群。弹性云项目经过了618这样大流量高并发的大促活动的考验。在本次演讲中，京东弹性云项目负责人永成将会首次独家分享15万Docker容器背后的技术挑战以及京东的解决方案，同时，还会重点分析京东在容器化进程中遇到的坑和经验。


4. 基于 Docker 的云处理服务：更新、扩容与自动容错
==================================================
 http://www.infoq.com/cn/presentations/cloud-processing-services-based-on-docker
又拍云在云处理服务升级过程中进行了容器化以及微服务化实践，过程中遇到了许多新的挑战，例如经常迁移的微服务如何做高效的负载均衡、动态发现、健康检查；如何在不影响线上业务的情况下做弹性伸缩、服务更新、故障处理；如何对服务进行快速灵活的权限控制、速率控制、参数检查等。这里主要分享我们团队在微服务化过程中为解决以上问题所做的一些工作。特别的，将着重介绍我们是如何利用 ngx_lua 实现动态的服务路由来很好地实现动态扩容、zero downtime 更新与自动容错的。 
```
负载均衡用nginx。
consul 注册的服务变化之后，自己实现居于 ngx_lua的动态负载均衡 
```

5. 携程基于容器的持续交付设计&实现
==================================
http://www.infoq.com/cn/presentations/design-and-implementation-of-the-continuous-delivery-of-ctrip
本次将介绍携程如何设计基于容器的持续交付，以容器镜像作为发布单元，实现开发、测试、生产环境的一致性，打造全新的灰度发布&扩容模式；分享Kubernetes与Mesos的取舍、网络模型的选择、Windows Server Container POC、运维监控等实战经验。
```
Chronos/Mesos  的动态调度  
pychronos
```

6.同程旅游5000个Docker实例上的CI实践
=====================================
http://www.infoq.com/cn/presentations/ci-practice-on-ly-travel-5000-docker-instances

同程旅游在引入Docker容器部署后在持续集成时带来了一些新的想法的同时也来新的挑战，如何的一个有数千个容器的实例的环境中做好CI，又如何通过容器的引入让开发真正的DevOps起来，其中涉及到基于容器的压缩式发布流程，容器的开发自扩容管理，Docker Image管理（量大、多版本等等），运维与开发在新的CI中的分工等。同程旅游将以上种种集成为一个管理平台，本次分享将会详细介绍同程旅游的基于容器的CI实践，重点谈坑和经验。 
```
web应用，mysql redis，mongodb等都运行在 docker。 cpu 利用率由20% 提高到 80%
```



7. 同程旅游支持亿级访问量的私有云服务架构探索
=============================================
http://www.infoq.com/cn/presentations/cloud-architecture-of-ly-support-billion-level-of-access
如何基于Docker容器技来解决基础环境虚拟化过重的问题，又是怎样使用云化的思维开发的缓存，消息等中间件撑起同程旅游海量订单交易的



8.  Kubernetes一周年背后的技术演进
=================================
http://www.infoq.com/cn/presentations/technology-evolution-of-kubernetes-in-first-anniversary
2015到2016年是Kubernetes正式release后经历过的完整一年。在此期间，容器技术圈初步形成了Google联盟与Mesosphere和Docker公司各领风骚的格局，Kubernetes项目也逐步完成了从Borg/Omega中的抽象理论模型到公认的容器集群化实践标准之间的自我进化。本次演讲将从一线开发者和维护者的角度，结合大规模作业调度与管理的理论演进，探讨Kubernetes项目在这段时间中技术视野与实现手段上的演化和发展，重点涵盖Kubernetes的核心功能实现、调度机制演化，以及更多前瞻性功能的深入分析等多方面细节。 


9. 面向容器平台的存储系统
=========================
http://www.infoq.com/cn/presentations/storage-system-for-container-platform
```
有容云  
comet ， docker volume plugin
```


10.  网易蜂巢Docker云计算平台架构演化
=====================================
http://www.infoq.com/cn/presentations/netease-docker-cloud-computing-platform-architecture-evolution

网易蜂巢是基于Docker 容器打造的云计算平台，以提高开发效率，降低研发成本为核心目标，提供极佳的用户体验，极速的性能，全面助力开发者开发云端应用。网易云之前多年稳定运行、支撑了众多重量级网易互联网产品（门户/新闻客户端、云音乐、考拉等）的线上业务运行。为了支持公有云的建设，我们对平台整体架构进行了优化演变，包括机房建设，基础设施，编排系统，容器服务，镜像仓库，管理服务等，同时也面临更多高难度的技术及工程复杂度的挑战，本分享主要介绍蜂巢云服务构架的演化过程及工程实践经验分享。
```
openstack构建aaS， PaaS
云网络基于openFow的浮动ip
云硬盘基于iscsi实现
Ceph
```

11. Kubernetes在企业中的场景运用及管理实践
==========================================
http://www.infoq.com/cn/presentations/application-and-management-practice-of-kubernetes-in-enterprise
```
时速云

网络- Flannel：   https://github.com/coreos/flannel  
   Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址。 

网络- Calico：  https://www.projectcalico.org/  
    这个开源项目是比较有趣的针对Neutron SDN的方案，当时投入了一些精力在其开源代码上，受到了很大的启发。Calico设计的思路其实非常简单直白，它在每台计算节点上安装了一个BGP Speaker，通过软件实现了Virtualized L3 Fabric。然后在架构上又引入了BGP Reflector解决full mesh的问题。


存储-  docker的volume plugin很多
etcd：用于服务发现的键值存储系统    https://github.com/coreos/etcd
```

12.  知乎 Docker 弹性计算平台的架构和实践
========================================
http://www.infoq.com/cn/presentations/zhihua-docker-elastic-computing-platform-architecture-and-practice

知乎规模的迅速增大，业务对计算资源的快速伸缩有了强烈的需求；内部服务化的架构推进，需要更好的环境隔离以及更细力度的计算资源隔离和调度能力。时下火热的 Docker 容器技术自然的成为我们构建弹性计算平台的基础方案。知乎采用 Mesos 进行底层资源的管理和调度，通过原生的 Docker 部署方式来管理容器实例，配合 Consul 和 Haproxy 进行服务的注册和接入，并和持续集成系统进行集成来提升业务的整体开发效率。目前，知乎绝大部分业务都已经平稳的运行在 Docker 弹性计算平台之上。 本次演讲介绍知乎 Docker 弹性计算平台的架构设计的来龙去脉，介绍将业务迁移到 Docker 平台上这一实施过程的实践经验和体会思考。 




13 云端基于Docker微服务应用的架构实践
=====================================
http://www.infoq.com/cn/presentations/docker-micro-service-application-architecture-practice
```
阿里云 工程师分享

容器编排 docker compose 
容器集群管理 docker swarm 。
阿里云 集成 
docker volumne plugin  支持不同的存储类型
服务注册 Spring Cloud
```

14 同程Docker 大规模应用之路
================================
http://www.infoq.com/cn/presentations/chinatechday-suzhou-docker



15 阿里聚石塔电商云容器服务应用和实践
======================================
http://www.infoq.com/cn/presentations/ali-electricity-supplier-cloud-container-service



