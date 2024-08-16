> 本博文来自我在博客园发表的[《使用ASM实现动态代理》](https://www.cnblogs.com/honger/p/6815322.html)

github地址：https://github.com/yomea/ASM
gitee地址：https://gitee.com/yomea/ASM
## 一、实现动态代理，首先得考虑有应该定义哪些类，根据JDK的动态代理思想，那么它就应该有一个生成代理的类
```java
package com.asm_core;

import java.io.PrintWriter;
import java.lang.reflect.Method;
import java.util.List;

import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.tree.ClassNode;
import org.objectweb.asm.tree.FieldInsnNode;
import org.objectweb.asm.tree.FieldNode;
import org.objectweb.asm.tree.InsnList;
import org.objectweb.asm.tree.InsnNode;
import org.objectweb.asm.tree.JumpInsnNode;
import org.objectweb.asm.tree.LabelNode;
import org.objectweb.asm.tree.LdcInsnNode;
import org.objectweb.asm.tree.MethodInsnNode;
import org.objectweb.asm.tree.MethodNode;
import org.objectweb.asm.tree.TypeInsnNode;
import org.objectweb.asm.tree.VarInsnNode;
import org.objectweb.asm.util.TraceClassVisitor;

import com.asm_core.logic.AddLogic;

/**
 * 生成代理对象
 * @author may
 *
 */
public class GenSubProxy {
    
    //逻辑接口
    private AddLogic logic = null;
    //被代理类的所有方法
    private Method[] methods = null;
    
    private String classNamePrefix = null;
    
    private String descInfoPrefix = null;
    
    private String logicPkg = null;
    
    public GenSubProxy(AddLogic logic) {
        
        String logicClassName = AddLogic.class.getName();
        
        this.logicPkg = logicClassName.substring(0, logicClassName.lastIndexOf(AddLogic.class.getSimpleName())).replace(".", "/");
                
        this.logic = logic;
        
    }
    
    
    public Object genSubProxy(Class<?> superClass) {
        
        //获得被代理类的方法集合
        methods = superClass.getDeclaredMethods();
        
        classNamePrefix = superClass.getName().substring(0, superClass.getName().lastIndexOf(superClass.getSimpleName()));
        
        descInfoPrefix = classNamePrefix.replace(".", "/");
        
        Object obj = null;
        try {
            PrintWriter pw = new PrintWriter(System.out, true);
            //生成ClassNode
            ClassNode cn = genClassNode(superClass);
            //ClassWriter.COMPUTE_FRAMES表示让asm自动生成栈图，虽然会慢上二倍。
            ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
            
            TraceClassVisitor tcv = new TraceClassVisitor(cw, pw);
            
            cn.accept(tcv);
            
            //cn.accept(cw);
            
            byte[] b = cw.toByteArray();
            
            MyClassLoader classLoader = new MyClassLoader(b);
            
            Class<?> proxy = classLoader.loadClass(classNamePrefix + "$Proxy");
            
            obj = proxy.newInstance();
            
            Method method = proxy.getDeclaredMethod("setLogic", AddLogic.class);
            
            method.invoke(obj, logic);
            
            Method meth = proxy.getDeclaredMethod("setMethods", Method[].class);
             
            meth.invoke(obj, new Object[] {methods});
             
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        return obj;
    }
    
    public ClassNode genClassNode(Class<?> superClass) {
        
        String superName = superClass.getName().replace(".", "/");
        
        ClassNode cn = new ClassNode(Opcodes.ASM5);
        
    
        //定义代理类的类名
        cn.name = descInfoPrefix + "$Proxy";
        //超类为当前被代理类
        cn.superName = superName;
        
        cn.access = Opcodes.ACC_PUBLIC;
        
        cn.version = Opcodes.V1_8;
        
        cn = proxyMethod(cn);
        
    
        
        
        return cn;
        
    }
    
    @SuppressWarnings("all")
    public ClassNode proxyMethod(ClassNode cn) {
        
        List<MethodNode> list = cn.methods;
        
        List<FieldNode> fields = cn.fields;
        //这里赋初始值必须是Integer, Float, Long, Double 或者 String，null
        fields.add(new FieldNode(Opcodes.ACC_PUBLIC, "logic", "L" + logicPkg + "AddLogic;", null, null));
        //添加methods字段
        fields.add(new FieldNode(Opcodes.ACC_PUBLIC, "methods", "[Ljava/lang/reflect/Method;", null, null));
        //添加方法setLogic，用于设置Logic
        MethodNode mn = new MethodNode(Opcodes.ACC_PUBLIC, "setLogic", "(L" + logicPkg + "AddLogic;)V", null, null);
        //下面的指令相当于this.logic = logic;
        InsnList insnList = mn.instructions;
        
        insnList.add(new VarInsnNode(Opcodes.ALOAD, 0));
        
        insnList.add(new VarInsnNode(Opcodes.ALOAD, 1));
        
        insnList.add(new FieldInsnNode(Opcodes.PUTFIELD, descInfoPrefix + "$Proxy", "logic", "L" + logicPkg + "AddLogic;"));
        
        insnList.add(new InsnNode(Opcodes.RETURN));
        
        mn.maxLocals = 2;//定义最大的本地变量
        
        mn.maxStack = 2;//定义最大的操作数栈
        
        list.add(mn);
        //添加一个setMethods方法，用于设置methods字段
        MethodNode meth = new MethodNode(Opcodes.ACC_PUBLIC, "setMethods", "([Ljava/lang/reflect/Method;)V", null, null);
        //这段指令相当于this.methods = methods;
        InsnList methList = meth.instructions;
        
        methList.add(new VarInsnNode(Opcodes.ALOAD, 0));
        
        methList.add(new VarInsnNode(Opcodes.ALOAD, 1));
        
        methList.add(new FieldInsnNode(Opcodes.PUTFIELD, descInfoPrefix + "$Proxy", "methods", "[Ljava/lang/reflect/Method;"));
        
        methList.add(new InsnNode(Opcodes.RETURN));
        
        meth.maxLocals = 2;
        
        meth.maxStack = 2;
        
        list.add(meth);//
        //添加构造方法
        MethodNode init = new MethodNode(Opcodes.ACC_PUBLIC, "<init>", "()V", null, null);
        //这里是调用父类构造器，相当于super();
        InsnList initList = init.instructions;
        
        initList.add(new VarInsnNode(Opcodes.ALOAD, 0));
        
        initList.add(new MethodInsnNode(Opcodes.INVOKESPECIAL, descInfoPrefix + "SayHello", "<init>", "()V", false));
        
        initList.add(new InsnNode(Opcodes.RETURN));
        
        init.maxLocals = 1;
        
        init.maxStack = 1;
        
        list.add(init);
        
        int count = 0;
        //循环创建需要代理的方法
        for (Method method : methods) {
            
            MethodNode methodNode = new MethodNode(Opcodes.ACC_PUBLIC,  method.getName(), DescInfo.getDescInfo(method), null, null);
            
//            System.err.println(DescInfo.getDescInfo(method));
            
            InsnList il = methodNode.instructions;
            
            //获得参数的类型
            Class<?>[] clazz = method.getParameterTypes();
            
            //计算出参数会占用掉的本地变量表长度，long，double类型占用两个slot
            int len = LocalLen.len(clazz);
            //获得这个方法的参数个数，不包括this
            int size = clazz.length;
            //或的返回值类型
            Class<?> rtClazz = method.getReturnType();
            
            il.clear();
            /**
             * 下面的一大段指令的意思是
             * int index = count;
             * Method method = methods[index];//从methods中获得对应的方法对象
             * Object[] objs = new Object[]{arg0,arg1,arg2....};用来存方法传过来的参数值的
             * try {
             *         logic.addLogic(method, objs);
             * } catch(Exception e) {
             *         e.printStackTrace();
             * }
             */
            il.add(new LdcInsnNode(count));//
            
            il.add(new VarInsnNode(Opcodes.ISTORE, len + 1));
            
            il.add(new VarInsnNode(Opcodes.ALOAD, 0));
            
            il.add(new FieldInsnNode(Opcodes.GETFIELD, descInfoPrefix + "$Proxy", "methods", "[Ljava/lang/reflect/Method;"));
            
            il.add(new VarInsnNode(Opcodes.ILOAD, len + 1));
            
            il.add(new InsnNode(Opcodes.AALOAD));
            
            il.add(new VarInsnNode(Opcodes.ASTORE, len + 2));//将栈顶的method存到局部变量表中
            
            //将参数长度推到栈顶
            il.add(new LdcInsnNode(size));
            
            il.add(new TypeInsnNode(Opcodes.ANEWARRAY, "java/lang/Object"));//new 出一个Object的数组
            
            il.add(new VarInsnNode(Opcodes.ASTORE, len + 3));//将数组存到本地变量表中
            
            int index = 1;
            
            //将参数值全都存到数组中
            for(int i = 0; i < size; i++) {
                
                il.add(new VarInsnNode(Opcodes.ALOAD, len + 3));//将数组推到栈顶
                
                il.add(new LdcInsnNode(i));//下标
                
                int opcode = OpcodeMap.getOpcodes(clazz[i].getName());//获得当前是什么类型的参数，使用什么样类型的指令
                //如果是long，double类型的index加2
                if(opcode == 22 || opcode == 24) {
                    
                    il.add(new VarInsnNode(opcode,index));//将long或者double参数推到栈顶
                    index += 2;
                } else {
                    
                    il.add(new VarInsnNode(opcode, index));//将参数推到栈顶
                    index += 1;
                }
                
                
                if(AutoPKG.auto(clazz[i].getName()) != null) {
                    
                    il.add(new MethodInsnNode(Opcodes.INVOKESTATIC, AutoPKG.auto(clazz[i].getName()), "valueOf", AutoPKG_valueOf.auto(clazz[i].getName()), false));
                    
                }
                
                
                il.add(new InsnNode(Opcodes.AASTORE));//将数据存到数组中
                
            }
            
            
            il.add(new VarInsnNode(Opcodes.ALOAD, 0));//
            
            il.add(new FieldInsnNode(Opcodes.GETFIELD, descInfoPrefix + "$Proxy", "logic", "L" + logicPkg + "AddLogic;"));
            
            il.add(new VarInsnNode(Opcodes.ALOAD, len+2));
            
            il.add(new VarInsnNode(Opcodes.ALOAD, len + 3));
            
            il.add(new MethodInsnNode(Opcodes.INVOKEINTERFACE, "" + logicPkg + "AddLogic", "addLogic", "(Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;", true));
            
            il.add(new TypeInsnNode(Opcodes.CHECKCAST, rtClazz.getName().replace(".", "/")));
            
            LabelNode label = new LabelNode();
            
            il.add(new JumpInsnNode(Opcodes.GOTO, label));
            //由于对栈图还是不太明白是啥意思，如果有知道的麻烦告知我一声
            //il.add(new FrameNode(Opcodes.F_SAME, 0, null, 0, null));
            
            il.add(new VarInsnNode(Opcodes.ASTORE, len + 4));
            
            il.add(new VarInsnNode(Opcodes.ALOAD, len + 4));
            
            il.add(new MethodInsnNode(Opcodes.INVOKEVIRTUAL, "java/lang/Exception", "printStackTrace", "()V", false));
            
            il.add(label);
            
            il.add(new InsnNode(OpcodeRt.getOpcodes(rtClazz.getName())));
            
            methodNode.maxLocals = 5+len;
            
            methodNode.maxStack = 5;
            
            list.add(methodNode);
            
            count ++;
        }
        
        return cn;
    }
    
    

}
```
## 二、有了生成代理的类，那么就还应该有个处理逻辑的接口
```java
package com.asm_core.logic;

import java.lang.reflect.Method;

/**
 * 这个类用来增加方法逻辑,类似于JDK代理的InvocationHandler
 * @author may
 *
 */
public interface AddLogic {
    
    /**
     * 
     * @param method 被代理对象的方法对象
     * @param args 被代理方法的参数
     * @throws Exception
     */
    public Object addLogic(Method method, Object[] args) throws Exception ;

}

```
## 三、如果方法参数中存在基本类型参数，要自动打包成Object[] args，写个基本类型对应包装类助手

```java
package com.asm_core;

import java.util.HashMap;
import java.util.Map;

public class AutoPKG {
    
    public static final Map<String, String> map = new HashMap<>();
    
    static {
        map.put("int", "java/lang/Integer");
        
        map.put("byte", "java/lang/Byte");
        
        map.put("short", "java/lang/Short");
        
        map.put("long", "java/lang/Long");
        
        map.put("boolean", "java/lang/Boolean");
        
        map.put("char", "java/lang/Character");
        
        map.put("float", "java/lang/Float");
        
        map.put("double", "java/lang/Double");
        
    }
    
    public static String auto(String type) {
        
        if(map.containsKey(type)) {
            
            return map.get(type);
        } else {
            
            
            return null;
        }
        
        
    }

}
```

## 四、基本类型对应包装类的valueOf方法的描述符
```java
package com.asm_core;

import java.util.HashMap;
import java.util.Map;

public class AutoPKG_valueOf {
    
    public static final Map<String, String> map = new HashMap<>();
    
    static {
        map.put("int", "(I)Ljava/lang/Integer;");
        
        map.put("byte", "(B)Ljava/lang/Byte;");
        
        map.put("short", "(S)Ljava/lang/Short;");
        
        map.put("long", "(J)Ljava/lang/Long;");
        
        map.put("boolean", "(Z)Ljava/lang/Boolean;");
        
        map.put("char", "(C)Ljava/lang/Character;");
        
        map.put("float", "(F)Ljava/lang/Float;");
        
        map.put("double", "(D)Ljava/lang/Double;");
        
    }
    
    public static String auto(String type) {
        
        if(map.containsKey(type)) {
            
            return map.get(type);
        } else {
            
            return null;
        }
        
        
    }

}

```
## 五、方法描述符生成助手
```java
package com.asm_core;

import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

/**
 * 用于生成字节码描述符的工具类
 * @author may
 *
 */
public class DescInfo {
    
    public static final Map<String, String> map = new HashMap<>();
    
    static {
        
        map.put("int", "I");
        
        map.put("byte", "B");
        
        map.put("short", "S");
        
        map.put("long", "J");
        
        map.put("boolean", "Z");
        
        map.put("char", "C");
        
        map.put("float", "F");
        
        map.put("double", "D");
        
        map.put("void", "V");
        
    }
    
    public static String getDescInfo(Method method) {
        
        Class<?>[]  pt = method.getParameterTypes();
        
        Class<?>  rt = method.getReturnType();
        
        String rtStr = "V";
        
        if(map.containsKey(rt.getName())) {
            
            rtStr = map.get(rt.getName());
            
        } else {
            /**
             * 如果不为空，那么就是数组
             */
            Class<?> clazz= rt.getComponentType();//如果原来的rt是int[][]
            Class<?> oldClazz = clazz;//int[]
            int count = 0;
            if(oldClazz != null) {
                rtStr = "";
                while(clazz != null) {
                    count ++;//2
                    oldClazz = clazz;
                    clazz= clazz.getComponentType();
                }
                for(int i = 0; i < count; i++) {
                    rtStr += "[";
                    
                }
                if(map.containsKey(oldClazz.getName())) {
                    rtStr += map.get(oldClazz.getName());
                } else {
                    rtStr += "L" + oldClazz.getName().replace(".", "/") + ";";
                }
            } else {
                
                rtStr = "L" + rt.getName().replace(".", "/") + ";";
            }
        }
        
        
        String descInfo = "(";
        
        for (Class<?> class1 : pt) {
            String name = class1.getName();
            
            if(map.containsKey(name)) {
                descInfo += map.get(name);
                
            } else {
                if(class1.getComponentType() != null) {
                    descInfo += class1.getName().replace(".", "/");
                    
                } else {
                    
                    descInfo += ("L" + name.replace(".", "/") + ";");
                }
            }
            
        }
        descInfo += ")" + rtStr;
        return descInfo;
        
    }
    

}
```
## 六、用于计算局部变量表的slot长度
```java
package com.asm_core;

/**
 * 计算本地变量表的长度，long，double类型会占用两个slot
 * @author may
 *
 */
public class LocalLen {
    
    
    public static int len(Class<?>[] clzz) {
        
        int count = 0;
        
        for (Class<?> class1 : clzz) {
            
            String str = class1.getName();
            if(str.equals("long") || str.equals("double")) {
                
                count += 2;
                
            } else {
                
                count ++;
            }
        }
        
        return count;
    }
    
}
```
## 七、根据不同类型使用不同字节码指令助手类
```java
package com.asm_core;

import java.util.HashMap;
import java.util.Map;

import org.objectweb.asm.Opcodes;

public class OpcodeMap {

    public static Map<String, Integer> map = new HashMap<>();
    
    static {
        
        
        map.put("int", Opcodes.ILOAD);
        
        map.put("byte", Opcodes.ILOAD);
        
        map.put("short", Opcodes.ILOAD);
        
        map.put("long", Opcodes.LLOAD);
        
        map.put("boolean", Opcodes.ILOAD);
        
        map.put("char", Opcodes.ILOAD);
        
        map.put("float", Opcodes.FLOAD);
        
        map.put("double", Opcodes.DLOAD);
        
        
    }
    
    public static int getOpcodes(String type) {
        
        if(map.containsKey(type)) {
            
            return map.get(type);
            
        } else {
            
            
            return Opcodes.ALOAD;
        }
        
    }
    
}
```
## 八、根据不同的返回类型使用不同字节码指令的助手类
```java
package com.asm_core;

import java.util.HashMap;
import java.util.Map;

import org.objectweb.asm.Opcodes;

public class OpcodeRt {

    public static Map<String, Integer> map = new HashMap<>();
    
    static {
        
        
        map.put("int", Opcodes.IRETURN);
        
        map.put("byte", Opcodes.IRETURN);
        
        map.put("short", Opcodes.IRETURN);
        
        map.put("long", Opcodes.LRETURN);
        
        map.put("boolean", Opcodes.IRETURN);
        
        map.put("char", Opcodes.IRETURN);
        
        map.put("float", Opcodes.FRETURN);
        
        map.put("double", Opcodes.DRETURN);
        
        map.put("void", Opcodes.RETURN);
        
        
    }
    
    public static int getOpcodes(String type) {
        
        if(map.containsKey(type)) {
            
            return map.get(type);
            
        } else {
            
            
            return Opcodes.ARETURN;
        }
        
    }
    
}
```
## 九、自定义类加载器
```java
package com.asm_core;

public class MyClassLoader extends ClassLoader {
    
    private byte[] b = null;
    
    public MyClassLoader(byte[] b) {
        
        this.b = b;
        
    }
    
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        
        return this.defineClass(name, b, 0, b.length);
    }

}
```
## 十、测试类

1】实现逻辑接口
```java
package com.asm_core.test;

import java.lang.reflect.Method;

import com.asm_core.logic.AddLogic;

/**
 * 实现了方法逻辑的类
 * @author may
 *
 */
public class AddLogicImpl implements AddLogic {
    
    private Object sayHello;
    
    public AddLogicImpl(Object sayHello) {
        
        this.sayHello = sayHello;
        
    }

    @Override
    public Object addLogic(Method method, Object[] args) throws Exception {
        
        System.out.println("Hello");
        Object obj = method.invoke(sayHello, args);//我们可以在调用目标方法的周围增加逻辑
        System.out.println("baby");
        return obj;
    }

}
```
2】被代理类
```java
package com.asm_core.test;

import java.util.Date;

/**
 * 需要进行代理的类
 * @author Administrator
 *
 */
public class SayHello {
    
    
    public void sayHello(String str_1, String str_2, int age, String[] args) {
        
        System.out.println(str_1 + " " + str_2 + "嘿嘿：" + age);
        
    }
    
    private String l() {
        System.out.println("private String l() {");
        return "";
        
    }
    
    public int[][] tt(int age, long l, double d) {
        System.out.println("public int[][] tt(int age) {");
        return new int[][]{};
    }
    
    public String[][] hh(long k, double d, Double dd) {
        System.out.println("public String[][] hh(long k, double d, Double dd) {");
        return null;
    }
    
    
    public String[][] hh(short age, byte[] arg, int a, float f, char c, long l, double d, int[][] ii, String str, String[][] ss, Date date) {
        System.out.println("public String[][] hh(short age, byte[] arg, int a, float f, char c, long l, double d, int[][] ii, String str, String[][] ss, Date date) {");
        return null;
    }
    
    /*public String[][] hh(Long l, Double d) {
        System.out.println("public String[][] hh(short age, byte[] arg, double d) {");
        return null;
    }*/


    
    
}
```
3】Test
```java
package com.asm_core.test;

import java.util.Date;

import com.asm_core.GenSubProxy;
import com.asm_core.logic.AddLogic;

public class Test {
    
public static void main(String[] args) {
        
        SayHello sayHello = new SayHello();
        
        AddLogic logic = new AddLogicImpl(sayHello);
        
        GenSubProxy genSubProxy = new GenSubProxy(logic);
        
        Object obj = genSubProxy.genSubProxy(SayHello.class);
        
        SayHello sh = (SayHello) obj;
        
        sh.hh((byte)1, new byte[]{}, 1, 1f, 's',1, 1, new int[][]{{12}}, "", new String[][]{{"sdf","s"}}, new Date());
        
        sh.sayHello("sg", "srt", 234, new String[]{});
        
    }

}
```
## 十一、总结

使用ASM实现动态代理，需要先学懂JVM虚拟机的字节码指令。在自己写字节码指令的时候，如果你忘记了某些代码的指令的实现，别忘记使用JDK的javap -c -v -private **.class。通过javap我们可以解决好多我们曾经感到疑惑的地方，比如为什么匿名内部类使用局部变量时这个局部变量不能变？为什么在字节码层面上不能直接将基本类型复制给Object类型？synchronized在字节码中如何表述的。。。。。。
