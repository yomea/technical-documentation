SPI的全称叫做Service Provider Interface，dubbo中大量使用这种服务发现机制，现在我们通过一个简单的例子来了解它是怎么运作的

```
@org.junit.Test
public void test1() {
    ProxyFactory protocol = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
}

```
上面一段代码表示我要获取ProxyFactory类型的服务，getAdaptiveExtension()表示自适应的获取一个服务实现

进入ExtensionLoader.getExtensionLoader方法，查看它做了什么

```
public static <T> ExtensionLoader<T> ExtensionLoader#getExtensionLoader(Class<T> type) {
    //类型不能为空
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
    //类型必须是接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    //检查这个接口是否被@SPI注解修饰
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
    //从缓存中获取已经加载过的ExtensionLoader
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        //没有缓存，重新创建一个
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```
从上面的代码中可以看到，dubbo首先会到缓存中寻找，如果找不到再创建一个，以下是ExtensionLoader的构造方法

```
private ExtensionLoader(Class<?> type) {
    this.type = type;
    //如果ExtensionFactory，那么对象工厂设置为空，如果是其他的类型，就需要加载ExtensionFactory
    //这个objectFactory一看就是用来创建对象的，它在哪里使用了呢？
    //它在ExtensionLoader的injectExtension方法中使用了，这个方法是用来依赖注入的
    //注入的逻辑，我们后面会分析
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```
好了，这么一看，ExtensionLoader的构造器没有做什么比较复杂的事情，除了type不是ExtensionFactory的情况，那我们继续看下getAdaptiveExtension方法吧

```
public T com.alibaba.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension() {
    //从缓存中获取自适应服务实现对象
    Object instance = cachedAdaptiveInstance.get();
    //没有
    if (instance == null) {
        //也不是因为创建服务出错导致的
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                //双重锁定检查
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //确实是没有，那么就创建吧
                        instance = createAdaptiveExtension();
                        //记录到缓存，以便下次直接使用
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
        }
    }

    return (T) instance;
}
```
创建自适应服务实现

```
private T com.alibaba.dubbo.common.extension.ExtensionLoader#createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
                                            |
                                            V
private Class<?> getAdaptiveExtensionClass() {
    //扫描服务实现
    getExtensionClasses();
    //如果在扫描类的阶段已经确定了自适应实例，那么直接返回
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //否则就通过SPI注解上提供的name去创建一个默认的服务实现
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```
扫描服务实现

