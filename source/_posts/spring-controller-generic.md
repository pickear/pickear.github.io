title: Spring 控制器泛型化参数
author: Dylan
date: 2018-10-26 16:47:25
tags:
  - spring
  - java
categories:
  - 编程语言
---
### 泛型的作用

&emsp;&emsp;在JDK1.5开始JAVA就引入了泛型，泛型的引入，能够减少我们在运行时的很多错误，让错误在编译期间就被发现。例如定义一个列表:List list = new ArrayList(); 我们可以往这个列表添加数值类型，同时也可以往列表添加字符串，在编译期间编译器不会给我们错误的提示。但在运行时，程序就会出错。当我们使用泛型定义列表:List<String> list = new ArrayList<>()时，我们只能往列表添加字符串，如果想往列表添加数值，编译器就会马上告诉你不允许。泛型只在编译期起作用，编译后泛型的信息就会被擦除。
  
### 参数泛型化

&emsp;&emsp;下面先给一段代码。
我们先定义一个用户类User:


```java

public class User{
	private String username;
	private String nickName;
	private String password;
	private String addr;
	
	##getter and setter

}
```

如果我们想通过用户名或者昵称来查询用户信息，在Controller里我们可以这样做:

```java
@RequestMapping(value = "/query",method = GET)
public String query(User user, Model model){
	List<User> users = userService.query(user);
	model.addAttribute("users",users);
	return "users";
}
```

我们知道，当我们在发起HTTP请求的时候，如果我们传递username或者nickName参数，Spring会自动帮我们将值绑定到user中，我们在query方法中直接使用就行，这个很容易。
接下来，我们想做一下分页功能，几乎每一个web站点都离不开分页。我们想查某一个国家的用户，并分页显示。要完成分页功能，首先我们定义一个分页类型.

```java
public class Page<P> {

    private final static int DEFAULT_SIZE = 15;
    private int page = 1;
    private int size = DEFAULT_SIZE;
    private long totalPages=1;
    private long totalCount = 0;
    private P param;
    
	##getter and setter
}
```

在分页对象里，我们定义了一个泛型P，因为我们想通过P来接收前端传过来的查询参数。并且，我们并不知道这个P的具体类型是什么。可能不同的业务逻辑里，这个参数类型实体是不同的。
很显示，我们分页查询，希望这样做:

```java
@RequestMapping(value = "/queryPage",method = GET)
public String queryPage(Page<User> page, Model model){
	List<User> users = userService.queryPage(page);
	model.addAttribute("page",page);
	return "users";
}
```

我们希望，在发起http请求时带上page=1,size=15,param.addr='中国'，服务端会返回15个中国用户的信息。但是结果却让我们大跌眼镜，事情并没有往我们想的方向发展，addr没能正确绑定到User里。
为什么呢?最主要原因是因为泛型运行时擦除的原因。在运行时，Spring无法知道Page的泛型类型是什么，只能用Object来初始化，而要将addr绑定到Object里，显然不行。

### 如何解决

&emsp;&emsp;很多时候，参数的泛型化是不可避免的，泛型能让代码更加优雅。当然，上面的例子我们可以避开泛型化处理分页也是很简单的。我们如何可以将数据绑定到泛型里呢？首先我们来了解一下Spring的数据绑定逻辑。
下面是Spring的ModelAttributeMethodProcessor类的源码：

