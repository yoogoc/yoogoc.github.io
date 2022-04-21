---
title: ent分析-runtime-create
tags:
- go
date: 2022-04-21 12:30:00
---

# ent runtime

通过简单的示例分析ent的运行时，目前只关心字段，暂不关心Edges

## 创建schema，生成ent代码

```go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Product holds the schema definition for the Product entity.
type Product struct {
	ent.Schema
}

// Fields of the Product.
func (Product) Fields() []ent.Field {
	return []ent.Field{
		field.Int64("id"),
		field.String("name"),
		field.String("pic"),
		field.String("code"),
		field.Bool("is_publish"),
		field.Int32("sort"),
		field.Time("created_at").Immutable().Default(time.Now),
		field.Time("updated_at").Default(time.Now).UpdateDefault(time.Now),
	}
}

// Edges of the Product.
func (Product) Edges() []ent.Edge {
	return nil
}
```

## 初始化driver

ent 对database/sql包进行了一次封装，把database/sql#DB结构体抽象到了ExecQuerier接口中

```go
// entgo.io/ent/dialect/driver.go
type ExecQuerier interface {
	ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
	QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
}

// Conn implements dialect.ExecQuerier given ExecQuerier.
type Conn struct {
	ExecQuerier
}

// Driver is a dialect.Driver implementation for SQL based databases.
// 这个结构体实际上是dialect.Driver的实现
type Driver struct {
	Conn
  // 这个字段是为了在buildersql的时候针对不同的数据库采用不同的语法或分隔符，比如pg的quote是",而mysql的是`
	dialect string
}

func Open(driver, source string) (*Driver, error) {
	db, err := sql.Open(driver, source)
	if err != nil {
		return nil, err
	}
	return &Driver{Conn{db}, driver}, nil
}
```

## 初始化client

1. 调用`ent/client.go:NewClient`

```bash
func NewClient(opts ...Option) *Client {
	// 创建配置对象
	cfg := config{log: log.Println, hooks: &hooks{}}
  // 通过Option方式传入自定义参数
	cfg.options(opts...)
  // 创建Client结构体
	client := &Client{config: cfg}
  // 初始化Client
	client.init()
	return client
}
```

整体流程还是非常简单的，接下来看下config都保存了什么内容

```bash
type config struct {
	// 执行db请求的驱动器
	driver dialect.Driver
	// 打出debug日志
	debug bool
	// debug模式使用的日志方法
	log func(...interface{})
	// 每个mutations执行时的hook
	hooks *hooks
}

type hooks struct {
  // product表执行时的hook
	Product    []ent.Hook
}
```

这个config内容不多，主要字段是driver和hook, 继续看下client结构体都保存了什么

```bash
type Client struct {
  // 配置
	config
	// 用于迁移db的schema结构体
	Schema *migrate.Schema
	// 与builders交互的client，我们这里只有一个ent schema，所以只有一个，下同
	Product *ProductClient
}
```

1. client自身的初始化

```bash
func (c *Client) init() {
  // 根据驱动器生成用于db迁移的Schema结构体
	c.Schema = migrate.NewSchema(c.driver)
  // 初始化ProductClient
	c.Product = NewProductClient(c.config)
}

// ent/migrate/migrate.go
// 初始化Schema
type Schema struct {
	drv dialect.Driver
}
func NewSchema(drv dialect.Driver) *Schema { return &Schema{drv: drv} }

// 初始化ProductClient
type ProductClient struct {
	config
}
func NewProductClient(c config) *ProductClient {
	return &ProductClient{config: c}
}
```

就这样，client初始化已经完成

## 事务的处理

ent使用`client.Tx(ctx)`或`client.BeginTx(ctx, opts)`开始一个事务，其区别在数BeginTx可以指定事务隔离级别

```go
func (c *Client) Tx(ctx context.Context) (*Tx, error) {
	if _, ok := c.driver.(*txDriver); ok {
		return nil, fmt.Errorf("ent: cannot start a transaction within a transaction")
	}
	tx, err := newTx(ctx, c.driver)
	if err != nil {
		return nil, fmt.Errorf("ent: starting a transaction: %w", err)
	}
	cfg := c.config
	cfg.driver = tx
	return &Tx{
		ctx:     ctx,
		config:  cfg,
		Product: NewProductClient(cfg),
	}, nil
}

