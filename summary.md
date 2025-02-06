## Chapter_one

### 1、视C++为一个语言集合：
包括：1）：C(即普通的C语言面向过程)；2）：OOP（面向对象-class）;
3）：泛型编程（template）；4）：STL(标准函数库)。

### 2、尽量用const、inline、enum代替#define:
对于单纯常量，我们可以用const或enum代替#define
对于类似于函数的宏，我们可以用inline代替#define

### 3、尽可能使用const:
char greeting[]=“Hello”;
char *p=greeting;   //non-c,non-c-data
const char *p=greeting; //non-c,c-data
char* const p=greeting; //c-pointer,n-c-data
const char* const p=greeting;   //c-pointer,c-data

**const最具威力的用法是面对函数声明的应用：**
const在函数前的声明可以避免一些非法操作。

**const成员函数：**
目的：为了确认该成员函数可用于const对象上。
两个理由：1）：他们可以使class中那些函数可以改变，不可以改变。2）：编写高效代码。

**在const与non-const成员函数中避免重复：**
在类中我们重载const与non-const的operator函数时，容易遇到重复的代码，我们要做的就是实现operator的功能并使用两次。
我们可以通过**常量性消除（casting away constness）**，只需要在non-const函数中使用const_cast、static_cast来转换（转型动作）。

