<p>
&nbsp;&nbsp;&nbsp;&nbsp;Boostrap类是tomcat的启动类，它有个main方法，用于启动tomcat，在研究其main方法前，我们先来看看这个类的静态块的处理逻辑
</p>


```
static {
        // Will always be non-null
        //获取用户目录
        String userDir = System.getProperty("user.dir");
        // Home first
        //Globals.CATALINA_HOME_PROP = catalina.home
        //从系统属性中获取tomcat的安装目录，这些属性可以在tomcat启动
        //Windows的bat处理文件或Linux的shell脚本时动态指定，或者调用java程序//时指定
        String home = System.getProperty(Globals.CATALINA_HOME_PROP);
        File homeFile = null;
        //如果不为空，那么就是找到了指定对应属性
        if (home != null) {
            File f = new File(home);
            try {
                //获取规范的文件对象,会解析路径中的./或者../
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                //如果发生错误，使用绝对路径
                homeFile = f.getAbsoluteFile();
            }
        }

        if (homeFile == null) {
            // First fall-back. See if current directory is a bin directory
            // in a normal Tomcat install
            File bootstrapJar = new File(userDir, "bootstrap.jar");
            //将bootstrap.jar所在的路径的上一层设置为tomcat的安装目录
            if (bootstrapJar.exists()) {
                File f = new File(userDir, "..");
                try {
                    homeFile = f.getCanonicalFile();
                } catch (IOException ioe) {
                    homeFile = f.getAbsoluteFile();
                }
            }
        }

        //最后只能降级处理，将当前用户目录作为tomcat的home目录
        if (homeFile == null) {
            // Second fall-back. Use current directory
            File f = new File(userDir);
            try {
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                homeFile = f.getAbsoluteFile();
            }
        }
        
        //将寻找的home目录设置为catalina的home目录
        catalinaHomeFile = homeFile;
        //CATALINA_HOME_PROP catalina.home, 将寻找的tomcat的home目录设置到系统属性中
        System.setProperty(
                Globals.CATALINA_HOME_PROP, catalinaHomeFile.getPath());

        // Then base catalina.base
        String base = System.getProperty(Globals.CATALINA_BASE_PROP);
        //如果没有指定catalina的base目录，那么将catalina的home目录设置为base目录
        if (base == null) {
            catalinaBaseFile = catalinaHomeFile;
        } else {
            //如果指定了catalina的基础路径，那么使用指定的路径
            File baseFile = new File(base);
            try {
                baseFile = baseFile.getCanonicalFile();
            } catch (IOException ioe) {
                baseFile = baseFile.getAbsoluteFile();
            }
            catalinaBaseFile = baseFile;
        }
        //catalinaBaseFile catalina.base
        //记录到系统属性中
        System.setProperty(
                Globals.CATALINA_BASE_PROP, catalinaBaseFile.getPath());
    }
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;Bootstrap的静态块解析好了catalina的home路径和基础路径，这些路径告诉了tomcat去何处寻找配置文件，加载jar包，加载web项目，下面便是Bootstrap的main方法
</p>

```
public static void main(String args[]) {
        //daemon引用的是Bootrap类型的对象
        if (daemon == null) {
            // Don't set daemon until init() has completed
            //如果还未初始化，那么创建Bootstrap对象，并进行初始化
            Bootstrap bootstrap = new Bootstrap();
            try {
            	//初始化bootstrap，
                bootstrap.init();
            } catch (Throwable t) {
                handleThrowable(t);
                t.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            // When running as a service the call to stop will be on a new
            // thread so make sure the correct class loader is used to prevent
            // a range of class not found exceptions.
            //给当前线程绑定tomcat自定义的类加载器
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }

        try {
            //默认的命令为start
            String command = "start";
            if (args.length > 0) {
                //获取命令
                command = args[args.length - 1];
            }
            
            if (command.equals("startd")) {
                //将命令修改为start
                args[args.length - 1] = "start";
                //加载
                daemon.load(args);
                //调用Boostrap的启动方法
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                //关闭tomcat
                daemon.stop();
            } else if (command.equals("start")) {
                //设置等待，用于阻塞，会创建在catalina中创建一个监听在8005的接口的socket，用于监听是否关闭
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
            } else if (command.equals("stop")) {
                //关闭服务
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                //仅仅加载
                daemon.load(args);
                if (null == daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }

    }
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;Boostrap的初始化
</p>

```
 public void init() throws Exception {
    	
    	//初始化URLClassloder类加载器，创建common类加载器，这个common类加载器可以说是tomcat中最顶层类加载器
        initClassLoaders();
        //给当前线程绑定类加载器
        Thread.currentThread().setContextClassLoader(catalinaLoader);
        //使用创建的自定义加载器catalinaLoader加载核心类，util，connector
        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        if (log.isDebugEnabled())
            log.debug("Loading startup class");
        //使用类加载器加载org.apache.catalina.startup.Catalina类，这个类是tomcat启动tomcat的核心类
        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
        //创建catalina对象
        Object startupInstance = startupClass.getConstructor().newInstance();

        // Set the shared extensions class loader
        if (log.isDebugEnabled())
            log.debug("Setting startup class properties");
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
        //给Catalina对象设置了一个shareLoader加载器，因为在配置文件中share加载并没有配置，所以这个加载实际上还是common加载器
        //包括上面的Server加载器
        method.invoke(startupInstance, paramValues);
        
        catalinaDaemon = startupInstance;

    }
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面的init方法创建了tomcat的common类加载器，并使用这个类加载器加载了核心包，util包等类，还创建了启动tomcat的核心对象catalina，并反射调用这个类的setParentClassLoader方法，设置类加载器，现在我们再来看看tomcat的类加载是怎么创建的。
</p>


```
 private void initClassLoaders() {
        try {
        	//从配置文件中获取类加载路径，然后构建成URLClassLoader
        	//这个配置文件通常是tomcat安装目录下conf目录下的catalina.properties文件
            commonLoader = createClassLoader("common", null);
            if( commonLoader == null ) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader=this.getClass().getClassLoader();
            }
            //加载server类加载器，通常为null，为null时返回的还是commonLoader类加载
            catalinaLoader = createClassLoader("server", commonLoader);
            //加载shared类加载器，通常为null，为null时返回的还是commonLoader类加载
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            handleThrowable(t);
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
    }
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bb3ffed292b6385298227977b3f12a68.png)

<p>
&nbsp;&nbsp;&nbsp;&nbsp;从上面的代码中可以看出对于tomcat而言，common，share，server类加载器的父子级关系，common是share与server的父加载器，share与server为同级关系，值得注意的是这三个类加载器都是URLClassLoader,对用配置文件catalina.properties只是指定他们的加载类的路径，比如：

common.loader="\${catalina.base}/lib","\${catalina.base}/lib/*.jar","\${catalina.home}/lib","\${catalina.home}/lib/*.jar"

假设我们tomcat的home路径是D:/tomcat
</p>


```
 private ClassLoader createClassLoader(String name, ClassLoader parent)
        throws Exception {

        String value = CatalinaProperties.getProperty(name + ".loader");
        //如果没有指定对应的类加载路径，那么直接返回父加载器，这里就是common类加载器
        if ((value == null) || (value.equals("")))
            return parent;
        //解析${}占位符，返回catalina.base或者catalina.home的值
        value = replace(value);

        List<Repository> repositories = new ArrayList<>();
        
        //返回类加载路径数组,比如D:/tomcat/lib D:/tomcat/lib/*.jar D:/tomcat/lib D:/tomcat/lib/*.jar
        String[] repositoryPaths = getPaths(value);

        for (String repository : repositoryPaths) {
            // Check for a JAR URL repository
            try {
                @SuppressWarnings("unused")
                URL url = new URL(repository);
                //如果URL类型的地址，那么设置仓库类型为RepositoryType.URL
                repositories.add(
                        new Repository(repository, RepositoryType.URL));
                continue;
            } catch (MalformedURLException e) {
                // Ignore
            }

            // Local repository
            //一系列jar类型的
            if (repository.endsWith("*.jar")) {
                repository = repository.substring
                    (0, repository.length() - "*.jar".length());
                repositories.add(
                        new Repository(repository, RepositoryType.GLOB));
            //指定某个jar类型的路径
            } else if (repository.endsWith(".jar")) {
                repositories.add(
                        new Repository(repository, RepositoryType.JAR));
            } else {
                //普通目录型的
                repositories.add(
                        new Repository(repository, RepositoryType.DIR));
            }
        }
        
        //(*1*)
        return ClassLoaderFactory.createClassLoader(repositories, parent);
    }
    
    
    //(*1*)
     public static ClassLoader createClassLoader(List<Repository> repositories,
                                                final ClassLoader parent)
        throws Exception {

        if (log.isDebugEnabled())
            log.debug("Creating new class loader");

        // Construct the "class path" for this class loader
        //URL集合，用于构建URLClassLoader
        Set<URL> set = new LinkedHashSet<>();

        if (repositories != null) {
            for (Repository repository : repositories)  {
                //如果是URL类型，那么很好办，如果存在!/这样字符串，那么替换成%21/，%21是这个英文叹号的URL编码
                if (repository.getType() == RepositoryType.URL) {
                    URL url = buildClassLoaderUrl(repository.getLocation());
                    if (log.isDebugEnabled())
                        log.debug("  Including URL " + url);
                    set.add(url);
                } else if (repository.getType() == RepositoryType.DIR) {
                    File directory = new File(repository.getLocation());
                    directory = directory.getCanonicalFile();
                    if (!validateFile(directory, RepositoryType.DIR)) {
                        continue;
                    }
                    //将普通目录路径，构建成URL对象
                    URL url = buildClassLoaderUrl(directory);
                    if (log.isDebugEnabled())
                        log.debug("  Including directory " + url);
                    set.add(url);
                } else if (repository.getType() == RepositoryType.JAR) {
                    File file=new File(repository.getLocation());
                    file = file.getCanonicalFile();
                    if (!validateFile(file, RepositoryType.JAR)) {
                        continue;
                    }
                    //构建成URL，替换路径中可能存在的!为%21
                    URL url = buildClassLoaderUrl(file);
                    if (log.isDebugEnabled())
                        log.debug("  Including jar file " + url);
                    set.add(url);
                } else if (repository.getType() == RepositoryType.GLOB) {
                    //如果是GLOB类型的仓库，那么就需要扫描这个路径下所有的jar包，将他们构建成URL加入到URL集合中
                    File directory=new File(repository.getLocation());
                    directory = directory.getCanonicalFile();
                    if (!validateFile(directory, RepositoryType.GLOB)) {
                        continue;
                    }
                    if (log.isDebugEnabled())
                        log.debug("  Including directory glob "
                            + directory.getAbsolutePath());
                    String filenames[] = directory.list();
                    if (filenames == null) {
                        continue;
                    }
                    for (int j = 0; j < filenames.length; j++) {
                        String filename = filenames[j].toLowerCase(Locale.ENGLISH);
                        if (!filename.endsWith(".jar"))
                            continue;
                        File file = new File(directory, filenames[j]);
                        file = file.getCanonicalFile();
                        if (!validateFile(file, RepositoryType.JAR)) {
                            continue;
                        }
                        if (log.isDebugEnabled())
                            log.debug("    Including glob jar file "
                                + file.getAbsolutePath());
                        URL url = buildClassLoaderUrl(file);
                        set.add(url);
                    }
                }
            }
        }

        // Construct the class loader itself
        final URL[] array = set.toArray(new URL[set.size()]);
        if (log.isDebugEnabled())
            for (int i = 0; i < array.length; i++) {
                log.debug("  location " + i + " is " + array[i]);
            }
        //使用准备好的URL集合构建URLClassLoader
        return AccessController.doPrivileged(
                new PrivilegedAction<URLClassLoader>() {
                    @Override
                    public URLClassLoader run() {
                        if (parent == null)
                            return new URLClassLoader(array);
                        else
                            return new URLClassLoader(array, parent);
                    }
                });
    }

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;Boostrap的load方法
</p>


```
 private void load(String[] arguments)
        throws Exception {

        // Call the load() method
        //将要被调用的方法名
        String methodName = "load";
        Object param[];
        Class<?> paramTypes[];
        if (arguments==null || arguments.length==0) {
            paramTypes = null;
            param = null;
        } else {
            //如果参数，那么将参数值设置进去
            paramTypes = new Class[1];
            paramTypes[0] = arguments.getClass();
            param = new Object[1];
            param[0] = arguments;
        }
        Method method =
            catalinaDaemon.getClass().getMethod(methodName, paramTypes);
        if (log.isDebugEnabled())
            log.debug("Calling startup class " + method);
        //调用catalina对象的load方法，这个我们到第二节去分析
        method.invoke(catalinaDaemon, param);

    }
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;Boostrap调用load方法之后，紧接着调用的方法是start方法，不出意外，start方法也是调用的catalina的start方法
</p>


```
public void start()
        throws Exception {
        if( catalinaDaemon==null ) init();

        Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
        method.invoke(catalinaDaemon, (Object [])null);

    }
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;经过以上分析，发现Boostrap的加载和启动都是交由catalina实例来执行，为什么要这么做呢？而且还是通过反射的方式，这里样的好处就是Boostrap与具体启动tomcat的实例解耦，它是一个门面，复杂的加载启动逻辑都交给了catalina，一下是Bootstrap的类图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/484f8174aa63c98da5cf70905ae7d5dd.png)

图中有这么几个属性：

- daemon：这是Bootstrap中一个静态属性，其引用的类型就是Bootstrap类型的变量
- catalinaBaseFile与catalinaHomeFile：在没有在系统属性中指定位置时，他们的值通常是相同的，一般指向tomcat的安装目录
- PATH_PATTERN：用于匹配解析catalina.properties指定的common，share，server加载器路径
- catalinaDaemon：用于引用catalina对象，这个对象用于真正启动，初始化，结束tomcat服务
- commonLoader：通用类加载器
- catalinaLoader：catalina类类加载器
- sharedLoader：共享类加载