```
private Map<String, Class<?>> getExtensionClasses() {
    //从缓存中找找看
    Map<String, Class<?>> classes = cachedClasses.get();
    //哦，好像没有
    if (classes == null) {
        synchronized (cachedClasses) {
            //我不信，我再找好看
            classes = cachedClasses.get();
            if (classes == null) {
                //好吧，确实没有，重新加载吧
                classes = loadExtensionClasses();
                //存起来
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

解析SPI注解，加载服务实现

```
private Map<String, Class<?>> loadExtensionClasses() {
    //获取SPI注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        //获取默认的服务实现名
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            //可以指定多个吗？
            String[] names = NAME_SEPARATOR.split(value);
            //毛线，不能指定多个，你给我分割干啥！还报错
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            //缓存一个默认的，用于下次用户获取服务实现时，如果没有指定名字，就用这个默认的
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    //加载META-INF/dubbo/internal/目录下的文件，文件名就是接口名
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    //加载META-INF/dubbo/下的接口服务实现
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    //加载META-INF/services/下的服务实现
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```
文件名用接口命名
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ac400bdff395e57e93392344c5e6fc9f.png)

文件的内容
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9967cdf50e0dc2f8936caaabb94dcb8b.png)

可以看到，接口实现是通过key=服务实现存放的，所以具体的解析就不说了，比较简单，我直接看到类的加载

```
private void com.alibaba.dubbo.common.extension.ExtensionLoader#loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    //如果定义的实现不是指定接口的实现类，报错
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error when load extension class(interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + "is not subtype of interface.");
    }
    //检查是否使用注解@Adaptive指定了默认自适应服务实现
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        //设置为默认的服务实现
        if (cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
            //发现了两个注释了@Adaptive的实现类，直接报错，不允许
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("More than 1 adaptive class found: "
                    + cachedAdaptiveClass.getClass().getName()
                    + ", " + clazz.getClass().getName());
        }
        //是否是装饰类
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        //添加到包装类集合中
        wrappers.add(clazz);
    } else {
        //如果不存在默认构造器，直接抛错
        clazz.getConstructor();
        //这个name就是解析文件时的key
        if (name == null || name.length() == 0) {
            //如果么有指定名字，那么从类的注解上获取
            //读取@Extension定义的名字，如果没有，那么就是实现类的类名并且去掉以接口名作为后缀的后的小写名字
            name = findAnnotationName(clazz);
            //如果没有名字，GG
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }
        //多个名字的情况，通过逗号分割
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            //获取@Activate注解，中文意思就是启用，我们在分析服务导出的时候可以看到监听器，filter都使用了这个注解
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                //缓存
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                //这里只会保存一个
                if (!cachedNames.containsKey(clazz)) {
                    cachedNames.put(clazz, n);
                }
                //查看缓存是否已经有了
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    //不允许重复
                    throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                }
            }
        }
    }
}
```

从上面的代码中可以看到，对于被@Adaptive标注的类和包装类都不会缓存的到extensionClasses集合中


```
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    //这里的cachedAdaptiveClass如果有值的话，一般是因为在加载服务实现的时候发现了被@Adaptive注释的类
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    //像我们所举的例子中ProxyFactory接口是没有的
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```


```
private Class<?> com.alibaba.dubbo.common.extension.ExtensionLoader#createAdaptiveExtensionClass() {
    //这个方法会发生代码的生成
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    //又是通过spi获取编译器，默认是使用javassist进行编译
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

现在我们来分析一下createAdaptiveExtensionClassCode方法

