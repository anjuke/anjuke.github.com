---
layout: post
title: "Aifang AJF框架"
date: 2013-03-20 12:00
author: wbsong
comments: true
categories:
---

## AJF - 爱房Java框架

### 远程代码仓库

####生产环境&开发环境都是 `git@git.corp.anjuke.com:aifang/aifang-java-site`
Java的项目目前比较少，所以大家都在一个仓库里面开发，所有人有全部的权限


### 目录结构
	
	所有项目的总的目录结构
	├── ajf-3rd //为了方便升级第三方类库，我们会对一些常用的包进行一些包装，这样当我们想换成另外一种实现方式的时候，就会很方便
	├── ajf-core //常用工具集，例如对缓存、JSON、日志、配置文件等的封装都在这里
	├── ajf-dal //数据访问层，在这里我们封装了JDBC & Mongo等常用的数据访问
	├── ajf-datasource //我们对数据源做了封装，可以实现一个JVM实例共享一个连接池，作用相当于Tomcat自带的JNDI的实现
	├── ajf-web //对于web层简单的封装
	├── app-inform //资讯相关的业务实现MVC中的C&V层
	├── app-loupan //楼盘相关的业务
	├── build //打包版本、发布版本、生成项目依赖等工具包
	├── domain-inform //资讯相关的业务，主要是对业务的一些封装
	├── domain-loupan //楼盘相关的业务实现
	├── service-loupan //楼盘相关的服务，主要是运行一些线程，例如从二手房同步房源
	
	build目录详情
	build
	├── aifang-pom.template.xml //爱房所有的项目都要引用的pom选项
	├── bin
	│   ├── build-bootstrap.jar //pom-gen.sh所依赖的jar包
	│   ├── deploy.sh //发布一组项目到爱房的代码仓库
	│   ├── install.sh //安装一组项目到本地仓库
	│   ├── jetty.sh //使用jetty插件启动本地的项目
	│   ├── pom-clean.sh //清除一组项目的pom.xml文件
	│   ├── pom-gen.bat //Windows下面生成一组项目的pom.xml文件
	│   └── pom-gen.sh //UnixLike下面生成一组项目的pom.xml文件 
	├── checkstyle
	│   └── aifang.xml
	├── groups //不同的项目集合
	│   ├── ajf.txt
	│   ├── alldev.txt
	│   ├── all.txt
	│   ├── build.txt
	
	一个典型的模块的目录结构，例如domain-loupan
	.
	└── com
    	└── aifang
        	└── domain
            	└── loupan
                	├── biz //对内外提供的接口，组装一些业务方法
                	├── dao //针对单个表增删查改
                	├── dto //输入输出的对象，某个方法可能会传递很多的参数，这个时候好的方案是采用对象传递
                	├── LoupanDomainModule.java //每个模块对外的统一入口，主要用来寻找spring的配置文件
                	├── model //数据库表字段和对象属性之间的对应关系
	
框架目录命名规范
	
	ajf-xxxxxx 属于框架的公共的类库
	app-xxxxxx MVC结构中处理C&V的部分
	domain-xxxxxx MVC结构中处理M的部分
	service-xxxxxx 主要运行一些常驻内存的线程
	
	
### 明白包是如何管理的
	之前如果是开发php等非编译型语言的工程师可能没法理解包管理的概念，因为每次运行都是将所有的源代码放到一起运行，但是向java等编译型语言在运行的时候，并不能使用源代码直接运行，而是要依赖于编译后的字节码，而且我们依赖于第三方的类库基本上都会有很多版本，且并不是所有的版本都适合我们自己的项目，所以对于包的管理就显得尤为重要。
	还有一点就是我们自己的代码随着时间的增长，模块越来越多，而且有些模块其实已经很成熟了，例如ajf-xxxxx其实已经很长时间没有修改了，但是我么有些业务逻辑的模块基本上几天就会修改一次，例如domain-loupan,domain-loupan是对ajf-xxxxx是有依赖关系的，我们之前的策略是每次编译我们我们所有的依赖与我们自己的项目，这样做的后果是每次调试，编译都很慢，后来我们对自己的项目也做了包的版本控制，这样每次只要编译少量的项目就可以了。 
	
#### 版本控制
	我们自己的版本控制分成两种，第一种dev-SNAPSHOP，作为开发调试测试阶段使用，第二种RELEASE版本，有具体的代号例如xx.xx.xx，作为生产环境的版本使用
