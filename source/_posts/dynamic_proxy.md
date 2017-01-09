---
title: 动态代理的实现与案例
date: 2017-01-09 10:00:00
author : 陈帅
tags: Java
---

关于动态代理与其在应用中的实现参考了[楼江航](http://agapple.iteye.com/blog/799827 "Title") 的文章，有关ASM的未及研究，本文主要总结以下内容：

        动态代理的背景

        动态代理的实现方式

        JDK和CGLIB的实现方式

        JDK和CGLIB的优劣对比

        应用中的使用场景

<!-- more -->


**代理**
---------

代理的概念不言而喻，它可以实现过滤请求、插入横切逻辑等功能，应用场景丰富多彩。
代理的方式分为静态代理和动态代理两种。

### **静态代理**

程序运行前代理类的字节码文件依然存在，需要程序员编写源文件。

    缺点：要针对于每一个类撰写代理类；对于单个被代理的类，如果需要被代理的方法很多，又加大了工作量。

    优点：直观，可读性较强。

### **动态代理**

程序运行时动态生成代理类的字节码文件，不需要程序员编写代理类java文件。

    缺点：由于是运行时动态生成的，因此可读性不是很强；而且受限于被代理类自身的属性（jdk需要提供接口，cglib需要是非私有类）。

    优点：代码更加简洁，解放了无谓的编码工作。

----------


**实现方式**
---------

让你来实现一个代理类，需要哪些上下文，有哪些解决方案~正是JDK和CGLIB两种解决方案的映射。

要生产一个类A的代理类，唯一需要了解的就是生成一个什么类，因此就有了基于该类的接口构造一个“A”，或者继承A生产一个“B”。（一开始刚接触jdk动态代理的时候我也很不解为什么要提供接口）。

至于如何生成一个class文件，在既定规则下你当然可以先生产java文件，再编译成字节码文件。而最好的做法是直接操作字节码文件，jdk和cglib生成字节码文件分别用的sun的ProxyGenerator和开源项目ASM字节码框架。

----------


**环境准备**
---------

### **引cglib和asm的jar包**

![环境准备](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_1.jpg)

### **target类**
接口：
```
public interface BookFacade {
    
    public void addBook();  

}
```
实现：
```
public class BookFacadeImpl implements BookFacade, Serializable {
	  
    private static final long serialVersionUID = 1L;

    @Override  
    public void addBook() {  
        System.out.println("增加图书方法。。。");  
    }  
  
}
```

### **代理类**
JDK代理类
```
public class BookFacadeProxy implements InvocationHandler {  
    private Object target;  
    /** 
     * 绑定委托对象并返回一个代理类 
     * @param target 
     * @return 
     */  
    public Object bind(Object target) {  
        this.target = target;  
        //取得代理对象  
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),  
                target.getClass().getInterfaces(), this);   //要绑定接口(这是一个缺陷，cglib弥补了这一缺陷)  
    }  
  
    @Override  
    /** 
     * 调用方法 
     */  
    public Object invoke(Object proxy, Method method, Object[] args)  
            throws Throwable {  
        Object result=null;  
        System.out.println("事物开始");  
        //执行方法  
        result=method.invoke(target, args);  
        System.out.println("事物结束");  
        return result;  
    }  
  
}
```
CGLIB代理类
```
public class BookFacadeProxyCglib implements MethodInterceptor {

    private Object target;

    public Object getInstance(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.target.getClass());
        // 回调方法
        // enhancer.setCallbackType(this.getClass());
        enhancer.setCallback(this);
        // 创建代理对象
        return enhancer.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("before run!");
        proxy.invokeSuper(obj, args);
        System.out.println("after run!");
        return null;
    }

}
```

### **client类**
```
public class TestProxy {  
    
    private static String outputFile = "/Users/apple/Downloads/out";
    //控制cglib生成的class文件持久化到本地硬盘
    static {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, outputFile);
    }
  
    public static void main(String[] args) throws IOException {
        TestProxy testProxy = new TestProxy();
        BookFacadeImpl bookProxy = testProxy.cglibProxyClient();
        bookProxy.addBook();
        // jdkToFile(bookProxy);
        // testProxy.testSerail();
    }  

    public void testSerail() throws IOException {
        BookFacadeImpl obj = new BookFacadeImpl();
        toFile(obj);
    }

    public void jdkProxyClient() {
        BookFacadeProxy proxy = new BookFacadeProxy();  
        BookFacade bookProxy = (BookFacade) proxy.bind(new BookFacadeImpl());  
        bookProxy.addBook();  
    }
    
    public BookFacadeImpl cglibProxyClient() {
        BookFacadeProxyCglib proxy = new BookFacadeProxyCglib();
        BookFacadeImpl bookProxy = (BookFacadeImpl) proxy.getInstance(new BookFacadeImpl());
        return bookProxy;
    }
    //JDK动态代理生成的字节码文件持久化
    public static void jdkToFile(BookFacadeImpl obj) throws IOException {
        Class clazz = obj.getClass();
        String className = clazz.getName();
        byte[] classFile = ProxyGenerator.generateProxyClass(className, BookFacadeImpl.class.getInterfaces());

        FileOutputStream fos = new FileOutputStream(outputFile);
        // ClassReader cr = new ClassReader(className);
        // byte[] bits = cr.b;
        fos.write(classFile);

    }
    //另一种JDK字节码文件持久化方法
    public static void toFile(BookFacadeImpl obj) throws IOException {
        Class clazz = obj.getClass();
        String className = clazz.getName();
        FileOutputStream fos = new FileOutputStream(outputFile);
        ClassReader cr = new ClassReader(className);
        byte[] bits = cr.b;
        fos.write(bits);

    }
}
```

## **JDK动态代理实现原理**
### **反编译后源文件**
```
public final class $Proxy0 extends Proxy
	implements BookFacade, Serializable
{

	private static Method m1;
	private static Method m3;
	private static Method m0;
	private static Method m2;

	public $Proxy0(InvocationHandler invocationhandler)
	{
		super(invocationhandler);
	}

	public final boolean equals(Object obj)
	{
		try
		{
			return ((Boolean)super.h.invoke(this, m1, new Object[] {
				obj
			})).booleanValue();
		}
		catch (Error ) { }
		catch (Throwable throwable)
		{
			throw new UndeclaredThrowableException(throwable);
		}
	}

	public final void addBook()
	{
		try
		{
			super.h.invoke(this, m3, null);
			return;
		}
		catch (Error ) { }
		catch (Throwable throwable)
		{
			throw new UndeclaredThrowableException(throwable);
		}
	}

	public final int hashCode()
	{
		try
		{
			return ((Integer)super.h.invoke(this, m0, null)).intValue();
		}
		catch (Error ) { }
		catch (Throwable throwable)
		{
			throw new UndeclaredThrowableException(throwable);
		}
	}

	public final String toString()
	{
		try
		{
			return (String)super.h.invoke(this, m2, null);
		}
		catch (Error ) { }
		catch (Throwable throwable)
		{
			throw new UndeclaredThrowableException(throwable);
		}
	}

	static 
	{
		try
		{
			m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {
				Class.forName("java.lang.Object")
			});
			m3 = Class.forName("proxy.BookFacade").getMethod("addBook", new Class[0]);
			m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
			m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
		}
		catch (NoSuchMethodException nosuchmethodexception)
		{
			throw new NoSuchMethodError(nosuchmethodexception.getMessage());
		}
		catch (ClassNotFoundException classnotfoundexception)
		{
			throw new NoClassDefFoundError(classnotfoundexception.getMessage());
		}
	}
}
```
### **代理类生成原理**
#### **Proxy的newProxyInstance**
生成代理类，然后把横切逻辑作为构造函数的入参去实例化该代理类。
```
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        if (h == null) {
            throw new NullPointerException();
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class cl = getProxyClass(loader, interfaces);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            Constructor cons = cl.getConstructor(constructorParams);
            return (Object) cons.newInstance(new Object[] { h });
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString());
        } catch (IllegalAccessException e) {
            throw new InternalError(e.toString());
        } catch (InstantiationException e) {
            throw new InternalError(e.toString());
        } catch (InvocationTargetException e) {
            throw new InternalError(e.toString());
        }
    }
```
#### **Proxy的getProxyClass**
相当于CGLIB的Enhance.create，主要做的是key和缓存的管理。
```
public static Class<?> getProxyClass(ClassLoader loader, 
                                         Class<?>... interfaces)
	throws IllegalArgumentException
    {
	// 如果目标类实现的接口数大于65535个则抛出异常（我XX，谁会写这么NB的代码啊？）
	if (interfaces.length > 65535) {
	    throw new IllegalArgumentException("interface limit exceeded");
	}

	//遍历target类的接口

	Class proxyClass = null;

	String[] interfaceNames = new String[interfaces.length];

	Set interfaceSet = new HashSet();	// for detecting duplicates

	for (int i = 0; i < interfaces.length; i++) {
	    
	    String interfaceName = interfaces[i].getName();
	    Class interfaceClass = null;
	    try {
		interfaceClass = Class.forName(interfaceName, false, loader);
	    } catch (ClassNotFoundException e) {
	    }
	    if (interfaceClass != interfaces[i]) {
		throw new IllegalArgumentException(
		    interfaces[i] + " is not visible from class loader");
	    }

		.......

	}

	// 把目标类实现的接口名称作为缓存（Map）中的key，相当于CGLIB的KeyFactory
	Object key = Arrays.asList(interfaceNames);

	Map cache;
	
	synchronized (loaderToCache) {
	    cache = (Map) loaderToCache.get(loader);
	    if (cache == null) {
		cache = new HashMap();
		loaderToCache.put(loader, cache);
	    }

	}

	synchronized (cache) {

	    do {
		// 根据接口的名称从缓存中获取对象
		Object value = cache.get(key);
		if (value instanceof Reference) {
		    proxyClass = (Class) ((Reference) value).get();
		}
		if (proxyClass != null) {
		    // 如果代理对象的Class实例已经存在，则直接返回
		    return proxyClass;
		} else if (value == pendingGenerationMarker) {
		    try {
			cache.wait();
		    } catch (InterruptedException e) {
		    }
		    continue;
		} else {
		    cache.put(key, pendingGenerationMarker);
		    break;
		}
	    } while (true);
	}

	try {
	       .......
		
		// 动态生成代理对象
		byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
		    proxyName, interfaces);
		try {
			// 根据代理类的字节码生成代理类的实例
		    proxyClass = defineClass0(loader, proxyName,
			proxyClassFile, 0, proxyClassFile.length);
		} catch (ClassFormatError e) {
		    throw new IllegalArgumentException(e.toString());
		}
	    }
	    // add to set of all generated proxy classes, for isProxyClass
	    proxyClasses.put(proxyClass, null);

	} 
	.......
	
	return proxyClass;
    }
```
#### **ProxyGenerator的generateProxyClass**
其实这里才是核心，但是关于字节码的处理就不做深究了，倒不如去研究asm框架，性能差距可不小。
```
public static byte[] generateProxyClass(final String name,
                                            Class[] interfaces)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces);
		// 动态生成代理类的字节码
        final byte[] classFile = gen.generateClassFile();

		// 持久化字节码，client中的持久化也就是抄的这里
        if (saveGeneratedFiles) {
            java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        FileOutputStream file =
                            new FileOutputStream(dotToSlash(name) + ".class");
                        file.write(classFile);
                        file.close();
                        return null;
                    } catch (IOException e) {
                        throw new InternalError(
                            "I/O exception saving generated file: " + e);
                    }
                }
            });
        }
        return classFile;
    }
```
### **横切逻辑的invoke逻辑**
JDK的动态代理生成比较简单。细心的话你会发现构建代理类的时候入参只有接口和classloader，因此JDK的动态代理类功能也比较单一。最后看下横切逻辑是何时执行的。

下面代码摘自于代理类中反编译后的addBook方法：

"h"即横切逻辑类，返回h中invoke实现，会发现依赖于源target实例，通过反射来调用target中相应的方法。而cglib在此也做了较大的优化。
```
public final void addBook()
	{
		try
		{
			super.h.invoke(this, m3, null);
			return;
		}
		catch (Error ) { }
		catch (Throwable throwable)
		{
			throw new UndeclaredThrowableException(throwable);
		}
	}
```
## **CGLIB动态代理实现原理**
### **反编译后的代理类**
![CGLIB反编译后的代理类1](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_2.jpg)
![CGLIB反编译后的代理类2](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_3.jpg)
![CGLIB反编译后的代理类3](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_4.jpg)

### **代理类简介**
有个初步的认识，简要介绍这9个标注的地方：

    ①回调函数，由于是static类型，可以在class生成之后实例化前注入，适用于相同的class不同的回调函数的应用场景，在有些screen中回调函数是带有属性的，因此不能生成class的时候就织入。

    ②用户定义的回调函数，织入class中，在诸多回调函数中拥有最高的优先级。

    ③持有代理类和target类;④持有代理类和target类中相应的方法，每个方法均含有两个该组类；⑤⑥代理类和target中对应方法的具体实现。③~⑥都是用来替换反射调用的解决方法中的关键点，下面有详述。

    ⑦非织入的回调函数处理逻辑，结合上面的描述可以总结，回调函数的优先级为：织入的回调函数>ThreadLocal回调函数>static回调函数，结合织入的回调函数的filter机制可以构建出更加强大的处理逻辑~

    ⑧横切逻辑的执行，由此可以看出，默认采用MethodProxy来实行“反射”调用，空间换时间，加速了代理类的执行速度。

    ⑨ThreadLocal或Static回调函数的注入，留给外界的接口，本文的案例也利用了该点功能。

### **代理类的生成流程**
![代理类UML](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_5.jpg)
ClassGenerator类控制生成的流程，具体实现在Enhancer中，并且Enhancer也提供了自定义的key。

### **KeyFactory**
相比较上面JDK的key，CGLIB中Key的生成方式比较独特，而生成Key的生成策略也是由Enhancer来决定，所以最终动态生成的类增加了一个EnhancerKeyFactory。
```
private static final EnhancerKey KEY_FACTORY =
      (EnhancerKey)KeyFactory.create(EnhancerKey.class);
```
### **ClassGenerator**
抽象类，控制class生成的流程，不提供具体的实现。如下流程可以看到，具体的实现都留给了子类Enhancer实现。
```
protected Object create(Object key) {
        try {
        	Class gen = null;
        	
            synchronized (source) {
                ClassLoader loader = getClassLoader();
                Map cache2 = null;
                cache2 = (Map)source.cache.get(loader);
                if (cache2 == null) {
                    cache2 = new HashMap();
                    cache2.put(NAME_KEY, new HashSet());
                    source.cache.put(loader, cache2);
                } else if (useCache) {
                    Reference ref = (Reference)cache2.get(key);
                    gen = (Class) (( ref == null ) ? null : ref.get()); 
                }
                if (gen == null) {
                    Object save = CURRENT.get();
                    CURRENT.set(this);
                    try {
                        this.key = key;
                        
                        if (attemptLoad) {
                            try {
                                gen = loader.loadClass(getClassName());
                            } catch (ClassNotFoundException e) {
                                // ignore
                            }
                        }
                        if (gen == null) {
                            byte[] b = strategy.generate(this);
                            String className = ClassNameReader.getClassName(new ClassReader(b));
                            getClassNameCache(loader).add(className);
                            gen = ReflectUtils.defineClass(className, b, loader);
                        }
                       
                        if (useCache) {
                            cache2.put(key, new WeakReference(gen));
                        }
                        return firstInstance(gen);
                    } finally {
                        CURRENT.set(save);
                    }
                }
            }
            return firstInstance(gen);
        } catch (RuntimeException e) {
            throw e;
        } catch (Error e) {
            throw e;
        } catch (Exception e) {
            throw new CodeGenerationException(e);
        }
    }
```
### **GeneratorStrategy**
正如其名，生成策略，提供给开发者扩展使用。

如下可以看到，类很简洁，默认使用DebuggingClassWriter的字节码处理策略，transform空实现，留做拓展。
```
ublic class DefaultGeneratorStrategy implements GeneratorStrategy {
    public static final DefaultGeneratorStrategy INSTANCE = new DefaultGeneratorStrategy();
    
    public byte[] generate(ClassGenerator cg) throws Exception {
        ClassWriter cw = getClassWriter();
        transform(cg).generateClass(cw);
        return transform(cw.toByteArray());
    }

    protected ClassWriter getClassWriter() throws Exception {
      return new DebuggingClassWriter(ClassWriter.COMPUTE_MAXS);
    }

    protected byte[] transform(byte[] b) throws Exception {
        return b;
    }

    protected ClassGenerator transform(ClassGenerator cg) throws Exception {
        return cg;
    }
}
```
### **CallBackFilter**
当类中每个方法所需要的拦截器都不尽相同的时候，CallBackFilter就派上用场了。
```
int index = filter.accept(actualMethod);
            if (index >= callbackTypes.length) {
                throw new IllegalArgumentException("Callback filter returned an index that is too large: " + index);
            }
            originalModifiers.put(method, new Integer((actualMethod != null) ? actualMethod.getModifiers() : method.getModifiers()));
            indexes.put(method, new Integer(index));
            List group = (List)groups.get(generators[index]);
            if (group == null) {
                groups.put(generators[index], group = new ArrayList(methods.size()));
            }
            group.add(method);
```
根据上面字节码处理逻辑可以总结出：

（1）CallBackTypes可知，仅仅过滤织入class的那些回调函数

（2）根据CallBackFilter的Accept函数来确定代理方法所需要执行的callBack下标。

（3）一个方法只能织入一个CallBack

### **MultiCallBack**
CallBack函数有多个子类，前面介绍的MethodInterCeptor只是其中一个，再介绍一个延迟加载的拦截器，应用面也是比较广的。

LazyLoader意为用到再加载，Bean的部分属性设置为延迟加载在某些应用场景会有出乎意料的效果（案例）

LazyLoader的原理可以从反编译的代码中总结出来：

![lazy loader](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_6.jpg)

    ①分别是回调函数和缓存的结果

    ②每个方法的执行都要被代理到回调方法中

    ③如果缓存中没有，则执行LoadObject方法，并缓存结果。

### **FastClass**
#### **前言**
如果说CGLIB优于JDK的一点在于ASM框架的优势，那么另一个优势就是替换了原有了反射调用的局限性，对比下面JDK和CGLIB在invoke的异同：

CGLIB：
![CGLIB](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_7.jpg)

JDK:
![JDK](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_8.jpg)

分析上面两段代码可以看到，CGLIB采用了Method的invokerSuper方法，走的并不是反射调用这一条道路。

#### **FastClass实现原理**
大致流程如下：

    执行invoke的时候cglib为代理类和taret类分别生成一份FastClass,主要用于Method名称和方法执行的映射

    代理类中每个方法都有MethodProxy，此类封装了代理类方法的上下文信息，用于和FastClass适配而已。

    代理类中的MethodProxy在执行invokerSuper时会依据方法名调用到代理类的FastClass对应的方法去执行。

#### **FastClass的初始化**

下面可以看到，初始化的时候会生成代理类和target类的FastClass
```
private void init()
    {
        /* 
         * Using a volatile invariant allows us to initialize the FastClass and
         * method index pairs atomically.
         * 
         * Double-checked locking is safe with volatile in Java 5.  Before 1.5 this 
         * code could allow fastClassInfo to be instantiated more than once, which
         * appears to be benign.
         */
        if (fastClassInfo == null)
        {
            synchronized (initLock)
            {
                if (fastClassInfo == null)
                {
                    CreateInfo ci = createInfo;

                    FastClassInfo fci = new FastClassInfo();
                    fci.f1 = helper(ci, ci.c1);
                    fci.f2 = helper(ci, ci.c2);
                    fci.i1 = fci.f1.getIndex(sig1);
                    fci.i2 = fci.f2.getIndex(sig2);
                    fastClassInfo = fci;
                    createInfo = null;
                }
            }
        }
    }
```
#### **反编译后的FastClass**
```
public class BookFacadeImpl$$EnhancerByCGLIB$$1adc12da$$FastClassByCGLIB$$e7355050 extends FastClass
{
    
    public int getIndex(Signature signature)
    {
        String s = signature.toString();
        s;
        s.hashCode();
        JVM INSTR lookupswitch 27: default 527
    //                   -2055565910: 236
    //                   -1725733088: 247
    //                   -1696758078: 258
    //                   -1457535688: 269
    //                   -1411812934: 280
    //                   -1026001249: 291
    //                   -894172689: 302
    //                   -623122092: 312
    //                   -419626537: 323
    //                   243996900: 334
    //                   374345669: 345
    //                   560567118: 356
    //                   811063227: 367
    //                   946854621: 377
    //                   973717575: 388
    //                   1116248544: 399
    //                   1221173700: 410
    //                   1230699260: 420
    //                   1365077639: 431
    //                   1517819849: 442
    //                   1584330438: 453
    //                   1826985398: 464
    //                   1902039948: 474
    //                   1913648695: 485
    //                   1972855819: 495
    //                   1984935277: 506
    //                   2011844968: 516;
           goto _L1 _L2 _L3 _L4 _L5 _L6 _L7 _L8 _L9 _L10 _L11 _L12 _L13 _L14 _L15 _L16 _L17 _L18 _L19 _L20 _L21 _L22 _L23 _L24 _L25 _L26 _L27 _L28
    //根据MethodProxy中的上下文信息适配合适的方法
    public Object invoke(int i, Object obj, Object aobj[])
        throws InvocationTargetException
    {
        //拦截器中传入的代理类实例，依次为载体执行相应的方法，如果执行的是target类FastClass的话便会内存泄露
        (BookFacadeImpl$$EnhancerByCGLIB$$1adc12da)obj;
        i;
        JVM INSTR tableswitch 0 26: default 392
    //                   0 128
    //                   1 143
    //                   2 147
    //                   3 159
    //                   4 169
    //                   5 191
    //                   6 201
    //                   7 206
    //                   8 226
    //                   9 237
    //                   10 248
    //                   11 259
    //                   12 272
    //                   13 276
    //                   14 286
    //                   15 291
    //                   16 296
    //                   17 301
    //                   18 316
    //                   19 320
    //                   20 332
    //                   21 336
    //                   22 341
    //                   23 355
    //                   24 378
    //                   25 382
    //                   26 387;
           goto _L1 _L2 _L3 _L4 _L5 _L6 _L7 _L8 _L9 _L10 _L11 _L12 _L13 _L14 _L15 _L16 _L17 _L18 _L19 
_L2:
        "CGLIB$SET_THREAD_CALLBACKS([Lnet/sf/cglib/proxy/Callback;)V";
        equals();
        JVM INSTR ifeq 528;
           goto _L29 _L30
```
#### **invoker vs invokerSuper**
Method中执行的是invokerSuper方法，因此会调用代理类的FastClass中相应方法，进而回调代理类中的方法而进行target中原方法的调用，还记得下面那段代码么，上文提到过~，FastClass正是回调的该方法从而调用了target中的addBook实现。
```
final void CGLIB$addBook$0()
    {
        super.addBook();
    }
```
有了上面的基础，来看看 invoker方法，该方法调用的是target类FastClass中相应的方法，试想如果你真的写成了invoker方法，最后回调的可不是CGLIB$addBook$0这个方法了，而是代理类中的addBook方法了，那么循环调用就开始了，内存会发生泄露。

你或许你已经发现了，CGLIB中很多field都是代理类和Target类双份的，其实target类感觉至此没有用，还会有不可预料到的风险。


### **CGLIB小结**
    动态生成了KEY、FASTCLASS、PROXYCLASS三种类型的CLASS

    拦截器的优先级：可织入>ThreadLocal>static

    ASM框架处理字节码+无反射调用改善了代理的效率

    丰富的扩展点，延伸出来的功能点很多，包括BeanMap、BeanCopy、延迟加载等相关功能。

    Enhancer暴露出来的功能点有：

![Func](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_9.jpg)
    
    设置属性

    生成CLASS或者实例

    对于已经生成的CLASS反射调用注入相应的拦截器
    
## **案例分析**
### **背景**
在学习业务的时候发现了应用中关于异步加载的代码，因此停下来研究了下，作者也沉淀了相关文档：http://agapple.iteye.com/blog/918898。
### **异步并行加载**
#### **业务上下文**
![业务](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_10.jpg)
#### **异步加载模板**
线程池无阻塞处理异步调用的逻辑

![模板](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_11.jpg)

#### **CGLIB代理类的生成逻辑**
![代理类生成](http://7o4zmy.com1.z0.glb.clouddn.com/shuai_12.jpg)

    模板是单例，因此缓存每个class，避免多余的开销

    局部变量不存在线程安全的问题，其实没必要设置在ThreadLocal回调函数中，但是CGLIB只提供了两种后处理方法，这也是没有办法的事情。

    延迟加载拦截器，带返回值的task，超时三秒。

不得不说，线程+CGLIB+延迟加载=异步并行加载，非常巧妙！
