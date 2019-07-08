title: spring-boot1.3.5支持jsp导致浏览器访问应用偶尔慢
author: Xiang Chuang
tags:
  - 这些年，那些坑
categories:
  - springboot
  - jsp
date: 2019-07-08 14:42:00
---
本文将阐述在基于spring-boot1.3.5日常开发中所遇到的一个疑惑的问题，下面将按照问题处理的过程进行行文！！！

一、项目简介：
	这儿有一个业务系统，我们称之为M，架构基于springboot1.3.5，同时提供了web能力，前后端分离。线上生产环境中，外部商户的【取现业务】会跳转到本系统的页面上进行操作，实际场景中更多的是手机端进行页面跳转，但也可以PC端网页版跳转，因为M是基于spring-mvc提供的能力（这是对项目的通用认知，但某些关键的细节点并没有包含在里面，以至于在后续问题分析中走了很多弯路，耗时2天半才定位到问题）。
    
二、问题现象
	业务运行过程中，外部商户吐槽声来了，为什么跳转到你们的页面那么慢，耗时3-4s甚至更久，且频次比较高（后经运维部门同时查看，发现超过10%的请求出现慢的情况）。由于外部商户是移动端app，因此业务跳转耗时3-4s反馈到用户那儿的便于点击确认操作后白板3-4s。这确实是一个糟糕的用户体验！另，据相关小伙伴反映，这是一个由来已久的问题，只是以前业务少，而且是自己的前端业务部门使用，有时忍忍就过去了。（这儿提到这一点，因为，如果能证实以前的慢和现在的慢问题一样，那么可以推导出并不是新的改动后其他操作带来的新问题，这对问题定位有指导性意义，但分析问题时总归是仓促的，这些信息极容易被丢掉）。
    
三、业务描述
	为了阐述问题的分析过程，业务场景必须清晰，下面简介一下该业务场景：
    
![upload successful](\images\pasted-88.png)

四、问题排查
	通过前面的描述，我们知道了业务场景、系统情况及问题现象，接下来，我们开始了问题排查的过程，这过程，是痛苦的！！！
