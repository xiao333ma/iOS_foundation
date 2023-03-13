源码地址 https://opensource.apple.com/tarballs 
https://github.com/gnustep  对应的 libs-base 是 OC 代码库
https://github.com/apple-oss-distributions  CF, objc4

xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc XXX.m

1. 一个 NSObject 对象占用多少内存？
    * allocWithZone: 最终调用 instanceSize 不足 16 字节，按 16 字节对齐(分配16字节)。class_getInstanceSize(class)最终调用 alignedInstanceSize（ivar size）对齐过的大小，取到的是 isa 为 8 字节（64位机器）
    * 前 8 个字节存放 isa_t, isa_t 为 union 类型。
    
2. 大端小端模式
    * 大端模式，是指数据的高字节保存在内存的低地址中，而数据的低字节保存在内存的高地址中，这样的存储模式有点儿类似于把数据当作字符串顺序处理：地址由小向大增加，数据从高位往低位放；这和我们的阅读习惯一致。
    * 小端模式，是指数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中，这种存储模式将地址的高低和数据位权有效地结合起来，高地址部分权值高，低地址部分权值低。
    * iOS ARM，X86 小端模式，从高地址开始读取。
    
    > 至于为什么要区分大小端，这是因为在计算机系统中，我们是以字节为单位的，每个地址单元都对应着一个字节，一个字节为 8bit。但是在C语言中除了8bit的char之外，还有16bit的short型，32bit的long型（要看具体的编译器），另外，对于位数大于 8位的处理器，例如16位或者32位的处理器，由于寄存器宽度大于一个字节，那么必然存在着一个如何将多个字节安排的问题。因此就导致了大端存储模式和小端存储模式。
    
3. Student 占用几个字节，（macOS/iOS 操作系统 16 字节对齐）内存对齐：结构体的大小必须是最大成员大小的倍数
    Bucket_size (16, 32, 48, 64...)
    class_getInstanceSize(至少需要多少内存) 采用 8 字节对齐，macOS/iOS 操作系统 16 字节对齐。
    class_getInstanceSize 调用 align_word 函数
    sizof 操作符号，编译期就知道大小，sizof（p）p有多大
    malloc_size(p) p 指向的内存有多大
    
    The GNU C Library (glibc) #MALLOC_ALIGNMENT i386 就是 16， 其他平台 2 * sizof(size_t)
    ```
    @interface Student: NSObject
    {
        @public
        int _age;
        int _no;
    }
    @end
    ```
    * Student 包含 isa 为 8 个字节，_age 为 4 字节，_no 为 4 字节。整个对象为 16 字节。
    * Student_IMP 结构体只存放成员变量，一是节省空间，二是方法是共享的，一份就够。方法放在类结构体。

4. OC对象包含3类：intance 对象，class 对象，meta-class 对象
    * instance 对象是类 alloc 出来的对象，每次调用 alloc 都会产生新的 instance 对象
    * instance 对象在内存中存储的信息包括：isa 指针，其他成员变量的值。
    * class 对象包括 isa 指针，supperclass 指针，property,instance method, protocol, ivar；[[xxx class] class] 始终返回 类对象
    * meta-class 和 class 的内存结构一样，包含的 isa 指针，supperclass 指针，property,class method, protocol, ivar，只不过有些内容为空
    * objc_getClass(const char *name) 根据字符串找到类对象 
    
5. isa、superclass 总结
    * instance 的 isa 指向 class (指向的解释 64bit 开始需要 &ISA_MASK)
    * class 的 isa 指向 meta-class
    * meta-class 的 isa 指向基类的meta-class
    * class 的 superclass 指向父类的 class（如果没有父类，superclass 指向 nil）
    * meta-class 的 superclass 指向父类 meta-class（基类的meta-class superclass 指向基类 class）
    * instance 调用对象方法（减号方法），isa 找到 class，方法不存在，就通过 superclass 找父类
    * class 调用类方法（加号方法），isa 找到 meta-class, 方法不存在，就通过 superclass 找父类
    ```
    @impletation NSObject (test)
    - (void)test {
    
    }
    @end
    
    @impetation Person: NSObject
     
    @end
    
    ``` 
    * [Person test], [NSObject test] 都能调用成功。实质是消息发送（@selector(test)），不关心是否是类方法

6. isa 指针
    * 从64bit开始，isa需要进行一次位运算，才能计算出真实地址 &ISA_MASK 
    
7. OC 的类信息存放在哪里
    * 对象方法、属性、成员变量信息、协议信息，存放在 class 对象中
    * 类方法，存放在 meta-class 对象中
    * 成员变量的**具体值**，存放在 instance 对象
    
8. iOS 如何实现 KVO
    * 利用runtime API 动态生成一个字类，并且让 instance 对象的 isa 指向子类
    * 当修改 instance 对象属性时， 对调用 Foundation 的 _NSSetXValueAndNotify
        * willchangeValueForKey:
        * 父类原来的 setter
        * didChangeValueForKey: 内部会触发 Oberserver 的监听方法
    * NSNotification 原理（http://cloverkim.com/ios_notification-principle.html）
        * 在NSNotificationCenter内部一共保存了两张表，一张用于保存添加观察者的时候传入的NotificationName的情况；一张用于保存添加观察者的时候没有传入NotificationCenter的情况。
        * 对于Named Table
            * 首先外层有一个Table，以通知名称为key。其value同样是一个Table。
            * 为了实现可以传入一个参数object用于只监听指定该对象发出的通知，及一个通知可以添加多个观察者。则内Table以传入的object为key，用链表来保存所有的观察者，并且以这个链表为value。
            * 在实际开发中，我们经常将object参数传nil，这个时候系统会根据nil自动产生一个key。相当于这个key对应的value（链表）保存的就是对于当前NotificationName没有传入object的所有观察者。当NotificationName被发送时，在链表中的观察者都会收到通知。
        * UnNamed Table
            * UNamed Table结构比Named Table简单得多。因为没有NotificationName作为key。这里直接就以object为key，比Named Table少了一层Table嵌套。
            * 如果在注册观察者时没有传入NotificationName，同时没有传入object，所有的系统通知都会发送到注册的对象里。
            
9. 如何手动触发 KVO
    * 手动调用 willChangeValueForKeys 和 didChangeValueForKey：
    * 直接修改成员变量不会触发 KVO，因为 KVO 的实质是调用 change 方法
    
10. KVC （键值编码）setvalue:forkey: 会不会触发 KVO
    * 会触发 KVO
    * 设置值的原理 setValue:ForKey: 查找顺序 1. setKey: 2. _setKey: 3. +(BOOL)accessInstanceVariablesDirectly 返回 YES 会访问成员变量。4. _key, _isKey, key, isKey
    * 底层 person->_age = 10 后 再触发 willchange didchange
    * valueForKey: 查找顺序 1. - (id)getKey 2.-(id)key 3.-(id)isKey 4.-(id)_key
    
11. Category 底层原理
    * category 方法通过 runtime 动态将分类方法合并到类对象中去
    * 编译完毕存放到 struct _category_t 结构里, 里面存储分类的对象方法，类方法，属性，协议信息，并没有存放成员变量的ivar，所以不支持添加成员变量
    * 所有的 Category 的方法，属性，协议数据，合并到一个大数组中
        * 越靠后边编译的 Categroy 数据，会在数组的前面。
    * 将合并后的分类数据（方法，属性，协议），插入到类原来数据的前面，因为代码（while(i--)）
    * 源代码顺序
        * objc-os.mm 
            * _objc_init, 
            * map_images, 
            * map_images_nolock
        * objc-runtime-new.mm 
            * _read_images
            * remethodizeClass 
            * attachCategories 
            * attchLists
            * realloc, memomove, memcpy 
    * Extension 在编译期，category 在运行期
    * memcpy, memmove 区别，memcpy 是一格一格挪动

12. Category Load 方法什么时候调用
    * load 方法在加在类和分类时候调用，所有的分类 +load 方法都会调用，load 方法调用不依赖 msg_send 所以本类，分类都会调用。而是找到 load method 地址（函数指针）直接调用
    ```
    struct loadable_class { //这个结构体只用来加载类
        Class cls;
        IMP method;
    }
    ```
    * 源码解读过程：objc-os.mm
        * _objc_init
        * load_images
        * prepare_load_methods
            * schedule_class_load
            * add_class_to_loadable_list
            * add_category_to_loadable_list
        * call_load_methods
            * call_class_loads
            * call_category_loads
            * (*load_method)(cls, SEL_load)
    * 本类调用顺序与 loadable_classes 数组顺序有关
    * 先调用类的 +load
        * 按照编译先后顺序调用（先编译，先调用）
        * 调用子类的 +load 之前先调用父类的 +load
    * 再调用分类的 +load
        * 按照编译先后顺序调用（先编译，先调用）
        
