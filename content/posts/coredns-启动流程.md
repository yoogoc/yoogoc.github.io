---

title: coredns-启动流程
tags:
- k8s
- dns
- go
date: 2021-03-08 22:42:00
---

# 1. load

入口文件位于 coremain/run.go#Run，在真正执行Run之前有一些init操作, 暂可称为Load阶段

## 1. core/dnsserver/register.go#init

```go
func init() {
	flag.StringVar(&Port, serverType+".port", DefaultPort, "Default port")

	caddy.RegisterServerType(serverType, caddy.ServerType{
		Directives: func() []string { return Directives },
		DefaultInput: func() caddy.Input {
			return caddy.CaddyfileInput{
				Filepath:       "Corefile",
				Contents:       []byte(".:" + Port + " {\nwhoami\nlog\n}\n"),
				ServerTypeName: serverType,
			}
		},
		NewContext: newContext,
	})
}
```

流程分析：

1. 从命令行参数`dns.port`获取Port,默认为 53
2. 将serverType="dns"注册至caddy，并配置插件链和默认配置文件Corefile及Context
3. 插件链默认为core/dnsserver/zdirectives.go#Directives，注意插件链是顺序执行的
4. 默认配置会开启whoami，log插件
5. 注册caddy服务需要提供context，其类型为`func(inst *Instance) Context`：

默认的NewContext为

```go
func newContext(i *caddy.Instance) caddy.Context {
	return &dnsContext{keysToConfigs: make(map[string]*Config)}
}

type dnsContext struct {
	keysToConfigs map[string]*Config
	configs []*Config
}

func (h *dnsContext) saveConfig(key string, cfg *Config) {
	h.configs = append(h.configs, cfg)
	h.keysToConfigs[key] = cfg
}

var _ caddy.Context = &dnsContext{}

func (h *dnsContext) InspectServerBlocks(sourceFile string, serverBlocks []caddyfile.ServerBlock) ([]caddyfile.ServerBlock, error) {
	...
	return serverBlocks, nil
}

func (h *dnsContext) MakeServers() ([]caddy.Server, error) {
	...
	return servers, nil
}
```

编译其检查了`var _ caddy.Context = &dnsContext{}`，InspectServerBlocks，MakeServers目前并没有执行，只是加载到了内存中

## 2. coremain/run.go#init

流程分析：

1. 设置默认加载的配置文件`caddy.DefaultConfigFile = "Corefile"`

2. 设置不输出caddy的init log`caddy.Quiet = true`

3. 解析从命令行加载的参数：

   a. conf: Corefile路径

   b. plugins: 显示安装的插件列表

   c. pidfile: 写入pid的路径

   d. version: 打印版本

   e. quiet: 不打印初始化日志

4. 注册caddyfileLoaders `caddy.RegisterCaddyfileLoader("flag", caddy.LoaderFunc(confLoader))`
5. 注册defaultCaddyfileLoader`caddy.SetDefaultCaddyfileLoader("default", caddy.LoaderFunc(defaultLoader))`

# Coredns Run

至此load结束，正式进入Run方法

## 1.coremain/run.go#Run

```go
func Run() {
	caddy.TrapSignals()
	flag.Parse()

	if len(flag.Args()) > 0 {
		mustLogFatal(fmt.Errorf("extra command line arguments: %s", flag.Args()))
	}

	log.SetOutput(os.Stdout)
	log.SetFlags(0) // Set to 0 because we're doing our own time, with timezone

	if version {
		showVersion()
		os.Exit(0)
	}
	if plugins {
		fmt.Println(caddy.DescribePlugins())
		os.Exit(0)
	}

	// Get Corefile input
	corefile, err := caddy.LoadCaddyfile(serverType)
	if err != nil {
		mustLogFatal(err)
	}

	// Start your engines
	instance, err := caddy.Start(corefile)
	if err != nil {
		mustLogFatal(err)
	}

	if !dnsserver.Quiet {
		showVersion()
	}

	// Twiddle your thumbs
	instance.Wait()
}
```



1. 设置TrapSignals
2. 加载serverType对应的corefile：`corefile, err := caddy.LoadCaddyfile(serverType)`
3. 开启caddy服务：`instance, err := caddy.Start(corefile)`

# Caddy Start

CoreDNS使用了 Caddy 提供的一些功能，因此需要开启caddy服务

