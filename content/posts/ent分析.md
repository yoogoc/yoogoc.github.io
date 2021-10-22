---
title: ent分析-ent codegen
tags:
- go
date: 2021-10-07 12:30:00
---

> 代码基于`commit:d199a7c26797e29ffbd5651673301bc936f99029`

# init-初始化schema

## 目的

生成数据表/go entity 的schema文件，用于定义表结构/实体模型

## 源码分析

1. cmd入口在`cmd/internal/base/base.go:66`

```go
func InitCmd() *cobra.Command {
	var target string
	cmd := &cobra.Command{
		Use:   "init [flags] [schemas]",
		Short: "initialize an environment with zero or more schemas",
		Example: examples(
			"ent init Example",
			"ent init --target entv1/schema User Group",
		),
		// 命令行参数校验是否开始于大写
		Args: func(_ *cobra.Command, names []string) error {
			for _, name := range names {
				if !unicode.IsUpper(rune(name[0])) {
					return errors.New("schema names must begin with uppercase")
				}
			}
			return nil
		},
		// 命令入口, 支持生成多schema
		Run: func(cmd *cobra.Command, names []string) {
			if err := initEnv(target, names); err != nil {
				log.Fatalln(fmt.Errorf("ent/init: %w", err))
			}
		},
	}
	// 生成路径默认当前路径下ent/schema目录
	cmd.Flags().StringVar(&target, "target", defaultSchema, "target directory for schemas")
	return cmd
}
```

2. 执行入口`cmd/internal/base/base.go:182`

```go
func initEnv(target string, names []string) error {
	// 创建目录
	if err := createDir(target); err != nil {
		return fmt.Errorf("create dir %s: %w", target, err)
	}
	for _, name := range names {
		// 校验name
		if err := gen.ValidSchemaName(name); err != nil {
			return fmt.Errorf("init schema %s: %w", name, err)
		}
		b := bytes.NewBuffer(nil)
		// 模板渲染
		if err := tmpl.Execute(b, name); err != nil {
			return fmt.Errorf("executing template %s: %w", name, err)
		}
		// 写入目标目录
		newFileTarget := filepath.Join(target, strings.ToLower(name+".go"))
		if err := os.WriteFile(newFileTarget, b.Bytes(), 0644); err != nil {
			return fmt.Errorf("writing file %s: %w", newFileTarget, err)
		}
	}
	return nil
}
```

校验name：

```go
func ValidSchemaName(name string) error {
	// Schema package is lower-cased (see Type.Package).
	pkg := strings.ToLower(name)
	// 是否为go关键字
	if token.Lookup(pkg).IsKeyword() {
		return fmt.Errorf("schema lowercase name conflicts with Go keyword %q", pkg)
	}
	// 是否为go原生类型
	if types.Universe.Lookup(pkg) != nil {
		return fmt.Errorf("schema lowercase name conflicts with Go predeclared identifier %q", pkg)
	}
	// 是否为ent关键字
	if _, ok := globalIdent[pkg]; ok {
		return fmt.Errorf("schema lowercase name conflicts ent predeclared identifier %q", pkg)
	}
	if _, ok := globalIdent[name]; ok {
		return fmt.Errorf("schema name conflicts with ent predeclared identifier %q", name)
	}
	return nil
}
```

# generate-根据schema生成go code

## 目的

根据schema生成predicate，crud及ent基础代码（如context、config、runtime、mutation、client等）

## 源码分析

1. cmd入口：`cmd/internal/base/base.go:117`