[!总结：](https://github.com/Cyxuan0311/Effective_Cpp/blob/master/image/const.png?raw=true)

### 4、确保对象被使用前已被初始化：
在类中，通常我们使用**列表初始化**的效率会高于函数内部的初始化。同时我们使用，即class的成员变量总是以声明的次序被初始化，所以我们进行初始化的过程中要保证变量的初始顺序与声明顺序一致。

static对象：其寿命从被构造出来到程序结束为止。

除了这些以外，我们需要编译A与B两个文件时，A构造函数中用到了B中的对象，那么初始化A和B的顺序就很重要了，这些变量称为（non-local static对象）
解决方法是：将每个non-local static对象搬到自己专属的函数内，并且该对象被声明为static，然后这些函数返回一个reference指向他所含的对象，用户调用这些函数，而不直接涉及这些对象（Singleton模式手法）

    
    原代码：
    "A.h"
    class FileSystem{
    public:
        std::size_t numDisks() const;
    };
    extern FileSystem tfs;
    "B.h"
    class Directory{
    public:
        Directory(params){
            std::size_t disks = tfs.numDisks(); //使用tfs
        }
    }
    Director tempDir(params);
    修改后：
    "A.h"
    class FileSystem{...}    //同前
    FileSystem& tfs(){       //这个函数用来替换tfs对象，他在FileSystem class 中可能是一个static，            
    static FileSystem fs;//定义并初始化一个local static对象，返回一个reference
    return fs;
    }
    "B.h"
    class Directory{...}     // 同前
    Directory::Directory(params){
    std::size_t disks = tfs().numDisks();
    }
    Directotry& tempDir(){   //这个函数用来替换tempDir对象，他在Directory class中可能是一个static，
    static Directory td; //定义并初始化local static对象，返回一个reference指向上述对象
    return td;
    }
    
这样做的原理在于C++对于函数内的local static对象会在“该函数被调用期间，且首次遇到的时候”被初始化。当然我们需要避免“A受制于B，B也受制于A”

总结：
\*为内置型对象进行手工初始化，因为C++不保证初始化他们。
\*构造函数最好使用初始化列初始化而不是复制，并且他们初始化时有顺序的。
\*为了免除跨文件编译的初始化次序问题，应该以local static对象替换non-local static对象。

## Chapter_two

### 5、了解C++默默编写并调用哪些函数：
通常情况下，一个类中如果我们没有声明一些基本的成员函数，编译器会自动声明默认的函数。这些函数都是public并inline的。

    
    如下：
    class Empty{
        public:
            Empty(){...}    //default构造函数
            Empty(const Empty &rhs){...}    //copy函数
            ~Empty(){...}   //析构函数
            Empty& operator=(const Empty& rhs){...}     //copy assignment操作符
    }
    

    针对copy assignment，给出以下例子：
    
    
        template<class T>
        class NameObject{
            public:
                NameObject(std::string &name,const T& value);   //如前，为申明operator=
            private:
                std::string &nameValue;
                const T objectValue;
        };

        std::string newDog("Persephone");
        std::string oldDog("Satch");
        NameObject<int> p(newDog,2);
        NameObject<int> s(oldDog,36);
        p=s;
    
针对该情况，编译器定义的operator=无能为力。只能使用自己定义的operator=。
总结：
\*编译器可以自动为class生成default构造函数，拷贝构造函数，拷贝赋值操作符，以及析构函数。

### 6、若不想使用编译器自动生成的函数，就该明确拒绝：
若我们不想要copy函数或copy assignment操作符，我们可以将其定义为private,如下：

    
    class HomeForSale{
        public:
            ...
        private:
            ...
            HomeForSale(const HomeForSale&);
            HomeForSale & operator=(const HomeForSale&);
    };
    
    
这一条主要是针对类设计者而言的，有一些类可能从需求上不允许两个相同的类，例如某一个类表示某一个独一无二的交易记录，那么编译器自动生成的拷贝和复制函数就是无用的，而且是不想要的。
总结：
\*可以将不需要的默认自动生成函数设置成delete(C++11标准)的或者弄一个private的父类并且继承下来。

### 7、为多态基类声明virtual析构函数：
其主要原因是如果基类没有virtual析构函数，那么派生类在析构的时候，如果是delete 了一个base基类的指针，那么派生的对象就会没有被销毁，引起内存泄漏。 例如：

    
    class TimeKeeper{
        public:
        TimeKeeper();
        ~TimeKeeper();
        virtual getTimeKeeper();
    };
    class AtomicClock:public TimeKeeper{...}
    TimeKeeper *ptk = getTimeKeeper();
    delete ptk;
    

除析构函数以外还有很多其他的函数，如果有一个函数拥有virtual 关键字，那么他的析构函数也就必须要是virtual的，但是如果class不含virtual函数,析构函数就不要加virtual了，因为一旦实现了virtual函数，那么对象必须携带一个叫做vptr(virtual table pointer)的指针，这个指针指向一个由函数指针构成的数组，成为vtbl（virtual table），这样对象的体积就会变大，例如：

    
    class Point{
        public://析构和构造函数
        private:
            int x, y
    };
    
本来上面那个代码只占用64bits(假设一个int是32bits)，存放一个vptr就变成了96bits，因此在64位计算机中无法塞到一个64-bits缓存器中，也就无法移植到其他语言写的代码里面了。
总结：
\*如果一个函数是多态性质的基类，应该有virtual 析构函数。
\*如果一个class带有任何virtual函数，他就应该有一个virtual的析构函数。
\*如果一个class不是多态基类，也没有virtual函数，就不应该有virtual析构函数。

### 8、别让异常逃离析构函数：

举一个例子，对于一个DBconnection，我们需要确保数据库连接总是关闭。

    
        如下：
        class DBConn{
            public:
                ...
                ~DBConn(){
                    db.close();
                }
            private:
                DBConnection db;
            };
    

一个更好的方式是重新设计DBConn接口，如下：

    
    class DBConn{
        public:
            ...
            void close(){
                db.close();
                closed=true;
            }
            ~DBConn(){
                if(!closed){
                    try{
                        db.close();
                    }
                    catch(...){
                        记录，记下catch的调用失败。
                    }
                }
            }
        private：
            DBConnection db;
            bool closed;
    };
    
    
这种做法就可以一方面将close的的方法交给用户，另一方面在用户忽略的时候还能够做“强迫结束程序”或者“吞下异常”的操作。相比较而言，交给用户是最好的选择，因为用户有机会根据实际情况操作异常。
总结：
\*析构函数不要抛出异常，因该在内部捕捉异常。
\*如果客户需要对某个操作抛出的异常做出反应，应该将这个操作放到普通函数（而不是析构函数）里面。

### 9、绝不在构造和析构过程中调用virtual函数：
举一个例子：

    
    class Transaction{
        public:
            Transaction();
            virtual void logTransaction() const=0;//因为类型的不同。
            ...
    };
    Transaction::Transaction(){
        ...
        logTransaction();   //最后的动作是记下这笔交易。
    }
    class BuyTranscation:public Transaction{
        public:
            virtual void logTransaction() const;
            ...
    };
    class SellTransaction:public Transaction{
        public:
            virtual void logTransaction() const;
            ...
    };
    
    或者有一个更难发现的版本：

    class Transaction{
        public:
            Transaction(){init();}
            virtual void logTransaction() const = 0;
        private:
            void init(){
                logTransaction();
            }
    };
这个时候代码会调用 Transaction 版本的logTransaction，因为在构造函数里面是先调用了父类的构造函数，所以会先调用父类的logTransaction版本，解决方案是不在构造函数里面调用，或者将需要调用的virtual弄成non-virtual的。

改进一下：

    class Transaction{
    public:
        explicit Transaction(const std::string& logInfo);
        void logTransaction(const std::string& logInfo) const; //non-virtual 函数
    }
    Transaction::Transaction(const std::string& logInfo){
    logTransaction(logInfo); //non-virtual函数
    }
    class BuyTransaction: public Transaction{
    public:
    BuyTransaction(parameters):Transaction(createLogString(parameters)){...} //将log信息传递给base class 构造函数
    private:
    static std::string createLogString(parameters); //注意这个函数是用来给上面那个函数初始化数据的，这个辅助函数的方法
    }

总结：
\*在构造和析构期间不要调用virtual函数，因为这类调用从不下降至派生类的版本。

### 10、令operator=返回一个reference to *this：

主要是为了支持连读和连写，例如：

    class Widget{
        public:
            Widget& operator=(int rhs){return *this;}
    }
    a = b = c;

总结：
\*令赋值（assignment）操作符返回一个reference to *this。

### 11、在operator中处理“自我赋值”：
举一个例子：

    class Bitmap{...};
    class Widget{
        ...
        private:
            Bitmap *pb;
    };

    以下是operator=的实现代码：
    Widget& Widget::operator=(const Widget &rhs){
        delete pb;  //停止使用当前的bitmap
        pb=new Bitmap(*rhs.pb); //使用rhs's bitmap的附件。
        return *this;
    }

但是，可能出现的问题是\*this与rhs是同一个对象。delete操作会将\*this与rhs都销毁。

解决方法：
    
    Widget& Widget::operator=(const Widget& rhs){
        if(this==&rhs)  return *this;   //do nothing.

        delete pb;
        pb=new Bitmap(*rhs.pb);
        return *this;
    }
尽量去发现异常。

我们可以采用一个效率更高的方法：

    class Widget{
        ...
        void swap(Widget& rhs);
        ...
    };
    Widget& Widget::operator=(const Widget& rhs){
        Widget temp(rhs);   //创造一个副本。
        swap(temp);
        return *this;
    }
总结：
\*确保当对象自我赋值的时候operator=有比较良好的行为，包括两个对象的地址，语句顺序，以及copy-and-swap。
\*确定任何函数如果操作一个以上的对象，而其中多个对象可能指向同一个对象时，仍然正确。

### 13、复制对象时勿忘其每一个成分：
考虑一个class表示顾客：

    void logCall(const std::string &funcName);  //制造一个log entry。
    class Customer{
        public：
            ...
            Customer(const Customer &rhs);
            Customer& operator=(const Customer& rhs);
            ...
        private:
            std::string name;
    };
    Customer::Customer(const Customer &rhs):name(rhs.name){
        logCall("Customer copy constructor");
    }
    Customer& Customer::operator=(const Customer &rhs){
        logCall("Customer copy assignment operator");
        name=rhs.name;
        return *this;
    }
    但是如果另一个成员变量加入：
    class Date{...};
    class Customer{
        public:
            ...
        private:
            std::string name;
            Date lastTranscation;
    };

此时copying函数执行的是“局部拷贝”，而且编译器是不会提示你的！！！
所以我们要同时修改copying函数

总结：
\*当编写一个copy或者拷贝构造函数，应该确保复制成员里面的所有变量，以及所有基类的成员。
\*不要尝试用一个拷贝构造函数调用另一个拷贝构造函数，如果想要精简代码的话，应该把所有的功能机能放到第三个函数里面，并且由两个拷贝构造函数共同调用。
\*当新增加一个变量或者继承一个类的时候，很容易出现忘记拷贝构造的情况，所以每增加一个变量都需要在拷贝构造里面修改对应的方法。

## Chapter_three

### 13、以对象管理资源：
主要是为了防止在delete语句执行前return，所以需要用对象来管理这些资源。这样当控制流离开f以后，该对象的析构函数会自动释放那些资源。 例如shared_ptr就是这样的一个管理资源的对象。他是在自己的析构函数里面做delete操作。所以如果自己需要管理资源的时候，也要在类内进行delete，通过对象来管理资源

总结：

\*建议使用shared_ptr。
\*如果需要自定义shared_ptr，请通过定义自己的资源管理类来对资源进行管理。

### 14、在资源管理类中小心copying行为：
在资源管理类里面，如果出现了拷贝复制行为的话，需要注意这个复制具体的含义，从而保证和我们想要的效果一样
思考下面代码在复制中会发生什么：

    class Lock{
    public:
    explicit Lock(Mutex *pm):mutexPtr(pm){
        lock(mutexPtr);//获得资源锁
    }
    ~Lock(){unlock(mutexPtr);}//释放资源锁
    private:
    Mutex *mutexPtr;
    }
    Lock m1(&m)//锁定m
    Lock m2(m1);//好像是锁定m1这个锁。。而我们想要的是除了复制资源管理对象以外，还想复制它所包括的资源（deep copy）。通过使用shared_ptr可以有效避免这种情况。

需要注意的是：copy函数有可能是编译器自动创建出来的，所以在使用的时候，一定要注意自动生成的函数是否符合我们的期望。

总结：
\*复制RAII对象（Resource Acquisition Is Initialization）必须一并复制他所管理的资源（deep copy）。
\*普通的RAII做法是：禁止拷贝，使用引用计数方法。

### 15、在资源管理类中提供对原始资源的访问：
例如：shared_ptr<>.get()这样的方法，或者->和*方法来进行取值。但是这样的方法可能稍微有些麻烦，有些人会使用一个隐式转换，但是经常会出错：

    class Font; class FontHandle;
    void changeFontSize(FontHandle f, int newSize){    }//需要调用的API

    Font f(getFont());
    int newFontSize = 3;
    changeFontSize(f.get(), newFontSize);//显式的将Font转换成FontHandle

    class Font{
    operator FontHandle()const { return f; }//隐式转换定义
    }
    changeFontSize(f, newFontSize)//隐式的将Font转换成FontHandle
    但是容易出错，例如
    Font f1(getFont());
    FontHandle f2 = f1;就会把Font对象换成了FontHandle才能复制

总结：
\*每一个资源管理类RAII都应该有一个直接获得资源的方法。
\*隐式转换对客户比较方便，显式转换比较安全，具体看需求。

### 16、成对使用new和delete时要采用相同形式：
总结：

\*即： 使用new[]的时候要使用delete[], 使用new的时候一定不要使用delete[]。

### 17、以独立语句将new的对象置入智能指针：
牢记**以对象管理资源**的智慧名言。
考虑下面的代码：

    int priority();
    void processWidget(shared_ptr<Widget> pw, int priority);
    processWidget(new Widget, priority());// 错误，这里函数是explicit的，不允许隐式转换（shared_ptr需要给他一个普通的原始指针
    processWidget(shared_ptr<Widget>(new Widget), priority()) // 可能会造成内存泄漏

    内存泄漏的原因为：先执行new Widget，再调用priority， 最后执行shared_ptr构造函数，那么当priority的调用发生异常的时候，new Widget返回的指针就会丢失了。当然不同编译器对上面这个代码的执行顺序不一样。所以安全的做法是：

    shared_ptr<Widget> pw(new Widget)
    processWidget(pw, priority())

总结：
\*凡是有new语句的，尽量放在单独的语句当中，特别是当使用new出来的对象放到智能指针里面的时候。

## Chapter_four

### 18、让接口容易被正确使用，不易被误用：
举一个例子:
    
    class Date{
        public:
            Date(int month,int day,int year);
            ...
    };
    容易使用户传入错误的参数，也容易导入不正确的类型。要思考用户有可能做出什么样子的错误。

一个办法是，以函数替换对象，表现每个月。另一个办法是，限制类型，可以使用const。
考虑下面的代码：

    Date(const Month &m, const Day &d, const Year &y);//注意这里将每一个类型的数据单独设计成一个类，同时加上const限定符
    为了让接口更加易用，可以对month加以限制，只有12个月份
    class Month{
    public:
    static Month Jan(){return Month(1);}//这里用函数代替对象，主要是方式第四条：non-local static对象的初始化顺序问题
    }

    而对于一些返回指针的问题函数，例如：
    Investment *createInvestment();//智能指针可以防止用户忘记delete返回的指针或者delete两次指针，但是可能存在用户忘记使用智能指针的情况，那么方法：
    std::shared_ptr<Investment> createInvestment();就可以强制用户使用智能指针，或者更好的方法是另外设计一个函数：
    std::shared_ptr<Investment>pInv(0, get)
总结：
\*“促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。
\*“阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。
\*shared_ptr支持定制删除器，从而防范dll问题，可以用来解除互斥锁等。

### 19、设计class犹如设计type:
应该带着**语言设计者当初设计语言内置类型**一样来设计class。

如何设计class：

1、新的class对象应该被如何创建和构造
2、对象的初始化和赋值应该有什么样的差别（不同的函数调用，构造函数和赋值操作符）
3、新的class如果被pass by value（以值传递），意味着什么（copy构造函数）
4、什么是新type的“合法值”（成员变量通常只有某些数值是有效的，这些值决定了class必须维护的约束条件）
5、新的class需要配合某个继承图系么（会受到继承类的约束）
6、新的class需要什么样的转换（和其他类型的类型变换）
7、什么样的操作符和函数对于此type而言是合理的（决定声明哪些函数，哪些是成员函数）
8、什么样的函数必须为private的
9、新的class是否还有相似的其他class，如果是的话就应该定义一个class template
10、你真的需要一个新type么？如果只是定义新的derived class或者为原来的class添加功能，说不定定义non-member函数或templates更好

总结：
\*class的设计就是type的设计。

### 20、宁以pass-by-reference-to-const替换pass-by-value:
考虑以下代码：

    class Person{
        public:
            Person();
            virtual ~Person();
            ...
        private:
            std::string name;
            std::string address;
    };
    class Student:public Person{
        public:
            Student();
            ~Student();
            ...
        private:
            std::string Schoolname;
            std::string SchoolAddress;
    };

使用pass-by-reference-to-const主要是可以提高效率，同时可以避免基类和子类的参数切割问题。

    bool validateStudent(const Student &s);//省了很多构造析构拷贝赋值操作
    bool validateStudent(s);

    subStudent s;
    validateStudent(s);//调用后,则在validateStudent函数内部实际上是一个student类型，如果有重载操作的话会出现问题
对于STL等内置类型，还是以值传递好一些。

总结：
\*尽量以pass-by-reference-to-const代替pass-by-value。前者通常比较高效，并且可以避免切割问题。
\*STL以及其他内置类型、STL的迭代器和函数对象。pass-by-value比较合适。

### 21、必须返回对象时，别妄想返回其reference:

考虑一个用以表示有理数的class:

    class Rational{
        public:
            Rational(int numberator=0,int denominator=1);
            ...
        private:
            int n,d;
            friend operator*(const Rational &lhs,const Rational &rhs);
    };

函数创捷新对象的途径有二：stack空间或在heap空间。

主要是很容易返回一个已经销毁的局部变量，如果想要在堆上用new创建的话，则用户无法delete，如果想要在全局空间用static的话，也会出现大量问题,所以正确的写法是：

    inline const Rational operator * (const Rational &lhs, const Rational &rhs){
        return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    }
当然这样写的代价就是成本太高，效率会比较低。
总结：
\*千万不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同时需要多个这样的对象。

### 22、将成员变量声明为private:
应该将成员变量弄成private，然后用过public的成员函数来访问他们，这种方法的好处在于可以更精准的控制成员变量，包括控制读写，只读访问等。

同时，如果public的变量发生了改变，如果这个变量在代码中广泛使用，那么将会有很多代码遭到了破坏，需要重新写

另外protected 并不比public更具有封装性，因为protected的变量，在发生改变的时候，他的子类代码也会受到破坏。

从封装的角度来看：只有两种访问权限，private（提供封装）、和其他。

总结：
\*切记将成员变量声明为private。可赋予客户访问数据的一致性、可细微划分访问控制、允许约束条件获得保证，并提供class作者以充分的实现弹性。
\*protected并不比public更具封装性。

### 23、宁以non-member、non-friend替换member函数：
区别如下：

    class WebBrowser{
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();
    }

    member 函数：
    class WebBrowser{
    public:
        ......
        void clearEverything(){ clearCache(); clearHistory();removeCookies();}
    }

    non-member non-friend函数：
    void clearBrowser(WebBrowser& wb){
        wb.clearCache();
        wb.clearHistory();
        wb.removeCookies();
    }
这里的原因是：member可以访问class的private函数，enums，typedefs等，但是non-member函数则无法访问上面这些东西，所以non-member non-friend函数更好

这里还提到了namespace的用法，namespace可以用来对某些便利函数进行分割，将同一个命名空间中的不同类型的方法放到不同文件中这也是C++标准库的组织方式，例如：

    "webbrowser.h"
    namespace WebBrowserStuff{
    class WebBrowser{...};
        //所有用户需要的non-member函数
    }

    "webbrowserbookmarks.h"
    namespace WebBrowserStuff{
        //所有与书签相关的便利函数
    }

总结：
\*non-member、non-friend函数替换member函数。这样可以增加封装性、包裹弹性和机能扩充性。

### 24、若所有参数皆需类型转换，请为此采用non-member函数：
例如想要将一个int类型变量和Rational变量做乘法，如果是成员函数的话，发生隐式转换的时候会因为不存在int到Rational的类型变换而出错：

    class Rational{
    public:
        const Rational operator* (const Rational& rhs)const;
    }
    Rational oneHalf;
    result = oneHalf * 2;
    result = 2 * oneHalf;//出错，因为没有int转Rational函数

    non-member函数
    class Rational{}
    const Rational operator*(const Rational& lhs, const Rational& rhs){}

总结：
\*如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member。

### 25、考虑写出一个不抛异常的swap函数：
swap函数即为交换两个对象的值。
代码：

    namespace std{
        template<typename T>
        void swap(T &a,T &b){
            T temp(a);
            a=b;
            b=temp;
        }
    }

只要T类型支持copy函数与copy assignment操作符即可完成。
我们可以提供一个支持Widget类的专属swap函数
代码：

    namespace WidgetStuff{
        ...
        template<typename T>
        class Widget{...};
        ...
        template<typename T>
        void swap(Widget<T>& a,Widget<T>& b){   //non-member swap函数，这里并不属于std命名空间。
            a.swap(b);
        }
    }
试着做以下事情：
1）：提供一个public swap函数。
2）：在class或template所在的命名空间中内提供一个non-member swap，并令他调用public swap成员函数。
3）：如果你正在编写一个class(而非class template)，为你的class特化std::swap。并令他调用你的swap成员函数。

总结：
\*当std::swap对你的类型效率不高时，提供一个swap成员函数，并保证该函数不抛出异常。
\*如果你提供一个member swap，也要提供一个non-member swap来调用前者。对于class（而非template），也请特化std::swap。
\*调用swap时应针对std::swap使用using声明式，然后调用swap不用声明。
\*不要盲目的在std template中加入新东西。

### Chapter_five

### 26、尽可能延后变量定义式的出现时间：
考虑一个加密函数：
代码：

    std::string encryptPassword(const std::string &password){
        using namespace std;
        string encrypted;
        if(password.length()<MinimumPasswordLength){
            throw logic_error("Password is too short");
        }
        ...
        return encrypted;
    }
    其中，encrypted变量过早定义，如果if成立，该变量将不被使用。

可以改进为：
代码：

    std::string encryptPassword(const std::string &password){
        using namespace std;
        if(password.length()<MinimumPasswordLength){
            throw logic_error("Password is too short");
        }
        string encrypted;
        ...
        return encrypted;
    }
    不过，该变量依然没有进行赋值，条款4指出在直接构造时赋值会高效一点。
    所以我们加入：
    void encrypt(std::string &s);
    并在string encrypted后加入该member函数。

另外，对于一个循环，我们考虑变量在循环外还是循环内时：
一般，循环外会好一点；但是如果循环的次数较高，循环内会好一点。

总结：
\*延后变量定义式的出现，可以提高程序的清晰度与效率。

### 27、尽量少采用转型动作：
回顾以下转型语法：
1）：C风格-（T）expression。
2）：函数风格-T（expression）。
3）：另外C++还提供四种新式转型：

    const_cast<T(expression)
    dynamic_cast<T(expression)
    reinterpret_cast<T(expression)
    static_cast<T>(expression)
    
    const_cast用于将对象的常量性消除。
    dynamic_cast用于“安全向下转型”，也就是用来决定某对象是否归属继承体系中的某个类型。
    reinterpret_cast执行低级转型。
    static_cast用来强迫隐式转换，例如将non-const对象转换为const对象，或将int转换double等。

给出一个例子：

    class Widget{
        public:
            explicit Widget(int size);
            ...
    };
    void doSomeWork(const Widget &w);
    doSomeWork(Widget(15)); //int函数风格->Widget.
    doSomeWork(static_cast<Widget>(15));    //intC++风格->Widget.

主要是因为：

1.从int转向double容易出现精度错误

2.将一个类转换成他的父类也容易出现问题

总结：
- 尽量避免转型，特别是在注重效率的代码中减少dynamic_cast，试着用无需转型的替代设计
- 如果转型是必要的，试着将他封装到函数背后，让用户调用该函数，而不需要在自己的代码里面转型
- 如果需要转型，使用新式的static_cast等转型，比原来的（int）好很多（更明显，分工更精确）


### 28、避免返回handles指向对象内部成分：
主要是为了防止用户误操作返回的值：

    修改前代码：
    class Rectangle{
        public:
            Point& upperLeft() const { return pData->ulhc; }
            Point& lowerRight() const { return pData->lrhc; }
    }
    如果修改成：
    class Rectangle{
        public:
            const Point& upperLeft() const { return pData->ulhc; }
            const Point& lowerRight() const { return pData->lrhc; }
    }
    则仍然会出现悬吊的变量，例如：
    const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft());