a、首先，我们跟进商户提供的流水号进行日志查询，发现商户请求的时间和我们底层系统M实际收到的时间确实相差了3-4s，于是，我们找到了入口系统Api，因为整个业务场景，是由Api进行的页面跳转，会不会是Api做了什么事，导致页面跳转慢了？
b、通过查询Api的日志，发现收到请求的时间确实和商户时间一致，我们似乎找到问题根源，即Api有问题，但当进一步排查时，发现Api和M的交互只是做了简单的302重定向，并无其他逻辑，且发现Api发起重定向的时间到M“接收”到请求便耗时了3-4s，于是，我们把矛头执行了网络问题，加之跳转过程是用的域名跳转，我们更深信这3-4s花在了网络上！
c、我们怀疑是否是网络问题，于是找到运维的小伙伴，一通排查，回复我们网络没问题，并给了我们ng日志
![upload successful](\images\pasted-89.png)
d、这个回复让我们不甘心，因为我们的Api正常的发起了跳转请求，我们的M3-4s后才接收到请求，那这耗时一定是在网络上啊，并多次反复追问信息中心域名等相关网络问题，后来，我通过PC页面直接进行测试（因为我也是该产品的用户），当我一顿操作并点击确认按钮后，咦，很快呀，接着我反复的重试、刷新、重试，果不其然偶尔出现了一笔3-4s的耗时，看来这真是偶尔出现呀（新那这不应该是应用问题呀），紧接着我事用谷歌浏览器看了请求情况，发现如下信息：
![upload successful](\images\pasted-90.png)，咦，这耗时花在了TTFB上面呀，然后百度一查，TTFB包含了tcp握手时间即服务端返回第一个字节的时间。哦，那这不正说明了网络问题嘛！同时我发现处理页面偶尔慢，某些纯后台服务或者图片也慢呀，于是又找到了运维同事，这已经大半天过去了（虽然中间处理问题断断续续）
e、这时，运维同事按照我们的方式测试，也确实出现了偶尔慢，他们抓包分析，发现当前网络正常呀，但有一个现象，慢的请求有三次重复ACK的现象，这是什么鬼？运维同事说这是正常的，TCP包乱序会重复发起ACK嘛，我将信将疑，但问题没说通啊，为什么慢，一小时过去了。这时更多的同事加入到问题分析，运维小伙伴为证明不是网络问题，在线上服务器直接ip访问页面，发现，也会出现慢，于是，我们也选择相信了网络没问题，但问题是什么？
f、这时我们测试过程中，又发现了某些图片资源404，某些图片资源即使存在但偶尔会慢，咦，是不是因为页面访问了资源而资源慢导致页面慢？这时其实问题的分析稍有偏差了，因为d的时候已经出现过类似问题，但没有包含资源的纯后台页面也慢啊！但花了近一天的分析，实在是太困惑，于是我们朝着这个方向去了。
g、我们把也慢扣了出来，并扔到了ng上，既然图片资源慢，我们改动页面，以至于后面仅仅是一个文本页面了，确实，访问不慢了（但前面说了，这已经偏离了，因为页面位于ng中，而不是项目中，但这也是分析的必经之地，因为这儿确实也存在问题），我们尝试加上css、js资源，慢又出现了，并且发现都是图表慢，但，这不应该了CDN上静态资源应该很快呀，但后面我们发现这些慢的图片并不位于CDN，而是在项目M中，同时也存在访问了CDN上不存在且并不需要依赖的资源。我们把图片映射做了一下ng映射（前面说的这是在ng上的页面），再次访问，正常了。于是结论出来了，至少，前端开发中，动静分离不合规，同时copy代码，无用的资源访问也出现了。这是今天耗时一天的分析结论。但。
h、第二天，上班后，按照昨日的分析，前端进行了标准化的优化，上线，但问题依旧（毕竟前面图片这个问题是问题，但不是这次问题的根本问题）。这时，大家伙都很无语了，不知道为什么了！运维小伙伴又将纯文本页面扔到M项目中，启动访问，发现，也会慢呀，什么情况！同时，相同页面单独扔到tomcat中启动就正常，哎，见鬼了。似乎无下文了。
i、这时M系统负责人突然说到，测试环境能重现慢的问题，（我似乎看到了一丝曙光），但本地启动没问题（这个。。。），，难道还是环境问题？我来到测试环境测试中，看着业务日志与gc日志，发现每次慢的时候就会做一次gc回收，咦，内存有问题？（实际是因为测试环境内存小，512M堆大小），但我又发现线上gc正常啊，但一点蛛丝马迹都得一试，把线上1.5G堆调成了4G堆，不过，问题依旧啊，似乎还慢的更频繁的样子（还是基于http访问又硬件负载均衡，正常业务跑步进来）。
j、心有不甘，但想着能重现问题，应该就能定位到问题，于是我开始了代码分析之路，和其他小伙伴从两个方向并行去分析，都耗时一天半了，心想着，两个思路一起搞，兴许快点。既然测试环境能重现，好吧，我试试本地是不是真重现不了，我打卡idea启动M，一顿尝试，好像真不可以呢。但我debug到了请求执行的环节，那，好吧，开启debug日志在测试环境看看，是哪个环节慢了？基于warn日志级别重启测试环境M，然后测试，发现，咦，真是的业务系统自己慢呢，已经收到请求了，在某个环节慢，而且，把页面扔到其他系统进行访问确没问题：
```
2019-07-05 19:05:38.955 DEBUG [http-nio-8004-exec-3] InternalNioInputBuffer:180-- Received [GET /creditCard/cardBinInfo.json HTTP/1.1
Host: 192.168.46.3:8004
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Mobile Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8
Cookie: _csrf=c25ca56a-799f-4e23-b179-b09abaf7fa0e; cavyboss-session=40c8e4d0-0d09-4952-8812-93a1f44b0068; mpay-session=0714ad72-0e83-43d9-8b87-b002c34a71f3

]
2019-07-05 19:05:38.956 DEBUG [http-nio-8004-exec-3] CoyoteAdapter:180-- The variable [uriBC] has value [/creditCard/cardBinInfo.json]
2019-07-05 19:05:38.957 DEBUG [http-nio-8004-exec-3] CoyoteAdapter:180-- The variable [semicolon] has value [-1]
2019-07-05 19:05:38.957 DEBUG [http-nio-8004-exec-3] CoyoteAdapter:180-- The variable [enc] has value [utf-8]
2019-07-05 19:05:42.640 DEBUG [http-nio-8004-exec-3] LegacyCookieProcessor:180-- Cookies: Parsing b[]: _csrf=c25ca56a-799f-4e23-b179-b09abaf7fa0e; cavyboss-session=40c8e4d0-0d09-4952-8812-93a1f44b0068; mpay-session=0714ad72-0e83-43d9-8b87-b002c34a71f3
2019-07-05 19:05:42.643 DEBUG [http-nio-8004-exec-3] CoyoteAdapter:180--  Requested cookie session id is 0714ad72-0e83-43d9-8b87-b002c34a71f3
2019-07-05 19:05:42.645 DEBUG [http-nio-8004-exec-3] RemoteIpValve:180-- Incoming request /creditCard/cardBinInfo.json with originalRemoteAddr '192.168.47.152', originalRemoteHost='192.168.47.152', originalSecure='false', originalScheme='http' will be seen as newRemoteAddr='192.168.47.152', newRemoteHost='192.168.47.152', newScheme='http', newSecure='false'
2019-07-05 19:05:42.645 DEBUG [http-nio-8004-exec-3] AuthenticatorBase:180-- Security checking request GET /creditCard/cardBinInfo.json
2019-07-05 19:05:42.646 DEBUG [http-nio-8004-exec-3] RealmBase:180--   No applicable constraints defined
2019-07-05 19:05:42.646 DEBUG [http-nio-8004-exec-3] AuthenticatorBase:180--  Not subject to any constraint
2019-07-05 19:05:42.653 DEBUG [http-nio-8004-exec-3] OrderedRequestContextFilter:114-- Bound request context to thread: org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper@5edc7f61
2019-07-05 19:05:42.654 DEBUG [http-nio-8004-exec-3] DispatcherServlet:863-- DispatcherServlet with name 'dispatcherServlet' processing GET request for [/creditCard/cardBinInfo.json]
2019-07-05 19:05:42.655 DEBUG [http-nio-8004-exec-3] Parameters:180-- Set encoding to UTF-8
2019-07-05 19:05:42.660 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.663 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.663 DEBUG [http-nio-8004-exec-3] EndpointHandlerMapping:301-- Looking up handler method for path /creditCard/cardBinInfo.json
2019-07-05 19:05:42.671 DEBUG [http-nio-8004-exec-3] EndpointHandlerMapping:311-- Did not find handler method for [/creditCard/cardBinInfo.json]
2019-07-05 19:05:42.671 DEBUG [http-nio-8004-exec-3] RequestMappingHandlerMapping:301-- Looking up handler method for path /creditCard/cardBinInfo.json
2019-07-05 19:05:42.692 DEBUG [http-nio-8004-exec-3] RequestMappingHandlerMapping:308-- Returning handler method [public java.lang.Object com.yiji.mobilepay.web.controller.CreditCardWithdrawControler.cardBinInfo(javax.servlet.http.HttpServletRequest,javax.servlet.http.HttpServletResponse,javax.servlet.http.HttpSession)]
2019-07-05 19:05:42.692 DEBUG [http-nio-8004-exec-3] DefaultListableBeanFactory:251-- Returning cached instance of singleton bean 'creditCardWithdrawControler'
2019-07-05 19:05:42.693 DEBUG [http-nio-8004-exec-3] DispatcherServlet:949-- Last-Modified value for [/creditCard/cardBinInfo.json] is: -1
2019-07-05 19:05:42.704 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.704 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.831 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResources(META-INF/services/com.alibaba.fastjson.serializer.AutowiredObjectSerializer)
2019-07-05 19:05:42.841 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.842 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.844 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.844 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.847 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.848 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.848 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.849 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.849 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.849 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.862 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findClass(com.yjf.common.lang.result.ViewResultInfoBeanInfo)
2019-07-05 19:05:42.862 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Returning ClassNotFoundException
2019-07-05 19:05:42.866 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findClass(com.yjf.common.lang.result.ViewResultInfoCustomizer)
2019-07-05 19:05:42.867 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Returning ClassNotFoundException
2019-07-05 19:05:42.869 DEBUG [dubbo-remoting-client-heartbeat-thread-1] HeartBeatTask:66--  [DUBBO] Send heartbeat to remote channel /192.168.46.3:20917, cause: The channel has no data-transmission exceeds a heartbeat period: 60000ms
2019-07-05 19:05:42.870 DEBUG [New I/O client worker #1-3] HeartbeatHandler:82--  [DUBBO] Receive heartbeat response in thread New I/O client worker #1-3
2019-07-05 19:05:42.874 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(Object.class)
2019-07-05 19:05:42.874 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.875 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(Object.class)
2019-07-05 19:05:42.876 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.876 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.877 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(String.class)
2019-07-05 19:05:42.878 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.879 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(String.class)
2019-07-05 19:05:42.879 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.880 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.881 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(StringBuilder.class)
2019-07-05 19:05:42.882 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.883 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(StringBuilder.class)
2019-07-05 19:05:42.883 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.884 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.887 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(com.class)
2019-07-05 19:05:42.887 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.888 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(com.class)
2019-07-05 19:05:42.889 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.889 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.890 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(java/lang/com.class)
2019-07-05 19:05:42.891 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.892 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(java/lang/com.class)
2019-07-05 19:05:42.892 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.893 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.894 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(com/yjf.class)
2019-07-05 19:05:42.894 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.895 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(com/yjf.class)
2019-07-05 19:05:42.896 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.896 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.897 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(com/yjf/common.class)
2019-07-05 19:05:42.898 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.899 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(com/yjf/common.class)
2019-07-05 19:05:42.899 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.900 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.901 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180-- getResource(com/yjf/common/util.class)
2019-07-05 19:05:42.901 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   Delegating to parent classloader org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8
2019-07-05 19:05:42.902 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     findResource(com/yjf/common/util.class)
2019-07-05 19:05:42.903 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--     --> Resource not found, returning null
2019-07-05 19:05:42.903 DEBUG [http-nio-8004-exec-3] WebappClassLoaderBase:180--   --> Resource not found, returning null
2019-07-05 19:05:42.908 DEBUG [http-nio-8004-exec-3] ToString:550-- classloader:TomcatEmbeddedWebappClassLoader
  context: ROOT
  delegate: true
----------> Parent Classloader:
org.springframework.boot.loader.LaunchedURLClassLoader@11a9e7c8

2019-07-05 19:05:42.910 DEBUG [http-nio-8004-exec-3] RequestResponseBodyMethodProcessor:225-- Written [ViewResultInfo{code=null,data=null,message=请通过网页正常步骤进行访问或者系统已超时,success=false}] as "application/json;charset=UTF-8" using [com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter@3220bfbc]
2019-07-05 19:05:42.911 DEBUG [http-nio-8004-exec-3] DispatcherServlet:1036-- Null ModelAndView returned to DispatcherServlet with name 'dispatcherServlet': assuming HandlerAdapter completed request handling
2019-07-05 19:05:42.911 DEBUG [http-nio-8004-exec-3] DispatcherServlet:997-- Successfully completed request
2019-07-05 19:05:42.911 DEBUG [http-nio-8004-exec-3] OrderedRequestContextFilter:104-- Cleared thread-bound request context: org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper@5edc7f61
2019-07-05 19:05:42.912 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.912 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.912 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.913 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
2019-07-05 19:05:42.913 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:125-- Opening RedisConnection
2019-07-05 19:05:42.913 DEBUG [http-nio-8004-exec-3] RedisConnectionUtils:205-- Closing Redis Connection
```
但是，这个环节去干什么了呢？源代码日志很少，看不出来呀！难道是分布式session问题，发现springboot分布式session我们内部事用redis实现的，后经一顿排查，发现上面日志中已经包含了redis的操作，但是是在慢的那个环节之后呀（不过提了个醒，以后redis要是抽风，这也是个风险点呢）。
k、现在知道问题就出在org.apache.catalina.connector.CoyoteAdapter这个类日志中显示的慢的那个环节了，但他究竟干了什么？貌似只是简单的一些操作啊，代码层次很深，今天也疲惫了，就没往深处看，为什么本地测试不出现问题？带着这些疑问，今天收班了。
l、今天没来是周六，但问题没解决，心有不甘呀，而且都能重现问题了，要是不定位到，好可惜。于是背着包来到了公司，反正今日无安排！来到公司，经过作为和今早放松了大脑，有准备整装待发了。本地不出问题，难道是打包环节不一样？两个私服不一致?于是我把测试环境的jar包拷了下来，java XXX -jar启动，咦，本地重现问题了！真是私服问题，包版本不一致？我又把本地项目用另一个私服打了jar包java XXX -jar启动，咦，问题也重现了，哦！原来是idea启动时候和java -jar有些区别，也许是classpath加载的区别吧，此处没深究，但很庆幸，本地重现问题，离问题定位再进一步
m、既然本地都重现问题了，虽然不能用idea断点测试，但我改改源码加加日志吧。ok！源码日志已加号，这下日志详细了吧，java XXX -jar启动，测试，啊！原来是这慢，还以为只是简单的map呢，被类没迷惑了：
```
2019-07-06 11:44:55.430 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- normaliztion start............................
2019-07-06 11:44:55.430 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- normaliztion end............................
2019-07-06 11:44:55.432 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- decodedURI start............................
2019-07-06 11:44:55.432 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- decodedURI end............................
2019-07-06 11:44:55.432 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check normaliztion start............................
2019-07-06 11:44:55.432 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check normaliztion end............................
2019-07-06 11:44:55.433 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check getUseIPVHosts start............................{}false
2019-07-06 11:44:55.433 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check getUseIPVHosts end............................{}localhost
2019-07-06 11:44:55.433 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check mapreu start............................{}
2019-07-06 11:45:00.570 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check mapreu end............................{}
2019-07-06 11:45:00.570 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check session start1............................{}
2019-07-06 11:45:00.570 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check session start4............................{}
2019-07-06 11:45:00.571 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check session start5............................{}
2019-07-06 11:45:00.571 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check session start6............................{}
2019-07-06 11:45:00.571 INFO  [http-nio-8004-exec-13] CoyoteAdapter:180-- check session start7............................{}
```
源码
```
// This will map the the latest version by default
            connector.getService().getMapper().map(serverName, decodedURI,
                    version, request.getMappingData());
```
此时内心激动，问题似乎更清晰了，打开idea，断点测试，往里面跟，发现了一个gateway页面，咦，这时我们定义的页面，全项目搜索一下，它干啥了！
源码：
```
 // Rule 1 -- Exact Match
        MappedWrapper[] exactWrappers = contextVersion.exactWrappers;
        internalMapExactWrapper(exactWrappers, path, mappingData);

        // Rule 2 -- Prefix Match
        boolean checkJspWelcomeFiles = false;
        MappedWrapper[] wildcardWrappers = contextVersion.wildcardWrappers;
        if (mappingData.wrapper == null) {
            internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting,
                                       path, mappingData);                                    
```
包含gateway的代码：
```
@Configuration
public class StargateConfigration {
	@Bean
	public ServletRegistrationBean dispatcherRegistration(StargateDispachServlet stargateDispachServlet) {
	    ServletRegistrationBean registration = new ServletRegistrationBean(
	    		stargateDispachServlet);
	    registration.addUrlMappings("/gateway");
	    return registration;
	}

```
咦，servlet，我顿时想起以前好像我们的spring-boot专门做了堆jsp的支持，于是来到我们的tomcat-staters，
```

	@Bean(name = "yijiEmbeddedServletContainerCustomizer")
	public EmbeddedServletContainerCustomizer embeddedServletContainerCustomizer() {
		
		return container -> {
			//1. disable jsp if possible
			if (!tomcatProperties.getJsp().isEnable()) {
				JspServlet jspServlet = new JspServlet();
				jspServlet.setRegistered(false);
				container.setJspServlet(jspServlet);
			}
```
难道是M系统启用了yiji.tomcat.jsp.enable=true？通过线上系统变量搜索，发现，确实，M启用了，其他系统没有，恍然大悟，这就是为什么M有问题而其他系统没问题的原因吧！于是我java -Dyiji.tomcat.jsp.enable=false -jar M，测试，果然，不慢了。。。问题终于定位到，spring-boot本身不推荐jsp，但提供了支持，这说明本身支持的方式就存在性能问题！
n、嗯，接下来，下掉jsp吧，本来这就是遗留问题，传统项目采用jsp，切换springboot后强行支持了jsp，但却有性能问题！还有安全漏洞！