```go
func GenerateCmd(postRun ...func(*gen.Config)) *cobra.Command {
	var (
		// Header: codegen头部信息
		// Target: 目标目录
		cfg       gen.Config
		// 存储驱动:默认sql,可选gremlin
		storage   string
		// additional features
		features  []string
		// 外部模板,支持的格式:dir=xxx,file=xxx,glob=xxx
		templates []string
		// 代码生成的id类型,默认int
		idtype    = IDType(field.TypeInt)
		cmd       = &cobra.Command{
			Use:   "generate [flags] path",
			Short: "generate go code for the schema directory",
			Example: examples(
				"ent generate ./ent/schema",
				"ent generate github.com/a8m/x",
			),
			Args: cobra.ExactArgs(1),
			Run: func(cmd *cobra.Command, path []string) {
				// 加载驱动和features
				opts := []entc.Option{
					entc.Storage(storage),
					entc.FeatureNames(features...),
				}
				// 将模板映射为Option结构体
				for _, tmpl := range templates {
					typ := "dir"
					if parts := strings.SplitN(tmpl, "=", 2); len(parts) > 1 {
						typ, tmpl = parts[0], parts[1]
					}
					switch typ {
					case "dir":
						opts = append(opts, entc.TemplateDir(tmpl))
					case "file":
						opts = append(opts, entc.TemplateFiles(tmpl))
					case "glob":
						opts = append(opts, entc.TemplateGlob(tmpl))
					default:
						log.Fatalln("unsupported template type", typ)
					}
				}
				// If the target directory is not inferred from
				// the schema path, resolve its package path.
				// 如果目标目录不在schema目录,需要找到一个合适的包路径
				if cfg.Target != "" {
					pkgPath, err := PkgPath(DefaultConfig, cfg.Target)
					if err != nil {
						log.Fatalln(err)
					}
					cfg.Package = pkgPath
				}
				cfg.IDType = &field.TypeInfo{Type: field.Type(idtype)}
				// 执行gen
				if err := entc.Generate(path[0], &cfg, opts...); err != nil {
					log.Fatalln(err)
				}
        // 生成后的hook
				for _, fn := range postRun {
					fn(&cfg)
				}
			},
		}
	)
	cmd.Flags().Var(&idtype, "idtype", "type of the id field")
	cmd.Flags().StringVar(&storage, "storage", "sql", "storage driver to support in codegen")
	cmd.Flags().StringVar(&cfg.Header, "header", "", "override codegen header")
	cmd.Flags().StringVar(&cfg.Target, "target", "", "target directory for codegen")
	cmd.Flags().StringSliceVarP(&features, "feature", "", nil, "extend codegen with additional features")
	cmd.Flags().StringSliceVarP(&templates, "template", "", nil, "external templates to execute")
	return cmd
}
```

2. 执行入口`entc/entc.go:53`：

```go
func Generate(schemaPath string, cfg *gen.Config, options ...Option) (err error) {
	// 默认输出目录为schema同级
	if cfg.Target == "" {
		abs, err := filepath.Abs(schemaPath)
		if err != nil {
			return err
		}
		// default target-path for codegen is one dir above
		// the schema.
		cfg.Target = filepath.Dir(abs)
	}
	for _, opt := range options {
		if err := opt(cfg); err != nil {
			return err
		}
	}
	// 默认存储为sql
	if cfg.Storage == nil {
		driver, err := gen.NewStorage("sql")
		if err != nil {
			return err
		}
		cfg.Storage = driver
	}
	// 确保包路径不会循环依赖
	undo, err := gen.PrepareEnv(cfg)
	if err != nil {
		return err
	}
	defer func() {
		if err != nil {
			_ = undo()
		}
	}()
	// 准备就绪,进入生成器方法
	return generate(schemaPath, cfg)
}
```

3. 生成入口`entc/entc.go:284`：

```go
func generate(schemaPath string, cfg *gen.Config) error {
	// 构造Graph
	graph, err := LoadGraph(schemaPath, cfg)
	if err != nil {
		// 如果开启了快照feature,则尝试恢复
		if err := mayRecover(err, schemaPath, cfg); err != nil {
			return err
		}
		// 尝试重新构造Graph
		if graph, err = LoadGraph(schemaPath, cfg); err != nil {
			return err
		}
	}
	// 格式化包名('-' => '_')
	if err := normalizePkg(cfg); err != nil {
		return err
	}
	return graph.Gen()
}
```