```xml
<!--build/aifang-pom.template.xml->
<!--我们每个模块都会有自己的版本-->
<properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <ajf.version>2.1.0</ajf.version>
        <ajf2.version>${ajf.version}</ajf2.version>
        <domain-common.version>2.1.0</domain-common.version>
        <domain-crm.version>2.1.0</domain-crm.version>
        <domain-dw.version>2.1.0</domain-dw.version>
        <domain-ifx.version>2.1.0</domain-ifx.version>
        <domain-inform.version>2.1.0</domain-inform.version>
        <domain-loupan.version>2.1.0</domain-loupan.version>
        <!-- dependency version -->
        <slf4j.version>1.5.6</slf4j.version>
        <commons-codec.version>1.7</commons-codec.version>
</properties>
<dependencies>
        <!-- test -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
        </dependency>
        <!-- logger -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
</dependencies>  

<!--app-loupan/pom.template.xml-->
<dependencies>
	<!--对于我们自己的稳定的模块直接已经发布的版本-->
    <dependency>
        <groupId>com.aifang.ajf</groupId>
        <artifactId>ajf-core</artifactId>
        <version>${ajf2.version}</version>
    </dependency>
	<!--对于楼盘自己的项目，采用project.version，也就是说和app-loupan一期编译-->    
    <dependency>
        <groupId>com.aifang.domain</groupId>
        <artifactId>domain-loupan</artifactId>
        <version>${project.version}</version>
    </dependency>
	<!--对于第三方的包，我们在项目里面不具体指定是那个版本，版本管理在build/aifang-pom.template.xml-->
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
    </dependency>   
</dependencies>    
     
```
 			
### 搭建环境
	安装Java，设置JAVA_HOME eg: export JAVA_HOME=/usr/local/jdk
	安装maven
	设置PATH 变量，将java的bin目录&maven的bin目录加入到PATH中
	安装zeromq-2.1.x & jzmq
	for mac 软链接 libjzmq.dylib 到 /usr/lib/java/
	for linux 软链接 libjzmq.so 到 /usr/lib/
	 
### 开始开发,例如开发楼盘项目

如何运行项目
	
	$ git clone git@git.corp.anjuke.com:aifang/aifang-java-site
	$ cd aifang-java-site
	$ ./build/bin/pom-gen.sh loupan
	$ mvn clean -Dmaven.test.skip=true install
	$ mvn jetty:run -pl app-loupan

如何从头开始一个模块
	
	一个模块可能需要新建新建几个表，需要一个对外统一的业务对象
	一个访问DB的模块需要3层，Model、Dao、Biz
	Model负责对DB的表进行结构的描述
	Dao负责对Model的包装，提供增删查改等基本的功能
	Biz负责将多个相关的Dao组合起来对外提供统一的服务，例如我们可以将对loupan的所有操作放到一起

如何创建一个Model，例如我们有一个cities表，需要创建一个Model

```java
@Entity //标记这个类是一个Model
@Table(name = "cities") //标记这个Model对应的数据的表名称是cities
@DBModule(DBType.NEWHOME) //标记这个Model所在的数据库是newhome_db,DBType在ajf-datasources有定义
public class CitiesModel  implements Serializable { //一定要实现Serializable，因为这个类的对象需要放到Memcache等缓存中


    private static final long serialVersionUID = -7021780954838273442L; //Java规范

    @Id //标记city_id这个字段是主键
    @Column(name = "city_id") //标记这个类的cityId字段对应cities.city_id字段，我们表字段&名称的规范是下划线作为单词的分割
    private int cityId; //类的属性和方法名称、类名称等使用java的驼峰规则

    @Column(name = "city_name") 
    private String cityName;
    
}
```

如何创建一个Dao，例如我们要对刚才的CitiesModel表进行一个包装提供基本的增删查改功能
我们需要2个步骤，A创建一个接口继承，B创建一个类，继承这个接口并且实现AbstractAJFGenericJDBCDao类 
```java
public interface CitiesDao extends AJFGenericDao<Integer, CitiesModel> {	

}

@Repository("citiesDao") //通过这个自动注解，我们就可以使用citiesDao这个名称来使用CitiesDaoImpl这个类的对象
public class CitiesDaoImpl extends
        AbstractAJFGenericJDBCDao<Integer, CitiesModel> implements CitiesDao {

}
```

如何创建一个业务对象
和Dao一样，我们需要创建一个接口&一个实体类
```java
public interface CitiesBiz {
    public List<CitiesModel> getAllCities();
}

@Service("citiesBiz") //让Spring生产一个citiesBiz的关于CitiesBizImpl这个类的对象
public class CitiesBizImpl implements CitiesBiz {

    @Resource //Spring会自动引用一个叫citiesDao的对象，我们刚才使用@Repository("citiesDao")定义的
    private CitiesDao citiesDao;

    @Override
    public List<CitiesModel> getAllCities() { //这里实现我们的业务逻辑
        return citiesDao.findAll();
    }
}
```