boundingBox会返回一个temp的新的，暂时的Rectangle对象，在这一整行语句执行完以后，temp就变成空的了，就成了悬吊的变量

总结：

- 尽量不要返回指向private变量的指针引用等
- 如果真的要用，尽量使用const进行限制，同时尽量避免悬吊的可能性
  
### 29、为“异常安全”而努力是值得的：
假设有一个class用来表示GUI菜单：

    class PrettyMenu{
        public:
            ...
            void ChangeBackground(std::istream &imgSrc);
            ...
        private:
            Mutex mutex;
            Iamge* bgImage;
            int imageChange;
    };
    以下是函数实现：
    void PrettyMenu::changeBackground(std::istream &imgSrc){
        lock(&mutex);
        delete bgImage;
        ++imageChanges++;
        bgImage=new Image(imgSrc);
        unlock(&mutex);
    }
    从“异常安全”的观点来看，该函数没有满足其中任何一个条件。
当异常被抛出时，带有异常安全性的函数会：
- 不泄露任何资源。
- 不允许数据败坏。

异常安全函数具有以下三个特征之一：

- 如果异常被抛出，程序内的任何事物仍然保持在有效状态下，没有任何对象或者数据结构被损坏，前后一致。在任何情况下都不泄露资源，在任何情况下都不允许破坏数据。
- 如果异常被抛出，则程序的状态不被改变，程序会回到调用函数的状态。
- 承诺绝不抛出异常。

