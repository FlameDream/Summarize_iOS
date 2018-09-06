##前言
iOS 4.0版本中，块不包含Objective-C中的新编程发现。 它们也存在于其他编程语言中（例如Javascript）和其他名称，例如Closures。 在iOS中，它们首次出现在4.0版本中，从那时起它们就已经被广泛接受和使用。 在随后的iOS版本中，Apple重新编写或更新了许多框架方法，因此它们采用了块，而且块似乎部分是代码编写方式的未来。 但他们到底有什么关系呢？

块是添加到C，Objective-C和C ++的语言级功能，它允许您创建不同的代码段，这些代码段可以传递给方法或函数，就像它们是值一样。 块是Objective-C对象，这意味着它们可以添加到NSArray或NSDictionary等集合中。 它们还能够从封闭范围中捕获值，使其类似于其他编程语言中的闭包或lambda

**总之：Block块是Objective-C的对象，块是一个独立的自治代码片段，总是存在于另一个编程结构的范围内。**
参考[Apple开发文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)

##一、iOS Block 概念
Apple文档介绍Block是Object-C的对象，Object-C的对象创建过程，确认对象的名字（变量名），创建对象（对象的堆栈区）。Block块其实和对象创建很类似，创建的规则不一样。下边介绍一下Block创建的规则和使用的方法。

Block定义分为两个模块。1.block变量（block：block代码块定义的声明的变量名） 2.表达式（block：独立自治的代码片）  

####1.Block 变量
Block块变量的定义和使用函数指针引用C函数的方式相同，您可以声明一个变量来跟踪块，如下所示：

> void (^simpleBlock)(void);
> 返回值类型 (^变量名)(参数列表)

####2.Block 表达式
Block块 表达式和C匿名函数的方法表达式相同。
> ^ int (int count) {
>       return count + 1;
>  };
> ^ 返回值类型 (参数列表) {表达式}

#### 3.Block块使用常见和各种情况

###### 3.1Block 表达式
Block表达式语法：
> ^ 返回值类型 (参数列表) {表达式}

例如：
```
^ int (int count) {    //返回值， 参数列表
return count + 1;
};
```
其中，可省略部分有：

* 返回类型，例：
```
^ (int count) {
return count + 1;
};
```
* 参数列表为空，则可省略，例：

```
^ {
NSLog(@"No Parameter");
};
```

###### 3.2Block 变量
声明Block类型变量语法：

>返回值类型 (^变量名)(参数列表) = Block表达式

例如，如下声明了一个变量名为blk的Block：

```
int (^blk)(int) = ^(int count) {
return count + 1;
};
```
当Block类型变量作为属性时，写作：
```
typedef int (^blk_k)(int);
@property(copy) blk_k blk;

-(void)initBlockProperty{
_blk = ^(int count) {
return count + 1;
};
}
-(void)test{
_blk();
}

```

当Block类型变量作为函数的参数时，写作：

```
- (void)func:(int (^)(int))blk {
NSLog(@"Param:%@", blk);
}
```

借助typedef可简写：

```
typedef int (^blk_k)(int);

- (void)func:(blk_k)blk {
NSLog(@"Param:%@", blk);
}
```

Block类型变量作返回值时，写作：
```
- (int (^)(int))funcR {
return ^(int count) {
return count ++;
};
}
```

借助typedef简写：
```
typedef int (^blk_k)(int);

- (blk_k)funcR {
return ^(int count) {
return count ++;
};
}
```

##二、Block底层原理

对Block进行行深入的分析，把block编译解析，查看底层Block是如何实现的，通过使用Clang把Block解析成C++源码
Block结构
```
int main() {
int count = 100;
void (^ blk)() = ^(){
NSLog(@"In Block:%d", count);
};
blk();
}
```
>clang -rewrite-objc 源码文件名

解析之main方法内的源码：
```
int main() {
int count = 100;
void (* blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, count));
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
}
```
从解析出来的源代码看，main内包含一个__main_block_impl_0的Block块，而定义的Block块被解析成__block_impl，接下来看一下这些内容Block块的具体内容。
```
// main_block 块
struct __main_block_impl_0 {
struct __block_impl impl;   //定义的那个block块
struct __main_block_desc_0* Desc;   // main Block的描述
int count;                          //定义的那个count

//  block 实现内部实现细节赋值等内容
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _count, int flags=0) : count(_count) {
impl.isa = &_NSConcreteStackBlock;
impl.Flags = flags;
impl.FuncPtr = fp;
Desc = desc;
}
};


// 主要是block块对count进行复制出来
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
int count = __cself->count; // bound by copy
// __cself相当于Objective-C中的self
NSLog((NSString *)&__NSConstantStringImpl__var_folders_d0_ts4pl5295tnfzz5ls4lj_z8w0000gp_T_main2_e76fdd_mi_0, count);
}

// block块的描述 
static struct __main_block_desc_0 {
size_t reserved;   // block块预留的内存
size_t Block_size;   // block块的内存
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

// Block块的结构体
struct __block_impl {   
void *isa;
int Flags;
int Reserved;
void *FuncPtr;
};
```
在Objective-C中，任何类的定义都是对象。类和类的实例（对象）没有任何本质上的区别。任何对象都有isa指针。block块中包含isa，验证苹果文档说Block是Object-C的对象。
* 注意事项：block容易造成循环引用，在block里面如果使用了self，然后形成强引用时，需要打断循环引用；在MRC下用_block，在ARC下使用__weak;

##三、关于block在内存中的位置

对于block块的存储位置（block入口的地址）可能存放在3个地方：代码区（全局区）、堆区、栈区（ARC情况下回自动拷贝到堆区、因此ARC下只有两个地方：代码区和堆区）。
![11111.png](https://upload-images.jianshu.io/upload_images/2664540-85193156f20af630.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



Block有三种类，根据Block对象创建时所处数据区不同而进行区别：
* **_NSConcreteStackBlock：**在栈上创建的Block对象
* **_NSConcreteMallocBlock：**在堆上创建的Block对象
* **_NSConcreteGlobalBlock：**全局数据区的Block对象
![222.png](https://upload-images.jianshu.io/upload_images/2664540-55ba822a91e585df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 代码区：不访问栈区的变量（如局部变量），且不访问堆区的变量（如用alloc创建的对象）时，此时block存放在代码区；
* 栈区：把栈区的代码复制到堆区，延长使用Block块的使用时间（Block传值）copy
* 堆区：如果访问了堆区的变量（如局部变量），或堆区的变量（如用alloc创建的对象），此时block存方在堆区

实际是放在栈区，在ARC情况下自动拷贝到堆区，如果不是ARC则存放在栈区，所在函数执行完毕就回释放，想再外面调用需要用copy指向它，这样就拷贝到了堆区，strong属性不会拷贝、会造成野指针错区。（需要理解ARC是一种编译器特性，即编译器在编译时在核实的地方插入retain、release、autorelease，而不是iOS的运行时特性）。
此外代码存在堆区时,需要注意，因为堆区不像代码区不变化，堆区是动态的（不断的创建销毁），当没有强指针指向的时候就会被销毁，如果再去访问这段代码时，程序就会崩溃！所以此种情况在定义block属性时需要指定为strong or copy。block是一段代码，即不可变，所以使用copy也不会深拷贝。

##总结
在使用Block过程中，常用的几种修饰词
1.避免在对象循环引用__weak、__strong
2.在block内使用变量使用__block
3.延长block的使用时间使用copy修饰