4. 构造Graph`entc/entc.go:27`:

```go
func LoadGraph(schemaPath string, cfg *gen.Config) (*gen.Graph, error) {
	// 加载映射后的schema结构体
	spec, err := (&load.Config{Path: schemaPath}).Load()
	if err != nil {
		return nil, err
	}
	cfg.Schema = spec.PkgPath
	if cfg.Package == "" {
		// default package-path for codegen is one package
		// before the schema package (`<project>/ent/schema`).
		// 默认包名
		cfg.Package = path.Dir(spec.PkgPath)
	}
	// 从schema中构造Graph
	return gen.NewGraph(cfg, spec.Schemas...)
}
```

5. 从schema路径加载Schema结构体`entc/load/load.go:49`：

```go
func (c *Config) Load() (*SchemaSpec, error) {
	// Config初始化只有Path,这里的load()方法会加载出所有schema的name,
	// 并返回schema包路径,用于buildTmpl渲染import
  // load()方法使用了很多的ast知识，暂时忽略细节
	pkgPath, err := c.load()
	if err != nil {
		return nil, fmt.Errorf("entc/load: load schema dir: %w", err)
	}
	if len(c.Names) == 0 {
		return nil, fmt.Errorf("entc/load: no schema found in: %s", c.Path)
	}
	var b bytes.Buffer
	// 构造可执行文件
	err = buildTmpl.ExecuteTemplate(&b, "main", struct {
		*Config
		Package string
	}{c, pkgPath})
	if err != nil {
		return nil, fmt.Errorf("entc/load: execute template: %w", err)
	}
	buf, err := format.Source(b.Bytes())
	if err != nil {
		return nil, fmt.Errorf("entc/load: format template: %w", err)
	}
	if err := os.MkdirAll(".entc", os.ModePerm); err != nil {
		return nil, err
	}
	target := fmt.Sprintf(".entc/%s.go", filename(pkgPath))
	if err := os.WriteFile(target, buf, 0644); err != nil {
		return nil, fmt.Errorf("entc/load: write file %s: %w", target, err)
	}
	defer os.RemoveAll(".entc")
	// 执行映射好的main文件,用于将schema下的schema definition文件序列化为[]byte,输出到std,
	// 然后entc主进程收集main文件(target)的stdout，这里依赖go环境
	out, err := run(target)
	if err != nil {
		return nil, err
	}
	spec := &SchemaSpec{PkgPath: pkgPath}
	// 将schema反序列化至spec结构体
	for _, line := range strings.Split(out, "\n") {
		schema, err := UnmarshalSchema([]byte(line))
		if err != nil {
			return nil, fmt.Errorf("entc/load: unmarshal schema %s: %w", line, err)
		}
		spec.Schemas = append(spec.Schemas, schema)
	}
	return spec, nil
}
```

6. 从Schema中构造Graph`entc/gen/graph.go:126`:

```go
func NewGraph(c *Config, schemas ...*load.Schema) (g *Graph, err error) {
	defer catch(&err)
	g = &Graph{c, make([]*Type, 0, len(schemas)), schemas}
	// 构造节点
	for i := range schemas {
		g.addNode(schemas[i])
	}
	// 构造边,并添加到相应node
	for i := range schemas {
		g.addEdges(schemas[i])
	}
	// 解析关联
	for _, t := range g.Nodes {
		check(resolve(t), "resolve %q relations", t.Name)
	}
	for _, t := range g.Nodes {
		check(t.setupFKs(), "set %q foreign-keys", t.Name)
	}
	// 构造索引
	for i := range schemas {
		g.addIndexes(schemas[i])
	}
	// 指定graph默认值,当前只有id type
	g.defaults()
	return
}
```

6.1. 构造节点：