func (c *Client) BeginTx(ctx context.Context, opts *sql.TxOptions) (*Tx, error) {
	if _, ok := c.driver.(*txDriver); ok {
		return nil, fmt.Errorf("ent: cannot start a transaction within a transaction")
	}
	tx, err := c.driver.(interface {
		BeginTx(context.Context, *sql.TxOptions) (dialect.Tx, error)
	}).BeginTx(ctx, opts)
	if err != nil {
		return nil, fmt.Errorf("ent: starting a transaction: %w", err)
	}
	cfg := c.config
	cfg.driver = &txDriver{tx: tx, drv: c.driver}
	return &Tx{
		ctx:     ctx,
		config:  cfg,
		Product: NewProductClient(cfg),
	}, nil
}
```

可以看到，两种方式开启一个事务和初始化client流程几乎相同。

## 通过`ProductClient`创建一个builder

```go
// Create returns a create builder for Product.
func (c *ProductClient) Create() *ProductCreate {
	mutation := newProductMutation(c.config, OpCreate)
	return &ProductCreate{config: c.config, hooks: c.Hooks(), mutation: mutation}
}

// CreateBulk returns a builder for creating a bulk of Product entities.
func (c *ProductClient) CreateBulk(builders ...*ProductCreate) *ProductCreateBulk {
	return &ProductCreateBulk{config: c.config, builders: builders}
}

// Update returns an update builder for Product.
func (c *ProductClient) Update() *ProductUpdate {
	mutation := newProductMutation(c.config, OpUpdate)
	return &ProductUpdate{config: c.config, hooks: c.Hooks(), mutation: mutation}
}

// UpdateOne returns an update builder for the given entity.
func (c *ProductClient) UpdateOne(pr *Product) *ProductUpdateOne {
	mutation := newProductMutation(c.config, OpUpdateOne, withProduct(pr))
	return &ProductUpdateOne{config: c.config, hooks: c.Hooks(), mutation: mutation}
}

// UpdateOneID returns an update builder for the given id.
func (c *ProductClient) UpdateOneID(id int64) *ProductUpdateOne {
	mutation := newProductMutation(c.config, OpUpdateOne, withProductID(id))
	return &ProductUpdateOne{config: c.config, hooks: c.Hooks(), mutation: mutation}
}

// Delete returns a delete builder for Product.
func (c *ProductClient) Delete() *ProductDelete {
	mutation := newProductMutation(c.config, OpDelete)
	return &ProductDelete{config: c.config, hooks: c.Hooks(), mutation: mutation}
}

// DeleteOne returns a delete builder for the given entity.
func (c *ProductClient) DeleteOne(pr *Product) *ProductDeleteOne {
	return c.DeleteOneID(pr.ID)
}

// DeleteOneID returns a delete builder for the given id.
func (c *ProductClient) DeleteOneID(id int64) *ProductDeleteOne {
	builder := c.Delete().Where(product.ID(id))
	builder.mutation.id = &id
	builder.mutation.op = OpDeleteOne
	return &ProductDeleteOne{builder}
}

// Query returns a query builder for Product.
func (c *ProductClient) Query() *ProductQuery {
	return &ProductQuery{
		config: c.config,
	}
}

// Get returns a Product entity by its id.
func (c *ProductClient) Get(ctx context.Context, id int64) (*Product, error) {
	return c.Query().Where(product.ID(id)).Only(ctx)
}

