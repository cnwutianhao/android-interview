# 操作符 + 号和 StringBuilder 性能差异

在 Java 中，`+` 号操作符用于字符串拼接是非常方便的，但从性能角度来看，当涉及到在一个循环中或者频繁操作时，使用 `+` 号拼接字符串的性能要明显低于使用 `StringBuilder`。

## 使用 `+` 号拼接字符串

当使用 `+` 操作符拼接两个字符串字面量时，Java 编译器会进行常量折叠（constant folding），将这两个字符串字面量直接合并成一个新的字符串字面量。

```java
String a = "a" + "b";
```

这里的 "a" 和 "b" 都是字符串字面量。在编译时直接合并成一个新的字符串字面量 "ab"。

当使用 `+` 操作符拼接非字面量或变量，Java 会在底层会创建一个新的 `StringBuilder` 对象，使用它来拼接字符串，然后将拼接后的结果转换回字符串。

例如，下面的代码：

```java
String b = "bcd";
String a = "a" + b;
```

会被优化成类似这样的代码：

```java
String b = "bcd";
StringBuilder sb = new StringBuilder();
sb.append("a");
sb.append(b);
String a = sb.toString();
```

这是因为 Java 中的 String 类是不可变的，每次使用 + 操作符拼接字符串时，实际上都会创建一个新的 String 对象和 `StringBuilder` 对象。

如果在一个循环或频繁调用的方法中使用 `+` 操作符，会产生大量的中间字符串对象，增加垃圾回收的压力，从而影响性能。


## 使用 `StringBuilder` 拼接字符串

`StringBuilder` 是一个可变的字符串对象，专门设计来构建或操作字符串的可变字符序列，它内部使用一个可变的字符数组来存储字符串，它允许在现有字符数组上进行修改，而不需要每次都创建一个新的字符串对象。

所以我们可以重复使用同一个 `StringBuilder` 对象，反复调用 `append()` 方法来拼接多个字符串，避免了多余对象的创建和销毁，使得 `StringBuilder` 在需要多次修改字符串内容的场景下更为高效。

```java
StringBuilder sb = new StringBuilder();
sb.append("a").append("bdc");
String result = sb.toString();
```

在这个例子中，a 和 bcd 是字面量，在编译时已知的，因此 `StringBuilder` 只创建了一个字符串对象（最终通过 `toString()` 方法返回的结果），而且在整个拼接过程中没有产生多余的临时字符串对象。


## 性能差异

### 临时对象的创建

使用 `+` 操作符时，每次拼接都可能涉及到创建新的字符串对象和复制字符数组，这会带来额外的性能开销，特别是在循环中，会创建大量的临时 `StringBuilder` 和字符串对象。

而 `StringBuilder` 由于其内部数组的可扩展性，可以在初始化时就分配足够的容量，减少多次扩容的性能损耗。可以重复使用同一个 `StringBuilder` 对象，反复调用 `append()` 方法来拼接多个字符串，减少了对象的创建和销毁成本。

### 内存分配

`StringBuilder` 通过预先分配一块内存来存储字符串，避免了 `+` 操作符可能引起的频繁内存分配和复制。

`StringBuilder` 在增长时有更加高效的内存分配和扩容机制，相比于循环中不断创建新的字符串对象，这种方式更节省内存。

### 总结

对于简单的、少量的字符串拼接，使用 `+` 操作符代码更简洁，且由于编译器优化，性能也是可接受的。

但对于大量的字符串拼接，特别是在循环或递归中，使用 `StringBuilder` 可以显著提高性能，因为它避免了每次拼接都创建新字符串对象的开销。

```java
// 较差的性能：在循环中使用字符串拼接
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;
}

// 更好的性能：在循环中使用StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

在第一个例子中，由于 `+` 操作符在循环中被使用，它会创建大量的临时字符串对象，这可能导致性能问题。而在第二个例子中，`StringBuilder` 可以避免频繁创建字符串对象，从而提高性能。