```go
// entc/gen/graph.go:126
func (g *Graph) addNode(schema *load.Schema) {
	// schema翻译为Type/Node/Ent
	t, err := NewType(g.Config, schema)
	check(err, "create type %s", schema.Name)
	g.Nodes = append(g.Nodes, t)
}

// entc/gen/type.go:191
func NewType(c *Config, schema *load.Schema) (*Type, error) {
	idType := c.IDType
	if idType == nil {
		idType = defaultIDType
	}
	typ := &Type{
		Config: c,
		ID: &Field{
			cfg:  c,
			Name: "id",
			def: &load.Field{
				Name: "id",
			},
			Type:      idType,
			StructTag: structTag("id", ""),
		},
		schema:      schema,
		Name:        schema.Name,
		Annotations: schema.Annotations,
		Fields:      make([]*Field, 0, len(schema.Fields)),
		fields:      make(map[string]*Field, len(schema.Fields)),
		foreignKeys: make(map[string]struct{}),
	}
	if err := ValidSchemaName(typ.Name); err != nil {
		return nil, err
	}
	for _, f := range schema.Fields {
		tf := &Field{
			cfg:           c,
			def:           f,
			Name:          f.Name,
			Type:          f.Info,
			Unique:        f.Unique,
			Position:      f.Position,
			Nillable:      f.Nillable,
			Optional:      f.Optional,
			Default:       f.Default,
			UpdateDefault: f.UpdateDefault,
			Immutable:     f.Immutable,
			StructTag:     structTag(f.Name, f.Tag),
			Validators:    f.Validators,
			UserDefined:   true,
			Annotations:   f.Annotations,
		}
		if err := typ.checkField(tf, f); err != nil {
			return nil, err
		}
		// User defined id field.
    // 如果从schema中单独定义id字段，那么则id字段取用户自定义的
		if tf.Name == typ.ID.Name {
			typ.ID = tf
		} else {
			typ.Fields = append(typ.Fields, tf)
			typ.fields[f.Name] = tf
		}
	}
	return typ, nil
}
```

6.2. 构造边,并添加到相应node`entc/gen/graph.go:259`:

```go
func (g *Graph) addEdges(schema *load.Schema) {
	// 根据schema获取Type结构体
	t, _ := g.typ(schema.Name)
	seen := make(map[string]struct{}, len(schema.Edges))
	for _, e := range schema.Edges {
		typ, ok := g.typ(e.Type)
		// edge的type必须在graph的nodes里
		expect(ok, "type %q does not exist for edge", e.Type)
		_, ok = t.fields[e.Name]
		// edge的名字不能与字段名相同
		expect(!ok, "%s schema can't contain field and edge with the same name %q", schema.Name, e.Name)
		_, ok = seen[e.Name]
		// 不能定义多个名字一样的edge
		expect(!ok, "%s schema contains multiple %q edges", schema.Name, e.Name)
		seen[e.Name] = struct{}{}
		switch {
		// Assoc only.
		// 正向关联,比如: edge.To("card", Card.Type).Unique()
		case !e.Inverse:
			t.Edges = append(t.Edges, &Edge{
				def:         e,
				Type:        typ,
				Name:        e.Name,
				Owner:       t,
				Unique:      e.Unique,
				Optional:    !e.Required,
				StructTag:   structTag(e.Name, e.Tag),
				Annotations: e.Annotations,
			})
		// Inverse only.
		// 反向关联,比如: edge.From("owner", User.Type).Ref("card")
		case e.Inverse && e.Ref == nil:
			// 必须要指定RefName
			expect(e.RefName != "", "missing reference name for inverse edge: %s.%s", t.Name, e.Name)
			t.Edges = append(t.Edges, &Edge{
				def:         e,
				Type:        typ,
				Name:        e.Name,
				Owner:       typ,
				Inverse:     e.RefName,
				Unique:      e.Unique,
				Optional:    !e.Required,
				StructTag:   structTag(e.Name, e.Tag),
				Annotations: e.Annotations,
			})
		// Inverse and assoc.
		// 仅同类型的edge会进入这里,比如:
		// edge.To("following", User.Type).From("followers")
		case e.Inverse:
			// e.Ref 指正向边
			ref := e.Ref
			// 必须不指定Ref名字
			expect(e.RefName == "", "reference name is derived from the assoc name: %s.%s <-> %s.%s", t.Name, ref.Name, t.Name, e.Name)
			// 必须同类型
			expect(ref.Type == t.Name, "assoc-inverse edge allowed only as o2o relation of the same type")
			// 因为是用一条语句表达了正反两条边,所以需要把两条边都加入切片
			t.Edges = append(t.Edges, &Edge{
				def:         e,
				Type:        typ,
				Name:        e.Name,
				Owner:       t,
				Inverse:     ref.Name,
				Unique:      e.Unique,
				Optional:    !e.Required,
				StructTag:   structTag(e.Name, e.Tag),
				Annotations: e.Annotations,
			}, &Edge{
				def:         ref,
				Type:        typ,
				Owner:       t,
				Name:        ref.Name,
				Unique:      ref.Unique,
				Optional:    !ref.Required,
				StructTag:   structTag(ref.Name, ref.Tag),
				Annotations: ref.Annotations,
			})
		default:
			panic(graphError{"edge must be either an assoc or inverse edge"})
		}
	}
}
```