// GetX is like Get, but panics if an error occurs.
func (c *ProductClient) GetX(ctx context.Context, id int64) *Product {
	obj, err := c.Get(ctx, id)
	if err != nil {
		panic(err)
	}
	return obj
}
```

product会有几种不同的builder，可以按照以下方式分类：

1. 是否批量：批量不会创建自己的mutation，但是会将传入的builder切片保存到自己的结构体里。
2. 是否支持hook：增改删相关可以执行hook，而查询相关方法是不能带hook的。
3. 基础builder和高级builder：我将（Create，Update，Delete，Query）成为基础builder，其余为高级builder。
4. 是否会panic：方法后带X的会panic。

每一个基础builder都会生成一个通过`newProductMutation`创建的mutation，先看下mutation结构：

```go
// 可以把一个mutation分成几个部分来对待：
type ProductMutation struct {
  // 1. 配置，与client相同
	config
  // 2. 当前操作类型
	op            Op
  // 3. 节点类型（固定字符串Product）
	typ           string
	// 4. 指针类型的field
	id            *int64
	name          *string
	pic           *string
	code          *string
	is_publish    *bool
	sort          *int32
	addsort       *int32
	created_at    *time.Time
	updated_at    *time.Time

  // 需要清空的字段
	clearedFields map[string]struct{}

	// 与oldValue有关
	done          bool
	// 旧值
	oldValue      func(context.Context) (*Product, error)
	// 谓语，也可以说是where条件
	predicates    []predicate.Product
}
```

现在再看ProductCreate这个builder，还是先看结构体：

```go
type ProductCreate struct {
	config
	mutation *ProductMutation
	hooks    []Hook
}
```

结构体依然朴实无华，还是看见常用方法，我们在create一个实体通常有如下操作：

```go
client.Product.Create().SetName("product1").Save(ctx)
```

create builder使用SetXX()或方法来设置值

```go
func (pc *ProductCreate) SetName(s string) *ProductCreate {
	pc.mutation.SetName(s)
	return pc
}

...

func (pc *ProductCreate) SetCreatedAt(t time.Time) *ProductCreate {
	pc.mutation.SetCreatedAt(t)
	return pc
}