总结：
- 异常安全函数即使发生异常也不会泄漏资源或是任何数据结构败坏。
- 强烈保证往往可以使用copy-and-swap实现。
- 函数提供的“异常安全保证”通常最高只等于其所调用之各个函数的“异常安全保证”中的最低者。
  
### 30、透彻了解inline的里里外外：
inline函数只是对编译器的一个申请，可以隐喻提出，也可以明确提出：
- 隐喻提出：将函数定义与class定义式中。
- 明确提出：在函数的定义式前加上inline。

inline 函数的过度使用会让程序的体积变大，内存占用过高

而编译器是可以拒绝将函数inline的，不过当编译器不知道该调用哪个函数的时候，会报一个warning

尽量不要为template或者构造函数设置成inline的，因为template inline以后有可能为每一个模板都生成对应的函数，从而让代码过于臃肿
同样的道理，构造函数在实际的过程中也会产生很多的代码，例如下面的：

    class Derived : public Base{
    public:
        Derived(){} // 看起来是空白的构造函数
    }
    实际上：
    Derived::Derived{
        //100行异常处理代码
    }

总结：
- inline函数尽量用在小型、被频繁调用的函数。
- 不要只因为function template出现在头文件，就将它们声明为inline。

### 31、将文件间的编译依存关系降至最低：
这个关系其实指的是一个文件包含另外一个文件的类定义等

