## package

### package 声明

Java 中的 package 声明和文件全路径一致，使用 `.` 分割文件夹。文件 `Vehicle.java` 的 package 声明：

```java
package cn.demo.basic;
```

其对应的文件目录结构如下：

![image.png](https://lucasji-obsidian-1322082073.cos.ap-shanghai.myqcloud.com/20240625101139.png)

Go 的 package 声明并没有层级结构，package 名称和 go 文件所在的目录名称保持一致。文件 `vehicle.go` 的 package 声明：

```go
package basic
```

![image.png](https://lucasji-obsidian-1322082073.cos.ap-shanghai.myqcloud.com/20240625101422.png)

### package 引用

Java 是面向对象的编程语言，一切引用以类为基本单位。在引用时要么引用一个具体的类

```java
import java.util.List;
```

要么使用 `*` 表示引用该 package 下的所有类

```java
import java.util.*;
```

Go 语言并不是一门面向对象的语言，所以即使 Go 中有和 Java 中 `Class` 类似概念的 `Struct`，`go` 文件和 `Struct` 之间没有必然的联系。

在 Go 语言中，只能 `import` 一个 package，而不是一个 `Struct`（类）

```go
import "github.com\xxx\demo"
```

## 源文件

Java 中一切皆为对象，类是一等公民，一个源文件对应与一个 `Class`。package 只是 `Class` 的目录，package 自身不能拥有函数或者变量。Java 的源文件以 `.java` 为后缀。

Go 的 package 使用更为灵活，对应的是一组变量、函数和类的集合。Go 的源文件以 `.go` 为后缀。

## 函数

> 在包 `mathematics` 下定义“相加”函数，该函数接收两个 `int` 类型的参数，返回两个参数相加的结果，返回值的类型同样为 `int`。

由于 Java 中类才是一等功能，函数和变量只能在类中定义，因此需要先定义一个类 `MyMath`，然后在该类中定义函数 `add`

```java
package cn.demo.mathematics;  
  
public class MyMath {  
  public static int add(int a, int b) {  
    return a + b;  
  }  
}
```

函数的使用

```java
package cn.demo;  
  
import cn.demo.mathematics.MyMath;

public class Main {  
  
  public static void main(String[] args) {  
    int a = 1;  
    int b = 1;  
    int add = MyMath.add(a, b);  
    System.out.println(add);  
  }
```

由于 `add` 函数被定义为静态方法，因此可以直接通过 `MyMath.add` 进行调用。如果 `add` 函数没有定义为静态方法的话，则先需要实例化 `MyMath` 对象 `MyMath myMath = new MyMath()`，再通过 `myMath` 对象调用 `add`：`myMath.add`。

在 [[#源文件]] 一节中，我们知道 Go 的 package 是一组变量、函数和类的集合。因此可以直接在 `go` 文件 `add.go` 中定义 `Add` 函数

```go
package mathematics  
  
func Add(x int, y int) int {  
    return x + y  
}
```

函数的使用

```go
package main  
  
import "fmt"  
import "proj-Go/cn/demo/mathematics"  
  
func main() {  
    var a = 1  
    var b = 2  
    var add = mathematics.Add(a, b)  
    fmt.Println(add)  
}
```

从 Java 和 Go 对于“相加”函数的定义和函数的使用可以看出很多区别。

1. 入参和返回值类型的位置：Java 中类型在变量命前面而 Go 相反。
2. 变量定义：Java 需要确定类型。Go 可以省略类型。
3. 变量赋值：Java 使用 `=`。Go 使用 `:=`。