func (pc *ProductCreate) SetNillableCreatedAt(t *time.Time) *ProductCreate {
	if t != nil {
		pc.SetCreatedAt(*t)
	}
	return pc
}
```

这里可能会有疑问：为什么有些字段会有Nillable setter，而有一些却没有，这里我们可以通过阅读源码中的模板得到([setter.tmpl](https://github.com/ent/ent/blob/cb6e0e063dbc88664a0377e7fde4796b593f9469/entc/gen/template/builder/setter.tmpl#L35)):

```go
{{ if and (not $f.Type.Nillable) (or $f.Optional $f.Default) (not (and $updater $f.UpdateDefault)) }}
```

`Type.Nillable`字段的意思**大概可以理解为**该**实体**字段是否是指针类型，那这个判断整体可以理解为：

如果 **字段不是指针类型**，且（**字段是不是必填** 或 **字段有默认值**）且 **不能是**（**更新操作**中 **该字段有更新默认值**）

更新操作的`ProductUpdate` ，`ProductUpdateOne`，`ProductDelete`和`ProductDeleteOne`中Setter方法与ProductCreate中类似，这里就不展开啰嗦了。

## builder构建完成，核心的执行方法

从ProductCreate Save开始：

```go
func (pc *ProductCreate) Save(ctx context.Context) (*Product, error) {
	var (
		err  error
		node *Product
	)
	// 根据Schema中定义Default方法对未赋值的字段进行赋默认值
	pc.defaults()
	// 根据有无hook将执行逻辑分为两个部分，但其最终都会调用sqlSave(ctx)方法
	if len(pc.hooks) == 0 {
    // 对字段进行检查，每个字段可能会有两种检查，第一是对不是optional的字段检查是否已经赋值
    // 第二种是对这个字段自定义的Validators进行检查
		if err = pc.check(); err != nil {
			return nil, err
		}
    // 真正有价值的东西，后面单独分析
		node, err = pc.sqlSave(ctx)
	} else {
    // 构造 Mutator ，使hook能通过slice形式进行递归调用，比如：Use(f, g, h)` 等于 `product.Hooks(f(g(h())))
		var mut Mutator = MutateFunc(func(ctx context.Context, m Mutation) (Value, error) {
			mutation, ok := m.(*ProductMutation)
			if !ok {
				return nil, fmt.Errorf("unexpected mutation type %T", m)
			}
			if err = pc.check(); err != nil {
				return nil, err
			}
			pc.mutation = mutation
			if node, err = pc.sqlSave(ctx); err != nil {
				return nil, err
			}
			mutation.id = &node.ID
			mutation.done = true
			return node, err
		})
		for i := len(pc.hooks) - 1; i >= 0; i-- {
			if pc.hooks[i] == nil {
				return nil, fmt.Errorf("ent: uninitialized hook (forgotten import ent/runtime?)")
			}
			mut = pc.hooks[i](mut)
		}
		if _, err := mut.Mutate(ctx, pc.mutation); err != nil {
			return nil, err
		}
	}
	return node, err
}
```

执行sqlSave：

```go
func (pc *ProductCreate) sqlSave(ctx context.Context) (*Product, error) {
	// 构造Product对象和sqlgraph.CreateSpec
	_node, _spec := pc.createSpec()
  // 构造sql并执行
	if err := sqlgraph.CreateNode(ctx, pc.driver, _spec); err != nil {
		if sqlgraph.IsConstraintError(err) {
			err = &ConstraintError{err.Error(), err}
		}
		return nil, err
	}
  // 更新id
	if _spec.ID.Value != _node.ID {
		id := _spec.ID.Value.(int64)
		_node.ID = int64(id)
	}
	return _node, nil
}
```

构造Spec（流程依旧朴实无华，代码即注释，不啰嗦了）:

```go
func (pc *ProductCreate) createSpec() (*Product, *sqlgraph.CreateSpec) {
	var (
		_node = &Product{config: pc.config}
		_spec = &sqlgraph.CreateSpec{
			Table: product.Table,
			ID: &sqlgraph.FieldSpec{
				Type:   field.TypeInt64,
				Column: product.FieldID,
			},
		}
	)
	if id, ok := pc.mutation.ID(); ok {
		_node.ID = id
		_spec.ID.Value = id
	}
	if value, ok := pc.mutation.Name(); ok {
		_spec.Fields = append(_spec.Fields, &sqlgraph.FieldSpec{
			Type:   field.TypeString,
			Value:  value,
			Column: product.FieldName,
		})
		_node.Name = value
 	}

  ...

	return _node, _spec
}
```

## 万事俱备只欠东风

终于从codegen生产的代码跳到了ent包内

```go
func CreateNode(ctx context.Context, drv dialect.Driver, spec *CreateSpec) error {
  // 所有的builder都会用这个通用的graph来构造和执行sql
	gr := graph{tx: drv, builder: sql.Dialect(drv.Dialect())}
  // creator 是为了create操作抽象成的结构体
	cr := &creator{CreateSpec: spec, graph: gr}
  // 构造和执行
	return cr.node(ctx, drv)
}
```

构造和执行,忽略了edges

```go
func (c *creator) node(ctx context.Context, drv dialect.Driver) error {
	var (
		...
    // 构造真正的InsertBuilder
		insert = c.builder.Insert(c.Table).Schema(c.Schema).Default()
	)
  // 填充字段值
	if err := c.setTableColumns(insert, edges); err != nil {
		return err
	}
  // 根据edges判断是否为多行sql，如果是则开启事务。
  // 无论怎样都会把creator的tx替换，如果开启事务：则结构体为https://github.com/ent/ent/blob/cb6e0e063dbc88664a0377e7fde4796b593f9469/dialect/sql/driver.go#Tx
  // 如果不开启：则结构体为https://github.com/ent/ent/blob/cb6e0e063dbc88664a0377e7fde4796b593f9469/dialect/dialect.go#NopTx
	tx, err := c.mayTx(ctx, drv, edges)
	if err != nil {
		return err
	}
	if err := func() error {
    // 拼接和执行sql
		if err := c.insert(ctx, insert); err != nil {
			return err
		}
		...
	}(); err != nil {
		return rollback(tx, err)
	}
	return tx.Commit()
}
```

拼接和执行sql：

```go
func (c *creator) insert(ctx context.Context, insert *sql.InsertBuilder) error {
	if opts := c.CreateSpec.OnConflict; len(opts) > 0 {
		insert.OnConflict(opts...)
		c.ensureLastInsertID(insert)
	}
	// 如果外部提供id值
	if c.ID.Value != nil {
		insert.Set(c.ID.Column, c.ID.Value)
		// In case of "ON CONFLICT", the record may exists in the
		// database, and we need to get back the database id field.
		if len(c.CreateSpec.OnConflict) == 0 {
      // 拼接sql，构造参数数组，这个query方法只是不停地拼字符串，就不展开啰嗦了
			query, args := insert.Query()
      // 执行sql，这个是各种驱动器实现的，就不深入分析了
			return c.tx.Exec(ctx, query, args, nil)
		}
	}
	return c.insertLastID(ctx, insert.Returning(c.ID.Column))
}
```

不提供id值的情况，需要从数据库取回id并赋值回CreateSpec