6.3 解析关联`entc/gen/graph.go:365`：

```go
func resolve(t *Type) error {
	for _, e := range t.Edges {
		switch {
		// 反向边
		case e.IsInverse():
			// 必须从e的node下的edges中找到相反的edge结构体
			ref, ok := e.Type.HasAssoc(e.Inverse)
			if !ok {
				return fmt.Errorf("edge %q is missing for inverse edge: %s.%s(%s)", e.Inverse, t.Name, e.Name, e.Type.Name)
			}
			// 两条边不能同时设为required
			if !e.Optional && !ref.Optional {
				return fmt.Errorf("edges cannot be required in both directions: %s.%s <-> %s.%s", t.Name, e.Name, e.Type.Name, ref.Name)
			}
			// 检查类型一致性
			if ref.Type != t {
				return fmt.Errorf("mismatch type for back-ref %q of %s.%s <-> %s.%s", e.Inverse, t.Name, e.Name, e.Type.Name, ref.Name)
			}
			// 设置edge的ref
			e.Ref, ref.Ref = ref, e
			// table 指的是需要加外键的表名
			table := t.Table()
			// Name the foreign-key column in a format that wouldn't change even if an inverse
			// edge is dropped (or added). The format is: "<Edge-Owner>_<Edge-Name>".
			// 以固定格式命名,即时反向也不会变
			column := fmt.Sprintf("%s_%s", e.Type.Label(), snake(ref.Name))
			// 确定关联类型
			switch a, b := ref.Unique, e.Unique; {
			// If the relation column is in the inverse side/table. The rule is simple, if assoc is O2M,
			// then inverse is M2O and the relation is in its table.
			case a && b:
				e.Rel.Type, ref.Rel.Type = O2O, O2O
			case !a && b:
				e.Rel.Type, ref.Rel.Type = M2O, O2M

			// If the relation column is in the assoc side.
			case a && !b:
				e.Rel.Type, ref.Rel.Type = O2M, M2O
				table = e.Type.Table()

			case !a && !b:
				e.Rel.Type, ref.Rel.Type = M2M, M2M
				table = e.Type.Label() + "_" + ref.Name
				c1, c2 := ref.Owner.Label()+"_id", ref.Type.Label()+"_id"
				// If the relation is from the same type: User has Friends ([]User).
				// give the second column a different name (the relation name).
				if c1 == c2 {
					c2 = rules.Singularize(e.Name) + "_id"
				}
				e.Rel.Columns = []string{c1, c2}
				ref.Rel.Columns = []string{c1, c2}
			}
			e.Rel.Table, ref.Rel.Table = table, table
			if !e.M2M() {
				e.Rel.Columns = []string{column}
				ref.Rel.Columns = []string{column}
			}
		// Assoc with uninitialized relation.
		case !e.IsInverse() && e.Rel.Type == Unk:
			switch {
			case !e.Unique && e.Type == t:
				e.Rel.Type = M2M
				e.Bidi = true
				e.Rel.Table = t.Label() + "_" + e.Name
				e.Rel.Columns = []string{e.Owner.Label() + "_id", rules.Singularize(e.Name) + "_id"}
			case e.Unique && e.Type == t:
				e.Rel.Type = O2O
				e.Bidi = true
				e.Rel.Table = t.Table()
			case e.Unique:
				e.Rel.Type = M2O
				e.Rel.Table = t.Table()
			default:
				e.Rel.Type = O2M
				e.Rel.Table = e.Type.Table()
			}
			if !e.M2M() {
				e.Rel.Columns = []string{fmt.Sprintf("%s_%s", t.Label(), snake(e.Name))}
			}
		}
	}
	return nil
}
```

