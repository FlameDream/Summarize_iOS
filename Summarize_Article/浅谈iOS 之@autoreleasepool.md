浅谈iOS 之@autoreleasepool

# 前言

在互联网时代，电子设备的内存管理是一个困扰的技术难点之一。随着iPhone手机技术的更新，在2011年之前使用手动引用计数MRC（Manual Reference Counting），在WWDC2011和iOS 5 引入了自动引用计数ARC（Auto Reference Counting），一个全新的内存管理机制诞生。而autoreleasepool是OC内存管理机制，在ARC的机制下会经常使用到@autoreleasepool自动释放池管理、优化内存。

# 一、基本概念

ARC下的产物，为了替代人工管理内存，大大的简化了iOS开发人员的内存管理工作；实质上是使用编译器替代人工在适当的位置插入release、autorelease等内存释放操作；

@autoreleasepool 自动释放池：

管理内存的池，把不需要的对象放在自动释放池中，自动释放这个池子内的对象。(简单，接下来会详细说明@autoreleasepool工作过程)

# 二、底层结构

在ARC中，看一下@autoreleasepool底层代码具体是什么。

### 1.查看@autoreleasepool{ }编译成C++代码

使用编译器clang编译main.m转化成main.cpp文件（在终端：clang  -rewrite-objc main.m）
```
    #import <Foundation/Foundation.h>
      int main(int argc, char * argv[]) {
          @autoreleasepool {
          }
    }
```

编译之后的main.cpp的代码，把主要的代码拷贝出来如下

``` 
extern "C" __declspec(dllimport)
    void * objc_autoreleasePoolPush(void);
extern "C" __declspec(dllimport)
     void objc_autoreleasePoolPop(void *);
struct __AtAutoreleasePool { 
        __AtAutoreleasePool()
         {
            atautoreleasepoolobj = objc_autoreleasePoolPush();
          }  
      ~__AtAutoreleasePool(){
          objc_autoreleasePoolPop(atautoreleasepoolobj);
        } 
     void * atautoreleasepoolobj;
  };
```

可以从以上代码看出来@autoreleasepool其实是objc_autoreleasePoolPush 和 objc_autoreleasePoolPop这两个方法组成的。

*总之：@autoreleasepool是由objc_autoreleasePoolPush 和 objc_autoreleasePoolPop方法构成的一个结构体。*

### 2.查看objc_autoreleasePoolPush和objc_autoreleasePoolPop

