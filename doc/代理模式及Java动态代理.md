#
# 代理模式及Java动态代理


## 关于代理

代理模式使用代理对象完成用户请求，屏蔽用户对真实对象的访问


![markdown](https://github.com/MingjianJin/JavaLearn/blob/master/pic/proxy/%E4%BB%A3%E7%90%86%E7%9A%84%E6%9C%AC%E8%B4%A8.png "markdown")

###
1. Subject(抽象主题)：声明真实主题和代理主题的公共接口
2. Proxy(代理)：包含真实主题的引用，对外提供服务，操作真实主题
3. RealSubject(真实主题)：实际的业务操作

## 代理与动态代理

**代理的本质**

![markdown](https://github.com/MingjianJin/JavaLearn/blob/master/pic/proxy/%E4%BB%A3%E7%90%86%E7%9A%84%E6%9C%AC%E8%B4%A8.png "markdown")


**动态代理的可能**

![markdown](https://github.com/MingjianJin/JavaLearn/blob/master/pic/proxy/java%E4%BB%A3%E7%A0%81%E5%8A%A8%E6%80%81%E5%88%9B%E5%BB%BA%E7%B1%BB%E7%9A%84%E5%8E%9F%E7%90%86%E5%9B%BE.png "markdown")


各种代理实现机制的核心差别：在什么层面上实现了代理
####
* 静态代理，代码层面的设计，实际是理解业务，在业务逻辑层的抽象
* JDK动态代理，语言层面实现，提供了标准的语法特性去实现
* ASM，是虚拟机底层的实现，需要开着了解byte code。在官方文档中，多次强调去查阅虚拟机的说明书，去理解bytecode
* javassist和cglib，应该是框架层面的实现

## 应用场景

1. Mybatis的Mapper接口实现
	* JavassistProxyFactory
2. Dubbo的动态代理
	* JavassitProxyFactory
	* JdkProxyFactory
3. Spring AOP的实现
	* jdk的实现：JdkDynamicAopProxy
	* cglib: ObjenesisCglibAopProxy
4. 全链路框架pinpoint
	* javaagint

## JAVA实现代理

用例场景描述

实际的行为，SayHello方法通过动态代理，执行10遍，并统计耗时

### 静态代理实现

```java
public class Proxy implements Subject{

    private RealSubject realSubject;
    public Proxy(){
        realSubject = new RealSubject();
    }

    private Long getNowMill(){
        Instant now = Instant.now();
        return now.getLong(ChronoField.MILLI_OF_SECOND)+1000*now.getEpochSecond();
    }
    @Override
    public void say(String words) {
        long starter = getNowMill();

        for(int i=0;i<10;i++){
            try {
                TimeUnit.MILLISECONDS.sleep(10L);
            }catch (InterruptedException e){

            }

            realSubject.say(words);
        }
        System.out.println(getNowMill()-starter);
    }

    public static void main(String... args){
        Proxy p = new Proxy();
        p.say("words");
    }
}

// RealSubject.java
public class RealSubject implements Subject{
    public void say(String words) {
        System.out.println(String.format("Hello, %s", words));
    }
}

// Subject.java
public interface Subject {
    void say(String words);
}
```

#### 动态代理

```java
public class ProxyObject implements InvocationHandler {

    private RealSubject realSubject;
    public ProxyObject(){
        realSubject = new RealSubject();
    }

    private Long getNowMill(){
        Instant now = Instant.now();
        return now.getLong(ChronoField.MILLI_OF_SECOND)+1000*now.getEpochSecond();
    }

    public void say(String words) {
        long starter = getNowMill();

        for(int i=0;i<10;i++){
            try {
                TimeUnit.MILLISECONDS.sleep(10L);
            }catch (InterruptedException e){

            }

            realSubject.say(words);
        }
        System.out.println(getNowMill()-starter);
    }

    public static void main(String... args){
        //通过代理类创建接口对象
        Subject subject = (Subject)Proxy.newProxyInstance(Subject.class.getClassLoader(), new Class[]{Subject.class},new ProxyObject());
        subject.say("words");
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Method method1 = this.getClass().getMethod(method.getName(),String.class);
        if(method1 != null){
            method1.invoke(this,args);
        }
        return null;
    }
}

// 实际的业务实现
// 此处不需要实现Subject接口
public class RealSubject {
    public void say(String words) {
        System.out.println(String.format("Hello, %s", words));
    }
}

// 接口定义
public interface Subject {
    void say(String words);
}
```

#### ASM实现
ASM提供了coreAPI和treeAPI两种API，对java类的字节码进行访问和修改，包括类、方法、元数据。但是很困难的地方是，没个接口访问的byte code是不同的，并且开发者有非常了解byte code。确实非常困难。不过效率是高的。


```java
public class ProxyASM {
    public static void main(String... args) {
        String name = "org.freeme.springboot.proxy.dynamic.basic.RealSubject";
        MyClassLoader cl = new MyClassLoader();
        ClassReader cr;
        try {
             cr= new ClassReader(name);

        }catch(IOException io){
            io.printStackTrace();
            return;
        }

        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        cr.accept(new RecordClassVisitor(cw), ClassReader.SKIP_DEBUG);

        Class<?> clazz = cl.defineClassForName(name,cw.toByteArray());
        try {
            clazz.getMethods()[1].invoke(clazz.newInstance(),"my god");
        }catch(InstantiationException e){
            e.printStackTrace();
        }catch(IllegalAccessException e){
            e.printStackTrace();
        }catch (InvocationTargetException e){
            e.printStackTrace();
        }
    }


}

class MyClassLoader extends ClassLoader{
    public MyClassLoader(){
        super(Thread.currentThread().getContextClassLoader());
    }

    public Class<?> defineClassForName(String name,byte[] data){
        return this.defineClass(name,data,0,data.length);
    }
}

class RecordClassVisitor extends ClassVisitor {
    public RecordClassVisitor(ClassVisitor cv) {super(Opcodes.ASM5,cv);}

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions){
        if("say".equals(name))
        {
            return new RecordMethodVisitor(super.visitMethod(access, name, desc, signature, exceptions));
        }
        return super.visitMethod(access, name, desc, signature, exceptions);
    }

}

class RecordMethodVisitor extends MethodVisitor
{
    public RecordMethodVisitor(MethodVisitor mv)
    {
        super(Opcodes.ASM5, mv);
    }

    public void visitCode() {
        /**
        对for循环的添加很困难
         * 方法执行之前植入代码
         * /**0: iconst_0
         1: istore_2
         2: iload_2
         3: bipush        10
         5: if_icmpge     39

         33: iinc          2, 1
         36: goto          2
         */
        super.visitMethodInsn(Opcodes.INVOKESTATIC, "org/freeme/springboot/proxy/dynamic/asm/Recode", "start", "()V", false);

        super.visitInsn(Opcodes.ICONST_0);
        super.visitVarInsn(Opcodes.ISTORE,2);
        super.visitVarInsn(Opcodes.ILOAD,2);
        super.visitIntInsn(Opcodes.BIPUSH,10);
        Label label = new Label();
        super.visitJumpInsn(Opcodes.IF_ICMPGE, label);
        super.visitCode();
        super.visitIincInsn(2,1);
        Label end = new Label();

        super.visitJumpInsn(Opcodes.GOTO,end);
        super.visitMaxs(2,2);

    }

    public void visitInsn(int opcode)
    {
        if(opcode == Opcodes.RETURN)
        {
            /**
             * 方法return之前，植入代码
             */
            super.visitMethodInsn(Opcodes.INVOKESTATIC, "org/freeme/springboot/proxy/dynamic/asm/Recode", "end", "()V", false);
        }
        super.visitInsn(opcode);
    }
}

```

#### Javassit实现
```java
public static long getNowMill(){
    Instant now = Instant.now();
    return now.getLong(ChronoField.MILLI_OF_SECOND)+1000*now.getEpochSecond();
}
public static void main(String... args) throws Exception{
    ClassPool cp = ClassPool.getDefault();
    CtClass cc = cp.get("org.freeme.springboot.proxy.dynamic.basic.RealSubject");
    CtMethod cm = cc.getDeclaredMethod("say");
    cm.addLocalVariable("starter",CtClass.longType);
    cm.insertBefore("System.out.println(\"start\");"
        + "starter = org.freeme.springboot.proxy.dynamic.javassist.ProxyJavassist.getNowMill();");
    cm.insertAfter("System.out.println(org.freeme.springboot.proxy.dynamic.javassist.ProxyJavassist.getNowMill()-starter);");
    Class c = cc.toClass();
    RealSubject realSubject = (RealSubject)c.newInstance();
    realSubject.say("world");
}
```

#### CGLIB
文档较少，但是从使用上，比asm和javaassist更容易上手。
```java
public class ProxyCglib {
    public static void main(String... args) throws Exception{
        new ProxyCglib().pro();
    }

    public static long getNowMill(){
        Instant now = Instant.now();
        return now.getLong(ChronoField.MILLI_OF_SECOND)+1000*now.getEpochSecond();
    }

    public void pro() throws Exception{
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(RealSubject.class);
        enhancer.setCallback((MethodInterceptor)(o, method, objects,proxy) -> {
            long starter = getNowMill();
            for(int i=0;i<10;i++)
                proxy.invokeSuper(o,objects);
            System.out.println(getNowMill()-starter);
            return "hello";
        });
        RealSubject realSubject = (RealSubject) enhancer.create();
        realSubject.say(" world");
    }
}
```

#### 参考资料
1. [Java动态代理机制详解](https://blog.csdn.net/luanlouis/article/details/24589193)
2. [ASM 4.0](https://asm.ow2.io/asm4-guide.pdf)
3. [AOP 的利器：ASM 3.0 介绍](https://www.ibm.com/developerworks/cn/java/j-lo-asm30/index.html)
4. [快速查看Java类字节码](https://www.jianshu.com/p/13dc7b08be3d)
5. [cglib](https://github.com/cglib/cglib/wiki/How-To)
6. [javassist](http://www.javassist.org/)