```
private String com.alibaba.dubbo.common.extension.ExtensionLoader#createAdaptiveExtensionClassCode() {
    StringBuilder codeBuilder = new StringBuilder();
    //获取接口所有public的方法
    Method[] methods = type.getMethods();
    boolean hasAdaptiveAnnotation = false;
    //检查是否存在被@Adaptive注解的方法
    for (Method m : methods) {
        if (m.isAnnotationPresent(Adaptive.class)) {
            hasAdaptiveAnnotation = true;
            break;
        }
    }
    // no need to generate adaptive class since there's no adaptive method found.
    //如果一个被@Adaptive注解的方法都没有，那么就报错
    if (!hasAdaptiveAnnotation)
        throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");
    //并且包名
    codeBuilder.append("package ").append(type.getPackage().getName()).append(";");
    //引入ExtensionLoader类
    codeBuilder.append("\nimport ").append(ExtensionLoader.class.getName()).append(";");
    //定义类名 当前接口类型$Adaptive implements 当前接口
    codeBuilder.append("\npublic class ").append(type.getSimpleName()).append("$Adaptive").append(" implements ").append(type.getCanonicalName()).append(" {");

    for (Method method : methods) {
        //获取返回类型
        Class<?> rt = method.getReturnType();
        //参数类型
        Class<?>[] pts = method.getParameterTypes();
        //异常列表
        Class<?>[] ets = method.getExceptionTypes();
        //获取犯法上的@Adaptive注解
        Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
        StringBuilder code = new StringBuilder(512);
        //如果没有被@Adaptive注解，那么方法体直接抛出不支持的操作
        if (adaptiveAnnotation == null) {
            code.append("throw new UnsupportedOperationException(\"method ")
                    .append(method.toString()).append(" of interface ")
                    .append(type.getName()).append(" is not adaptive method!\");");
        } else {
            int urlTypeIndex = -1;
            for (int i = 0; i < pts.length; ++i) {
                //找出URL参数位置
                if (pts[i].equals(URL.class)) {
                    urlTypeIndex = i;
                    break;
                }
            }
            // found parameter in URL type
            if (urlTypeIndex != -1) {
                // Null Point check
                //拼接参数判断逻辑，如果url这个参数为空，直接抛错
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                        urlTypeIndex);
                code.append(s);
                //给局部变量url赋值
                s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
                code.append(s);
            }
            // did not find parameter in URL type
            //如果在参数上没有找到明确的url参数
            else {
                String attribMethod = null;

                // find URL getter method
                //那么从参数对象中寻找是否有获取Url的方法
                LBL_PTS:
                for (int i = 0; i < pts.length; ++i) {
                    Method[] ms = pts[i].getMethods();
                    for (Method m : ms) {
                        String name = m.getName();
                        if ((name.startsWith("get") || name.length() > 3)
                                && Modifier.isPublic(m.getModifiers())
                                && !Modifier.isStatic(m.getModifiers())
                                && m.getParameterTypes().length == 0
                                && m.getReturnType() == URL.class) {
                            urlTypeIndex = i;
                            attribMethod = name;
                            break LBL_PTS;
                        }
                    }
                }
                //如果没有找到可以获取url的参数报错
                if (attribMethod == null) {
                    throw new IllegalStateException("fail to create adaptive class for interface " + type.getName()
                            + ": not found url parameter or url attribute in parameters of method " + method.getName());
                }

                // Null point check
                //判断那个包含url的参数是否为空
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                        urlTypeIndex, pts[urlTypeIndex].getName());
                code.append(s);
                //调用那个包含url的参数的获取url的方法，如果获取不到值，报错
                s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                        urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
                code.append(s);
                //给局部变量url赋值
                s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
                code.append(s);
            }
            //获取自适应注解上配置的值，用于指定在url的参数key
            String[] value = adaptiveAnnotation.value();
            // value is not set, use the value generated from class name as the key
            //如果没有指定，将类名进行驼峰转点，比如UserService => user.service
            if (value.length == 0) {
                char[] charArray = type.getSimpleName().toCharArray();
                StringBuilder sb = new StringBuilder(128);
                for (int i = 0; i < charArray.length; i++) {
                    if (Character.isUpperCase(charArray[i])) {
                        if (i != 0) {
                            sb.append(".");
                        }
                        sb.append(Character.toLowerCase(charArray[i]));
                    } else {
                        sb.append(charArray[i]);
                    }
                }
                value = new String[]{sb.toString()};
            }
            
            boolean hasInvocation = false;
            for (int i = 0; i < pts.length; ++i) {
                //如果参数类型是Invocation，比如invoker的参数就是这个
                if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
                    // Null Point check
                    //invocation参数不能为空
                    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                    code.append(s);
                    //获取将要调用的方法名
                    s = String.format("\nString methodName = arg%d.getMethodName();", i);
                    code.append(s);
                    hasInvocation = true;
                    break;
                }
            }
            //默认的服务实现名，在SPI注解上定义的默认名字
            String defaultExtName = cachedDefaultName;
            String getNameCode = null;
            for (int i = value.length - 1; i >= 0; --i) {
                if (i == value.length - 1) {
                    if (null != defaultExtName) {
                        //判断当前@Adative使用的场景
                        if (!"protocol".equals(value[i]))
                            //用于invoker的
                            if (hasInvocation)
                                //获取extName
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                //添加通用的获取extName的逻辑
                                getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                        else
                            //通过url中获取协议类型的extName
                            getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                    } else {
                        if (!"protocol".equals(value[i]))
                            if (hasInvocation)
                                getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                            else
                                getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                        else
                            getNameCode = "url.getProtocol()";
                    }
                } else {
                    if (!"protocol".equals(value[i]))
                        if (hasInvocation)
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                        else
                            getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                    else
                        getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
                }
            }
            //给局部变量extName赋值
            code.append("\nString extName = ").append(getNameCode).append(";");
            // check extName == null?
            //检查extName，不能为空
            String s = String.format("\nif(extName == null) " +
                            "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                    type.getName(), Arrays.toString(value));
            code.append(s);
            //通过ExtensionLoader获取服务实现
            s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                    type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
            code.append(s);

            // return statement
            //添加返回语句
            if (!rt.equals(void.class)) {
                code.append("\nreturn ");
            }
            //调用服务实现方法
            s = String.format("extension.%s(", method.getName());
            code.append(s);
            for (int i = 0; i < pts.length; i++) {
                if (i != 0)
                    code.append(", ");
                code.append("arg").append(i);
            }
            code.append(");");
        }
        //拼接方法
        codeBuilder.append("\npublic ").append(rt.getCanonicalName()).append(" ").append(method.getName()).append("(");
        for (int i = 0; i < pts.length; i++) {
            if (i > 0) {
                codeBuilder.append(", ");
            }
            //拼接方法参数
            codeBuilder.append(pts[i].getCanonicalName());
            codeBuilder.append(" ");
            codeBuilder.append("arg").append(i);
        }
        //拼接throws异常列表
        codeBuilder.append(")");
        if (ets.length > 0) {
            codeBuilder.append(" throws ");
            for (int i = 0; i < ets.length; i++) {
                if (i > 0) {
                    codeBuilder.append(", ");
                }
                codeBuilder.append(ets[i].getCanonicalName());
            }
        }
        codeBuilder.append(" {");
        //将上面拼接的方法体，拼接进去
        codeBuilder.append(code.toString());
        codeBuilder.append("\n}");
    }
    codeBuilder.append("\n}");
    if (logger.isDebugEnabled()) {
        logger.debug(codeBuilder.toString());
    }
    return codeBuilder.toString();
}
```