那么如何实现解耦呢,通常是将实现定义到另外一个类里面，如下：

    原代码：
    class Person{
    private
        Dates m_data;
        Addresses m_addr;
    }

    添加一个Person的实现类，定义为PersonImpl，修改后的代码：
    class PersonImpl;   //class声明式。
    class Person{
    private:
        shared_ptr<PersonImpl> pImpl;
    }
在上面的设计下,就实现了解耦，即“实现和接口分离”

与此相似的接口类还可以使用全虚函数

    class Person{
    public:
        virtual ~Person();
        virtual std::string name() const = 0;
        virtual std::string birthDate() const = 0;
    }
然后通过继承的子类来实现相关的方法

这种情况下这些virtual函数通常被成为factory工厂函数

总结：
- 应该让文件依赖于声明而不依赖于定义，可以通过上面两种方法实现。
- 程序头文件应该有且仅有声明。
  
## Chapter_six

### 32、确定你的public继承塑模出is-a关系：
public继承体现的是一种单向的一般化，
例如：

    class Person{...};
    class Student:public Person{...};
    即学生也是人，但人不一定是学生。

这里经常会出的错误是，将父类可能不存在的功能实现出来，例如：

    class Bird{
    virtual void fly();
    }
    class Penguin:public Bird{...};//企鹅是不会飞的
    如下：
    class Brid{
        ...
    };
    class FlyingBrid:public Brid{
        public:
            virtual void fly();
            ...
    };
    class Penguin:public Brid{
        ...
    };

总结：
- public继承中，意味着每一个Base class的东西一定适用于他的derived class。
  
### 33、避免遮掩继承而来的名称：
考虑以下代码：

    class Base{
        public:
            virtual void mf1() = 0;
            virtual void mf1(int);
            virtual void mf2();
            void mf3();
        void mf3(double);
    }
    class Derived:public Base{
        public:
            virtual void mf1();
            void mf3();
    }

这种问题可以通过：

    using Base::mf1;
    或者
    virtual void mf1(){//转交函数
        Base::mf1();
    }
    来解决，但是尽量不要出现这种遮蔽的行为

总结：
- derived class 会遮蔽Base class的名称。
- 可以通过using 或者转交函数来解决。

### 34、区分接口继承和实现继承：
举一个例子说明：

    class Shape{
        public:
            virtual void draw() const=0;
            virtual void error(const std::string &msg);
            int objectID() const;
            ...
    };
    class Rectangle:public Shape{...};
    class Ellipse:public Shape{...};
pure virtual 函数式提供了一个接口继承，当一个函数式pure virtual的时候，意味着所有的实现都在子类里面实现。不过pure virtual也是可以有实现的，调用他的实现的方法是在调用前加上基类的名称：

    class Shape{
    virtual void draw() const = 0;
    }
    ps->Shape::draw();

总结：
- 接口继承和实现继承不同，在public继承下，derived classes总是继承base的接口。
- pure virtual函数只具体指定接口继承
- 简朴的（非纯）impure virtual函数具体指定接口继承以及缺省实现继承
- non-virtual函数具体指定接口继承以及强制性的实现继承

### 35、考虑virtual函数以外的其他选择：

令客户通过public non-virtual成员函数间接调用private virtual函数，被称为non-virtual interface方法（NVI）。
例子如下：

    class GameCharacter{
        public:
            int healthValue() const{
                ...
                int retVal=doHealthValue();
                ...
                return retVal;
            }
        ...
        private:
            virtual int doHealthValue() const{
                ...
            }
    };
这种方法的优点在于事前工作和事后工作，这些工作能够保证virtual函数在真正工作之前之后被单独调用。

但是，我们还可以采用一个更简单的方法，要求每个人物的构造函数接收一个指针，指向一个健康计算函数。

    class GameCharacter;    //前置声明。
    int defaultHealthCalc(const GameCharacter &gc);
    class GameCharacter{
        public:
            typedef int (*HealthCalcFunc)(const GameCharacter&);
            explicit GameCharacter(HealthCalcFunc hcf=defaultHealthCalc)
                :healthFunc(hcf){}
            int healthValue() const{
                return healthFunc(*this);
            }
            ...
        private:
            HealthCalcFunc healthFunc;
    };

这个就是常见的Strategy设计模式。

同一人物类型的不同实体可以有不同的健康计算函数：

    class EvilBadGuy:public GameCharacter{
        public:
            explicit EvilBadGuy(HealthCalcFunc hcf=defaultHealthCalc)
                :GameCharacter(hcf){...}
            ...
    };
    int loseHealthQuickly(const GameCharacter&);
    int loseHealthSlowly(const GameCharacter&);

    EvilBadGuy ebg1(loseHealthQuickly);
    EvilBadGuy ebg2(loseHealthSlowly);

    如果将函数指针换成函数对象的话，会有更具有弹性的效果：

    typedef std::tr1::function<int (const GameCharacter&)> HealthCalcFunc;
    在这种情况下，HealthCalcFunc是一个typedef，他的行为更像一个函数指针，表示“接受一个reference指向const GameCharacter，并且返回int*”。

总结：这一节表示当我们为了解决问题而寻找某个特定设计方法时，不妨考虑virtual函数的替代方案
- 使用NVI手法，他是用public non-virtual成员函数包裹较低访问性（private和protected）的virtual函数。
- 将virtual函数替换成“函数指针成员变量”，这是strategy设计模式的一种表现形式。
- 以tr1::function成员变量替换virtual函数，因而允许使用任何可调用物（callable entity）搭配一个兼容与需求的签名式。
- 将继承体系内的virtual函数替换成另一个继承体系内的virtual函数。
- 将机能从成员函数移到class外部函数，带来的一个缺点是：非成员函数无法访问class的non-public成员。
- tr1::function对象就像一般函数指针，这样的对象可接纳“与给定之目标签名式兼容”的所有可调用物（callable entities）。
  
### 36、绝不重新定义继承而来的non-virtual函数：
考虑以下代码：

    class B{
        public:
            void mf();
            ...
    };

    class D:public B{
        public:
            void mf();
            ...
    };

    D x;

    B *pB = &x; pB->mf(); //调用B版本的mf
    D *pD = &x; pD->mf(); // 调用D版本的mf

