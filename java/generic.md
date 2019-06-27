# 泛型编程（Generic Programming） #

泛型是一种表示类或方法行为对于未知类型的类型约束的方法。

泛型是一种编译器机制，主要是在编译器层面实现的，它对于Java虚拟机（JVM）是透明的，通过该机制可以创建通用的代码并参数化（或模板化）剩余部分，从而以一种一般化方式创建（和使用）一些类型的实体（比如类或接口和方法）。这种编程方法被称为泛型编程。

泛型本质上是提供类型的"类型参数"，它们也被称为参数化类型（parameterized type）或参量多态（parametric polymorphism）。

一般使用大写字母作为类型参数。Java库使用`E`表示集合的元素类型，使用`K`和`V`分别表示字典键和值的类型，使用`T`（或`U`或`S`）表示任何类型。

## 泛型类 ##

```java
public class ClassName<T[, U...]> { ... }
```

泛型类是带有一个或多个类型参数的类。类型参数出现在类名后包括在一对尖括号（`<>`）中。在类定义中可以使用类型参数指定方法的返回类型以及属性和局部变量的类型。

通过替换类型参数为具体类型来实例化泛型类，可以认为泛型类是一般类的工厂。

## 泛型方法 ##

```java
public <T> T methodName(T... a)
{
    // ...
}
```

可以在一般类中通过类型参数定义泛型方法，类型参数出现在修饰符后返回类型前。

调用泛型方法时，将一对尖括号（`<>`）包括的具体类型放在点操作符后泛型方法名前。

## 受限类型参数 ##

受限类型参数用于表示有限制（或范围）的类型参数。

```java
<T extends BoundingType[ & BoundingType1 ...]>
```

表示`T`应该是`BoundingType`的子类型。`T`和`BoundingType`都可以是类或接口。类型参数可以有多个限制，多个`BoundingType`用`&`分隔，至多只有一个限制为类，且作为限制的类必须在限制列表的开始。

## 类型擦除 ##

类型擦除是 Java 编译器用来支持使用泛型的一项技术。使用泛型定义参数化的类时，Java编译器会自动创建一个对应的原始类型（raw type），原始类型的名称为泛型类型的名称（移除类型参数），类型参数被擦出并被它们的第一个限制类型（或没有限制时为`Object`）替换。

为了性能，应该将标记接口（没有方法的接口）放在限制列表的尾部。

调用泛型类的方法时，如果返回值的类型被擦除，那么编译器自动对返回值进行类型转换（cast）。

获取泛型类的属性时，如果属性的类型被擦除，那么编译器自动进行类型转换。

java泛型的编译机制：

+ jvm没有泛型的概念，只有原始类和普通方法
+ 所有的类型参数都被它们的限制替换
+ 生成桥接方法（bridge method）维护多态性
+ 插入类型转换（cast）维护类型安全

对于jvm，参数类型和返回值类型指定一个方法。

## 泛型的限制 ##

1. 不能用原始类型替换类型参数

2. 运行时类型查询仅支持原始类型。JVM中所有对象都是原始类型，所有类型查询都返回原始类型。使用`instanceof`查询对象是否属于泛型类型时编译器会报错，而使用`cast`将变量转换为泛型类型时编译器会发生警告。同样，`getClass()`方法总是返回原始类型。

3. 不能创建泛型类数组。数组记录元素类型，并且添加类型不一致的元素时会抛出`ArrayStoreException`异常。
    * 注意仅仅创建泛型类数组非法，可以声明类型为泛型类数组的变量，但是不能以泛型类数组初始化它。

4. 创建泛型类型的可变参数时编译器会发出警告，可以向调用包含泛型可变参数的方法的方法添加注解（`@SuppressWarnings(“unchecked”)`），或者对包含泛型可变参数的方法添加`@SafeVarargs`注解。

5. 不能实例化类型参数，如`new T(...)`是非法的，类型擦除后成为`new Object()`。

    ```java
    // 变通方法有两种
    // 一种是使用反射（`Class`类本身就是泛型类）
    public static <T> Pair<T> makePair(Class<T> cl)
    {
        try { return new Pair<>(cl.newInstance(), cl.newInstance()); }
        catch (Exception ex) { return null; }
    }

    Pair<String> p = Pair.makePair(String.class);

    // java8之后，可以使用构造器表达式
    // `Supplier<T>`为不带参数返回值为`T`的函数的函数接口
    public static <T> Pair<T> makePair(Supplier<T> constr)
    {
        return new Pair<>(constr.get(), constr.get());
    }

    Pair<String> p = Pair.makePair(String::new);
    ```

6. 不能构建泛型数组。

7. 不能在泛型类的静态属性或静态方法中使用类型参数。

8. 不能抛出或捕获泛型类对象，泛型类的限制不能是`Throwable`，但是在异常声明中可以使用类型参数。

9. 可以使用类型参数跳过编译器对检查型异常的检查。使用泛型类、类型擦除和`@SupressWarnings`注解可以跳过java类型系统的关键部分。

10. 注意类型擦除导致的异常。

## 泛型类的继承规则 ##

泛型不是协变的，无论`S`和`T`是否有关系，`GenericClass<S>`和`GenericClass<T>`之间都没有关系。

## 通配符 ##

通配符使用问号（`?`）表示类型参数，是一种表示未知类型的类型约束的方法，用于解决泛型不具有协变性，和数组行为不一致引起的混乱。

分为三类：

+ 无限定的通配符：`<?>`
+ 子类限定的通配符：`<? extends SubType>`
+ 父类限定的通配符：`<? super SuperType>`

通配符为一个泛型类所指定的类型集合提供了一个有用的类型范围。例如，对泛型类`ArrayList`而言，对于任意引用类型`T`，`ArrayList<? extends SubType>`类型是`ArrayList<T>`的父类，是原始类型`ArrayList`的子类。

父类限定的通配符（`<? super SuperType>`）用于写泛型对象（安全的更改器方法，即set方法），子类限定的通配符（`<? extends SubType>`）用于读取泛型对象（安全的访问器方法，即get方法）。无限制的通配符仅支持get方法，且返回值类型为`Object`，常用于简单的操作。

仍以泛型类`ArrayList`为例，父类限定的通配符的继承关系为：对于任意引用类型`T`，`ArrayList<? super SuperType>`类型是`ArrayList<T>`的父类，是`ArrayList<?>`的子类，而`ArrayList<?>`是原始类型`ArrayList`的子类。

通配符不是类型变量，不能使用`?`作为类型，可以使用助手方法进行通配符捕获。助手方法是一个泛型方法，用类型参数`T`表示通配符`?`。

```java
public static <T> void swapHelper(Pair<T> p) {
    T t = p.getFirst();
    p.setFirst(p.getSecond());
    p.setSecond(t);
}

public static void swap(Pair<?> p) { swapHelper(p); }
```