13. +intialize 底层原理
    * intialize 在类第一次接受到消息时执行，只执行一次，实质就是通过 objc_msgSend()。（思路：intialize 是在 objc_msgSend 第一次发消息时调用，也许是在 objc_msgSend 函数内部，也许是在寻找方法过程中调用）因为基于消息发送所以具有以下特点：
        * 如果子类没有实现 +intialize, 会调用父类 initialize (所以父类+initialize可能调用多次，多次是因为 objc_msgSend 的方法查找的顺序，也就是子类没有就找父类的方法使用)
        * 如果分类实现 +intialize, 就覆盖类本身 +initialize 调用
    * 调用顺序
        * 先调用父类+intialize再调用子类+intialize
        * 如果分类实现 +intialize, 就覆盖类本身 +initialize 调用，不影响父类调用
    * 源代码(代码递归执行，先给父类发消息)
        ``` 
        if(intialize && !cls->isIntiaze) {
            runtimeLock.unlockRead();
            _class_initialize(_class_getNonMetaClass(cls, inst));
            runtimeLock.read();
        }
        
        void callIntialize(Class cls) {
            ((void)(*)(Class, SEL))objc_msgSend(cls, SEL_initialize);
        }
        ```
    
    * 源码解读过程
        * objc-msg-arm64.s
            * objc_msgSend
        * objc-runtime-new.mm
            * class_getInstanceMethod
            * loopUpImpOrNil
            * loopUpImpOrForward
            * _class_intialize
            * callInitialize
            * objc_msgSend(cls, SEL_initialize)
    * load, initialize 方法区别
        * 调用方式（1， load 是根据函数地址直接调用，2，initialize 是通过 objc_msgSend 调用）
        * 调用时刻 （1. load 是runtime 加在类、分类的时候调用只一次，intial 是类第一次接收消息的时候调用，每个类只会intialize一次，父类可能被调用多次）
        * 调用顺序
    
14. Category 关联对象
    * objc4 源码解读：objc_references.mm
    * 不能直接添加成员变量，可以间接实现成员变量（思路：1. 全局变量，但会导致创建多个实例是值被覆盖，2. 通过 全局 Dictionary 实现每个实例唯一一个，但当属性过多时会导致出现多个 Dictionary，多线程问题）
    * setAssociateObject 的key，只用 `void *aKey`，但默认值是 0，会导致多个key 重复，改良void *aKey = &aKey, 意思就是 aKey 中存放自己的地址，保证 key 唯一。（思路：1. 全局变量为key，如：void *aKey = &aKey 不加 static 会导致外部直接通过 extern 后就能访问，2. 加 static 后作用域仅在当前文件，但不优雅。3.直接用字符串 @"key"，也就是用字符串的地址，不优雅。4.最后是 @selector(key)。5. _cmd = @selector(key)）
    * setAssociateObject 同样适用于类对象，因为参数是 id，也可以用 setAssociateObject([xxx class])
    * 关联一个 weak 对象，可内部通过 deallocbloack 或者包装对象的方式进行。
        ```
        - (id)weakObi {
            return objc_getAssociatedobject(self, SWEAK_KEY);
        }
        - (void) setWeakObj:(id)weakobj {
            [weakObj setBlock:^() {
            objc_setAssociatedObject(self, &WEAK_KEY, nil, OBJC_ASSOCIATION_ASSIGN);
        }];
            objc_setAssociatedobject(self, SWEAK_KEY, weakobj, OBJC_ASSOCIATION_ASSIGN);
        }
        - (DeallocBlock)block {
            return objc_getAssociatedObject(self, &BLOCK_KEY);
        }
        - (void)setBlock: (DeallocBlock)block {
            objc_setAssociatedObject(self, &BLOCK_KEY, block, OBJC_ASSOCIATION_COPY_NONATOMIC);
        }
        #pragma clang diagnostic push #pragma clang diagnostic ignored "-Wobjc-protocol-method-implementation"
        - (void) dealloc {
            self.block ? self.block() : nil;
            objc_setAssociatedObject(self, &BLOCK_KEY, nil, OBJC_ASSOCIATION_ASSIGN);
        }
        #pragma clang diagnostic pop
        
        // 先定义一个壳类
        @interface NJWeakWrapper : NSObject
        @property (nonatomic, weak) id obj;
        @end
        @implementation NJWeakWrapper
        @end

        // 然后用壳包一层
        - (id)someWeakProp
        {
            NJWeakWrapper *wrapper = objc_getAssociatedObject(self, @selector(someWeakProp));
            return wrapper.obj;
        }

        - (void)setSomeWeakProp:(id)obj
        {
            NJWeakWrapper *wrapper = nil;
            if (obj) {
                wrapper = [[NJWeakWrapper alloc] init];
                wrapper.obj = obj;
            }
            objc_setAssociatedObject(self, @selector(someWeakProp), wrapper, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        }
        ```
    * setAssociateObject 传进去的 value 并不会添加到本来的 ivars 里，没有存放在 instance对象也没有存储在类对象里
    * 实现关联对象技术的核心对象有 (AssociationsManager, AssociationsHashMap, ObjectAssociationMap, ObjcAssociation)
    * ` void objc_setAssociation(id object, const void * key, id value, objc_policy policy)`
    * AssociationsManager 管理 `AssociationsHashMap *_map`, `AssociationsHashMap<disguised_ptr_t, ObjectAssociationMap>`,  `AssociationsMap<void *, ObjectAssociation>`, 
        ```
        struct ObjectAssociation {
            unintptr_t _policy
            id _value
        }
        ```
        DISGUISE(object) 就是 disguised_ptr_t，const void *key 对应 AssociationsMap 中的 key。
        传入 nil 对 map.earse,也就是会把这一对移除
        objc_removeAssociation 就是擦除了某个 object 对应的 ObjectAssociationMap
    * `void objc_setAssociation` 没有弱引用（weak）效果
    * object 对象销毁，会自动销毁对应的关联对象，涉及内存管理知识。
    
15. block 的原理，（封装了函数调用以及调用环境的OC对象）
    ```
    struct __block_impl {
        void *isa;
        int Flags;
        int Reserved;
        void *FuncPtr; // block 的函数的具体实现，在 block 内断点，汇编首地址就是，放在 text 段
    };
    
    struct __main_block_impl_0 {
        struct __block_impl impl; // 注意不是指针，相当于把结构体放过来，内存地址于 __block_impl 一致。
        struct __main_block_desc_0* Desc;
        int age; // auto 自动变量，值传递，因为可能会消失
        int *height; // static 静态变量，指针传递，因为不会消失
        MJPerson *self; // 会捕获 self ，因为 self 是局部变量
        // c++ 构造函数
        // c++ 语法 age(_age) 表示将_age赋值给 age 成员变量
        __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, Person *_self, int flags=0) : age(_age), self(_self) {
            // 说明 block 为该对象
            impl.isa = &_NSConcreteStackBlock;
            impl.Flags = flags;
            impl.FuncPtr = fp;
            Desc = desc;
        }
    };
    // 封装了block执行逻辑的函数
    static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
        int age = __cself->age; // bound by copy
        int *height = __cself->height; // bound by copy

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_rg_jsprpr1n1dg179131z16jn8r0000gn_T_main_b60f00_mi_0, age, (*height));
    }
    static struct __main_block_desc_0 {
        size_t reserved;
        size_t Block_size;
    } __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
    // 定义 block 变量，以下为省略后的代码
    int age = 10;
    static height = 3;
    // block 指向 结构体
    void(*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA, age, &height);
    age = 20;
    // 指向block内部的代码
    block->FuncPtr(block);
    // 
    ```
    * block 是 OC 对象，内部有 isa；封装了函数地址（实现）
    * 局部变量：auto 自动变量，离开作用域自动销毁，值传递，block 捕获到内部，方式是 struct 添加一个成员变量；stack 静态变量，指针传递，也是添加一个成员变量（原因自动变量有可能随作用域消失，静态变量不会，跨函数访问会导致局部变量消失）；对象减号方法，block 内的 self 会被捕获，因为 self 也是局部变量，减号方法内对应 C 语言默认传入两个参数 self 和 _cmd ; block 内访问成员变量如 _name, 并不会捕获 _name , 而是捕获 self 然后再访问 self->_name;
    * 全局变量不需要捕获（原因可以随时访问到）
    * 一切以运行时的结果为准，`__NSStackBlock__` 时机运行可能不是，clang 从某个版本开始中间代码不再是 C++，而是某个中间产物。
        * `__NSGlobalBlock__`(data 段) , `__NSStackBlock__` （栈）, `__NSMallocBlock__`（堆） 是对应的 `_NSConcreateXXXBlock`，最终都是继承自 NSBlock 类型
    * `__NSGlobalBlock__` 没有访问 auto 变量;`__NSStackBlock__` 访问了 auto 变量(但打印是 mallocblock，是因为 ARC 的存在，ARC 关闭后就正常了显示为 stack);  `__NSMallocBlock__` 是 `__NSStackBlock__` 调用了 copy; 
    ```
    // 如下代码 MRC 情况下打印随机,因为 block 的 struct 因为在栈上，超出作用域其他值会被冲掉； ARC 自动帮做了 copy 后 age 打印会为10
    int age = 10
    block = ^{
        printf("%d", age);
    }
    ```
    * 以上三种block，stackblock copy 后到堆变 mallocblcok ，globalblock copy后不会发生变化，mallocstack copy 后引用计数增加。
    * 在 ARC 环境下，编译器会根据情况自动将栈上 block 复制到堆上
        * block 作为自动返回值（return blcok）会 copy
        * 被 `__strong` 修饰时，OC 对象 ARC 默认是 `__strong` 修饰
        * block 作为 Coacoa API 中方法名包含 usingblock: 时
        * copy,strong 修饰 block 在 ARC 没有区别
    * 堆空间的 block 会对实例对象进行一次 retain 操作，保证实例对象不会被销毁。如果实例对象有 `__weak` 修饰后不影响销毁。
    * `__weak` 问题解决
        * OC 转 C++ 可能遇到 cannot create `__weak` refrence 
        * 解决 `xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc main.m` (意思是需要运行时支持)
    * `__block` 的内存管理
        * 当 block 内部访问 auto 变量时
            * 只要 block 在栈上，不管 auto 变量是不是被 `__strong` 对外侧变量都不会产生强引用，无法给实例对象保命。
        * 一旦 copy， `__main_block_copy_0` 函数内部会调用 _Block_object_assign 函数会根据变量修饰符（`__strong`, `__weak`)做出相应的策略。针对是否时对象类型 _main_block_desc 内部会有是否包含 `__main_block_copy_0` `dispose` 的区分
            * `_Block_object_assign` 函数会对 `__block` 变量形成强引用（retain）或者弱引用（**注意：这里仅限于 ARC 时会 reatain，MRC时不会 reatain**）
        * 如果block 从堆上移除
            * 会调用 block 内部的 dispose 函数
            * dispose 函数内部调用 _Block_object_dispose 函数
            * _Block_object_dispose 函数会自动释放引用的 auto 变量，类似 release，-1 操作。
            * 以上两个函数与内存管理强相关，也就是非对象类型不会出现
        * `__block` 可以解决block 内部无法修改 auto 变量值的问题
        * `__block` 不能修饰全局变量，静态变量（static）
        * 编译器会把 `__block` 变量包装成一个对象，（`__Block_byref_age_0 *age`)
        ```
            __block int age = 10;
            ^{
                NSLog(@"%d", age);
            }();
            struct __main_block_impl_0 {
                struct __block_impl impl;
                struct __main_block_desc_0* Desc;
                __Block_byref_age_0 *age; // by ref
            }
            struct __Block_byref_age_0 {
                void *__isa;
                __Block_byref_age_0 *__forwarding; // 把自己地址传给该成员变量
                int __flags;
                int __size; // 
                int age; // 使用饿值
            }
        ```
    * 解决循环引用问题 ARC
        * weak，unsafe_unreatined 关键字修饰
        * `__block` 也能解决就是在内部用完后将捕获到的变量赋值为 nil；（缺点：必须要调用 block） 
    * 解决循环引用问题 MRC
        * unsafe_unreatined 关键字修饰
        * `__block` 也能解决，原因是该关键字修饰后在 MRC 情况下不会对外部变量进行捕获，也就是不会对外部对象进行 retain 
    * weak, strong dance 是为了保证在 block 整个运行环境内，strong self 就是强行保命，防止运行过程中 self 突然被释放。另外 strong self 可以直接通过 self->variable，weakself 访问成员变量会报错。
    * block 与 swift closure 的不同
        > 闭包其实是，将代码跟代码所处于的环境做为一个整体来看待。周围的环境，表现为代码所使用的数据。在有些语言中，这个概念叫代码块（block），匿名函数(lambda)等等。数据跟代码不再人为割裂开来，统一起来看待。闭包就会是很自然的概念。数据可以传递，从一个地方传递到另一个地方，并且以后再使用。闭包从某个角度来说，也是数据，当然也可以传递，从一个函数传递到另一个函数，也可以保持下来，以后再调用。
        * OC中默认截获变量，Swift默认截获变量的引用。它们都会强引用被截获的变量。
        * Swift中没有__block修饰符，但是多了截获列表。通过把截获的变量标记为weak避免引用循环
        * 两者都有Weak-Strong Dance，不过这一点上OC的写法更简单。
        * 在使用可选类型时，要明确闭包截获了可选类型还是实例变量。这样才能正确判断是否发生循环引用。
        
    