参照苹果开源代码找到objc_autoreleasePoolPush和objc_autoreleasePoolPop两个方法，两个方法在[NSObject.mm](https://link.jianshu.com/?t=http://NSObject.mm)中实现（苹果开源代码：[https://opensource.apple.com/tarballs/objc4](https://opensource.apple.com/tarballs/objc4/)/）

```
void *objc_autoreleasePoolPush(void){    
    if (UseGC) return nil;
    return AutoreleasePoolPage::push();
}
void objc_autoreleasePoolPop(void *ctxt){    
    if (UseGC) return;    // fixme rdar://9167170   
    if (!ctxt) return;  
    AutoreleasePoolPage::pop(ctxt);
}
```
从上面可以发现，C++类AutoreleasePoolPage才是实际的实现所在，找到AutoreleasePoolPage：

```
class AutoreleasePoolPage 
{
#define POOL_SENTINEL nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif      // 通过查询  PAGE_MAX_SIZE =  4096
    static size_t const COUNT = SIZE / sizeof(id);

    magic_t const magic;                              // 验证码
    id *next;                                                  //栈顶地址
    pthread_t const thread;                          // 所属线程
    AutoreleasePoolPage * const parent;    //父节点
    AutoreleasePoolPage *child;                  // 子节点
    uint32_t const depth;                             //page的复杂度                   
    uint32_t hiwat;
    ……
```
去除了一些不重要的代码，可以看出这是一个典型的双向列表结构，每个Page大小为4096 Byte，所以AutoreleasePool实质上是一个双向AutoreleasePoolPage列表；接下来分析一下自动释放池的工作过程：

### 3.创建自动释放池
void* objc_autoreleasePoolPush()内部实际调用的是AutoreleasePoolPage::push()函数，其实现如下：
```
//  objc_autoreleasePoolPush()内部实际调用
void * objc_autoreleasePoolPush(void)
{
    if (UseGC) return nil;
    return AutoreleasePoolPage::push();
}

//  AutoreleasePoolPage::push()的 push方法

static inline void *push() 
    {
        id *dest = autoreleaseFast(POOL_SENTINEL);
        assert(*dest == POOL_SENTINEL);
        return dest;
    }

static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {  // 如果页面不为空并且存在
            return page->add(obj);    // 添加对象到自动释放池入栈
        } else if (page) {     // 如果自动释放池页存在 且 页面满了
            return autoreleaseFullPage(obj, page);  // 
        } else {      // 
            return autoreleaseNoPage(obj);
        }
    }

```
从以上开源代码可以看出，hotPage()是找出当前的正在使用的page
1.hotPage存在且未满，AutoreleasePoolPage对象作为自动释放池加入栈中
2.hotPage存在且hotPage页面满了，AutoreleasePoolPage创建新的Page并把对象添加到栈中
3.hotPage不存在。添加一个新的AutoreleasePoolPage页面添加对象

######3.1.hotPage不存在，执行的方法
```
   id *autoreleaseNoPage(id obj)
    {
        // No pool in place.
        assert(!hotPage());

        if (obj != POOL_SENTINEL  &&  DebugMissingPools) {
            // We are pushing an object with no pool in place, 
            // and no-pool debugging was requested by environment.
            _objc_inform("MISSING POOLS: Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }

        // Install the first page.
        AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
        setHotPage(page);

        // Push an autorelease pool boundary if it wasn't already requested.
        if (obj != POOL_SENTINEL) {
            page->add(POOL_SENTINEL);
        }

        // Push the requested object.
        return page->add(obj);
    }

```

#####3.2.hotPage存在且hotPage页面满，执行的方法
```
static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        assert(page == hotPage()  &&  page->full());

        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());

        setHotPage(page);
        return page->add(obj);
    }

```

#####3.3.hotPage存在且未满，执行添加对象
```
 id *add(id obj)
    {
        assert(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }
```
综上可以看出在添加自动释放池，所有操作都是对双向堆栈AutoreleasePoolPage的一个创建和添加的操作。


### 4.销毁自动释放池
 首先autoreleasepool的释放工作交给objc_autoreleasePoolPop方法，bjc_autoreleasePoolPop方法如下，自动释放主要交给AutoreleasePoolPage::pop(ctxt);进行
```
void
objc_autoreleasePoolPop(void *ctxt)
{
    if (UseGC) return;

    // fixme rdar://9167170
    if (!ctxt) return;

    AutoreleasePoolPage::pop(ctxt);
}

```

自动释放的方法如下，更具传入的token，查找需要删除的那个页面，进行删除操作。
```
static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        if (token) {
            page = pageForPointer(token);
            stop = (id *)token;
            assert(*stop == POOL_SENTINEL);
        } else {
            // Token 0 is top-level pool
            page = coldPage();
            assert(page);
            stop = page->begin();
        }

        if (PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        // hysteresis: keep one empty child if this page is more than half full
        // special case: delete everything for pop(0)
        // special case: delete everything for pop(top) with DebugMissingPools
        if (!token  ||  
            (DebugMissingPools  &&  page->empty()  &&  !page->parent)) 
        {
            page->kill();
            setHotPage(nil);
        } else if (page->child) {
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }

```

释放自动释放池内内存，双向堆栈中，删除一个AutoreleasePoolPage，根据这个AutoreleasePoolPage对象找到，通过while循环找到AutoreleasePoolPage下方的对象，就像二叉树找到叶子节点。通过节点，首先记录这个节点的地址，找出这个节点的父节点。通过父节点把子节点置空，删除这个节点的指针指向，在通过delete删除对象A的内存空间。通过while循环，直到删除到最初的节点。
```
void kill() 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        AutoreleasePoolPage *page = this;
        while (page->child) page = page->child;

        AutoreleasePoolPage *deathptr;
        do {
            deathptr = page;
            page = page->parent;
            if (page) {
                page->unprotect();
                page->child = nil;
                page->protect();
            }
            delete deathptr;
        } while (deathptr != this);
    }

```

#总结
看到这里，相信你应该对 Objective-C 的内存管理机制有了更进一步的认识。通常情况下，我们是不需要手动添加 autoreleasepool 的，使用线程自动维护的 autoreleasepool 就好了。根据苹果官方文档中对 [Using Autorelease Pool Blocks](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html#//apple_ref/doc/uid/20000047-CJBFBEDI) 的描述，我们知道在下面三种情况下是需要我们手动添加 autoreleasepool 的：

1.  如果你编写的程序不是基于 UI 框架的，比如说命令行工具；
2.  如果你编写的循环中创建了大量的临时对象；
3.  如果你创建了一个辅助线程。
浅谈iOS 之@autoreleasepool