7. 根据Graph生成真正文件`entc/gen/graph.go:179`：

```go
func (g *Graph) Gen() error {
	// 构造生成器
	var gen Generator = GenerateFunc(generate)
	// 叠加hook
	for i := len(g.Hooks) - 1; i >= 0; i-- {
		gen = g.Hooks[i](gen)
	}
	return gen.Generate(g)
}

// generate is the default Generator implementation.
func generate(g *Graph) error {
	var (
		assets   assets
		external []GraphTemplate
	)
	// 获取root templates和全局拓展模板
	templates, external = g.templates()
	// 从node开始遍历
	for _, n := range g.Nodes {
		assets.dirs = append(assets.dirs, filepath.Join(g.Config.Target, n.Package()))
		// 逐个模板生成
		for _, tmpl := range Templates {
			b := bytes.NewBuffer(nil)
			if err := templates.ExecuteTemplate(b, tmpl.Name, n); err != nil {
				return fmt.Errorf("execute template %q: %w", tmpl.Name, err)
			}
			assets.files = append(assets.files, file{
				path:    filepath.Join(g.Config.Target, tmpl.Format(n)),
				content: b.Bytes(),
			})
		}
	}
	// 全局模板和全局拓展模板生成
	for _, tmpl := range append(GraphTemplates, external...) {
		if tmpl.Skip != nil && tmpl.Skip(g) {
			continue
		}
		if dir := filepath.Dir(tmpl.Format); dir != "." {
			assets.dirs = append(assets.dirs, filepath.Join(g.Config.Target, dir))
		}
		b := bytes.NewBuffer(nil)
		if err := templates.ExecuteTemplate(b, tmpl.Name, g); err != nil {
			return fmt.Errorf("execute template %q: %w", tmpl.Name, err)
		}
		assets.files = append(assets.files, file{
			path:    filepath.Join(g.Config.Target, tmpl.Format),
			content: b.Bytes(),
		})
	}
	// 删掉不需要的Features相关文件
	for _, f := range AllFeatures {
		if f.cleanup == nil || g.featureEnabled(f) {
			continue
		}
		if err := f.cleanup(g.Config); err != nil {
			return fmt.Errorf("cleanup %q feature assets: %w", f.Name, err)
		}
	}
	// Write and format assets only if template execution
	// finished successfully.
	if err := assets.write(); err != nil {
		return err
	}
	// We can't run "imports" on files when the state is not completed.
	// Because, "goimports" will drop undefined package. Therefore, it's
	// suspended to the end of the writing.
	return assets.format()
}

```

至此，entc完成了代码生成的全部工作，由此可以看出，entc并未真正有关于orm的操作，所以可以把entc称为ent前端。entc generate的流程大致可以有如下表示：

1. 将我们写好的schema转化为entc所需要的graph结构
2. 确定此次生成器需要的模板
3. 逐个生成orm文件

