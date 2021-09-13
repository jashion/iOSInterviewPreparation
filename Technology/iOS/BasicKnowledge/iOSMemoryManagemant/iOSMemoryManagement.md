# iOS内存管理面试题

## 1.内存管理

#### 1.1讲一下iOS内存管理的理解？

参考：

- iOS系统把App的运行内存分成栈区，堆区，全局静态区等等，其中除了堆区由程序员自己管理，剩下的内存都是由系统分配和释放。
- iOS使用引用计数的方式管理由程序员分配的内存，即对象刚创建时rc = 1，被别的对象强引用一次rc += 1，被持有的对象释放一次rc -= 1，直到对象的rc = 0，则该对象会被释放和回收所分配的内存空间。
- ARC是在MRC的基础上由编译器自动键入retain/release/autorelease方法，同时运行时库辅助完成，在降低程序崩溃，内存泄漏等风险的同时，很大程度减少了开发程序的工作量。
- iOS系统在32位系统上，对象的引用计数都是放在关联的全局哈希表sideTable里面，对象的引用计数的增删都需要访问全局的哈希表进行处理，对于经常进行引用计数操作的对象这是不少的消耗，所以，苹果在64位系统上采取了taggedPointer和nonpointer_isa，这两种方法来提高对象的存取效率。taggedPointer是苹果针对NSNumber这种小对象进行的优化，直接将对象的值存放在变量指针分配的内存上，也就是说，变量内存地址放的不是指针，而是真正的值，所以，该值是在栈上分配的而不是堆上，省去了为对象分配内存，计算引用计数等各种操作，提高了运行效率（注意：因为在栈上分配，更像是值类型，由系统释放，不需要引用计数）；而nonpointer_isa则会在isa指针上存放除了真正的isa内存地址之外，还存放了nonpointer的标志，是否有c++的析构函数，是否有关联对象，是否已经初始化，是否正在被释放，对象的引用计数等等一些和对象相关的信息。对于对象的引用计数来说，可以直接在指针上操作，不用访问全局的哈希表，只有当指针不够存储对象引用计数才会去操作全局的哈希表，提高了运行效率。（还可以进一步聊聊编译器怎么知道什么时候插入retain/release方法？rc的存储和访问？等等）

#### 1.2App运行内存分区？

参考：

栈区：由编译器分配和释放，一般存放函数地址，入参，局部变量等等，类似于数据结构的栈。

堆区：由程序员分配和释放，比如：iOS引用类型分配在堆区，值类型分配在栈区，若造成内存泄漏的对象，则会在程序退出时由系统回收，类似于数据结构的链表。

全局/静态区：存放全局变量和static修饰的静态变量，分初始化区和未初始化区，程序退出时由系统释放。

文字常量区：存放字符串常量和const修饰的常量，程序退出时由系统释放。

代码区：存放程序的二进制代码。

#### 1.3内存管理默认的关键字是哪几个？

参考：

引用类型：atomic，strong（MRC：retain），readWrite

值类型：atomic，assign，readWrite

#### 1.4BAD_ACCESS在什么情况会出现？

参考：

iOS应用在报BAD_ACCESS的意思是因为程序访问了不应该访问的内存：已释放的内存，系统保留不能访问的内存。

#### 1.5如何检测内存泄漏？

参考：

Xcode自带的工具：

Memory Leaks（检测哪些没有被引用并且不能不再次使用应该释放的内存）

Alloctions（检测哪些虽然有被引用但是不会再使用的内存）

Analyse（静态分析：逻辑错误，内存管理错误，声明错误，Api调用错误）

Debug Memory Graph（各个对象引用关系的图形化）

第三方工具：

