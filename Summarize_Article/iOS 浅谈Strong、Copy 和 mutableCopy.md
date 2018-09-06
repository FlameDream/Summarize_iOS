    在iOS的王国里，Strong、Copy和mutableCopy在我们的使用过程中，缺一不可的。接下来来介绍一下它们之间的关系和注意点。
    
    一、Copy和mutableCopy之拷贝
    在生活中，时常会用到一个词“拷贝”。例如：微信群里、朋友圈很多人会分享同一篇的文章或者是活动内容，这是一种“拷贝”。这就是类似于浅拷贝“Copy”。当你一个好朋友对你说，把你那个项目给我拷一份，这就是类似于深拷贝”mutableCopy“。
    
    浅拷贝（Copy）：指针拷贝，不产生新的对象，源对象的引用计数器+1；
    
    深拷贝（mutableCopy）：对象的拷贝，会产生新的对象，源对象的引用计数器不变。
    
    判断是浅拷贝和深拷贝就看两个变量的内存地址是否一样，一样就是浅拷贝，不一样就是深拷贝，也可以改变一个变量的其中一个属性值看两者的值是否发生变化。
    
    系统原生的对象深浅拷贝区别
    NSObject类提供Copy和mutableCopy方法，通过这两个方法即可拷贝已有对象的副本，主要的系统原生对象有：NSString和NSMutableString、NSArray和NSMutableArray、NSDictionary和NSMutableDictionary、NSSet和NSMutableSet，NSValue和NSNumber只遵守的NSCoping协议。
    
    注意：基本数据类型（assign修饰），没有对应的指针（假象），直接赋值操作，无需copy操作。
    
    二、Copy和Strong区别
    在OC中经常会碰到定义一个属性property，使用copy、strong这两个词。
    
    在系统原生对象中：NSString和NSMutableString、NSArray和NSMutableArray、NSDictionary和NSMutableDictionary、NSSet和NSMutableSet的使用这两个词的区别。NSString、NSArray、NSDictionary、NSSet都使用Copy这个修饰词，而NSMutableString、NSMutableArray、NSMutableDictionary和NSMutableSet则使用Strong修饰。
    
    @property 中的copy 参数的作用：
    
    在属性的setter实现中对赋值对象做一次copy操作，将copy操作的结果赋值给属性。
    
    情况一：属性是不可变类型的：如：NSString、NSArray、NSDictionary、NSSet
    
    如果赋值对象是可变的，那么将一个不可变的副本赋值给属性。
    
    如果赋值对象是不可变的，那么不会产生新的副本，只是对复制对象引用计数器加1。
    
    情况二：属性是可变类型的，建议不要使用copy参数
    
    可变类型的属性会根据需求对其内容进行修改，使用copy属性的对象类型是不可变的。如果修改这个属性，编译是不会报错，但是运行会奔溃。因为尝试修改一个不可变的对象。
    
    情况三：自定义对象类型，一般情况下不会对自定义的对象使用copy参数。
    
    必须遵守<NSCopying>协议，实现CopyWithZone:方法，才能调用Copy方法，创建副本。
    
    
    
    总之：Strong、Copy 和 mutableCopy这三者之间，需要从底层考虑。OC是一个动态的面向对象语言，C语言是OC的底层实现。Strong主要是对一个堆对象添加一个引用点，Copy和MutableCopy主要是从拷贝内容着手，是否创建新的内容空间。