即使不考虑这种代码层的差异，如果这样重定义的话，也不符合之前的“每一个D都是一个B”的定义。

总结：
- 绝对不要重新定义继承而来的non-virtual函数。

### 37、绝不从新定义继承而来的缺省参数值：
首先virtual函数系动态绑定，缺省参数值是静态绑定。
而静态绑定又是前期绑定，动态绑定又是后期绑定。

考虑以下代码：

    class Shape{
        public:
            enum ShapeColor{Red,Green,Blue};
            virtual void draw(ShapeColor color=Red) const =0;
            ...
    };
    class Rectangle:public Shape{
        public:
            virtual void draw(ShapeColor color=Green) const;    //赋予不同的缺省参数值。
            ...
    };
    class Circle:public Shape{
        public:
            virtual void draw(ShapeColor color) const;;
    };

    现在考虑以下：
    
    Shape* ps;
    Shape* pc=new Circle;
    Shape* pr=new Rectangle;

    其中ps、pc、pr的静态类型为Shape*;
    而pc的动态类型为Circle*,pr的动态类型为Rectangle*;
    所以pr->draw();
    调用Rectangle::draw(Shape::Red);
    由于缺省参数。

总结：
- 绝对不要重新定义一个继承而来的缺省参数值。
  
### 38、通过复合塑模出has-a或“根据某物实现出”：
举一个例子：

    class Address{....};
    class PhoneNumber{....};
    class Person{
        public:
            ...
        private:
            std::string name;
            Address address;
            PhoneNumber voiceNumber;
            PhoneNumber faxNumber;
    }

    还可以用STL库中的set与list比较：
    set不可以由list继承而来，但是可以由set可以has a list;

总结：
- 复合的意义与public继承完全不同。

### 39、明智而小心的使用private继承：
首先，意义上:
private继承，例如class D以private继承class B，用意就是采用class B内已经办妥的一些特性。
所以，private继承意味着（根据某物实现）。尽可能的使用复合，必要时才使用private继承。

总结：
- private 继承意味着is implemented in terms of， 通常比复合的级别低，但是当derived class 需要访问protect base class 的成员，或者需要重新定义继承而来的virtual函数时，这么设计是合理的。
- 和复合不同，private继承可以造成empty base最优化，这对致力于“对象尺寸最小化”的程序库开发者而言，可能很重要。

### 40、明智而小心的使用多重继承：
多重继承很容易造成名字冲突：

    class BorrowableItem{
    public:
        void checkOut();
    };
    class ElectronicGadget{
        bool checkOut()const;
    };
    class MP3Player:public BorrowableItem, public ElectronicGadget{...};
    MP3Player mp;
    mp.checkOut();//歧义，到底是哪个类的函数
    只能使用：
    mp.BorrowableItem::checkOut();
在实际应用中, 经常会出现两个类继承与同一个父类，然后再有一个类多继承这两个类：

    class Parent{...};
    class First : public Parent(...);
    class Second : public Parent{...};
    class last:public First, public Second{...};
当然，多重继承也有他合理的用途，例如一个类刚好继承自两个类的实现。
总结：
- 多重继承容易产生歧义
- virtual继承会增加大小、速度、初始化复杂度等成本，如果virtual base class不带任何数据，将是最具使用价值的情况
- 多重继承的使用情况：当一个类是“public 继承某个interface class”和“private 继承某个协助实现的class”两个相结合的时候。
  
## Chapter_seven

### 41、了解隐式接口和编译期多态：
面向对象编程总是以**显示接口**与**运行期多态**解决问题。
但是在template与泛型编程的世界中，总是以**隐式接口**与**编译期多态**。

    例如：
    template<typename T>
    void doprocessing(T &w){
        if(w.size()>10&&w!=someNastyWidget){
            T temp;
            temp.normalize();
            temp.size(w);
        }
    }

因为会涉及到operator的重载等等。

总结：
- class和template都支持接口和多态。
- 对class而言接口是显示的，以函数签名为中心。多态则是通过virtual函数发生于运行期。
- 对template参数而言，接口是隐式的，基于有效表达式。多态则是通过template具象化和函数重载解析发生于编译期。

### 42、了解typename的双重意义：

考虑以下代码：

    template<typename C>
    void print2nd(const C& container){  //打印容器的第二元素。
        if(container.size()>=2){
            C::const_iterator iter(container.begin());
            ++iter;
            int value=*iter;
            std::cout<<value;        
        }
    }

所以，在任何时候想要在template中指定一个嵌套从属类型名称（dependent names，依赖于C的类型名称），前面必须添加typename

总结：
- 声明template参数，前缀关键词class和typename可互换。
- 需要使用typename标识嵌套从属类型名称，但不能在base class lists（基类列）或者member initialization list（成员初始列）内以它作为base class修饰符，template class Derived : public typename Base ::Nested{}//错误的！！！！

### 43、学习处理模板化基类内的名称：
源代码：

    class CompanyA{
        public:
            void sendCleartext(const std::string& msg);
            ....
    }
    class CompanyB{....}

    template <typename Company>
    class MsgSender{
    public:
        void sendClear(const MsgInfo& info){
            std::string msg;
            Company c;
            c.sendCleartext(msg);
        }
    }
    template<typename Company>//想要在发送消息的时候同时写入log，因此有了这个类
    class LoggingMsgSender:public MsgSender<Company>{
        public:
            void sendClearMsg(const MsgInfo& info){
            //记录log
                sendClear(info);//无法通过编译，因为找不到一个特例化的MsgSender<company>
            }   
    }

解法一：

    template <> // 生成一个全特例化的模板
    class MsgSender<CompanyZ>{  //和一般的template，但是没有sendClear,当Company==CompanyZ的时候就没有sendClear了
        public:
            void sendSecret(const MsgInfo& info){....}
    }

解法二（使用this）:

    template<typename Company>
    class LoggingMsgSender:public MsgSender<Company>{
        public:
            void sendClearMsg(const MsgInfo& info){
            //记录log
                this->sendClear(info);//假设sendClear将被继承
            }
    }

解法三（使用using）：

    template<typename Company>
    class LoggingMsgSender:public MsgSender<Company>{
    public:

    using MsgSender<Company>::sendClear; //告诉编译器，请他假设sendClear位于base class里面

    void sendClearMsg(const MsgInfo& info){
        //记录log
        sendClear(info);//假设sendClear将被继承
        }
    }

解法四（指明位置）：

    template<typename Company>
    class LoggingMsgSender:public MsgSender<Company>{
        public:
            void sendClearMsg(const MsgInfo& info){
            //记录log
            MsgSender<Company>::sendClear(info);//假设sendClear将被继承
        }
    }

### 44、将与参数无关的代码抽离template:
主要是会让编译器编译出很长的臃肿的二进制码，所以要把参数抽离，看以下代码：
举一个例子：

    template<typename T,std::size_t n>
    class SquareMatrix{
        public:
            ...
            void invert();  //求逆矩阵
    }；
    SquareMatrix<double,5> sml;
    sml.invert();
    SquareMatrix<double,10> smy;
    smy.invert();
