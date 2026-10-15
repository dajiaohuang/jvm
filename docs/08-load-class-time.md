# 类加载的时机

## 类的生命周期

类从被加载到虚拟机内存开始，到卸载出内存为止，它的整个生命周期包括以下 7 个阶段：

- 加载
- 验证
- 准备
- 解析
- 初始化
- 使用
- 卸载

验证、准备、解析 3 个阶段统称为连接。

![Load Class](https://cdn-doocs.oss-cn-shenzhen.aliyuncs.com/gh/doocs/jvm@main/images/loadclass.png)

加载、验证、准备、初始化和卸载这 5 个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班地开始（注意是“开始”，而不是“进行”或“完成”），而解析阶段则不一定：它在某些情况下可以在初始化后再开始，这是为了支持 Java 语言的运行时绑定。

## 类加载过程中“初始化”开始的时机

Java 虚拟机规范没有强制约束类加载过程的第一阶段（即：加载）什么时候开始，但对于“初始化”阶段有着严格的规定。以下按 [Java SE 8 JVM 规范 §5.5](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.5) 列出 6 类初始化触发条件：

- 执行 `new` 时初始化所引用的类；执行 `getstatic`、`putstatic` 或 `invokestatic` 时，初始化声明解析后字段或方法的类或接口。Java 源码读取常量变量通常不产生 `getstatic`，不会因此触发初始化。
- 调用类库中某些反射方法时触发初始化；不是所有反射操作都会初始化类或接口。
- 初始化一个类时，如果其父类还没有初始化，则需要先初始化父类。
- 初始化一个类时，它直接或间接实现的、声明了默认方法的接口也需要先初始化；不是所有父接口都会因此初始化。
- 虚拟机启动时，用于需要指定一个包含 `main()` 方法的主类，虚拟机会先初始化这个主类。
- 首次调用由方法句柄解析产生、引用种类为 `REF_getStatic`、`REF_putStatic`、`REF_invokeStatic` 或 `REF_newInvokeSpecial` 的方法句柄时，初始化声明相应成员的类或接口。此规则不限于 JDK 1.7 的动态语言支持。

上述条件需要与规范规定的递归初始化规则一起理解。下文用**主动引用**与**被动引用**区分是否触发相关类型的初始化。

## 被动引用演示 Demo

### Demo1

```java
/**
 * 被动引用 Demo1:
 * 通过子类引用父类的静态字段，不会导致子类初始化。
 *
 * @author ylb
 *
 */
class SuperClass {
    static {
        System.out.println("SuperClass init!");
    }

    public static int value = 123;
}

class SubClass extends SuperClass {
    static {
        System.out.println("SubClass init!");
    }
}

public class NotInitialization {

    public static void main(String[] args) {
        System.out.println(SubClass.value);
        // SuperClass init!
    }

}
```

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。

### Demo2

```java
/**
 * 被动引用 Demo2:
 * 通过数组定义来引用类，不会触发此类的初始化。
 *
 * @author ylb
 *
 */

public class NotInitialization {

    public static void main(String[] args) {
        SuperClass[] superClasses = new SuperClass[10];
    }

}
```

这段代码会创建数组对象和对应的数组类，但不会因此初始化数组元素类型 `SuperClass`。数组类由虚拟机创建，直接继承自 `java.lang.Object`；数组类的创建不等于对元素类型进行初始化。

### Demo3

```java
/**
 * 被动引用 Demo3:
 * 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。
 *
 * @author ylb
 *
 */
class ConstClass {
    static {
        System.out.println("ConstClass init!");
    }

    public static final String HELLO_BINGO = "Hello Bingo";

}

public class NotInitialization {

    public static void main(String[] args) {
        System.out.println(ConstClass.HELLO_BINGO);
    }

}
```

编译通过之后，常量存储到 NotInitialization 类的常量池中，NotInitialization 的 Class 文件中并没有 ConstClass 类的符号引用入口，这两个类在编译成 Class 之后就没有任何联系了。

## 接口的加载过程

接口加载过程与类加载过程稍有不同。

类初始化前，其父类及直接或间接实现的、声明了默认方法的接口必须先初始化。接口初始化本身不要求先初始化其父接口，不能简单概括为“只有真正用到父接口字段才初始化”；具体条件与顺序见上述 JVM 规范 §5.5。
