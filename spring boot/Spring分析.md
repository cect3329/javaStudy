我看不下去了。。。

见博客https://blog.csdn.net/nuomizhende45/article/details/81158383

# 1 IOC容器的创建 

```java
package SpringDemo;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public interface MessageService {
    String getMessage();
}

class MessageServiceImp implements MessageService{
    public String getMessage() {
        return "hello,world";
    }
}

class App{
    public static void main(String[] args) {
        ApplicationContext ioc = new ClassPathXmlApplicationContext("application.xml"); // 从该方法进入，深入剖析
        System.out.println("Context ok");
        MessageService me = ioc.getBean(MessageService.class);
        System.out.println(me.getMessage());
        BeanDefinition
    }
}
```

# 1.1  ClassPathXmlApplicationContext("xxx");

```java
ClassPathXmlApplicationContext类 
```

```java
//调用方法
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
//真正调用的方法
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent); //如果有父容器，就先创建父容器
		setConfigLocations(configLocations);//解析传入的xml路径
		if (refresh) {
			refresh(); //关键方法，重新刷新==>也就是开始创建容器
		}
	}
```

# 1.2 refresh() 

```java
AbstractApplicationContext类
```

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader 
      implements ConfigurableApplicationContext {
    ......
        public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			prepareRefresh(); //创建容器前的准备工作
			//ApplicationContext 不是继承 了 beanFactory  而是持有一个 BeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();//返回一个Bean的制造工厂

			prepareBeanFactory(beanFactory);

			try {
				postProcessBeanFactory(beanFactory);

				invokeBeanFactoryPostProcessors(beanFactory);

				registerBeanPostProcessors(beanFactory);

				initMessageSource();

				initApplicationEventMulticaster();

				onRefresh();

				registerListeners();

				finishBeanFactoryInitialization(beanFactory);

				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				destroyBeans();

				cancelRefresh(ex);

				throw ex;
			}

			finally {
				resetCommonCaches();
			}
		}
	}

}
```

## 1 prepareRefresh()  创建bean容器前的准备工作

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    ....
        //将状态转换为激活状态
        protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();//记录启动的时间	
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) { //日志
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties(); //校验xml配置文件

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
}
```

## 2 	obtainFreshBeanFactory()

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    ....
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
       refreshBeanFactory(); //创建BeanFactory
       return getBeanFactory(); //返回BeanFactory
    }
}
```

### 1 refreshBeanFactory()

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
	protected final void refreshBeanFactory() throws BeansException {
        //应用中可以有多个 BeanFactory
		if (hasBeanFactory()) { //当前ApplicationContext 中是否有BeanFactory
			destroyBeans(); //销毁所有Bean
			closeBeanFactory(); //关闭BeanFactory
		}
		try {
            //使用DefaultListableBeanFactory 是因为它差不多是最底的BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory(); //创建一个新的BeanFactory
			beanFactory.setSerializationId(getId());//用于 BeanFactory的序列化
            //是否允许Bean覆盖，是否允许Bean循环引用
			customizeBeanFactory(beanFactory);
            //加载Bean到 BeanFactory
            //为什么是load BeanDefinitions呢？ ===>Bean可以看作是BeanDefinitions的实例化
			loadBeanDefinitions(beanFactory);
            //持有beanFactory 
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
    
    //什么是Bean覆盖===> 在配置文件中定义bean 使用了相同的id或者name，默认情况allowBeanDefinitionOverriding =null.如果在同一配置文件中重复了
    //会抛错，在不同配置文件中重复了，会覆盖
    //什么是循环引用===>A依赖B，B依赖A : A->B->A；或者A依赖B，B依赖C，C依赖A： A->B->C->A ；但不能在A的构造方法中依赖B，B的构造方法中依赖A，会错误
    //customizeBeanFactory(beanFactory)
    protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		if (this.allowBeanDefinitionOverriding != null) { //Bean覆盖
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.allowCircularReferences != null) {//循环引用
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
}
```

#### 1 loadBeanDefinitions（）

```java

public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
    ....
        //通过XmlBeanDefinitionReader实例 来为BeanFactory创建Bean实例
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
            // 为给定的BeanFactory创建一个XmlBeanDefinitionReader实例对象
            XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

            // Configure the bean definition reader with this context's resource loading environment.
            beanDefinitionReader.setEnvironment(this.getEnvironment());
            beanDefinitionReader.setResourceLoader(this);
            beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

            // Allow a subclass to provide custom initialization of the reader,
            // then proceed with actually loading the bean definitions.
            initBeanDefinitionReader(beanDefinitionReader);
        //重点方法 使用Read对象来加载xml配置
            loadBeanDefinitions(beanDefinitionReader);
        }
    
    //加载xml配置
    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
}
```

- xmlBeanDefinitionReader 的方法 loadBeanDefinitions

```java
@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		for (Resource resource : resources) { //将所有的配置文件转换成一颗DOM树
			count += loadBeanDefinitions(resource);
		}
		return count;
	}
//loadBeanDefinitions(resource);
int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
//找到该方法实现类
//XmlBeanDefinitionReader类
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();

		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
//return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
//int count = registerBeanDefinitions(doc, resource);
//返回从当前配置文件中加载了多少数量的Bean
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
//documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
//类 DefaultBeanDefinitionDocumentReader
@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());//从根节点开始解析
	}
```

- 开始解析

```java
//类 DefaultBeanDefinitionDocumentReader
protected void doRegisterBeanDefinitions(Element root) {
    //为什么定义父节点 因为 <beans>里面也可以嵌套beans
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
```



### 2 BeanDefiniton 类

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    //存放了Bean 的相关信息
	//但该类中并没有定义获取到BeanDefinition实例的方法（getInstance())
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON; 
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
	int ROLE_APPLICATION = 0;
	int ROLE_SUPPORT = 1;
	int ROLE_INFRASTRUCTURE = 2;
	void setParentName(@Nullable String parentName);
	//父Bean的名字 ==> bean继承 ≠ java继承
	@Nullable
	String getParentName();
	void setBeanClassName(@Nullable String beanClassName);
	@Nullable
	String getBeanClassName();
	void setScope(@Nullable String scope);
	@Nullable
	String getScope();
	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();
	void setDependsOn(@Nullable String... dependsOn);
	@Nullable
	String[] getDependsOn();
	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();
	void setPrimary(boolean primary);
	boolean isPrimary();
	void setFactoryBeanName(@Nullable String factoryBeanName);
	@Nullable
	String getFactoryBeanName();
	void setFactoryMethodName(@Nullable String factoryMethodName);
	@Nullable
	String getFactoryMethodName();
	ConstructorArgumentValues getConstructorArgumentValues();
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}
	MutablePropertyValues getPropertyValues();
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}
	void setInitMethodName(@Nullable String initMethodName);
	@Nullable
	String getInitMethodName();
	void setDestroyMethodName(@Nullable String destroyMethodName);
	@Nullable
	String getDestroyMethodName();
	void setRole(int role);
	int getRole();
	void setDescription(@Nullable String description);
	@Nullable
	String getDescription();
	ResolvableType getResolvableType();
	boolean isSingleton();
	boolean isPrototype();
	boolean isAbstract();
	@Nullable
	String getResourceDescription();
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();
}
```







# 2 Spring源码分析 视频