[MLeaksFinder](http://wereadteam.github.io/2016/02/22/MLeaksFinder/)

[FBRetainCycleDetector](https://www.jianshu.com/p/bdce04214cf3)

[PLeakSniffer](https://link.jianshu.com/?t=https://github.com/music4kid/PLeakSniffer)

#### 1.6说一下悬垂指针和野指针

参考：

悬垂指针：指向已释放的内存，会造成BAD_ACCESS。

野指针：没有初始化的指针，指向未知的内存。

## 2.ARC

#### 2.1说一下ARC的运行机制？

参考：

ARC的核心是引用计数，即对象刚创建时rc = 1，被别的对象强引用一次rc += 1，被持有的对象释放一次rc -= 1，直到对象的rc = 0，则该对象会被释放和回收所分配的内存空间。ARC是在MRC的基础上由编译器自动键入retain/release/autorelease方法，同时运行时库辅助完成，在降低程序崩溃，内存泄漏等风险的同时，很大程度减少了开发程序的工作量。在编译期间ARC采用的是更底层C的接口实现retain/release/autorelease，这样性能更好，并且进行了一些优化操作，比如省去同一上下文成对多余的retain/release，减少函数调用，提高性能等等。

#### 2.2ARC遵循的原则？

- 不能使用retain, release,retainCount和autorelase等函数。

  原因：因为ARC有效时，内存管理的工作交给了编译器，编译器在键入这些函数的时候调用的是更底层C实现的借口，性能更高，所以没必要多此一举，并且编译器也明确禁止调用这些函数，一旦在ARC有效时调用这些函数，会引起编译错误。

- 不可以使用NSAllocateObject和NSDeallocateObject等类。

  原因：在ARC有效时，内存管理的工作都交给了编译器，其中就包括对象内存的分配和回收，和第一条一样，也是没必要的操作，所以，编译器干脆就直接禁止了，会报编译错误。

- 必须遵守内存管理方法的命名规则。

  原因：无论ARC是否有效，对于对象生成/持有的方法必须遵循以下的命名规则：

  1.调用方持有对象，以alloc/new/copy/mutableCopy开头的方法。

  2.以init开头的方法，必须是实例方法，并且一定要返回对象，该对象不注册在自动释放池中。

- 不需要显示调用dealloc。

  原因：无论ARC是否有效，对象被释放时，系统会自动调用dealloc方法，不需要手动调用。区别在于，ARC无效时，需要在dealloc方法里面调用[super dealloc]，而ARC有效时会自动对此进行处理。

- 使用@autoreleasePool代替NSAutoreleasePool。

  原因：ARC有效时使用@autoreleasePool代替NSAutoreleasePool，直接调用会报编译错误。

- 不可以使用NSZone。

  原因：运行时系统已忽略了区域的概念，所以，这个对象也被禁止调用。

- 对象型变量不能作为C语言结构体的成员

  原因：C语言规则上没有方法来管理结构体的生存周期。因为结构体是值类型，在栈上分配，由系统负责释放。但是，对象是引用类型，在堆上分配，在ARC有效时由编译器管理其生命周期。

- 显示转换"id"和"void *"

  原因：在ARC有效时，会报编译错误，因为要明确指出所有权的转移，可以使用\__bridge关键字进行显示的转换。

#### 2.3ARC内存管理规则

- 自己生成的对象，自己持有
- 非自己生成的对象，自己也能持有
- 自己持有的对象不再需要时释放
- 非自己持有的对象无法释放

#### 2.4说一下iOS对象的rc怎么储存？

参考：

iOS对象有三种类型：taggedPointer，nonpointer_isa和非nonpointer_isa对象。taggedPointer是在栈上分配的，类型更像值类型，所以没有rc；nonpointer_isa则比较复杂，首先，如果对象的rc没有超过nonpointer_isa所分配的储存范围，则rc的值就储存在nonpointer_isa指针所分配的地方；如果对象的rc超过nonpointer_isa所分配的储存范围，则rc的值储存在nonpointer_isa指针所分配的地方和全局的哈希表的sideTable里面；非nonpointer_isa对象，rc存在全局哈希表sideTable里面。sideTable是一个结构体，包含引用计数表，weak表和自旋锁，而全局的哈希表包含多个sideTable，主要是为了避免多个对象同时操作sideTable，造成资源竞争。

#### 2.5说一下ARC在编译器和运行期做了哪些工作？

参考：

编译期：根据上下文在适当的地方插入retain/release，并且会根据上下文省略多余的retain/release对。

运行期：

1.weak修饰的对象，在运行时释放时被置为nil。

2.返回非持有对象的方法在MRC有效时，是通过autorelease来实现延时释放，而ARC有效时换成了objc_autoreleaseReturnValue，该方法会判断接下来是否要执行objc_retainAoutoreleasedReturnValue方法，如果是，则设置全局标志位并直接返回对象，而不放入autoreleasePool；否则，将对象加入autoreleasePool。而objc_retainAoutoreleasedReturnValue也不直接retain对象，而是判断全局的标志位，如果是，则直接返回对象；如果不是，则retain对象再返回。通过这样优化，就省去了一对retain/release操作，并且没有用到autoreleasePool，调用的速度更快。

#### 2.6说一下retain和release的实现？

参考：

taggedPointer：没有retain和release的概念。

没有开启nonpointer_isa：

1.在全局的哈希列表中，以对象的内存地址作为key，通过相关的哈希计算找到对应的SideTable。

2.SideTable是一个结构体，包含引用计数表，weak表和锁，再以对象的内存地址作为key，通过相关的哈希计算找到对应的引用计数内存地址。

3.最后进行增删操作。

开启nonpointer_isa：

1.通过相应的操作，判断引用计数是否溢出。

2.没有溢出，则对象的引用计数的增删直接在指针对应的地址操作。

3.溢出，则在对应的SideTable操作，步骤和没有开启nonpointer_isa一样。

#### 2.7说一下dealloc的实现？

参考：

1.dealloc会调用objc_object::rootDealloc方法：

（1）判断isTaggedPointer，如果是，则直接返回；如果不是，则走下一步。

（2）判断是否是nonpointer并且没有弱引用，没有关联对象，没有C++析构函数，没有强引用，如果是，则直接调用C函数free(this)释放内存；如果不是，则走下一步。

（3）调用object_dispose函数

2.object_dispose函数：

（1）判断对象是否为nil，如果是，则返回nil；如果不是，走下一步。

（2）调用objc_destructInstance函数。

（3）最后调用free函数释放内存。

3.objc_destructInstance函数：

（1）判断是否有C++析构函数，如果有，则执行。

（2）判断是否有关联的对象，如果有，则清除关联对象。

（3）调用clearDeallocating函数。

4.clearDeallocating函数：

（1）判断是否是没有开启nonpointer，如果是，则调用sidetable_clearDeallocating函数。

（2）判断是否有弱引用或则强引用，如果有，则调用clearDeallocating_slow函数。

5.sidetable_clearDeallocating和clearDeallocating_slow函数：

（1）找到对象所在的SideTable，并上锁。

（2）找到对象的弱引用表，清除掉。

（3）擦除对象的强引用表。

（4）开锁。

6.dealloc流程结束。

#### 2.8在MRC下如何重写Getter和Setter方法？

参考：

```objective-c
@property (nonatomic, strong) NSString *brand;
//@property (nonatomic, copy) NSString *brand;

//setter
-(void)setBrand:(NSString *)brand{
//如果实例变量指向的地址和参数指向的地址不同
    if (_brand != brand)
    {
        //将实例变量的引用计数减一
        [_brand release];
       //将参数变量的引用计数加一,并赋值给实例变量
        _brand = [brand retain];  //如果是copy修饰符，则_brand = [brand copy];
    }
}

//getter
-(NSString *)brand{
    //将实例变量的引用计数加1后,添加自动减1
    //作用,保证调用getter方法取值时可以取到值的同时在完全不需要使用后释放
    return [[_brand retain] autorelease];
}

//MRC下 手动释放内存 可重写dealloc但不要调用dealloc  会崩溃
-(void)dealloc{
    [_string release];
    //必须最后调用super dealloc
    [super  dealloc];
}
```

## 3.关键字

#### 3.1\__weak修饰的变量是否会被注册到autoreleasePool里面？

参考：

Apple LLVM版本8.0.0之前：

\__weak修饰的变量会注册到autoreleasePool里延时释放。

Apple LLVM版本8.0.0之后：

\__weak修饰的变量不会注册到autoreleasePool里，超过作用域会直接释放。

#### 3.2简要说一下autoreleasePool的数据结构？

#### 3.3\__weak和\_unsafe_unretain的区别？

#### 3.4什么情况使用weak关键字，相比assign有什么不同？

#### 3.5为什么已经有了ARC还是需要autoreleasePool的存在？

#### 3.6\__weak修饰的变量，如何实现在变量没有强引用后自动过置为nil?

#### 3.7说一下strong,retain, copy,assign,weak,unsafe_unretain的理解

#### 3.8函数返回一个对象，会将对象加入autoreleasePool里面吗？为什么？

#### 3.9说一下copy和mutableCopy有什么区别？浅复制和深复制以及集合类怎么实现深复制？

#### 3.10讲一下@dynamic关键字

#### 3.11autoreleasePool的释放时机

#### 3.12property的作用是什么，有哪些关键词？

#### 3.13父类的property是如何查找的？

#### 3.14怎么使用copy关键字？

#### 3.15使用copy修饰可变对象，比如NSMutableArray，会有什么问题？

#### 3.16如何让自己的类使用copy修饰符？如何重写带copy关键字的setter？

#### 3.17@property的本质是什么？ivar, getter,setter是如何生成并添加到这个类里面？

#### 3.18@protocol和category中如何使用@prpperty

#### 3.19@property中有哪些属性关键字？

#### 3.20用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？

#### 3.21对飞机和对象的copy操作

#### 3.22集合类对象的copy和mutableCopy

#### 3.23@synthesize合成实例变量的规则是什么？假如property名为foo，存在一个名为_foo的实例变量，那么还会自动合成新变量吗？

#### 3.24在有了自动合成属性属性实例变量后，@synthesize还有哪些使用场景？



































