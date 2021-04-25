---
title: java学习笔记-spring datasource加载driver分析
tags:
- java
date: 2020-10-12 22:53:00
---

默认HikariDataSource

1. 创建测试项目，引入相关包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.2.17</version>
</dependency>
```



2. spring boot会在启动时DataSourceConfiguration.class,由于spring-boot-starter-jdbc中包含hikari包，则会加载HikariDataSource：

   ```java
   // source in: DataSourceConfiguration.java#static class Hikari
   	/**
   	 * Hikari DataSource configuration.
   	 */
   	@Configuration(proxyBeanMethods = false)
   	@ConditionalOnClass(HikariDataSource.class)
   	@ConditionalOnMissingBean(DataSource.class)
   	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
   			matchIfMissing = true)
   	static class Hikari {

   		@Bean
   		@ConfigurationProperties(prefix = "spring.datasource.hikari")
   		HikariDataSource dataSource(DataSourceProperties properties) {
   			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
   			if (StringUtils.hasText(properties.getName())) {
   				dataSource.setPoolName(properties.getName());
   			}
   			return dataSource;
   		}
   	}

   ```
   ```java
	protected static <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
		return (T) properties.initializeDataSourceBuilder().type(type).build();
	}
	```

3. 查看DataSourceConfiguration上下文可得结论：无论哪种数据源/连接池都需要调用其静态方法createDataSource来初始化

4. 进入DataSourceProperties#initializeDataSourceBuilder方法，这里会真正的创建DataSource

   ```java
   	public DataSourceBuilder<?> initializeDataSourceBuilder() {
   		return DataSourceBuilder.create(getClassLoader()).type(getType()).driverClassName(determineDriverClassName())
   				.url(determineUrl()).username(determineUsername()).password(determinePassword());
   	}
   ```

   a. 通过`getClassLoader()).type(getType())`加载出HikariDataSource

   b. 设置Driver及相关参数

5. 由上代码可以知道determineDriverClassName()方法中会有确定驱动的算法

   ```java
   public String determineDriverClassName() {
   		if (StringUtils.hasText(this.driverClassName)) {
   			Assert.state(driverClassIsLoadable(), () -> "Cannot load driver class: " + this.driverClassName);
   			return this.driverClassName;
   		}
   		String driverClassName = null;
   		if (StringUtils.hasText(this.url)) {
   			driverClassName = DatabaseDriver.fromJdbcUrl(this.url).getDriverClassName();
   		}
   		if (!StringUtils.hasText(driverClassName)) {
   			driverClassName = this.embeddedDatabaseConnection.getDriverClassName();
   		}
   		if (!StringUtils.hasText(driverClassName)) {
   			throw new DataSourceBeanCreationException("Failed to determine a suitable driver class", this,
   					this.embeddedDatabaseConnection);
   		}
   		return driverClassName;
   	}
   ```

   显而易见的算法：

   1. 优先级最高的是直接指定spring.datasource.driver-class-name
   2. 其次是spring.datasource.url
   3. 如果都没指定，则会从embeddedDatabaseConnection中取得
   4. embeddedDatabaseConnection的加载逻辑是NONE，H2，DERBY，HSQL中按序循环，如果能找到相关类，就返回相对的embeddedDatabaseConnection
   5. determine**()方法都是类似的就不展开说明了



6. properties.initializeDataSourceBuilder().type(type).build() =》

   ```java
   private void bind(DataSource result) {
   		ConfigurationPropertySource source = new MapConfigurationPropertySource(this.properties);
   		ConfigurationPropertyNameAliases aliases = new ConfigurationPropertyNameAliases();
   		aliases.addAliases("driver-class-name", "driver-class");
   		aliases.addAliases("url", "jdbc-url");
   		aliases.addAliases("username", "user");
   		Binder binder = new Binder(source.withAliases(aliases));
   		binder.bind(ConfigurationPropertyName.EMPTY, Bindable.ofInstance(result));
   	}
   ```

7. bind方法会绑定相应bean，其中会调用HikariConfig#setDriverClassName方法

   ```java
      public void setDriverClassName(String driverClassName)
      {
         checkIfSealed();

         Class<?> driverClass = attemptFromContextLoader(driverClassName);
         try {
            if (driverClass == null) {
               driverClass = this.getClass().getClassLoader().loadClass(driverClassName);
               LOGGER.debug("Driver class {} found in the HikariConfig class classloader {}", driverClassName, this.getClass().getClassLoader());
            }
         } catch (ClassNotFoundException e) {
            LOGGER.error("Failed to load driver class {} from HikariConfig class classloader {}", driverClassName, this.getClass().getClassLoader());
         }

         if (driverClass == null) {
            throw new RuntimeException("Failed to load driver class " + driverClassName + " in either of HikariConfig class loader or Thread context classloader");
         }

         try {
            driverClass.getConstructor().newInstance();
            this.driverClassName = driverClassName;
         }
         catch (Exception e) {
            throw new RuntimeException("Failed to instantiate class " + driverClassName, e);
         }
      }

   ```

   这里会尝试driverClass.getConstructor().newInstance();，即实例化一个h2/pg Driver对象，打开org.h2.Driver/org.postgresql.Driver 可以看到有static代码块初始化registerDriver操作