```go
func Start(cdyfile Input) (*Instance, error) {
	inst := &Instance{serverType: cdyfile.ServerType(), wg: new(sync.WaitGroup), Storage: make(map[interface{}]interface{})}
	err := startWithListenerFds(cdyfile, inst, nil)
	if err != nil {
		return inst, err
	}
	signalSuccessToParent()
	if pidErr := writePidFile(); pidErr != nil {
		log.Printf("[ERROR] Could not write pidfile: %v", pidErr)
	}

	// Execute instantiation events
	EmitEvent(InstanceStartupEvent, inst)

	return inst, nil
}
```

## 1. 初始化caddy.Instance实例

```go
func startWithListenerFds(cdyfile Input, inst *Instance, restartFds map[string]restartTriple) error {
...
  instances = append(instances, inst)
...
  err = ValidateAndExecuteDirectives(cdyfile, inst, false)
...
  slist, err := inst.context.MakeServers()
...
  err = startServers(slist, inst, restartFds)
..
  return nil
}
```



## 2. 将这个newInstance启动

```go
func ValidateAndExecuteDirectives(cdyfile Input, inst *Instance, justValidate bool) error {
....
sblocks, err := loadServerBlocks(stypeName, cdyfile.Path(), bytes.NewReader(cdyfile.Body()))
...
sblocks, err = inst.context.InspectServerBlocks(cdyfile.Path(), sblocks)
...
return executeDirectives(inst, cdyfile.Path(), stype.Directives(), sblocks, justValidate)
}
```

## 3. 解析CoreFile，并加载server blocks

```
func loadServerBlocks(serverType, filename string, input io.Reader) ([]caddyfile.ServerBlock, error) {
   validDirectives := ValidDirectives(serverType)
   serverBlocks, err := caddyfile.Parse(filename, input, validDirectives)
...
	return serverBlocks, nil
}
```

1. 找到所有可用插件
2. 解析corefile，加载为serverBlocks

```
type ServerBlock struct {
	Keys   []string
	Tokens map[string][]Token
}
```

## 4. 通过load阶段定义的NewContext的InspectServerBlocks方法重写/检查serverBlocks

```
func (h *dnsContext) InspectServerBlocks(sourceFile string, serverBlocks []caddyfile.ServerBlock) ([]caddyfile.ServerBlock, error) {
	for ib, s := range serverBlocks {
		for ik, k := range s.Keys {
			za, err := normalizeZone(k)
.....
			s.Keys[ik] = za.String()
.....
			h.saveConfig(keyConfig, cfg)
		}
	}
	return serverBlocks, nil
}
```



CoreDNS在此步骤中做了两个事情

1. 检查并重写serverBlock.Key
2. 将serverBlock 中的Token转换，并保存至context中

## 5. 执行插件初始化

```
func executeDirectives(inst *Instance, filename string,
	directives []string, sblocks []caddyfile.ServerBlock, justValidate bool) error {

	storages := make(map[int]map[string]interface{})
	for _, dir := range directives {
		for i, sb := range sblocks {
			var once sync.Once
			if _, ok := storages[i]; !ok {
				storages[i] = make(map[string]interface{})
			}
			for j, key := range sb.Keys {
				// Execute directive if it is in the server block
				if tokens, ok := sb.Tokens[dir]; ok {
.........
					setup, err := DirectiveAction(inst.serverType, dir)
.........
					err = setup(controller)
.........
					storages[i][dir] = controller.ServerBlockStorage
				}
			}
		}
.........
	}

	return nil
}
```

此方法的主要目的是加载每个插件的setup方法

## 6. 构造Server List

执行NewContext.MakeServers(), 看起来代码很多，实际上做的事只有一件：根据不同的addr初始化正确的Server实例（会注册插件）

```
func (h *dnsContext) MakeServers() ([]caddy.Server, error) {
...
	groups, err := groupConfigsByListenAddr(h.configs)
	if err != nil {
		return nil, err
	}

	var servers []caddy.Server

	for addr, group := range groups {
		switch tr, _ := parse.Transport(addr); tr {
		case transport.DNS:
			s, err := NewServer(addr, group)
			if err != nil {
				return nil, err
			}
			servers = append(servers, s)

		case transport.TLS:
			s, err := NewServerTLS(addr, group)
			if err != nil {
				return nil, err
			}
			servers = append(servers, s)

		case transport.GRPC:
			s, err := NewServergRPC(addr, group)
			if err != nil {
				return nil, err
			}
			servers = append(servers, s)

		case transport.HTTPS:
			s, err := NewServerHTTPS(addr, group)
			if err != nil {
				return nil, err
			}
			servers = append(servers, s)
		}

	}

	return servers, nil
}
```

## 7.监听服务

至此加载完毕，等待请求