下面说说spring-boot如何支持jsp吧。其实就是多注册一个JspServlet!
1、添加依赖
```
<dependency>
            <groupId>org.apache.tomcat.embed</groupId>
            <artifactId>tomcat-embed-jasper</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>jstl</artifactId>
        </dependency>
```
2、配置文件中指定jsp文件存放路径
```
#路径,在webapp文件夹下新建文件夹WEB-INF,在往下建文件夹jsp
spring.mvc.view.prefix=/WEB-INF/jsp/
#文件名的后缀,例如:index.jsp,放在jsp文件夹下
spring.mvc.view.suffix=.jsp
```
![upload successful](\images\pasted-91.png)
3、作为框架，我们默认不支持jsp，因为有性能损耗
```
if (!tomcatProperties.getJsp().isEnable()) {
				JspServlet jspServlet = new JspServlet();
				jspServlet.setRegistered(false);
				container.setJspServlet(jspServlet);
			}
```
4、业务系统根据自身情况决定是否启动jsp支持，启用，意味着接受性能损耗！

心得：
  本地问题分析历史2天半（中途很多时间在处理其他事情），心理感受颇多，分析线上类似问题一定要思路清晰，先网路再应用，另外一定要有主线，如果能本地重现问题，那可是天大的耗时，源码跟踪，源码添加日志可有效帮我们分析问题，总之，分析跨系统、跨部门、框架级问题时，心态很重要！