由于代码的重复，就会导致代码暴涨。

所以我们可以编写一个带数值参数的函数：

    template<typename T>
    class SquareMatrixBase{
        protected:
            ...
            void invert(std::size_t matrixSize);
            ...
    };
    template<typename T,std::size_t n>
    class SquareMatrix:private SquareMatrixBase<T>{
        private:
            using SquareMatrixBase<T>::invert;  //避免base版的invert。
        public:
            ...
            void invert(){
                this->invert(n);    //制造一个inline的调用。
            }
    };
当然因为矩阵数据可能会不一样，例如5x5的矩阵和10x10的矩阵计算方式会不一样，输入的矩阵数据也会不一样，采用指针指向矩阵数据的方法会比较好：

    template<typename T>
    class SquareMatrixBase{
        protected:
            SquareMatrixBase(std::size_t n,T* pMem):size(n),pData(pMem){ }
            void setDataPtr(T* ptr){pData=ptr;}
            ...
        private:
            std::size_t size;
            T* pData;
    };

    template<typename T,std::size_t n>
    class SquareMatrix:private SquareMatrixBase<T>{
        public:
            SquareMatrix():SquareMatrixBase<T>(n,data){ }
            ...
        private:
            T data[n*n];
    };

总结：
- templates生成多个classes和多个函数，所以任何template代码都不该与某个造成膨胀的template参数产生依赖关系。
- 因非类型模板参数（non-type template parameters）而造成的代码膨胀，往往可以消除，做法是以函数参数后者class成员变量替换template参数。
- 因类型参数（type parameters）而造成的代码膨胀，往往可以降低，做法是让带有完全相同的二进制表述的具现类型，共享实现码。
  
### 45、运用成员函数模板接受所有兼容类型：

考虑以下代码：

    template<typename T>
    class SmartPtr{
        public:
            explicit SmartPtr(T *realPtr);
            ... 
    };
    SmartPtr<Top> pt1=SmartPtr<Middle>(new Middle);     //将SmartPtr<Middle>转化为SmartPtr<Top>
    SmartPtr<Top> pt2=SmartPtr<Botttom>(new Bottom);    //将SmartPtr<Bottom>转化为SmartPtr<Top>
    SmartPtr<const Top> pct2=pt1;   //将SmartPtr<Top>转化为SmartPtr<const Top>
编译的M与T完全为不同的模板。
由于我们的关注点在于如何smart指针的构造函数，但由于我们所需要的数量没有止境。
我们可以编写一个构造模板：

    template<typename T>
    class SmartPtr{
        public:
            template<typename U>
            SmartPtr(const SmartPtr<U>& other); //copy函数。
            ...
    };
    以上代码的意思为：可以由类型T生成类型U的智能指针。

但由于我们只想将B->T,而不是T->B。
我们可以考虑：

    template<typename T>
    class SmartPtr{
        public:
            template<typename U>
            SmartPtr(const SmartPtr<U>& other)  //为了生成copy构造函数
            :heldPtr(other.get()){....}
                T* get() const { return heldPtr; }
        private:
            T* heldPtr;                        //这个SmartPtr持有的内置原始指针
    };

总结：
- 使用成员函数模板生成“可接受所有兼容类型”的函数。
- 如果还想泛化copy构造函数、操作符重载等，同样需要在前面加上template。

### 46、需要类型转换时请为模板定义非成员函数：
考虑以下代码：

    template<typename T>
    class Rational{
        public:
            Rational(const T& numberator=0,const T& denominator=1);
            const T numerator() const;
            const T denominator() const;
            ...
    };
    template<typename T>
    const Rational<T> operator*(const Rational<T> &lhs,const Rational<T> &rhs){...}

像第24条一样，当我们进行混合类型算术运算的时候，会出现编译通过不了的情况

    template<typename T>
    const Rational<T> operator* (const Rational<T>& lhs, const Rational<T>& rhs){....}

    Rational<int> oneHalf(1, 2);
    Rational<int> result = oneHalf * 2; //错误，无法通过编译

解决方法：使用friend声明一个函数,进行混合式调用

    template<typename T>
    class Rational{
        public:
            friend const Rational operator*(const Rational& lhs, const Rational& rhs){
                return Rational(lhs.numerator()*rhs.numerator(), lhs.denominator() * rhs.denominator());
            }
    };
    template<typename T>
    const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>&rhs){....}
总结：
- 当我们编写一个class template， 而他所提供的“与此template相关的”函数支持所有参数隐形类型转换时，请将那些函数定义为classtemplate内部的friend函数。

### 47、请使用trait class表现类型信息：
traits是一种允许你在编译期间取得某些类型信息的技术，或者受是一种协议。这个技术的要求之一是：他对内置类型和用户自定义类型的表现必须是一样的。

    template<typename T>
    struct iterator_traits;  //迭代器分类的相关信息
    //iterator_traits的运作方式是，针对某一个类型IterT，在struct iterator_traits<IterT>内一定声明//某个typedef名为iterator_category。这个typedef 用来确认IterT的迭代器分类
    一个针对deque迭代器而设计的class大概是这样的
    template<....>
    class deque{
        public:
        class iterator{
            public:
            typedef random_access_iterator_tag iterator_category;
        }
    }
    对于用户自定义的iterator_traits，就是有一种“IterT说它自己是什么”的意思
    template<typename IterT>
    struct iterator_traits{
    typedef typename IterT::iterator_category iterator_category;
    }
    //iterator_traits为指针指定的迭代器类型是：
    template<typename IterT>
    struct iterator_traits<IterT*>{
        typedef random_access_iterator_tag iterator_category;
    }
综上所述，设计并实现一个traits class：
- 确认若干你希望将来可取得的类型相关信息，例如对迭代器而言，我们希望将来可取得其分类.
- 为该信息选择一个名称（例如iterator_category）.
- 提供一个template和一组特化版本(例如iterator_traits)，内含你希望支持的类型相关信息.
  
在设计实现一个traits class以后，我们就需要使用这个    

    traits class：

    template<typename IterT, typename DistT>
    void doAdvance(IterT& iter, DistT d, std::random_access_iterator_tag){ iter += d; }//用于实现random access迭代器
    template<typename IterT, typename DistT>
    void doAdvance(IterT& iter, DistT d, std::bidirectional_iterator_tag){ //用于实现bidirectional迭代器
        if(d >=0){
            while(d--)
                ++iter;
        }
        else{
            while(d++)
                --iter;
        }
    }

    template<typename IterT, typename DistT>
    void advance(IterT& iter, DistT d){
        doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
    }

使用一个traits class:
- 建立一组重载函数（像劳工）或者函数模板（例如doAdvance），彼此间的差异只在于各自的traits参数，令每个函数实现码与其接受traits信息相应.
- 建立一个控制函数（像工头）或者函数模板（例如advance），用于调用上述重载函数并且传递traits class所提供的信息.

### 48、认识template元编程:
模板元编程（TMP）是编写template-based C++程序并执行于编译期的过程。
TMP编译可以将工作从运行期转到编译期。

现在我们引进一个代码：

    template<typename IterT,typename DistT>
    void advance(IterT &iter,DistT d){
        if(iter is a random access iterator){
            iter+=d;
        }else{
            if(d>=0){
                while(d--)
                    ++iter;
            }else{
                while(d++)
                    --iter;
            }
        }
    }