上面Dao&Biz的使用都是在同一个模块内的情况，真实环境中，我们可能会使用其他模块的对象，例如我们在开发楼盘项目的时候，我们可能会使用到资讯的相关模块。例如还是上面的例子，我们引用资讯的一个叫articleBiz的ArticleBiz类的对象

```java
@Resource //Spring会自动引用一个叫citiesDao的对象，我们刚才使用@Repository("citiesDao")定义的
private CitiesDao citiesDao;

@Module(InformDomainModule.class) //这里注意和上面是稍微有些不一样的，Module注解是我们自己实现的
private ArticleBiz articleBiz;
```

理解模块是什么。随着我们网站越来越大，功能也越来越复杂，所以解决这种复杂局面的最好的方法就是分模块，我们一般是按照业务的维度来拆分的。例如我们将现有的业务拆分成domain-loupan,domain-inform,domain-crm等。拆分完成后我们可能在做某个项目的时候会引用多个模块的对象，这个时候我们不希望通过include的方式将多个模块合并起来，如果使用include的方式，那么我们其实只做到了代码的分离，我们还希望做的更好一点，能做到物理隔离，每个模块有自己的配置文件，有自己的Context。模块里面的对象通过Module注解对外提供对象。

如何创建一个模块，例如domain-loupan要对外提供服务

- 首先我们每个模块都会有自己的基本的命名空间，例如domain-loupan的命名空间是com.aifang.domain.loupan，我们会在模块的根命名空间下面创建一个 LoupanDomainModule的接口
```java
package com.aifang.domain.loupan;

public interface LoupanDomainModule {

}
```

- 然后我们会在项目的resources目录里面创建和上面一样的包结构，并且创建一个名为LoupanDomainModule**Context**.xml的文件，注意这里和上面相比多了一个**Context**单词

	
``` 
└── resources
    ├── com
    │   └── aifang
    │       └── domain
    │           └── loupan
    │               ├── LoupanDomainModuleContext.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans ...>
	<!-- 如果项目中有用到数据库，需要包含ajf的DB实现-->
	<import resource="classpath:ajfDaoApplicationContext.xml" />

	<!-- 为了方便的使用spring，我们使用了spring的自动注解模式，通过在类的上面进行Service，Repository注解生产默认类名首字母小写的对象ID-->
	<context:component-scan base-package="com.aifang.domain.loupan" />

	<!--有些时候，自动注解不能满足需求，我们需要手动注册对象到spring中-->
	<bean id="cache.DEFAULT" class="com.aifang.ajf.spring.MemcachedCacheFactoryBean"
		lazy-init="true">
		<property name="name" value="DEFAULT" />
		<property name="configName" value="loupan-memcached" />
	</bean>


	<!--加载配置文件-->
	<!-- 加载的顺序，最先加载的有效，和V2的顺序是反的-->
	<!---1 /home/www/config/local/domain-loupan.properties->
	<!---2 /home/www/config/java/domain-loupan.properties->
	<!---3 classpath:META-INF/config/local/domain-loupan.properties->
	<!---4 classpath:META-INF/config/default/domain-loupan.properties->
	<!-- property config -->
	<bean id="loupanDomainProps" class="com.aifang.ajf.spring.ConfigPropertiesFactoryBean">
		<property name="name" value="domain-loupan" />
	</bean>

	<bean
		class="org.springframework.beans.factory.config.PreferencesPlaceholderConfigurer">
		<property name="ignoreUnresolvablePlaceholders" value="true" />
		<property name="properties" ref="loupanDomainProps" />
	</bean>
</beans>

```

- 需要注意的是Spring里面的对象有2种创建方式，第一种通过component-scan+Service\Repository方式自动化，第二种自己手动配置 通过 <bean id="xxx" class="xxx">


### 框架常用命令
	
	$ build/bin/pom-gen.sh [project-group] [-v xxx] //生成一组项目，如果不指定-v的版本，那么就是 dev-SNAPSHOP
	$ billd/bin/deploy.sh [project-group] [-v xxx] //发布一组项目到爱房自己的仓库，project-group在 build/group/
	$ build/bin/jetty.sh [project] //使用jetty运行一个项目，pom里面需要配置插件
	$ mvn jetty:run //对于有些项目可以直接使用jetty来调试
	$ mvn eclipse:clean eclipse:eclipse //生成eclipse的项目文件 .project、.classpath、.settings
	$ mvn clean -Dmaven.test.skip=true install //将项目打包安装到自己的本地仓库，位于 ~/.m2/repository
	$ mvn dependency:tree //查看项目包之间的依赖关系
	
	


    			