16. runtime 底层原理
    * isa 详细了解，在 arm64 架构前，isa就是一个普通的指针，存放类对象，元类对象的地址，从 arm64架构开始，对isa进行了优化，变成了一个共用体（union）结构，还使用位域来存储更多的信息。
    * 掩码MASK用来取值和设置值
    * 类对象，元类对象，最后3位一定是0，由 isa_mask 决定，也就是实例对象&ISA_MASK 的地址是真正的 Class 对象的地址
    ```
        以下为 arm64 情况下：
        #     define ISA_MASK        0x0000000ffffffff8ULL
        #     define ISA_MAGIC_MASK  0x000003f000000001ULL
        #     define ISA_MAGIC_VALUE 0x000001a000000001ULL
        #     define ISA_HAS_CXX_DTOR_BIT 1
        #     define ISA_BITFIELD                                         
                uintptr_t nonpointer        : 1;  // 0 代表普通指针，存储 Class, Meta-Class 对象的内存地址 ，1 代表优化过，使用位域存储更多信息                         
                uintptr_t has_assoc         : 1; // 是否有设置过关联对象，如果没有，释放时会更快            
                uintptr_t has_cxx_dtor      : 1; // 是否有 C++ 的析构函数（.cxx_destruct），如果没有，释放时会更快                         
                uintptr_t shiftcls          : 33; // 存储着 Class, Meta-Class 对象的内存地址信息
                uintptr_t magic             : 6; // 用于在调试时分辨对象是否未完成初始化                         
                uintptr_t weakly_referenced : 1;  // 是否被弱引用指向过                        
                uintptr_t unused            : 1;  // 正在释放                      
                uintptr_t has_sidetable_rc  : 1;  // 引用计数是否过大无法存储在 isa 中，如果为1，那么引用计数会存储在一个叫 SideTable 的类属性中                        
                uintptr_t extra_rc          : 19  // 引用计数数字，rc 是 retain_count 的简称，里面的值是引用计数减一
        #     define ISA_HAS_INLINE_RC    1
        #     define RC_HAS_SIDETABLE_BIT 44
        #     define RC_ONE_BIT           (RC_HAS_SIDETABLE_BIT+1)
        #     define RC_ONE               (1ULL<<RC_ONE_BIT)
        #     define RC_HALF              (1ULL<<18)
    ```
    * Class 的 结构
    * class_rw_t 的 methods，properties，protocols 是二维数组，是它是有 shiftcls & DATA_FAST_MASK 得到。二维数组的好处是方便分类扩充，每个分类的方法都是一个 list
    * wwdc 20 objc 增加 class_rw_ext_t , 新版的rumtime利用了懒加载的机制，在类的methods，properties等需要修改时，才初始化class_rw_ext_t这块“dirty+memory”存储这些列表，这样就减少了在旧版 rumtime 中 90% 的类在 rw 中直接复制ro中数据浪费的内存。
    * class_rw_ext_t 何时开辟
        * 装载分类时；runtime动态添加方法时；runtime添加property时；runtime添加protocol时
        
    * method_t 是对方法、函数的封装
        ```
        struct method_t {
            SEL name; // 函数名
            const char *types // 编码
            MethodListIMP imp; // 具体函数实现
        }
        ```
    * `- (int)test:(int)age height:(float)h` 编码结果为 `i24@0:8i16f20`，其中 i 代表 int，@ 代表对象，f 代表 float ，：代表 SEL，因为对于OC方法调用实际就是 C 函数，隐藏了 self 和 _cmd 两个参数。具体数字 24 代表这个方法占用字节数，0代表self从0 index 开始，8代表从8 index 开始 
    * 方法缓存 cache_t 是一个散列表（算法），提高方法查找速度，objc_cache.m cache::insert
        ```
        struct cache_t {
            struct bucket_t *_buckets; // 散列表
            mask_t _mask; // 散列表长度减1后的值，如果该值为3，说明表长度是4，扩容策略 旧长度*2
            mast_t occupied; // 已经缓存的方法数量
        }
        @selector(test) & _mast = index, 这个索引肯定小于等于 mask 值
        
        objc_cache.m
        mask_t m = capacity - 1;
        mask_t begin = cache_hash(sel, m);
        
        void cache_t::insert(SEL sel, IMP imp, id receiver) {
                capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE; // 扩容策略
        }
        
        static inline mask_t cache_hash(SEL sel, mask_t mask) 
        {
            uintptr_t value = (uintptr_t)sel;
            return (mask_t)(value & mask);
        }
        
        static inline mask_t cache_next(mask_t i, mask_t mask) {
            return i ? i-1 : mask; // 出现散列碰撞，则 index - 1 处理，判断 key 与查找的是否相等
        }
        
        struct bucket_t {
        // explicit_atomic 显示原子性，目的是为了能够 保证 增删改查时 线程的安全性
            explicit_atomic<SEL> _key; // SEL 作为 key
            explicit_atomic<uintptr_t> _imp; // 函数的内存地址
        }
        ```
        * 介绍 OC 消息机制。OC 的方法调用：消息机制，给方法调用者发送消息，3个阶段：消息发送；动态方法解析；消息转发
            * OC 的方法查找顺序：先从 cache 找，然后从 class_rw_t 中找，然后从父类中找，找到的方法都会放到当前类的缓存中，最后调用 resolveInstanceMethod 给机会添加，如果还没有就消息转发
            ```
            void c_other(id self, SEL _cmd) {
            
            }
            
            + (BOOL)resolveInstanceMethod:(SEL)sel {
                if (sel == @selector(test)) {
                    class_addMethod(self, self, (IMP)c_other, "v16@0:8");
                    return YES;
                }
                return [super resolveInstanceMethod:sel];
            }
            ```
        * 进入消息转发阶段会进入 `__forwading__`
            ```
            -/+ (id)forwardingTargetForSelector:(SEL)aSelector {
                if (aSelector == @selector(test) {
                    return nil; // 返回 nil，以下方法签名处理。返回其他对象，则其他对象处理
                }
                return [super forwardingTargetForSelector:aSelector];
            }
            
            // 方法签名：返回值类型，参数类型
            -/+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
                if (aSelector == @selector(test) {
                    return [NSMethodSignature signatureWithObjCTypes:"v16@0:8"]; // 返回值为nil，因为 doesNotRecognizerSelector: crash；如果有返回值，则进入以下方法，数字可省略，可以为 "v@:"
                }
                return [super methodSignatureForSelector:aSelector]
            }
            
            // NSInvocation 封装一个方法调用，包含：方法调用者，方法名，方法参数
            // aInvocation.target 方法调用者
            // aInvocation.selector 方法名
            // [aInvocation getArgument:NULL at Index:0]
            // 找不到的方法，都会走到这里
            -/+ (void)forwardInvocation:(NSInvocation *)anInvocation {
                anInvocation.target = [Cat new];
                [anInvocation invoke];
            }
            ```
            
        * `@dynamic` 提醒编译器不要自动生成 setter 和 getter 的实现，不要生产成员变量，但不影响 setter 和 getter 的声明。
        * super 关键字告诉对象先从父类查找方法,消息接收着仍是当前对象
            ```
            [super run];
            
            struct objc_super = {
                id receiver; // 消息接收者
                Class super_class; // 父类
            };
            // 说明消息接收着仍为当前对象
            struct objc_super arg = {self, [Person class]};
            
            objc_msgSendSuper{arg, @selector(run)};
            ```
        * isKindOfClass, isMemberOfClass
            ```
            // 想获取元类用 object_getClass()
            + (Class)class {
                return self;
            }

            - (Class)class {
                return object_getClass(self);
            }
            // 当前类对象是否是元类对象的
            + (BOOL)isMemberOfClass:(Class)cls {
                return self->ISA() == cls;
            }
            // 当前实例对象是否是这个类对象的，
            - (BOOL)isMemberOfClass:(Class)cls {
                return [self class] == cls;
            }
            // 当前的实例对象是不是这个类对象，或者父类对象
            + (BOOL)isKindOfClass:(Class)cls {
            // 根类的父类对象是 nil
                for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass()) {
                    if (tcls == cls) return YES;
                }
                return NO;
            }

            - (BOOL)isKindOfClass:(Class)cls {
                for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
                    if (tcls == cls) return YES;
                }
                return NO;
            }
            ```
            
        * 以下代码执行结果：1.编译没问题，2 打印。栈区由高地址向地址增长，结构体内第一个成员变量在最低位地址。
            ```
            VeiwController.m
            // 打印结果是 "123"
            // 内存结构图
            - (void)viewDidLoad {
                [super viewDidLoad];
                
                NSString *obj2 = @"123";
                id cls = [Persion class];
                void *obj = &cls;
                [(__bridge id)obj print];
            }
            
            // 打印结果是 "ViewController"，原因是 super 调用问题，super 会被变化为 
            //  struct objc_super = {
            //    id receiver; // 消息接收者 ，低地址（也就是 self）
            //    Class super_class; // 父类，高地址（也就是 UIViewController）
            // };
            - (void)viewDidLoad {
                [super viewDidLoad];
                            
                id cls = [Persion class];
                void *obj = &cls;
                NSString *obj2 = @"123";
                [(__bridge id)obj print];
            }
            
            ```
            
        * super 的本质 objc_msgSendSuper2(), 虽然传入结构为 currentclass ，内部实现会取 current 的 superclass。clang C++ 重写后是 objc_msgSendSuper2，说明 clang 重写与实际略有差异，不影响分析。
        * LLVM 中间代码 （IR） 
            * OC 在变为机器代码之前，会被 LLVM 编译器转为中间代码（Intermediate Representation）(IR)
                * `clang -emit-llvm -S main.m` 可生成中间代码 .ll
            * 中间代码语法（https://llvm.org/docs/LangRef.html)
                * @ - 全局变量；
                * % - 局部变量；
                * alloca - 在当前执行的函数的堆栈帧中分配内存，当该函数返回到其调用者时，将释放内存
                * i32 - 32 位4字节的整数
                * align - 对齐
                * load - 读出，store 写入
                * icmp - 两个整数值比较，返回布尔值
                * br - 选择分之，根据条件来转向 label，不根据条件跳转的话类似 goto
                * label 代码标签
                * call - 调用函数
        * OC 是一门动态性比较强的编程语言，允许很多操作推迟到程序运行时在进行；OC 的动态性就是 runtime 来支撑和实现，runtime 是一套 C 语言的 API，封装了很多动态性相关的函数，平时编写的 OC 代码都是转换成了 Runtime API 进行调用。
        * Runtime 的实际用法：利用关联对象给分类添加属性；遍历类的成员变量（修改textfield的占位文字颜色，字典转模型，自动归档解档）；交换方法实现（hook）；利用消息转发机制解决方法找不到 crash 的问题；事件埋点上报；
        * method_exchangeImplementation(method1, method2) 交换的是方法的具体实现，也就是函数指针 IMP，一旦该方法调用，就会立即清理掉 cache_t（调用 flushcache）
        * load 虽然执行一次，添加 dispatch_once 靠谱的原因，有可能谁不小心主动调用 +load