其中我们可以使用typeid来进行类型比较，从而让**iter is a random access iterator**成立。

让我们再加一个例子：

    使用TMP计算阶乘：
    template<unsigned n>
    struct Factorial{
        enum{value=n*Factorial<n-1>::value};
    };
    //特殊情况
    template<>
    struct Factorial<0>{
        enum{value=1;}
    };
    明显的是，程序在编译期计算阶乘。

总结：
- TMP(模板元编程)可将工作从运行期转到编译期，从而实现早期错误检测与更高的效率。

## Chapter_eight

### 49、了解new-handler的行为：

当new无法申请到新的内存的时候，会不断的调用new-handler，直到找到足够的内存,new_handler是一个错误处理函数：

    namespace std{
        typedef void(*new_handler)();
        new_handler set_new_handler(new_handler p) throw();
    }
一个设计良好的new-handler要做下面的事情：
- 让更多内存可以被使用.
- 安装另一个new-handler，如果目前这个new-handler无法取得更多可用内存，或许他知道另外哪个new-handler有这个能力，然后用那个new-handler替换自己
- 卸除new-handler
- 抛出bad_alloc的异常
- 不返回，调用abort或者exit
  
new-handler无法给每个class进行定制，但是可以重写new运算符，设计出自己的new-handler
此时这个new应该类似于下面的实现方式：

    void* Widget::operator new(std::size_t size) throw(std::bad_alloc){
        NewHandlerHolder h(std::set_new_handler(currentHandler));      // 安装Widget的new-handler
        return ::operator new(size);                                   //分配内存或者抛出异常，恢复global new-handler
    }
总结：
- set_new_handler允许客户制定一个函数，在内存分配无法获得满足时被调用。
- Nothrow new是一个没什么用的东西。

### 50、了解new和delete的合理替换时机：

为什么要替换编译器提供的operator new和operator delete？
- 用来检查运行上的错误。
- 为了强化效能。
- 为了收集使用上的统计数据。

写一个定制型operator new

    static const int signature=0xDEADBEEF;
    typedef unsigned char Byte;
    //这段代码还有一些问题，如下：
    void* operator new(std::size_t size) throw(std::bad_alloc){
        using namespace std;
        size_t realsize=size+2*sizeof(int); //增加大小，使能够塞入两个signature.

        void* pMem=malloc(realSize);
        if(!pMem)
            throw bad_alloc();
        *(static_cast<int*>(pMem))=signature;
        *(reinterpret_cast<int*>(static_cast<Byte*>(pMem)+realSize-sizeof(int)))=signture;

        return static_cast<Byte*>(pMem)+sizeof(int);
    }

这个operator new的缺点主要在于它疏忽了身为这个特殊函数所应该的具有的“坚持C++规矩”的态度。

**字节对齐：**
参考链接：
    [链接](Txtfile\字节对齐.txt)

- 用来检测运用上的错误，如果new的内存delete的时候失败掉了就会导致内存泄漏，定制的时候可以进行检测和定位对应的失败位置
- 为了强化效率（传统的new是为了适应各种不同需求而制作的，所以效率上就很中庸）
- 可以收集使用上的统计数据
- 为了增加分配和归还内存的速度
- 为了降低缺省内存管理器带来的空间额外开销
- 为了弥补缺省分配器中的非最佳对齐位
- 为了将相关对象成簇集中起来
  
总结：
- 有许多理由需要写个自定德new和delete,包括改善效能、对heap运用mistakes进行调试、搜集heap使用信息。
  
### 51、编写new和delete时需固守成规：
operator new的返回值很简单，如果分配成功，返回一个指针，指向一块足够大的内存，如果分配失败，返回bad_alloc。但是也不是这么简单，operator new实际上会多次尝试调用new-handing函数。其中当指向new-handing函数的指针是null,operator new才会抛出异常。

    考虑以下的operator new伪码：
    void* operator new(std::size_t size) throw(std::bad_alloc){
        using namespace std;
        if(size==0)
            size=1;
        while(true){
            尝试分配 size bytes;
            if(分配成功)
                return (一个指针，指向分配得来的内存);
            //分配失败：找出目前的new-handler函数(如果有的话)
            new_handler globalHandler=set_new_handler(0);
            set_new_handler(globalHandler);
            if(globalHandler)
                (*globalHandler)();
            else
                throw bad_alloc();
        }
    }
重写new的时候要保证49条的情况，要能够处理0bytes内存申请等所有意外情况.
重写delete的时候，要保证删除null指针永远是安全的.

### 52、写了placement new也要写placement delete:

1. **placement new和placement delete的概念**：placement new是一种特殊的`new`表达式，它允许在已分配的内存上构造对象，其语法为`new (place) T(args)`，`place`是指向已分配内存的指针，`T`是要构造的对象类型，`args`是构造函数的参数。与之对应的placement delete，用于在对象析构后释放相关资源，其作用是配合placement new，确保内存管理的完整性。
2. **写placement delete的必要性**：当使用placement new在特定位置构造对象时，如果构造过程中抛出异常，且没有对应的placement delete，那么已分配的内存可能无法正确释放，从而导致资源泄漏。例如，在一个需要频繁在特定内存区域创建和销毁对象的程序中，如果没有正确配对的placement delete，随着时间的推移，会造成大量内存浪费，最终可能导致程序因内存不足而崩溃。
3. **placement delete的实现和调用**：placement delete的函数签名通常为`void operator delete(void* ptr, void* place)`，`ptr`是指向要释放内存的指针，`place`是之前placement new中指定的内存位置。编译器会在对象构造失败时自动调用placement delete来释放已分配但未成功构造对象的内存。用户在编写自定义的placement new时，必须同时提供对应的placement delete，以保证内存管理的正确性。
4. **常见错误及影响**：如果只提供placement new而忽略了placement delete，可能会在异常情况下导致内存泄漏。这不仅会降低程序的性能，还可能引发难以调试的错误，尤其是在长时间运行或对内存要求严格的程序中，这种错误可能会逐渐积累，最终导致系统故障。 

## Chapter_nine:

### 53、不要轻忽编译器的警告：
- 严肃对待编译器发出的warning， 努力在编译器最高警告级别下无warning
- 同时不要过度依赖编译器的警告，因为不同的编译器对待事情的态度可能并不相同，换一个编译器警告信息可能就没有了

### 54、让自己熟悉包括TR1在内的标准程序库：
其实感觉这一条已经有些过时了，不过虽然过时，但是很多地方还是有用的

- smart pointers
- tr1::function ： 表示任何callable entity（可调用物，只任何函数或者函数对象）
- tr1::bind是一种stl绑定器
- Hash tables例如set，multisets， maps等
- 正则表达式
- tuples变量组
- tr1::array：本质是一个STL化的数组
- tr1::mem_fn:语句构造上与成员函数指针一样的东西
- tr1::reference_wrapper： 一个让references的行为更像对象的东西
- 随机数生成工具
- type traits
  
### 55、让自己熟悉Boost:

主要是因为boost是一个C++开发者贡献的程序库，代码相对比较好.