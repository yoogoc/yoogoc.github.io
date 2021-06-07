---
title: coredns-如何实现k8s服务发现
tags:
- k8s
- dns
- coredns
- go
date: 2021-06-07 18:54:00
---

# 为什么要用coredns，而不是kube-dns

> https://coredns.io/2018/11/27/cluster-dns-coredns-vs-kube-dns/

官网的这篇文章给出了很详细的理由，而且k8s官方也推荐用

# k8s基于dns的服务发现

> k8s dns 规范
>
> https://github.com/kubernetes/dns/blob/master/docs/specification.md

将CoreDNS的service的clusterIP设置到kubelet-config中的clusterDNS，这样，在每个pod启动的时候，kubelet会自动的生成容器的`/etc/resolv.conf`, 例如：

```
nameserver 169.254.20.10
search istio-system.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

解释：

1. nameserver：就是coredns的service的clusterIP
2. search： 用来补全hostname，假设有名为`kiali`的service，则在当前命名空间下，可以通过`kiali`访问到服务，其他命名空间可以通过`kiali.istio-system`访问
3. options：如果查询的域名包含的点“.”，不到5个，那么进行DNS查找，将使用非完全限定名称（或者叫绝对域名），如果你查询的域名包含点数大于等于5，那么DNS查询，默认会使用绝对域名进行查询

# CoreDNS基于插件的服务

一个简单的Corefile：

```
.:54 {
    log
    errors
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
        endpoint http://localhost:8001
    }
}
```

解释：启动一个端口为54的dns服务，解析所有host，并启动插件：log，errors，kubernetes

> k8s插件的dsl参照官网
>
> https://coredns.io/plugins/kubernetes/

# CoreDNS kubernetes插件处理流程

## 1. 入口

`plugin/kubernetes/handler.go`

ServeDNS方法为插件入口方法，经过具体处理后，将应答消息写入`dns.ResponseWriter`，然后返回成功

## 2. 执行步骤

1. ServeDNS中，构造state后执行按类型执行响应的解析方法

```go
switch state.QType() {
	case dns.TypeA:
		records, err = plugin.A(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeAAAA:
		records, err = plugin.AAAA(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeTXT:
		records, err = plugin.TXT(ctx, &k, zone, state, nil, plugin.Options{})
	case dns.TypeCNAME:
		records, err = plugin.CNAME(ctx, &k, zone, state, plugin.Options{})
	case dns.TypePTR:
		records, err = plugin.PTR(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeMX:
		records, extra, err = plugin.MX(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeSRV:
		records, extra, err = plugin.SRV(ctx, &k, zone, state, plugin.Options{})
	case dns.TypeSOA:
		if qname == zone {
			records, err = plugin.SOA(ctx, &k, zone, state, plugin.Options{})
		}
	case dns.TypeAXFR, dns.TypeIXFR:
		return dns.RcodeRefused, nil
	case dns.TypeNS:
		if state.Name() == zone {
			records, extra, err = plugin.NS(ctx, &k, zone, state, plugin.Options{})
			break
		}
		fallthrough
	default:
		// Do a fake A lookup, so we can distinguish between NODATA and NXDOMAIN
		fake := state.NewWithQuestion(state.QName(), dns.TypeA)
		fake.Zone = state.Zone
		_, err = plugin.A(ctx, &k, zone, fake, nil, plugin.Options{})
	}
```

2. 以dns.TypeA为例，看下plugin.A的处理流程

```go
func A(ctx context.Context, b ServiceBackend, zone string, state request.Request, previousRecords []dns.RR, opt Options) (records []dns.RR, err error) {
  //检查特殊的 apex.dns 目录，寻找将以 A 或 AAAA 形式返回的记录。
	services, err := checkForApex(ctx, b, zone, state, opt)
	if err != nil {
		return nil, err
	}

	dup := make(map[string]struct{})

  // 轮询找到的所有记录
	for _, serv := range services {

		what, ip := serv.HostType()

		switch what {
		case dns.TypeCNAME:
			if Name(state.Name()).Matches(dns.Fqdn(serv.Host)) {
				// x CNAME x is a direct loop, don't add those
				continue
			}

			newRecord := serv.NewCNAME(state.QName(), serv.Host)
			if len(previousRecords) > 7 {
				// don't add it, and just continue
				continue
			}
			if dnsutil.DuplicateCNAME(newRecord, previousRecords) {
				continue
			}
			if dns.IsSubDomain(zone, dns.Fqdn(serv.Host)) {
				state1 := state.NewWithQuestion(serv.Host, state.QType())
				state1.Zone = zone
				nextRecords, err := A(ctx, b, zone, state1, append(previousRecords, newRecord), opt)

				if err == nil {
					// Not only have we found something we should add the CNAME and the IP addresses.
					if len(nextRecords) > 0 {
						records = append(records, newRecord)
						records = append(records, nextRecords...)
					}
				}
				continue
			}
			// This means we can not complete the CNAME, try to look else where.
			target := newRecord.Target
			// Lookup
			m1, e1 := b.Lookup(ctx, state, target, state.QType())
			if e1 != nil {
				continue
			}
			// Len(m1.Answer) > 0 here is well?
			records = append(records, newRecord)
			records = append(records, m1.Answer...)
			continue

		case dns.TypeA:
      // a类型的记录根据host去重
			if _, ok := dup[serv.Host]; !ok {
				dup[serv.Host] = struct{}{}
				records = append(records, serv.NewA(state.QName(), ip))
			}

		case dns.TypeAAAA:
			// nada
		}
	}
	return records, nil
}

```

3. 查询符合条件的service

```go
func checkForApex(ctx context.Context, b ServiceBackend, zone string, state request.Request, opt Options) ([]msg.Service, error) {
	if state.Name() != zone {
		return b.Services(ctx, state, false, opt)
	}

	// 如果区名本身被解析，我们伪造查询以解析一个特殊条目，这相当于NS解析。
	old := state.QName()
	state.Clear()
	state.Req.Question[0].Name = dnsutil.Join("apex.dns", zone)

	services, err := b.Services(ctx, state, false, opt)
	if err == nil {
		state.Req.Question[0].Name = old
		return services, err
	}

	state.Req.Question[0].Name = old
	return b.Services(ctx, state, false, opt)
}
```

4. 真正与k8s api-server交互的是Records方法：

```
func (k *Kubernetes) Records(ctx context.Context, state request.Request, exact bool) ([]msg.Service, error) {
	r, e := parseRequest(state.Name(), state.Zone)
	if e != nil {
		return nil, e
	}
	if r.podOrSvc == "" {
		return nil, nil
	}

	if dnsutil.IsReverse(state.Name()) > 0 {
		return nil, errNoItems
	}

	if !wildcard(r.namespace) && !k.namespaceExposed(r.namespace) {
		return nil, errNsNotExposed
	}
	// 在此之前都是做校验，下面才是真正的核心方法：判断当前请求的是pod地址还是service，并请求api-server获取对应kv

	if r.podOrSvc == Pod {
		pods, err := k.findPods(r, state.Zone)
		return pods, err
	}

	services, err := k.findServices(r, state.Zone)
	return services, err
}
```

至此已完成k8s插件解析逻辑，整体流程还是非常清晰的，我表达的可能过分简化，如果想更详细的了解，还是看看代码

## 调试k8s插件

1. 本地开启kubectl proxy,将api-server代理至本地8001端口下
2. 设置corefile中k8s插件的endpoint
3. 调试模式启动coredns
4. 使用dig请求解析，因为我们本地的resolv.conf是没有search的，所以需要解析完整的地址：

```sh
dig -p 54 xxx.xx.svc.cluster.local
```