17. Runloop: 运行循环，在程序运行过程中循环做一些事情。应用范畴：定时器，PerformSelector; GCD Async mainqueue; 事件响应，手势识别，界面刷新；网络请求；AutoreleasePool;
    * 如果没有 runloop，举例：命令行工具无 runloop ，代码执行完毕立即退出。
    * 如果有了 runloop，不会马上退出，而是保持运行状态。
    * 基本作用：
        * 保持程序的持续运行；
        * 处理 App 中的各种事件；
        * 节省 CPU 资源，提高程序程序性能：该做事时做事，该休息时休息；
    * iOS 中有两套可访问的 Runloop，Foundation 中的 NSRunloop, Core Foundation 中的 CFRunloopRef。主要包括五大类：CFRunLoopRef; CFRunLoopModeRef; CFRunLoopSourceRef; CFRunLoopTimerRef; CFRunLoopObserverRef;
        * CFRunloopModeRef 代表 RunLoop 的运行模式
            * 一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source0/Source1/Timer/Observer
            * RunLoop 启动时只能选择其中一个 Mode，作为 currentMode
            * 如果需要切换Mode，只能退出当前Loop，再选择一个Mode进入
                * 不同组的 Source0/Source1/Timer/Observer能分割开来，互不影响
            * 如果 Mode 里没有任何 Source0/Source1/Timer/Oberver, runloop 会立刻退出
            * 常见的2中Mode，NSDefaultRunLoopMode, UITrackingRunLoopMode
        * OC Foundation 与 CF Foundation 拿到的 runloop 地址值不同，因为NSRunloop 是对 CFRunloop 的一层包装
        ```
        struct __CFRunLoopMode {
            CFMutableSetRef _sources0;
            CFMutableSetRef _sources1;
            CFMutableArrayRef _observers;
            CFMutableArrayRef _timers;
        }
        typedef struct __CFRunLoop * CFRunLoopRef;
        struct __CFRunLoop {
            CFMutableSetRef _commonModes;
            CFMutableSetRef _commonModeItems;
            CFRunLoopModeRef _currentMode;
            CFMutableSetRef _modes;
        };
        ```
        * Source0
            * 触摸点击事件
            * PerformSelector:onThread:
        * Source 1
            * 基于 Port 的线程间的通信
        * Timers
            * NSTimer
            * performSelector:afterDelay:
        * Observers
            * 用于监听 RunLoop 的状态
            * UI 刷新 （BeforWaiting)
            * AutoreleasePool (BeforWaiting)
        ```
            CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {};
            
            CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
        ```
        * dipatch_async(dispatch_get_main_queue(), ^{}) 是依赖 runloop 的，被 `__CFRunLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__` ，和 JS EventLoop 异步处理方式一致。
        * JS Event Loop: 异步执行的运行机制如下。（同步执行也是如此，因为它可以被视为没有异步任务的异步执行。）
            * 所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。主线程从"任务队列"中读取事件，这个过程是循环不断的，所以整个的这种运行机制又称为Event Loop（事件循环）。
            * 主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。
            * 一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。
            * 主线程不断重复上面的第三步。
        * 调用 `__CFRunLoopServiceMachPort`（等待别的消息来唤醒线程） 使 runloop 休眠，CPU 让出时间片，mach_msg 内核 API
        * RunLoop 休眠的实现原理：**用户态**调用 mach_msg() 与**内核态**进行通信，内核决定使其进入休息；等有事件进入时，内核态通过 mach_msg() 与用户态进行通信，是用户态进行消息处理
            * 等待，休息
                * 没有消息就让线程休息
                * 有消息就唤醒线程
        * 线程保活: [NSRunloop Current] addPort:[NSMachPort new]; 
            ```
            - (void)viewDidLoad {
                __weak typeof(self) weakSelf = self;
                self.thread = [NSThread alloc] initWithBlock:^{
                    [[NSRunLoop currentLoop] addPort:[NSPort new] forMode: defaultMode];
                    // 该方法相当于开启了无限的循环，无法被停下
                    [[NSRunLoop currentLoop] run];
                    NSLog(@"end");
                    
                    
                    // 优化方式，
                    while(weakSelf && !weakself.isStopped) {
                        [NSRunLoop currentLoop] runMode:NSDefaultRunLoopMOde BeforDate:[NSDate distanceFuture]];
                    }
                }];
            }
                // 停止本次 runloop
                CFRunLoopStop(CFRunLoopGetCurrent());
            
            ```
            * NSThread addTarget: 是强引用，会造成无法是释放，可采用 NSThread initWithBlock
            * 该情况下线程仍然没有消失，原因是代码被卡在了 addPort: 这里，若想停止，则停止该线程的 runloop（CFRunLoopStop(CFGetCurrentRunLoop)）
    * runloop 实现的功能： [深入理解runloop](https://blog.ibireme.com/2015/05/18/runloop/)
        * autorealeasePool
        * 事件响应：
            *  苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。
            * 当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event， Event, 先告诉source1（mach_port）,source1 唤醒 RunLoop, 然后将事件 Event 分发给 source0, 然后由source0来处理。并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。
            * _UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。
            * SpringBoard 其实是一个标准的应用程序，这个应用程序用来管理 iOS 的主屏幕；
            * source1 是苹果用来监听 mach port 传来的系统事件的，source0 是用来处理用户事件的。source1 收到系统事件后，会回调 source0，所以最终这些事件都是由 source0 处理的。
        * 手势识别： 
              * 当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。
              * 苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个 Observer 的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行 GestureRecognizer 的回调。
              * 当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。
        * 响应链：
            * 响应连：由最基础的view向系统传递，first view -> super view -> ... -> view controller -> window -> Application -> AppDelegate
            * 传递链：有系统向最上层view传递，Application -> window -> root view -> ... -> first view
        * 界面刷新
            *  当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。
            * 苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。
        * 定时器
            * NStimer 注册到 runloop 后，不会太准确回调，错过就等下一次。CADisplayLink 是和屏幕刷新率一致的定时器。如果一帧被跳过就会卡顿。
        * PerformSeletor
            * 当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。
            *  当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。
        * AFNetworking 2.0 时代；生成 runloop ，通过添加 NSMachPort 方式，并不接受事件，只为了不退出。当需要这个线程执行任务则将任务放到该 runloop。不同线程间 Notification 的转发处理
        * AsyncDisplayKit；将计算放于其他线程，在主线程 runloop 添加Observer，监听 runloop beforwait 和 exit，收到回调，遍历队列，更新。
            
18. 多线程
    * 同步，异步，并发，并行，串行；dispatch_sync 和 dispatch_async 用来控制是否要开启新的线程； 
        * 同步和异步主要影响：能不能开启新的线程
            * 同步：在**当前**线程中执行任务，**不具备**开启新线程的能力
            * 异步：在**新的**线程中执行任务，**具备**开启新线程的能力
        * 并发和串行主要影响：任务的执行方式
            * 并发：多个任务同时执行
            * 串行：一个任务执行完毕后，再执行下一个任务
        * 线程与队列说不清道不明的关系： 线程是代码执行的路径，队列则是用于保存以及管理任务的，线程负责去队列中取任务进行执行。
    * 使用 sync 函数往**当前串行**队列中添加任务，会阻塞当前的串行队列（产生死锁）。 阻塞当前线程立即执行任务，但任务已被添加到当前队列队尾，当前任务没有执行完毕，导致无法取出任务，死锁。
    * 子线程延时执行任务；子线程执行完任务就会退出，在退出的线程上再执行任务会出现 crash 现象。
        ```
        dispatch_async(queue, ^{
            nslog(1);
            [self performselector:test withobject:nil afterdelay:.0]
            nslog(3);
            
            // 这行代码可以被注释的原因是 perform after 相当于已经往runloop添加了定时器
            // nsport 相当于添加 source1
            // runloop 无 source，timer，oberver 就会退出
            // [nsrunloop currentrunloop] addport:[nsport new] formode:nsdefaultmode
            [[nsrunloop currentloop] runmode:nsdefaultloopmode befordate:[nsdate distancefuture]]
        });
        ```
    * iOS 线程同步方案
        * 锁 OSSpinlock 自旋锁，等待锁的线程会处于**忙等状态**，一直占有 CPU 资源，存在优先级反转问题。低优先级拿到锁做事，高优先级一直占用 CPU 导致低优先级无法完成任务释放锁，那么会处于死锁状态。（另外方式可以直接让线程休眠，方式会更好）
        * os_unfair_lock 用于取代 OSSpinlock，Low-Level lock(无事做就睡觉) 从 iOS 10 开始才支持，从底层调用看，等待 os_unfair_lock 的锁的线程处于休眠状态，并非忙等。汇编调用 syscall）`import <os/lok.h>`，忘记解锁，会导致死锁
        * pthread_mutex 叫做 互斥锁，等待锁的线程处于休眠状态（汇编调用 syscall） `import <pthread>`，同一线程反复加锁，也会导致死锁。
        * pthread_mutex 改变 attri 改为 递归锁也可解决问题。递归锁，允许同一个线程反复加锁。
        * pthread_mutex 添加条件，pthread_cond_wait() 后睡眠并且会解锁。pthread_cond_signal() 会唤醒，可应用于生产者消费者模式,线程唤醒
        
        * NSLock 对 pthread_mutex 普通模式的封装；NSRecursiveLock 是对 pthread_mutex 递归模式的封装；NSCondition 是对 pthread_mutex 条件锁的封装；NSConditionLock 是对 NSCodition 的进一步封装
        * 线程的任务一旦执行完毕，生命周期就结束，无法再使用；保住线程的命为什么用 runloop，用强指针不就可以吗？答：准确来讲，使用runloop是为了保持线程处于激活状态。
        * 串行队列也可
        * dispatch_semaphore 叫做信号量，信号量的初始值，可以用来控制线程并发访问的最大数量。
        * `@synchronized` 是对 mutex 的封装；源码 objc-sync.mm , 内部 hash 表，对象地址为key
        * 什么情况使用自旋锁划算
            * 预计线程等待锁的时间很短
            * 加锁的代码（临界区）经常被调用，但竞争情况很少发生
            * CPU 资源不紧张
            * 多核处理器
        * 什么情况互斥锁划算
            * 预计线程等待锁的时间较长
            * 单核处理器
            * 临界区有 IO 操作
            * 临界区竞争激烈
        * atomic 用于保证属性 setter，getter的原子性操作，相当于在getter和setter里加锁；可以参考 objc-accessors.mm；它并不能保证使用属性的过程是线程安全的
    * iOS 的读写安全方案，IO 操作，文件操作
        * 同一时间，只能有一个线程在写入；同一时间，允许有多个线程进行读的操作；同一时间，不允许既有写的操作，又有读的操作。
        * 上面场景就是典型的“多读单写”，经常用于文件等数据的读写操作，iOS 的实现方案有：
            * pthread_rwlock: 读写锁，等待锁的线程会进入休眠
            * dispatch_barrier_async: 栅栏，
                * 这个函数传入的并发队列必须是自己通过 dispatch_queue_create 创建的
                * 如果传入的是一个串行或是一个全局的并发队列，那这个函数便等同于 dispatch_async 函数的效果
                ```
                // 初始化队列
                dispatch_queue_t rwqueue = dispatch_queue_create("rw_queue", DISPATCH_QUEUE_CONCURRENT);
                // 读
                dispatch_async(rwqueue, ^{
                        
                });
                // 写
                dispatch_barrier_async(rwqueue, ^{
        
                });
                
                ```

19. 内存管理
    * NSTimer, CADisplayLink 注意的问题，他们是对 target 强引用的。强引用是在 Timer 内部实现的，外部传入 weakself 无用，直接用 block 那个方法解决。
    * 创建代理类，代理类弱引用 target，再通过消息转发解决。NSProxy 效率更高，会直接忽略消息搜索和查找的阶段，直接就转发。
    * NSTimer 依赖 runloop , 如果 runloop 的任务过于繁琐，可能会导致 NSTimer 不准时
    * GCD 的定时器更加准时
    * iOS 程序的内存布局：地址由低到高分别是：保留区，代码段（`__TEXT__`)，数据段（`__DATA__` 包含字符串常量，已初始化数据，未初始化数据)，堆（heap 向高地址区增长），栈（stack 向低地址区增长），内核区。
        * 栈：函数调用的开销
        * 堆：通过 alloc，malloc，calloc 等动态分配的空间，分配的内存空间地址越来越大
    * Tagged Pointer
        * 从 64bit开始，iOS 引入了tagged pointer 技术，用于优化 NSNumber, NSDate, NSString等小对象的存储
        * 在没有使用 tagged pointer 之前，NSNumber 等对象需要动态分配内存、维护引用计数等，NSNumber 指针存储的是堆中 NSNumber 对象的地址值
        * 使用 tagged pointer 之后，NSNumber 指针里面存储的数据变成了 Tag + Data，也就是将数据直接存储在了指针中
        * 当指针不够存储数据时，才会使用动态分配内存的方式来存储数据
        * objc_msgSend 能识别 Tagged Pointer，比如 NSNumber 的 intvalue 方法，直接从指针提取数据，节省了以前的调用开销
        * iOS 平台指针地址最高有效位为1，说明为 tagged pointer。mac 平台 最低有效位为 1 说明为 tagged pointer。
    * OC 对象的内存管理
        * 内存泄漏，该释放的对象没有释放 场景举例：
            * 代理的属性关键字设置为strong造成的内存泄漏
            * CoreGraphics框架里申请的内存忘记释放，使用静态分析可以轻易发现
            * CoreFoundation框架里申请的内存忘记释放，使用静态分析可以轻易发现。
            * NSTimer 不正确使用造成的内存泄漏
                * NSTimer重复设置为NO的时候，不会引起内存泄漏
                * NSTimer重复设置为YES的时候，有执行invalidate就不会内存泄漏，没有执行invalidate就会内存泄漏，在 timer的执行方法里调用invalidate也可以。
                * 中间target：控制器无法释放，是因为timer对控制器进行了强引用，使用类方法创建的timer默认加入了runloop，所以，timer只要不持有控制器，控制器就能释放了。
                * 通知和KVO、block循环引用 、NSThread
            * Instrument leaks 实现思路
                * Leaks的实现思路是搜索所有可能包含指向malloc内存块指针的内存区域，比如全局数据内存块，寄存器和所有的栈。如果malloc内存块的地址被直接或者间接引用，则是reachable的，反之，则是leaks.
                * Leaked memory: Memory unreferenced by your application that cannot be used again or freed (also detectable by using the Leaks instrument).
                * Abandoned memory: Memory still referenced by your application that has no useful purpose
                * Cached memory: Memory still referenced by your application that might be used again for better performance.
                * 其中 Leaked memory 和 Abandoned memory 都属于应该释放而没释放的内存，都是内存泄露，而 Leaks 工具只负责检测 Leaked memory，而不管 Abandoned memory。在 MRC 时代 Leaked memory 很常见，因为很容易忘了调用 release，但在 ARC 时代更常见的内存泄露是循环引用导致的 Abandoned memory，Leaks 工具查不出这类内存泄露，应用有限。
            * MLeaksFinder 开源库，为基类 NSObject 添加一个方法 -willDealloc 方法，该方法的作用是，先用一个弱指针指向 self，并在一小段时间(3秒)后，通过这个弱指针调用 -assertNotDealloc，而 -assertNotDealloc 主要作用是直接中断言。缺点只能基于 UIViewController
            
        * 一个新创建的 OC 对象引用计数默认时1， 当引用计数减为0，OC对象就会销毁，释放其占用的内存
        * 调用 retain 会让 OC 对象的引用计数 +1， 调用 release 会让 OC 对象的引用计数 -1
        * 内存管理的经验：当调用 alloc、new、copy、mutableCopy 方法返回一个对象，在不需要这个对象时，要调用 release 或者 autorelease 来释放它
        * 想拥有某个对象，就让它的引用计数+1；不想再拥有某个对象，就让它的引用计数-1
        * 可以通过 `extern void _objc_autoreleasePoolPrint(void);` 查看自动释放池的情况
        * copy 对可变集合对象进行操作就是深拷贝（内容复制，有产生新对象），返回不可变集合对象，对不可变集合对象进行操作就是浅拷贝（指针复制，没有产生新对象）；mutableCopy 对集合类型都是深拷贝，返回可变集合对象。
    * NSZombie
        * `__dealloc_zombie`背后的逻辑 对其进行符号断点
            * object_getClass 获取当前对象的class;
            * class_getName 获取当前对象class对应的字符串；
            * 进行字符串拼接_NSZombie_%s，通过objc_lookUpClass寻找是否已经存在zombie的类，比如_NSZombie_Student;
            * 如果不存在的话，则重新创建，先获取通过objc_lookUpClass模板类_NSZombie_；
            * 然后通过objc_duplicateClass，基于类_NSZombie_，复制一个副本_NSZombie_Student；
            * 不同于 objc_allocateClassPair 方法，该方法是基于一个父类，重建一个子类，而 objc_duplicateClass 只是拷贝一个副本，不是父类子类的关系，所以在之前获取 _NSZombie_Student 的 superclass 的是 nil；
            * 然后通过objc_destructInstance将该对象的关联全部解除，对象的内存其实没有回收掉；
            * 最后调用object_setClass将原对象Student的isa指向了_NSZombie_Student；
    
    * Address Sanitizer（地址消毒剂）![地址](https://easeapi.com/blog/84-address-sanitizer.html)
        * Address Sanitizer替换了malloc和free的实现。当调用malloc函数时，它将分配指定大小的内存A，并将内存A周围的区域标记为”off-limits“。当free方法被调用时，内存A也被标记为”off-limits“，同时内存A被添加到隔离队列，这个操作将导致内存A无法再被重新malloc使用。代码中所有的内存访问操作都被编译器转换为如下形式：
        ```
        // Before
        *address = ...;  // or: ... = *address;

        // After
        if (IsMarkedAsOffLimits(address)) {
          ReportError(address);
        }
        *address = ...;  // or: ... = *address;
        ```
        * 当访问到被标记为”off-limits“的内存时，Address Sanitizer就会报告异常。
        * Address Sanitizer可以用来检测如下内存使用错误:
            * 内存释放后又被使用；
            * 内存重复释放；
            * 释放未申请的内存；
            * 使用栈内存作为函数返回值；
            * 使用了超出作用域的栈内存；
            * 内存越界访问；
        *Address Sanitizer vs Zombie Objects
            * 我们知道Xcode还有一个内存检测工具Zombie Objects。那么他们有什么区别？

            * 开启Zombie Objects后，dealloc将会被hook，被hook后执行dealloc，内存并不会真正释放，系统会修改对象的isa指针，指向_NSZombie_前缀名称的僵尸类，将该对象变为僵尸对象。

            * 僵尸类做的事情比较单一，就是响应所有的方法：抛出异常，打印一条包含消息内容及其接收者的消息，然后终止程序。
            * 由此可见，Zombie Objects是无法检测内存越界的，Address Sanitizer比Zombie Objects有更强大的捕捉内存问题的能力。当然，Address Sanitizer的不足上面也说了，会占用较多的内存资源，对性能影响较大。
        
    * 引用计数存储
        * 在 64 bit 中，引用计数可以直接存储在优化过的 isa 指针中，也可以存储在 Side Table 类中
        * 当一个对象要释放时，会自动调用 dealloc，接下来的调用轨迹是
            * dealloc, _objc_rootDealloc , rootDealloc, object_dispose, objc_destructInstance、free
        ```
        typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,RefcountMapValuePurgeable> RefcountMap;
        
        struct SideTable {
            spinlock_t slock;
            RefcountMap refcnts; // 是一个存放着对象引用计数的散列表, key为 objc_object。
            weak_table_t weak_tabel; // 弱引用表散列表，key 是 （原对象&mask），弱引用表清除
        }
        
        objc-object.h （objc_object::release(), objc_object::retain(), objc_object::rootRetainCount(), objc_object::rootDealloc()）
        
        inline uintptr_t 
        objc_object::rootRetainCount()
        {
            if (isTaggedPointer()) return (uintptr_t)this;

            sidetable_lock();
            isa_t bits = __c11_atomic_load((_Atomic uintptr_t *)&isa().bits, __ATOMIC_RELAXED);
            if (bits.nonpointer) { // 优化过的 isa
                uintptr_t rc = bits.extra_rc; 
                if (bits.has_sidetable_rc) { // 引用计数除存储在 extra_rc 中，sidetable 也有
                    rc += sidetable_getExtraRC_nolock();
                }
                sidetable_unlock();
                return rc;
            }

            sidetable_unlock();
            return sidetable_retainCount();
        }
        
        size_t 
        objc_object::sidetable_getExtraRC_nolock()
        {
            ASSERT(isa().nonpointer);
            SideTable& table = SideTables()[this];
            RefcountMap::iterator it = table.refcnts.find(this);
            if (it == table.refcnts.end()) return 0;
            else return it->second >> SIDE_TABLE_RC_SHIFT;
        }
        
        void *objc_destructInstance(id obj)
        {
            if (obj) {
                // Read all of the flags at once for performance.
                bool cxx = obj->hasCxxDtor();
                bool assoc = obj->hasAssociatedObjects();

                // This order is important.
                if (cxx) object_cxxDestruct(obj); // 清除成员变量
                if (assoc) _object_remove_associations(obj, /*deallocating*/true);
                obj->clearDeallocating(); // 将指向当前对象的弱指针置为 nil
            }

            return obj;
        }
        ```
    * ARC 做了什么
        * 在编译期，ARC用的是更底层的C接口实现的retain/release/autorelease，这样做性能更好
        * 运行期：主要是指 weak 关键字。weak 修饰的变量能够在引用计数为0 时被自动设置成 nil，显然是有运行时逻辑在工作的。
        * 运行期：为了保证向后兼容性，ARC 在运行时检测到类函数中的 autorelease 后紧跟其后 retain，此时不直接调用对象的 autorelease 方法，而是改为调用 objc_autoreleaseReturnValue。  若那段代码要在返回对象上执行 retain 操作，则设置全局数据结构中的一个标志位，而不执行 autorelease 操作，与之相似，如果方法返回了一个自动释放的对象，而调用方法的代码要保留此对象，那么此时不直接执行 retain ，而是改为执行 objc_retainAoutoreleasedReturnValue函数。此函数要检测刚才提到的标志位，若已经置位，则不执行 retain 操作，设置并检测标志位，要比调用 autorelease 和 retain 更快。
    * weak 的实现原理
        * 创建以 objc_initweak 为入口，内部调用 storeWeak，通过引用计数表 sidetable 散列表中的 weak_table_t 全局 weak hash 表实现存储，以对象地址为key，weak 修饰的变量为 value 进行存储，如果对象不是 taggedpointer 类型，会对对象的位域添加 weak 的标识，该标识在 对象 dealloc 的时候使用。
        * dealloc 阶段最终通过 key 查找到 weak_table 中的 weak_entry_t 对用弱引用指针进行 memset  操作，也就是赋值为 nil。这个阶段也会判断该对象 isa 指针是否有 has_sidetable_rc, 决定对引用计数表进行擦除。
        * 释放时的顺序 dealloc, _objc_rootDealloc , rootDealloc, object_dispose, objc_destructInstance、free
        * 调用最终以下函数实现销毁
        ```
        NEVER_INLINE void
        objc_object::clearDeallocating_slow()
        {
            ASSERT(isa().nonpointer  &&  (isa().weakly_referenced || isa().has_sidetable_rc));

            SideTable& table = SideTables()[this];
            table.lock();
            if (isa().weakly_referenced) {
                weak_clear_no_lock(&table.weak_table, (id)this);
            }
            if (isa().has_sidetable_rc) {
                table.refcnts.erase(this);
            }
            table.unlock();
        }
        ```
    * 自动释放池
        * 主要底层数据结构：`__AtAutoreleasePool`、`AutoreleasePoolPage`
        * 调用了 autorelease 的对象最终都是通过 AutoreleasePoolPage 对象管理的
        * 每个 autoreleasepoolpage 占用 4096 个字节，除了用来存放它内部的成员变量，剩下的空间用来存放autorelease 对象的地址。
        * 所有的 autoreleasepoolpage 对象通过双向链表的形式连接起来，存满了就新开辟 page 进行存储
        * 调用 push 方法会将一个 POOL_BOUNDARY 入栈(不是指的堆区，还是栈区，指的结构)，并且返回其存放的内存地址
        * 调用 pop 方法时会传入一个 POOL_BOUNDARY 的内存地址，会从最后一个入栈的对象开始发送release消息，直到遇到这个 POOL_BOUNDARY
        * `id *next` 指向了下一个能存放 autorelease 对象地址的区域
        * hot page 当前在使用的 page
        ```
        NSObject.mm
        class AutoreleasePoolPage : private AutoreleasePoolPageData
        
        struct AutoreleasePoolPageData
        {
            magic_t const magic;
            __unsafe_unretained id *next;
            objc_thread_t const thread;
            AutoreleasePoolPage * const parent;
            AutoreleasePoolPage *child;
            uint32_t const depth;
            uint32_t hiwat;
        }
        
        id * begin() {
            return (id *) ((uint8_t *)this+sizeof(*this)); // 当前page的容量，也就是接下来存其他对象的起始地址
        }
        ```
    * runloop 和 autorelease
        * iOS 在主线程的 Runloop 中注册了2个 Oberver
        * 第1个 Observer 监听 kCFRunLoopEntry 事件，会调用 objc_autoreasePoolPush()
        * 第2个 Oberver 监听 kCFRunLoopBeforWaiting 事件，会调用 objc_autoreasePoolPop()、objc_autoreleasePoolPush()
        * 监听 kCFRunLoopBeforExit 事件，会调用 objc_autoreasePoolPop()
    * 子线程 autoreleasepool，在子线程中原本是没有自动释放池的，但是如果有runloop或者autorelease对象的时候，就会自动的创建自动释放池，通过 autoreleasepoolnopage 每个autorelease创建的时候都会监听当前线程的销毁方法，在线程退出时调用 tls_dealloc 方法。
    * 每使用一次`__weak`对象，运行时系统都会将其指向的原始对象先retain，之后保存到自动释放池中（ AutoReleasePoolPage的add() 函数）。因此如果大量调用`__weak`对象，则会重复进行此工作。不仅耗费无意义的性能（重复存储同一对象），还会使内存在短时间内大量增长。
    ```
            id
        objc_loadWeak(id *location)
        {
            if (!*location) return nil;
            return objc_autorelease(objc_loadWeakRetained(location));
        }
    ```
    
    * 局部变量在出了作用域就会释放吗？1. ARC 是，2. MRC 需要自管理
    
20. 性能优化
    * 在屏幕成像的过程中，CPU和GPU起着至关重要的作用
    * 在iOS中是双缓冲机制，有前帧缓存、后帧缓存
        * CPU: 对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制（Core Graphics）
        * GPU: 纹理的渲染
    * 卡顿产生的原因
        * 垂直信号到来前，CPU 和 GPU 没有完成工作，显示上一帧信息
        * 卡顿解决的主要思路
            * 尽可能减少CPU、GPU资源消耗
            * 按照60FPS的刷帧率，每隔16ms就会有一次VSync信号
    * 卡顿优化 - CPU 
        * 尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用CALayer取代UIView
        * 不要频繁地调用UIView的相关属性，比如frame、bounds、transform等属性，尽量减少不必要的修改
        * 尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性
        * Autolayout会比直接设置frame消耗更多的CPU资源
        * 图片的size最好刚好跟UIImageView的size保持一致
        * 控制一下线程的最大并发数量
        * 尽量把耗时的操作放到子线程
            * 文本处理（尺寸计算、绘制）
            * 图片处理（解码、绘制）
    * 卡顿优化 - GPU
        * 尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示
        * GPU能处理的最大纹理尺寸是4096x4096，一旦超过这个尺寸，就会占用CPU资源进行处理，所以纹理尽量不要超过这个尺寸
        * 尽量减少视图数量和层次
        * 减少透明的视图（ `alpha < 1` ），不透明的就设置opaque为YES
        * 尽量避免出现离屏渲染
    * 离屏渲染
        * 在OpenGL中，GPU有2种渲染方式
            * On-Screen Rendering：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
            * Off-Screen Rendering：离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作
        * 离屏渲染消耗性能的原因
            * 需要创建新的缓冲区
            * 离屏渲染的整个过程，需要多次切换上下文环境，先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕
        * 哪些操作会触发离屏渲染
            * 光栅化，layer.shouldRasterize = YES
            * 遮罩，layer.mask
            * 圆角，同时设置layer.masksToBounds = YES、layer.cornerRadius大于0
                * 考虑通过CoreGraphics绘制裁剪圆角，或者叫美工提供圆角图片
            * 阴影，layer.shadowXXX
                * 如果设置了layer.shadowPath就不会产生离屏渲染
    * 卡顿检测
        * 平时所说的“卡顿”主要是因为在主线程执行了比较耗时的操作
        * 可以添加Observer到主线程RunLoop中，通过监听RunLoop状态切换的耗时，以达到监控卡顿的目的

21. 启动优化
    * 静态库和动态库的区别(https://zhuanlan.zhihu.com/p/71372182)
        * 可执行文件大小不一样
            * 静态链接的可执行文件要比动态链接的可执行文件要大得多，因为它将需要用到的代码从二进制文件中“拷贝”了一份，而动态库仅仅是复制了一些重定位和符号表信息。
        * 占用磁盘大小不一样
            * 如果有多个可执行文件，那么静态库中的同一个函数的代码就会被复制多份，而动态库只有一份，因此使用静态库占用的磁盘空间相对比动态库要大。
        * 扩展性与兼容性不一样
            * 如果静态库中某个函数的实现变了，那么可执行文件必须重新编译，而对于动态链接生成的可执行文件，只需要更新动态库本身即可，不需要重新编译可执行文件。正因如此，使用动态库的程序方便升级和部署。
        * 依赖不一样
            * 静态链接的可执行文件不需要依赖其他的内容即可运行，而动态链接的可执行文件必须依赖动态库的存在。所以如果你在安装一些软件的时候，提示某个动态库不存在的时候也就不奇怪了。
            * 即便如此，系统中一般存在一些大量公用的库，所以使用动态库并不会有什么问题。
        * 复杂性不一样
            * 相对来讲，动态库的处理要比静态库要复杂，例如，如何在运行时确定地址？多个进程如何共享一个动态库？当然，作为调用者我们不需要关注。另外动态库版本的管理也是一项技术活。
        * 加载速度不一样
            * 由于静态库在链接时就和可执行文件在一块了，而动态库在加载或者运行时才链接，因此，对于同样的程序，静态链接的要比动态链接加载更快。所以选择静态库还是动态库是空间和时间的考量。但是通常来说，牺牲这点性能来换取程序在空间上的节省和部署的灵活性时值得的。再加上局部性原理，牺牲的性能并不多。
    * App 启动可以分为：冷启动，热启动
    * 启动分几个阶段 ![URL](https://mp.weixin.qq.com/s/3-Sbqe9gxdV6eI1f435BDg)
    * APP启动时间的优化，主要是针对冷启动进行优化
    * 通过添加环境变量可以打印出APP的启动时间分析（Edit scheme -> Run -> Arguments）
        * DYLD_PRINT_STATISTICS 设置为1
        * 如果需要更详细的信息，那就将 DYLD_PRINT_STATISTICS_DETAILS 设置为1
    * APP的冷启动可以概括为3大阶段 dyld, runtime, main
        * dyld, Apple的动态链接器，可以用来装载Mach-O文件（可执行文件、动态库等）; 启动APP时，dyld所做的事情有
            * 装载 APP 的可执行文件，同时会递归加载所有依赖的动态库
            * 对每个二进制做 bind 和 rebase，主要耗时在 Page In，影响 Page In 数量的是 objc 的元数据，Rebase 修正内部(指向当前mach-o文件)的指针指向； 而bind指向的是镜像外部的资源指针。之所以需要Rebase，是因为刚刚提到的ASLR使得地址随机化，导致起始地址不固定，另外由于Code Sign，导致不能直接修改Image。Rebase的时候只需要增加对应的偏移量即可。待Rebase的数据都存放在`__LINKEDIT`中。
            * 当dyld把可执行文件、动态库都装载完毕后，会通知Runtime进行下一步的处理
        * runtime, 启动APP时，runtime所做的事情有
            * 调用 map_images 进行可执行文件内容的解析和处理
            * +load 和静态初始化被调用，除了方法本身耗时，这里还会引起大量 Page In
            * 在 load_images 中调用 call_load_methods ，调用所有 Class 和 Category 的 +load 方法
            * 进行各种 objc 结构的初始化（注册Objc类 、初始化类对象等等）
            * 调用 C++ 静态初始化器和 `__attribute__((constructor))` 修饰的函数
            * 到此为止，可执行文件和动态库中所有的符号(Class，Protocol，Selector，IMP，…)都已经按格式成功加载到内存中，被runtime 所管理

        * 总结
            * APP的启动由dyld主导，将可执行文件加载到内存，顺便加载所有依赖的动态库
            * 并由runtime负责加载成objc定义的结构
            * 所有初始化工作结束后，dyld就会调用main函数
            * 接下来就是UIApplicationMain函数，AppDelegate的application:didFinishLaunchingWithOptions:方法
            
        * dyld 是启动的辅助程序，是 in-process 的，即启动的时候会把 dyld 加载到进程的地址空间里，然后把后续的启动过程交给 dyld。dyld 主要有两个版本：dyld2 和 dyld3。

        * dyld2 是从 iOS 3.1 引入，一直持续到 iOS 12。dyld2 有个比较大的优化是 dyld shared cache[1]，什么是 shared cache 呢？

        * shared cache 就是把系统库(UIKit 等)合成一个大的文件，提高加载性能的缓存文件。
        * iOS 13 开始 Apple 对三方 App 启用了 dyld3，dyld3 的最重要的特性就是启动闭包，闭包里包含了启动所需要的缓存信息，从而提高启动速度。
        * dyld2 和 dyld3 的主要区别：解析动态库的依赖关系，找到 bind & rebase 的指针地址，找到 bind 符号的地址, 注册 objc 的 Class/Method 等元数据，对大型工程来说，这部分耗时会很长
        * iOS 13 开始 Apple 对三方 App 启用了 dyld3，dyld3 的最重要的特性就是启动闭包，闭包里包含了启动所需要的缓存信息，从而提高启动速度。
        * wwdc22 dyld 新的特性来了，page-in linking。优点：减少脏内存， 干净的 DATA_CONST （意味可以回收重建，减少内存压力）
        * 支持 page-in linking 的动态库，在启动时的 fixups 阶段，并没有真正执行 fixups。而是等到 page-in 的时候才执行。
    * APP的启动优化
        * 按照不同的阶段
            * dyld
                * 减少动态库、合并一些动态库（定期清理不必要的动态库）
                * 减少Objc类、分类的数量、减少Selector数量（定期清理不必要的类、分类）
                * 减少C++虚函数数量
                * Swift尽量使用struct
            * runtime
                * 用+initialize方法和dispatch_once取代所有的 `__attribute__((constructor))`、C++静态构造器、ObjC的 `+load`
            * main
                * 在不影响用户体验的前提下，尽可能将一些操作延迟，不要全部都放在finishLaunching方法中
                * 按需加载
                
22. 安装包瘦身
    * 安装包（IPA）主要由可执行文件、资源组成
    * 资源（图片、音频、视频等）
        * 采取无损压缩
        * 去除没有用到的资源： https://github.com/tinymind/LSUnusedResources
    * 可执行文件瘦身
        * 编译器优化
            * Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES
            * 去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO， Other C Flags添加-fno-exceptions
        * 利用AppCode（https://www.jetbrains.com/objc/）检测未使用的代码：菜单栏 -> Code -> Inspect Code
        * 编写LLVM插件检测出重复代码、未被调用的代码
    * 生成LinkMap文件，可以查看可执行文件的具体组成
    * 可借助第三方工具解析LinkMap文件： https://github.com/huanxsd/LinkMap
    
23. 设计模式
    * 是一套被反复使用、代码设计经验的总结
    * 使用设计模式的好处是：可重用代码、让代码更容易被他人理解、保证代码可靠性
    * 一般与编程语言无关，是一套比较成熟的编程思想
    
    * 设计模式可以分为三大类
        * 创建型模式：对象实例化的模式，用于解耦对象的实例化过程
            * 单例模式、工厂方法模式，等等
        * 结构型模式：把类或对象结合在一起形成一个更大的结构
            * 代理模式、适配器模式、组合模式、装饰模式，等等
        * 行为型模式：类或对象之间如何交互，及划分责任和算法
            * 观察者模式、命令模式、责任链模式，等等
    * 七大设计法则（https://juejin.cn/post/6975728866430025741）
        * 我了解过的，七大设计原则的目的是降低对象之间的耦合，增加程序的可复用性、可扩展性和可维护性。可以说是面向对象程序设计必须要遵守的七大准则。
        * 七大设计原则有**开放封闭原则**，对扩展开放，对修改关闭，意思就是不修改已经写好的代码，只能写新的代码来扩展新的功能，它能够降低每次改动对系统带来的风险。**里氏替换原则**，子类可以扩展父类的功能，但不能改变父类原有的功能，具体的使用就是子类不要去重写父类中的方法，它规定了继承的使用方法，可以减少多态的滥用，也减少了父子类的耦合程度。**依赖倒置原则**，抽象不应该依赖细节，细节应该依赖抽象；简单来说就是它提倡我们使用接口编程，这样能够降低类之间的耦合程度，有利于代码的升级扩展。**单一职责原则**，一个类应该有且仅有一个引起它变化的原因，也就是一个类应该只做一件事，这样可以提高代码的阅读性，提高类的独立性。**接口隔离原则**，一个类对另一个类的依赖应该建立在最小的接口上；客户端程序不应该依赖它不需要的接口方法，也就是一个接口要精简单一，太复杂的接口应该拆分成小接口，它能够降低接口的复杂度，从而提高接口以及实现类的内聚。**迪米特法则**，一个软件实体应当尽可能少地与其他实体发生相互作用，每个类应该都只依赖于成员属性、方法的入参和返回值，其他的类都不应该直接产生依赖。**合成复用原则**，它要求在软件复用时，要尽量先使用组合或者聚合等关联关系来实现，其次才考虑使用继承关系来实现，因为继承复用是会增加两个类的耦合程度的，而合成复用是不会增加耦合程度的，更加的灵活，有利于降低代码的耦合程度。

24. crash 收集、处理
    > dSym指的是Debug Symbols。 也就是调试符号，我们常常称为符号表文件。 符号对应着类、函数、变量等，这个符号表文件是内存与符号如函数名，文件名，行号等的映射，在崩溃日志分析方面起到了举足轻重的作用。
    * 发生 crash，会将 crash report 存储在设备上。会有描述信息。low memory 无堆栈信息。
    * 源码转机器码会生成 Debug 符号表。Debug 符号表是映射表。存于 binary 信息或 dSYM 中。app+dSYM+UUID 必须是一套，否则无法工作。
    * archive 后 binary和 dSYM 存储。
    * 上传的 dSYM 对 testflight 用户和愿意分析诊断的客户很有必要。
    * 当 app crash 时，一个没有被符号化的 crash report 会存在设备。
    * 获取后通过 Xcode 通过 dSYM 和 二进制信息符号化。
    * App Store 可下载部分crash report。
    * App Store 会对crash分组聚合。
    * 未符号化，在堆栈信息看不到方法名和函数名。
    * atos 命令符号化 crash report
    * 分析crash report
    * 每个都有header，incident id: crash report 唯一ID。crashreprot key：匿名设备ID 。beta id：testflight。process: 发生crash进程名。可以和CFBundleExecutable匹配上。

25. OOM 总结：
    * 特征1：OOM根据主要场景分为 FOOM 和 BOOM，也就是前后台
    * 特征2：OOM的崩溃在手机本地会以 Jetsam 形式存在，FOOM对用户体验影响比较大
    * 特征3：OOM的 SIGKILL 无法在程序运行中被捕获
    * 如何确定OOM产生：可以通过排除法的手段去确定是否有 OOM 的产生
    * 线上监控：有了能够判断出OOM的手段后，可以进行线上监控，监控采用在程序静止时的单一线程中进行
    * 监控的具体手段是：采用mach内核中的vm region相关函数可以获得具体dirty内存节点信息
    * 符号化：通过 user_tag 将 vm region 的内存节点符号化，便于提取详细信息
    * 建立内存快照：引用关系通过遍历内存节点的相互引用建立快照
    * 指定合理的数据上报策略：Memory Graph上报流程图
    * 后台分析：跟埋点上报分析差不多类似
    
26. Pod install 流程：
    * 读取Podfile文件，分析有哪些第三方依赖库及对应的版本
    * 加载源文件，每个xxx.podspec.json文件都包含一个源代码的索引，这些索引包含一个Git地址和Git tag，它们以commit SHAs的方式存在在 ~/Library/Caches/CocoaPods中，这个路径中文件的创建是由Core gem负责的，CocoaPods将依照Podfile、.podspec和缓存文件的信息将源文件下载到pods目录中
    * 生成Pods.xcodeproj，每次pod install指向，如果检测到改动时，CocoaPods会利用Xcodeproj gem组件对Pods.xcodeproj进行更新，该文件如果不存在，则用默认配置生成，否则会将已有的配置项加载至内存中
    * 安装第三方库，当CocoaPods往工程中添加一个三方库时，不只是添加代码这么简单，还会添加很多内容，由于每个第三方库有不同的Target，因此对于每个库，都会有几个文件需要添加，每个target都需要:
   
27.  `.xcodeproj` 和 `.xcworkspace` 不同：
    * `.xcodeproj`包含文件（代码/资源）、设置和从这些文件和设置构建产品的 Target 。工作区包含可以相互引用的项目。`.xcworkspace` 项目集合，当项目之间存在相关性时，组织
    * 两者都负责构建整个项目，但级别不同。
    * Xcode 3 引入 subproject，父子关系，父可以引用子 target，反之不行。
    * Xcode 4 引入 xcworkspace, 兄弟关系，意味着在一个 workspace 里任意项目可以引用其他

28. po 和 p 区别
    * p 和 po 在 lldb 使用时都涉及到编译代码，不同的是 po 在编译代码执行阶段获取的是 description 字符串描述。p 则是直接对变量进行 formatter，
    * p 打印原始变量的值或引用的值，po 尝试为该对象调用 -description 并打印返回的字符串
            