```java
public class ModelAttributeMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {

	protected final Log logger = LogFactory.getLog(getClass());

	private final boolean annotationNotRequired;


	/**
	 * Class constructor.
	 * @param annotationNotRequired if "true", non-simple method arguments and
	 * return values are considered model attributes with or without a
	 * {@code @ModelAttribute} annotation.
	 */
	public ModelAttributeMethodProcessor(boolean annotationNotRequired) {
		this.annotationNotRequired = annotationNotRequired;
	}


	/**
	 * Returns {@code true} if the parameter is annotated with
	 * {@link ModelAttribute} or, if in default resolution mode, for any
	 * method parameter that is not a simple type.
	 */
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
	}

	/**
	 * Resolve the argument from the model or if not found instantiate it with
	 * its default if it is available. The model attribute is then populated
	 * with request values via data binding and optionally validated
	 * if {@code @java.validation.Valid} is present on the argument.
	 * @throws BindException if data binding and validation result in an error
	 * and the next method parameter is not of type {@link Errors}.
	 * @throws Exception if WebDataBinder initialization fails.
	 */
	@Override
	public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		String name = ModelFactory.getNameForParameter(parameter);
		Object attribute = (mavContainer.containsAttribute(name) ? mavContainer.getModel().get(name) :
				createAttribute(name, parameter, binderFactory, webRequest));

		if (!mavContainer.isBindingDisabled(name)) {
			ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
			if (ann != null && !ann.binding()) {
				mavContainer.setBindingDisabled(name);
			}
		}

		WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
		if (binder.getTarget() != null) {
			if (!mavContainer.isBindingDisabled(name)) {
				bindRequestParameters(binder, webRequest);
			}
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
				throw new BindException(binder.getBindingResult());
			}
		}

		// Add resolved attribute and BindingResult at the end of the model
		Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
		mavContainer.removeAttributes(bindingResultModel);
		mavContainer.addAllAttributes(bindingResultModel);

		return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
	}

	/**
	 * Extension point to create the model attribute if not found in the model.
	 * The default implementation uses the default constructor.
	 * @param attributeName the name of the attribute (never {@code null})
	 * @param methodParam the method parameter
	 * @param binderFactory for creating WebDataBinder instance
	 * @param request the current request
	 * @return the created model attribute (never {@code null})
	 */
	protected Object createAttribute(String attributeName, MethodParameter methodParam,
			WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {

		return BeanUtils.instantiateClass(methodParam.getParameterType());
	}

	/**
	 * Extension point to bind the request to the target object.
	 * @param binder the data binder instance to use for the binding
	 * @param request the current request
	 */
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		((WebRequestDataBinder) binder).bind(request);
	}

	/**
	 * Validate the model attribute if applicable.
	 * <p>The default implementation checks for {@code @javax.validation.Valid},
	 * Spring's {@link org.springframework.validation.annotation.Validated},
	 * and custom annotations whose name starts with "Valid".
	 * @param binder the DataBinder to be used
	 * @param methodParam the method parameter
	 */
	protected void validateIfApplicable(WebDataBinder binder, MethodParameter methodParam) {
		Annotation[] annotations = methodParam.getParameterAnnotations();
		for (Annotation ann : annotations) {
			Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
			if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
				Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
				Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
				binder.validate(validationHints);
				break;
			}
		}
	}

	/**
	 * Whether to raise a fatal bind exception on validation errors.
	 * @param binder the data binder used to perform data binding
	 * @param methodParam the method argument
	 * @return {@code true} if the next method argument is not of type {@link Errors}
	 */
	protected boolean isBindExceptionRequired(WebDataBinder binder, MethodParameter methodParam) {
		int i = methodParam.getParameterIndex();
		Class<?>[] paramTypes = methodParam.getMethod().getParameterTypes();
		boolean hasBindingResult = (paramTypes.length > (i + 1) && Errors.class.isAssignableFrom(paramTypes[i + 1]));
		return !hasBindingResult;
	}

	/**
	 * Return {@code true} if there is a method-level {@code @ModelAttribute}
	 * or, in default resolution mode, for any return value type that is not
	 * a simple type.
	 */
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (returnType.hasMethodAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(returnType.getParameterType())));
	}

	/**
	 * Add non-null return values to the {@link ModelAndViewContainer}.
	 */
	@Override
	public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		if (returnValue != null) {
			String name = ModelFactory.getNameForReturnValue(returnValue, returnType);
			mavContainer.addAttribute(name, returnValue);
		}
	}

}
```

这里不具体分析源码，我们知道，该Processor通过supportsParameter判断是否要做数据绑定，通过resolveArgument来处理数据绑定。我们可以自定义一个Processor,来影响Spring的数据绑定逻辑。

首先，我们定义一个注解，该注解用于指定泛型的具体类型:

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface PageParam {

    Class paramClazz() default HashMap.class;
}
```

我们再定义一个Processor，用于实现数据绑定:

```java
/**
 * @author dylan
 * @date 2018/10/26
 */
public class QueryPageArgumentResolver extends ModelAttributeMethodProcessor {

    /**
     * Class constructor.
     *
     * @param annotationNotRequired if "true", non-simple method arguments and
     *                              return values are considered model attributes with or without a
     *                              {@code @ModelAttribute} annotation.
     */
    public QueryPageArgumentResolver(boolean annotationNotRequired) {
        super(annotationNotRequired);
    }

    public QueryPageArgumentResolver(){
        super(false);
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
	    //如果参数打上了PageParam注解，就用此Processor处理
        return parameter.hasParameterAnnotation(PageParam.class);
    }

    @Override
    protected Object createAttribute(String attributeName, MethodParameter methodParam, WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {
		
		//创建对象，实际上是Page对象
        Object attribute = BeanUtils.instantiateClass(methodParam.getParameterType());
		//获取PageParam注解
        PageParam pageParam = methodParam.getParameterAnnotation(PageParam.class);
		//获取PageParam注解指定的paramClazz。
        Class clazz = pageParam.paramClazz();
		//通过反射实例化注解的类型，并set到Page对象里。该反射使用了joor反射库。
        ((Page)attribute).setParam(Reflect.on(clazz).create().get());
        return attribute;
    }

    @Override
    protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
        ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
        ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
        servletBinder.bind(servletRequest);
    }
```

要让该Processor起作用，需要添加到Spring中，这里用了Spring boot，实现如下:

```java
@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        super.addArgumentResolvers(argumentResolvers);
        argumentResolvers.add(new QueryPageArgumentResolver());
    }

}
```

此时，Controller就可以这样定义了:

```java
@RequestMapping(value = "/queryPage",method = GET)
public String queryPage(@PageParam(paramClazz = User.class) Page<User> page, Model model){
	List<User> users = userService.queryPage(page);
	model.addAttribute("page",page);
	return "users";
}
```

当我们发起请求时，主要参数带有param.addr，地址就可以正确绑定到泛型里的User对象下。





