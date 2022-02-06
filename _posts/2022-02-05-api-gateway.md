---
title: API Gateway
author: tjufc
date: 2022-02-5
category: Jekyll
layout: post
---

- [API Gateway](#api-gateway)
	- [概述](#概述)
	- [典型案例](#典型案例)
	- [百度BFE](#百度bfe)
		- [概述](#概述-1)
		- [源码结构梳理](#源码结构梳理)
		- [模块插件机制](#模块插件机制)
		- [流量转发](#流量转发)
		- [条件表达式](#条件表达式)
		- [监控](#监控)
		- [日志](#日志)
		- [其它](#其它)
	- [美团](#美团)
		- [概述](#概述-2)
	- [Traefik](#traefik)

# API Gateway

## 概述

以下是nginx官网对API Gateway的一段介绍：
[refer](https://www.nginx.com/learn/api-gateway/)

> An API gateway is the conductor that organizes the requests being processed by the microservices architecture to create a simplified experience for the user. 面向微服务。<br>
> It’s a translator, taking a client’s many requests and turning them into just one, to reduce the number of round trips between the client and application. 服务(接口)编排能力。<br>
> An API gateway is set up in front of the microservices and becomes the entry point for every new request being executed by the app. It simplifies both the client implementations and the microservices app. 接入层。<br>

<span style="display:block;text-align:center">![nginx api gateway](/images/api_gateway/nginx_gw.png){: height="100%" width="100%"}</span>
<div style="font-size:14px;text-align:center">Nginx for API Gateway</div>

## 典型案例

+ 百度[BFE](https://github.com/bfenetworks/bfe)
+ 美团
  + [Oceanus](https://tech.meituan.com/2018/09/06/oceanus-custom-traffic-routing.html)
  + [Shepherd](https://mp.weixin.qq.com/s/iITqdIiHi3XGKq6u6FRVdg)
+ [Traefik](https://github.com/traefik/traefik)
+ [Lura](https://github.com/luraproject/lura) 看起来主要是用来做服务编排的。

## 百度BFE

学习内容：原理；（按模块）设计思想、源码

参考材料：

+ [《深入理解BFE》](https://github.com/baidu/bfe-book)

### 概述

+ 定位：百度内部的七层负载均衡接入层
+ 功能：接入和转发、流量调度、WAF、数据分析
+ 接入层技术发展
  + HTTPS：证书维护、性能
  + 安全：DDoS攻击；防御规则检查；计算资源；0 day场景。
  + 数据：接入层的优势？后端延迟、错误；用户所有的流量数据。
  + 控制系统：自动化、智能化。云原生化。
+ 负载均衡：四层负载均衡(BGW) VS 七层负载均衡(BFE)

<span style="display:block;text-align:center">![百度的负载均衡架构](/images/bfe/百度的负载均衡架构.png){: height="50%" width="50%"}</span>
<div style="font-size:14px;text-align:center">百度的负载均衡架构</div>

### 源码结构梳理

<span style="display:block;text-align:center">![bfe源码结构](/images/bfe/uml_bfe.png)</span>
<div style="font-size:14px;text-align:center">bfe源码结构</div>

`BfeServer`顶层对象

+ `BfeServer`对应一个监听协程，每个监听协程针对每个请求开启1个处理协程。单个bfe进程可以配置多个`BfeServer`。
+ `BfeServer`通过`WaitGroup`控制请求处理协程，并实现优雅重启。
+ `Serve()`方法负责执行整个请求处理及响应。其中，`conn`对象负责实现基础网络功能，`ReverseProxy`对象负责实现路由、负载均衡等核心功能。完整流程详见[请求处理流程及响应](https://github.com/baidu/bfe-book/blob/version1/implementation/life_of_a_request/life_of_a_request.md)一节，代码参见`ReverseProxy.ServeHTTP()`。
+ `BfeServer`依赖回调框架能力(`bfe_module`包)，注册并顺序执行回调链，实现请求的定制化处理。
+ `bfe_basic.Request`和`bfe_basic.Response`：相当于请求上下文，特别是前者，在其它的web框架中我们也可以看到命名类似于*Context*的对象。它们贯穿了整个`conn`的生命周期，我们在这个协程的任何地方访问它都是最低成本的。

### 模块插件机制

**准备**

+ 参考章节：
  + [模块插件机制](https://github.com/baidu/bfe-book/blob/version1/design/module/module.md)
  + [模块框架](https://github.com/baidu/bfe-book/blob/version1/implementation/model_framework/model_framework.md)
  + [如何开发BFE扩展模块](https://github.com/baidu/bfe-book/blob/version1/develop/how_to_write_module/how_to_write_module.md)
+ 代码模块
  + `bfe_module`
  + `bfe_modules`、`bfe_modules.mod_waf`

**回调模型**

BFE通过*回调*来执行插件功能。

+ 回调框架(BfeCallbacks)
+ 回调点(CallbackPoint)：单个请求维度定义了9个回调点。
+ 回调链(HandlerList)：实现上是个链表。
+ 回调(接口)类型：`RequestFilter`等5个回调接口。
+ 回调返回类型(没定义，暂且称之为HandlerReturnType)

以下是`ReverseProxy.ServeHTTP`请求处理流程中，某个回调点的代码：

```go
	// ...

	// Callback for HandleBeforeLocation
	// srv.CallBacks是回调框架BfeCallbacks对象
	// 获取回调点HandleBeforeLocation对应的回调链
	hl = srv.CallBacks.GetHandlerList(bfe_module.HandleBeforeLocation)
	if hl != nil {
		// 依次执行回调(函数)
		// 逻辑上，在特定回调点有特定的回调(接口)类型
		retVal, res = hl.FilterRequest(basicReq)
		basicReq.HttpResponse = res
		// 根据回调返回类型进行处理
		switch retVal {
		case bfe_module.BfeHandlerClose:
			// close the connection directly (with no response)
			action = closeDirectly
			return
		case bfe_module.BfeHandlerFinish:
			// close the connection after response
			action = closeAfterReply
			basicReq.BfeStatusCode = bfe_http.StatusInternalServerError
			return
		case bfe_module.BfeHandlerRedirect:
			// ...
```

回调函数的添加实现(`AcceptFilter`示例)：

```go
// AddAcceptFilter adds accept filter to handler list.
func (hl *HandlerList) AddAcceptFilter(f interface{}) error {
	callback, ok := f.(func(session *bfe_basic.Session) int)
	if !ok {
		return fmt.Errorf("AddAcceptFilter():invalid callback func")
	}

	hl.handlers.PushBack(NewAcceptFilter(callback))
	return nil
}
```

上述的回调模型实现了模块可插拔，那么具体的功能模块该如何实现呢？我们继续来看。

**模块**

我们以非常重要的waf模块为例，深入研究一下模块内部的实现原理([WAF简介](https://zhuanlan.zhihu.com/p/97396469))。这部分内容可以按照以下3个部分进行梳理：

+ 模块配置
+ 回调的注册与实现
+ 模块状态

waf模块配置示例：

```json
{
    "Version": "2019-12-10184356",
    "Config": {
        "example_product": [
            {
                "Cond": "req_path_prefix_in(\"/rewrite\", false)",
                "BlockRules": [
	            "RuleBashCmd"
                ]
            }
        ]
    }
}
```

大部分的模块功能都可以拆分为2部分，即：条件(`Cond`)、动作(`BlockRules`，在其它模块也叫`Actions`等)。那么，一个模块的逻辑可以描述为：如果命中条件，就执行动作。

waf模块实现了`bfe_module.RequestFilter`回调：

```go
func (m *ModuleWaf) handleWaf(req *bfe_basic.Request) (int, *bfe_http.Response) {
	// 获取RuleList
	rules, ok := m.ruleTable.Search(req.Route.Product)
	if !ok {
		return bfe_module.BfeHandlerGoOn, nil
	}
	for _, rule := range *rules {
		// 首先，判断条件是否命中
		if !rule.Cond.Match(req) {
			continue
		}
		m.state.CheckedReq.Inc(1)
		// BlockRules - 拦截模式
		for _, blockRule := range rule.BlockRules {
			blocked, err := m.handler.HandleBlockJob(blockRule, req)
			if err != nil {
				m.state.BlockedRuleError.Inc(1)
				log.Logger.Debug("ModuleWaf.handleWaf() block job err=%v, rule=%s", err, blockRule)
				continue
			}
			if blocked {
				req.ErrCode = ErrWaf
				m.state.HitBlockedReq.Inc(1)
				return bfe_module.BfeHandlerFinish, nil
			}
		}
		// CheckRules - 观察模式
		for _, checkRule := range rule.CheckRules {
			// ...
		}
		break
	}
	return bfe_module.BfeHandlerGoOn, nil
}
```

几个点：

+ 用到了我们之前介绍过的[条件表达式](#条件表达式)
+ waf规则的处理分为*拦截模式*(`BlockRules`)和*观察模式*(CheckRules)。这是因为waf规则上线通常需要先开启观察模式验证，后开启拦截模式拦截攻击流量。
+ waf模块状态`ModuleWafState`主要记录了各种规则命中的数据，用于统计。这个信息也非常重要，往往需要采集到后台进行统一监控。详见[监控](#监控)一节。

模块的注册是`bfe_module.BfeModule`接口中`Init`规定的，waf模块出了注册回调，还添加了监控。

```go
func (m *ModuleWaf) Init(cbs *bfe_module.BfeCallbacks, whs *web_monitor.WebHandlers, cr string) error {
	// ...

	// 回调注册
	err = cbs.AddFilter(bfe_module.HandleFoundProduct, m.handleWaf)
	if err != nil {
		return fmt.Errorf("%s.Init(): AddFilter(m.handleWaf): %v", m.name, err)
	}

	err = web_monitor.RegisterHandlers(whs, web_monitor.WebHandleMonitor, m.monitorHandlers())
	if err != nil {
		return fmt.Errorf("%s.Init(): RegisterHandlers(m.monitorHandlers): %v", m.Name(), err)
	}

	// ...
}
```

**实现细节**

+ `bfe_module.HandlerList`内表示回调类型的类型成员事实上并没有什么用。回调接口类型是通过`switch type {}`动态判断的。
+ `mod_waf.wafRule`和`mod_waf.WafRule`的区别：命名接近。前者负责检查条件是否命中，后者负责实现动作执行。

### 流量转发

**转发模型**

参考章节：[BFE的转发模型](https://github.com/baidu/bfe-book/blob/version1/design/model/model.md)

+ 租户(Tenant/Product)：大致上，一个域名对应一个租户。百度内部可能叫“产品线”。
+ 集群(Cluster)：一个租户可以对应多个集群，一个租户维护一个路由转发表。一个集群一般按照不同的IDC再划分多个子集群。
+ 实例：`ip:port`

**实现机制**

设计为2个核心能力

+ [请求路由](https://github.com/baidu/bfe-book/blob/version1/implementation/routing/routing.md)：负责通过域名、Vip、path等特征，根据路由规则，获取租户->集群。代码上对应`bfe_route`包，规则的解析则依赖`bfe_route_conf`包。
+ [负载均衡](https://github.com/baidu/bfe-book/blob/version1/implementation/balancing/balancing.md)：已知下游集群的情况下，根据负载均衡策略，获取子集群->实例。代码上对应`bfe_balancer`包。

**实现细节**

+ (根据域名)获取租户 `HostTable.findHostRoute`

```go
func (t *HostTable) findHostRoute(host string) (route, error) {
	// ...
	match, ok := t.hostTrie.Get(strings.Split(string_reverse.ReverseFqdnHost(hostnameStrip(host)), "."))
	if ok {
		// get route success, return
		return match.(route), nil
	}
  // ...
}
```

使用了一棵域名[前缀树](https://zhuanlan.zhihu.com/p/28891541)来查找，从根节点到叶子节点为一个完整域名。从根节点往下依次是各级[域名](https://baike.baidu.com/item/%E9%A1%B6%E7%BA%A7%E5%9F%9F%E5%90%8D/2152551)，到了叶子节点则是租户。

前缀树的结构：和普通的树区别在于节点的children使用了一个映射，key为前缀。

```go
type trieChildren map[string]*Trie

type Trie struct {
	Entry      interface{}
	SplatEntry interface{} // to match xxx.xxx.*
	Children   trieChildren
}
```

根据完整域名(`path`)查找租户的算法：

```go
func (t *Trie) Get(path []string) (entry interface{}, ok bool) {
	// 递归终止条件
	if len(path) == 0 {
		return t.getEntry()
	}

	// key是当前前缀
	key := path[0]
	// newPath做为递归查找的输入路径
	newPath := path[1:]

	res, ok := t.Children[key]
	if ok {
		// 递归查找
		entry, ok = res.Get(newPath)
	}

	// ...
}
```

+ 获取集群 `HostTable.LookupCluster`

路由规则的检查和动作执行代码如下：

```go
// LookupCluster find clusterName with given request.
func (t *HostTable) LookupCluster(req *bfe_basic.Request) error {
	// match advanced route rules 获取路由规则表
	rules, ok := t.productAdvancedRouteTable[req.Route.Product]
	if !ok {
		req.Route.ClusterName = ""
		req.Route.Error = ErrNoProductRule
		return req.Route.Error
	}

	// matching route rules 规则检查
	for _, rule := range rules {
		// 条件表达式描述的规则
		if rule.Cond.Match(req) {
			clusterName = rule.ClusterName
			break
		}
	}

	// ...
}
```

可见，`productAdvancedRouteTable`中预先(显然是在server初始化阶段)注册了一些列分流条件，在这里根据请求(参数)进行逐个进行判断，直至找到符合条件的集群。
关于分流条件的实现细节，详见[条件表达式](#条件表达式)一节。

### 条件表达式

**准备**

+ 参考章节：[BFE的路由转发机制—条件表达式](https://github.com/baidu/bfe-book/blob/version1/design/route/route.md)
+ 代码模块：`bfe_basic/condition`
+ 背景知识：[AST(抽象语法树)](https://juejin.cn/post/6844904035271573511)

**实现机制**

如书中所述：

> 条件表达式在BFE的内部数据结构，是一个中缀表达式形式的二叉树。二叉树的非叶子节点代表了操作符。叶子节点代表条件原语。

<span style="display:block;text-align:center">![BFE条件表达式](/images/bfe/BFE条件表达式.png){: height="80%" width="80%"}</span>
<div style="font-size:14px;text-align:center">BFE条件表达式实现机制</div>

条件表达式对于用户相当于一套语言，底层引擎需要支持对这套语言的*编译*(*解释*)。这类问题往往通过AST来解决。

条件表达式的编译流程实现：

```go
func Build(condStr string) (Condition, error) {
	// 词法解析&语法解析，生成语法树
	node, identList, err := parser.Parse(condStr)
	if err != nil {
		return nil, err
	}

	// ...

	// 构建执行树
	return build(node)
}
```

主要包括2个步骤：

+ 词法解析&语法解析，生成语法树。以下简称*分析*
+ 构建执行树。以下简称*生成*

其中，构建执行树(`build(node)`)的实现：

```go
func build(node parser.Node) (Condition, error) {
	switch n := node.(type) {
	case *parser.CallExpr:
		return buildPrimitive(n)
	case *parser.UnaryExpr:
		return buildUnary(n)
	case *parser.BinaryExpr:
		return buildBinary(n)
	case *parser.ParenExpr:
		return build(n.X)
	default:
		return nil, fmt.Errorf("unsupported node %s", node)
	}
}
```

可见，这棵AST的节点被分为了3类：

+ **叶子节点——原语**：这类节点包含可以直接执行的*条件原语*。*分析*环节中将节点实例化为`parser.CallExpr`类型；*生成*环节实例化为`condition.PrimitiveCond`类型。
+ **叶子结点—一元运算**：这类节点需要在*条件原语*的基础上，进行一次一元逻辑运算，例如：`非`。*分析*环节对应`parser.UnaryExpr`，*生成*环节对应`condition.UnaryCond`。
+ **非叶子结点**：这类节点需要对左右2个子节点，进行一次二元逻辑运算，例如：`与/或/非`。*分析*环节中实例化为`parser.BinaryExpr`，*生成*环节中实例化为`parser.BinaryCond`。

**实现细节**

+ `parser`包*分析*部分的代码看起来像是基于`go`包改造的。
+ *条件原语*的实现：`condition.PrimitiveCond`又被拆分成`Matcher`和`Fetcher`的组合，它们分别负责参数的获取和匹配。这样可以方便将已有的对象迅速组合出新的原语。

例如：

```go
// buildPrimitive builds primitive from PrimitiveCondExpr.
// if failed, b.err is set to err, return Condition is nil
// if success, b.err is nil
func buildPrimitive(node *parser.CallExpr) (Condition, error) {
	switch node.Fun.Name {
	case "default_t":
		return &DefaultTrueCond{}, nil
	// ...
	case "req_vip_in":
		matcher, err := NewIpInMatcher(node.Args[0].Value)
		if err != nil {
			return nil, err
		}
		return &PrimitiveCond{
			name:    node.Fun.Name,
			node:    node,
			fetcher: &VIPFetcher{},
			matcher: matcher,
		}, nil
	case "req_vip_range":
		// ...
  }
```

有兴趣可以看看具体Matcher和Fetcher的实现。

### 监控

**准备**

+ 参考章节：
  + [监控机制](https://github.com/baidu/bfe-book/blob/version1/design/monitor/monitor.md)
  + [如何开发BFE扩展模块](https://github.com/baidu/bfe-book/blob/version1/design/monitor/monitor.md)
+ go [atomic包](https://juejin.cn/post/6844904053042839560)
+ go反射

本节我们仍旧以`mod_waf`的实现为例，进行学习。

**实现机制**

[监控机制](https://github.com/baidu/bfe-book/blob/version1/design/monitor/monitor.md)一节一共讨论了2种监控方法：

+ 基于日志监控：
  + 打印日志，监控日志内容。
  + 优点：信息完善。缺点：耗资源 -> IO操作，读取、解析、匹配...
+ 维护内部状态：每秒处理请求数、并发连接数、命中策略数...

bfe的服务状态信息(如上)，是基于内存进行统计(metrics)，并依赖一个web server(后台协程)对外暴露。这就引出了本节讨论的2个主要模块`metrics`和`web_monitor`。

首先，`metrics`实现了基于内存的服务状态存储和计算

+ 服务状态分为`Counter`/`Gauge`/`State`三类，它们使用go语言的`atomic`机制保证操作的原子性。
+ `MetricsData`基于通过互斥锁(`Mutex`)实现了`Diff`接口，用于计算每个`interval`之间的数据变化。例如：最近20s内的请求pv。
+ `Metrics`对象负责实现服务状态的UI。用户可以基于上述3类服务状态定义任意形式的结构体(即`metricStruct`)，`Metrics`内部通过go反射来解析这个结构体。

这段代码展示了`Metrics`是如何利用go反射，将用户定义的任意由`Counter/Gauge/State`三类状态组合成的结构体`metricStruct`，解析成内部数据结构的。

```go
// initMetrics initializes metrics struct
func (m *Metrics) initMetrics(s interface{}) {
	m.counterMap = make(map[string]*Counter)
	m.gaugeMap = make(map[string]*Gauge)
	m.stateMap = make(map[string]*State)

	// 利用反射，获取s中变量的type和value
	t := reflect.TypeOf(s).Elem()
	v := reflect.ValueOf(s).Elem()

	// 对每个变量进行遍历。实际上，每个变量即时用户想要维护的一个内部状态。
	for i := 0; i < v.NumField(); i++ {
		field := t.Field(i)
		value := v.Field(i)

		// track created counters
		name := m.convert(field.Name)

		// 对s中变量类型进行枚举，它必然是Counter/Gauge/State
		switch mType := field.Type.Elem().Name(); mType {
		case TypeState:
			v := new(State)
			m.stateMap[name] = v
			value.Set(reflect.ValueOf(v))

		case TypeCounter:
			v := new(Counter)
			m.counterMap[name] = v
			value.Set(reflect.ValueOf(v))

		case TypeGauge:
			v := new(Gauge)
			m.gaugeMap[name] = v
			value.Set(reflect.ValueOf(v))
		}
	}
}
```

其次，`web_monitor`实现了对外暴露数据的后台web server,比较简单：

+ 基于go标准http包，其中`webHandler`是`http.Handler`接口的具体实现。
+ 接口路由基于2层map实现。第一层通过`WebHandlerType`进行枚举，第二层直接用key查询。
+ `RegisterHandler`接口实现回调注册。

最后，我们来看一下`mod_waf`是如何依靠上述2个模块实现监控的。在[模块插件机制](#模块插件机制)一节中我们知道`ModuleWafState`用于维护waf模块的内部状态，它分别关联了1个`Metrics`对象和全局`WebMonitor`对象。

定义模块自身的内部状态结构体：

```go
type ModuleWafState struct {
	CheckedReq *metrics.Counter // record how many requests check waf rule

	HitBlockedReq  *metrics.Counter // record how many requests check waf rule
	HitCheckedRule *metrics.Counter // hit checked rule

	BlockedRuleError *metrics.Counter //err times of check blocked rule
	CheckedRuleError *metrics.Counter // err times of check checked rule
}
```

实现暴露状态数据的回调接口，并在`Init`接口中向`WebMonitor`注册：

```go
func (m *ModuleWaf) Init(cbs *bfe_module.BfeCallbacks, whs *web_monitor.WebHandlers, cr string) error {
	// ...

	// 监控相关
	err = web_monitor.RegisterHandlers(whs, web_monitor.WebHandleMonitor, m.monitorHandlers())
	if err != nil {
		return fmt.Errorf("%s.Init(): RegisterHandlers(m.monitorHandlers): %v", m.Name(), err)
	}

	// ...
}
```

### 日志

**准备**

+ 参考章节：[日志机制](https://github.com/baidu/bfe-book/blob/version1/design/log/log.md)

**实现机制**

首先，BFE对于不同用途的日志做出明确区分，这点是值得重视的。日志本身实现难度并不大，但是很多业务中的日志却是五花八门，个人觉得大多时候都是设计上的懒惰。

我们主要看 *访问日志(access log)*。它被实现为一个模块`mod_access`，实现方法我们已经在`mod_waf`介绍过了，它们大体相同。这里直接看回调代码：

```go
func (m *ModuleAccess) requestLogHandler(req *bfe_basic.Request, res *bfe_http.Response) int {
	byteStr := bytes.NewBuffer(nil)

	// 这一段是在拼装access日志
	for _, item := range m.reqFmts {
		switch item.Type {
		case FormatString:
			byteStr.WriteString(item.Key)
		case FormatTime:
			onLogFmtTime(m, byteStr)
		default:
			// fmtHandlerTable这个表里存储了item.Type和对应信息的获取方法
			handler, found := fmtHandlerTable[item.Type]
			if found {
				h := handler.(func(*ModuleAccess, *LogFmtItem, *bytes.Buffer,
					*bfe_basic.Request, *bfe_http.Response) error)
				h(m, &item, byteStr, req, res)
			}
		}
	}

	// 这里是实际打印日志，详见log4go
	m.logger.Info(byteStr.String())

	return bfe_module.BfeHandlerGoOn
}
```

其次，我们简单学习一下*log4go*这个包。代码结构如下：

<span style="display:block;text-align:center">![log4go代码结构](/images/bfe/uml_log4go.png)</span>
<div style="font-size:14px;text-align:center">log4go代码结构</div>

log4go的主要设计是基于`chan`来异步写日志，即：用户只负责向`chan`中分发日志(消息)，log4go通过后台协程去消费这个`chan`中的数据，串行地向日志文件中进行写操作。

实现上，核心接口是`LogWriter`和`LogCloser`，前者负责用户*写日志*操作，后者负责关闭日志。

`TimeFileLogWriter`是一个具体的实现。首先看下初始化消费协程的逻辑：

```go
func NewTimeFileLogWriter(fname string, when string, backupCount int, enableCompress bool) *TimeFileLogWriter {
	// ...

	// 开启后台协程，真正向文件中写入日志。这样所有写入操作都是串行的。
	go func() {
		defer func() {
			if w.file != nil {
				w.file.Close()
			}
		}()

		for rec := range w.rec {
			// 检查一下消息是否为终止信号
			if w.EndNotify(rec) {
				return
			}

			// 日志切分处理
			if w.shouldRollover() {
				if err := w.intRotate(); err != nil {
					fmt.Fprintf(os.Stderr, "NewTimeFileLogWriter(%q): %s\n", w.filename, err)
					continue
				}
			}

			// Perform the write
			var err error
			if rec.Binary != nil {
				_, err = w.file.Write(rec.Binary)
			} else {
				_, err = fmt.Fprint(w.file, FormatLogRecord(w.format, rec))
			}
			if err != nil {
				fmt.Fprintf(os.Stderr, "NewTimeFileLogWriter(%q): %s\n", w.filename, err)
			}
		}
	}()

	return w
}
```

终止信号是什么呢？我们先看下`LogWrite`接口中`Close`，以及`LogClose`接口的具体实现：

```go
// Close waits for dump all log and close chan
func (w *TimeFileLogWriter) Close() {
	w.WaitForEnd(w.rec)
	close(w.rec)
}

// notyfy the logger log to end
func (lc *LogCloser) EndNotify(lr *LogRecord) bool {
	if lr == nil && lc.IsEnd != nil {
		lc.IsEnd <- true
		return true
	}
	return false
}

// add nil to end of res and wait that EndNotify is call
func (lc *LogCloser) WaitForEnd(rec chan *LogRecord) {
	rec <- nil
	if lc.IsEnd != nil {
		<-lc.IsEnd
	}
}
```

不难发现，实际上是传入一个`nil`对象(`LogRecord`类型)。后台协程通过`EndNotify`获取到`nil`终止信号，向`lc.IsEnd`通道内发出终止标记，并直接返回结束；主协程通过`WairForEnd`阻塞在`<-lc.IsEnd`这里，一旦收到后台协程发出的终止标记后，即表示剩余日志完成处理，关闭日志通道`w.rec`。

### 其它

+ 优雅重启
  + 向监听协程发出close信号，`bfe_server.Serve`捕获信号后等待超时并终止`Accept`。
  + 开启同步协程，等待所有连接的处理协程结束(使用`WaitGroup`)，然后发出结束通知；同时开启超时协程(使用`time.After`)。
  + 循环监听同步协程和超时协程，直到有一个发生后，结束服务。

```go
// ShutdownHandler is signal handler for QUIT
func (srv *BfeServer) ShutdownHandler(sig os.Signal) {
	shutdownTimeout := srv.Config.Server.GracefulShutdownTimeout
	log.Logger.Info("get signal %s, graceful shutdown in %ds", sig, shutdownTimeout)

	// notify that server is in graceful shutdown state
	close(srv.CloseNotifyCh)

	// close server listeners
	srv.closeListeners()

	// waits server conns to finish
	connFinCh := make(chan bool)
	go func() {
		srv.connWaitGroup.Wait()
		connFinCh <- true
	}()

	shutdownTimer := time.After(time.Duration(shutdownTimeout) * time.Second)

Loop:
	for {
		select {
		// waits server conns to finish
		case <-connFinCh:
			log.Logger.Info("graceful shutdown success.")
			break Loop

		// wait for shutdown timeout
		case <-shutdownTimer:
			log.Logger.Info("graceful shutdown timeout.")
			break Loop
		}
	}

	// shutdown server
	log.Logger.Close()
	os.Exit(0)
}
```

## 美团

没看到有开源的，因此仅对官方技术文档做一个整理。

### 概述

<span style="display:block;text-align:center">![美团API网关整体架构](/images/api_gateway/美团API网关整体架构.jpeg)</span>
<div style="font-size:14px;text-align:center">美团API网关整体架构</div>


如图，整体架构主要包括2个部分：

+ [Oceanus](https://tech.meituan.com/2018/09/06/oceanus-custom-traffic-routing.html)：相当于美团统一前端，负责将流量转发到业务集群。
+ [Shepherd](https://mp.weixin.qq.com/s/iITqdIiHi3XGKq6u6FRVdg)：对业务线API Gateway需求进行了抽象，以服务(API管理)和SDK的形式输出能力。

以下进行展开介绍。

**Oceanus**

这个材料里面我们只能看到定制化路由这部分，关于安全、登陆等常见功能并未做介绍。

+ 基于*OpenResty*开发实现。
+ 策略查询：`Host+location_path` & `appkey`
+ 策略规则设计：`condition`支持简单的规则引擎。
+ 策略更新：使用worker进程间的共享内存，定时更新。

<span style="display:block;text-align:center">![Oceanus策略查询设计](/images/api_gateway/Oceanus策略查询设计.png){: height="80%" width="80%"}</span>
<div style="font-size:14px;text-align:center">Oceanus策略查询设计</div>


<span style="display:block;text-align:center">![Oceanus策略查询设计](/images/api_gateway/Oceanus策略规则设计.png)</span>
<div style="font-size:14px;text-align:center">Oceanus策略规则设计</div>

**Shepherd**

+ 服务编排：依托公司内部自研的编排中间件 [海盗](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651748475&idx=3&sn=23b517c9c5173a6585ddc7bfd23a878a&chksm=bd12a1368a65282071cec8ce73f16f86de546b2e22eae0af46cd8b6b88521b2ed14e32fb32e9&scene=21#wechat_redirect)。

## Traefik