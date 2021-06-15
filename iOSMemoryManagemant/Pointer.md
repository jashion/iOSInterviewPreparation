# 指针优化

##### TaggedPointer

苹果为了节省内存和提高执行效率，提出了NSTaggedPointer。TaggedPointer专门用来储存小对象，比如：NSNumber，NSDate等；TaggedPointer不是指针，而是值；TaggedPointer访问速度提高了3倍，创建速度提高了106倍。TaggedPointer的结构如下：

![](/Users/huangjinhua/Downloads/Learn/Github/iOSInterviewPreparation/iOS内存管理/TaggedPointer.png)

TaggedPointer由标志位，值和其他组成。

其中标志位最高位用来判断当前的指针是否是TaggedPointer。最高位为1，则表示当前指针是TaggedPointer；最高位为0，则表示当前指针不是TaggedPointer。标志位剩下的三位则用来表示TaggedPointer当前存的值是什么类型，比如010为NSString类型，011为NSNumber类型等等。

```objectivec
enum
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,
    ...
}
```

接下来的56位用来存储变量的值，但是，如果当前变量的值超过了这个范围，则TaggedPointer会变成普通指针，系统会另外开辟一块内存存储当前变量的值。

至于剩下的4bit，在TaggedPointer储存的是字符串时，表示的是字符串中字符的个数。

（PS：objc4-818.2版本苹果对TaggedPointer指针做了混淆，要得到真正的指针需要和全局变量objc_debug_taggedpointer_obfuscator做异或操作。）

##### NONPointer__ISA

我们知道一个对象必定包含isa指针，实例对象的isa指向类对象，类的isa指向元类对象，元类对象的isa指向根元类。NONPointer__ISA指针，不仅仅保存了isa的地址，还保存了一些其他额外的信息，比如：最高位用来判断当前的指针是否是nonpointer，对象的引用计数，有没有弱引用，有没有关联对象等等。NONPointer__ISA指针也是苹果为了节省内存，提高效率而使用的技术，毕竟64位仅仅用来存储指针有点浪费，把对象的一些关键信息一起储存在指针上，不仅可以节省内存，还可以提高访问速度，比如：获取对象的引用计数等等。

![](/Users/huangjinhua/Downloads/Learn/Github/iOSInterviewPreparation/iOS内存管理/ISA.png)

| nonpointer        | 0表示普通的isa指针，1表示nonpointer_isa                                                                                               |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------- |
| has_assoc         | 表示该对象是否有关联对象，如果没有，则析构时会更快                                                                                                   |
| has_cxx_dtor      | 表示该对象是否有C++或ARC的析构函数，如果没有，则析构时更快                                                                                            |
| shifcls           | 真正的isa指针，指向该对象的类                                                                                                            |
| magic             | 表示对象是否未完成初始化                                                                                                                |
| weakly_referenced | 表示该对象是否有weak对象引用，如果没有，则析构时更快                                                                                                |
| unused            | 表示该对象是否正在析构                                                                                                                 |
| has_sidetable_rc  | 表示该对象的引用计数是否过大无法储存在isa指针，                                                                                                   |
| extra_rc          | 表示该对象的引用计数（objc4-818.2版本做了一些调整，在开启nonpointer时extra_rc和sideTable上的引用计数都是真正的引用计数；没有开启nonpointer时sideTable上的引用计数需要+1返回，具体看源码。） |

---

### 参考：

[深入理解Tagged Pointer](http://blog.devtang.com/2014/05/30/understand-tagged-pointer/)

[Objective-C 引用计数原理 ](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)