对于ProxyFactory接口它为其生成的代码如下

```
package java.com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ProxyFactory$Adaptive implements com.alibaba.dubbo.rpc.ProxyFactory {

    public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws com.alibaba.dubbo.rpc.RpcException {
        //检查url，url不能为空
        if (arg2 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg2;
        //获取需要的接口服务实现是谁，如果没有指定proxy，那么就使用默认值javassist
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        //通过ExtensionLoader获取，所以说为啥默认的ProxyFactory是JavassistProxyFactory
        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        //通过JavassistFactory创建invoker的代码我们在之前的章节已经分析过了
        return extension.getInvoker(arg0, arg1, arg2);
    }

    public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0, boolean arg1) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        //创建代理的方法，我们在第6节中分析参数回调的时候讲过，此处不再赘述
        return extension.getProxy(arg0, arg1);
    }

    //这个方法与上面的方法一样，只是少了一个指定是否代理泛化接口的参数
    public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0);
    }
}
```
ExtensionLoader#getExtension

```
//name：服务实现类名
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    //获取默认的服务实现
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    //从缓存中获取对应的实例
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //创建
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

创建Extension

```
private T com.alibaba.dubbo.common.extension.ExtensionLoader#createExtension(String name) {
    //从缓存中获取对应名字的实现类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        //从缓存中获取
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            //直接调用默认构造器创建
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //对这个创建的实例进行依赖注入
        injectExtension(instance);
        //使用装饰类进行装饰，然后对装饰类进行依赖注入
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```
dubbo的依赖注入

```
private T ExtensionLoader#injectExtension(T instance) {
    try {
        //这个objectFactory是ExtensionFactory类型，它的默认实现类是AdaptiveExtensionFactory
        //因为这个工厂对象的上有一个@Adaptive，所以它会被设置为默认的实现
        if (objectFactory != null) {
            //循环当前服务实现类的包括父类的所有public的set方法
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    /**
                     * Check {@link DisableInject} to see if we need auto injection for this property
                     */
                     //如果当前方法方法上标注了@DisableInject，那么表示这个方法不允许依赖注入
                    if (method.getAnnotation(DisableInject.class) != null) {
                        continue;
                    }
                    //获取参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        //获取属性名
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        //调用对象工厂构建对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            //注入
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```
下面就是我们使用到的默认的对象工厂，可以看到它被@Adaptive注解修饰

```
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        //获取ExtensionFactory的ExtensionLoader
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        //获取所有服务实现，除了被@Adaptive，是装饰类功能的类
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        //除了当前对象工厂外，其他的对象工厂
        //SpringExtensionFactory：用于从spring容器中获取bean，很明显这个name就是对标spring的beanName
        //SpiExtensionFactory：通过SPI的方式获取自适应实现类
        for (ExtensionFactory factory : factories) {
            //循环调用创建
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```
接下来，我们再来研究一下被@Activate注解修饰的服务实现

```
//url：url地址，key：url上某个参数的key，@Activate注解的value就是指定的这个key
public List<T> com.alibaba.dubbo.common.extension.ExtensionLoader#getActivateExtension(URL url, String key) {
    return getActivateExtension(url, key, null);
}
                                                    |
                                                    V
public List<T> getActivateExtension(URL url, String key, String group) {
    //获取参数值
    String value = url.getParameter(key);
    return getActivateExtension(url, value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), group);
}
                                                    |
                                                    V
public List<T> getActivateExtension(URL url, String[] values, String group) {
    List<T> exts = new ArrayList<T>();
    List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
    //不包含 -defaul 这样的属性
    if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
        //加载spi实现类，这个方法在上面已经分析过了
        getExtensionClasses();
        //循环extName -> @Activate
        for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
            String name = entry.getKey();
            Activate activate = entry.getValue();
            //判断注解定义的组是否包好当前传递进来的组名
            if (isMatchGroup(group, activate.group())) {
                //如果extName获取服务实现
                T ext = getExtension(name);
                //这个names是在方法调用的时候传递进来的，它是url是指定的某个参数
                if (!names.contains(name)
                        //不包含 -extName
                        && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
                        //@Activate的value属性上指定的key在url上存在且不为空
                        && isActive(activate, url)) {
                    //将符合条件的服务实现记录
                    exts.add(ext);
                }
            }
        }
        //进行排序
        Collections.sort(exts, ActivateComparator.COMPARATOR);
    }
    List<T> usrs = new ArrayList<T>();
    for (int i = 0; i < names.size(); i++) {
        String name = names.get(i);
        //不是以 - 开始
        if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
                //不包含 -name 属性
                && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
            //default
            if (Constants.DEFAULT_KEY.equals(name)) {
                //记录服务实现
                if (!usrs.isEmpty()) {
                    exts.addAll(0, usrs);
                    usrs.clear();
                }
            } else {
                //获取服务实现
                T ext = getExtension(name);
                usrs.add(ext);
            }
        }
    }
    //记录服务实现
    if (!usrs.isEmpty()) {
        exts.addAll(usrs);
    }
    return exts;
}
```

总结：

通过ExtensionLoader加载META-INF/services/，META-INF/dubbo/，META-INF/dubbo/internal/下的服务。SPI服务实现类必须有@SPI注解，@SPI注解可以指定默认的自适应接口实现，

如果某个实现类上面标注了@Adaptive，那么这个类优先被选中为自适应的默认实现，如果没有找到标注为@Adaptive的实现类，那么会通过javasist进行动态生成一个接口实现，
这个动态生成的实现类会实现被@Adaptive标注的方法，没有被@Adaptive标注的方法默认实现为抛出不支持的异常，被@Adaptive标注的方法，必须要有URL参数或者对应传入的bean有
getUrl方法，否则抛错，@Adaptive注解上需要指定从URl中获取什么样的参数值的参数名，如果url上面没有指定对应参数名的值，那么使用@SPI注解上指定的默认扩展名去获取接口实现。

@Activate：这个注解主要用于标识每个SPI实现类被激活，比如我们使用的Filter，只有被标注了@Activate注解的才能被用于构建过滤器链。

@Extension：这个注解已被废弃，用来指定扩展实现的名字，通常我们在配置SPI实现的时候就会指定SPI扩展名