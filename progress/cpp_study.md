
1. 移动语义的目的
    "移动语义是为了高效转移资源所有权，既能提升性能，也能让某些资源（如独占指针、锁）在语义上禁止拷贝。
    "移动语义允许我们把资源的所有权从一个对象'转移'到另一个对象，而不是复制一份。

2. 右值引用 T&& 与 移动资源的关系
    右值引用 T&& 的作用：
        它是一种类型，用来绑定到右值（更准确说是 xvalue 或 prvalue）
        它本身不会自动移动任何东西，只是"告诉编译器：这个对象可以被'偷'资源"。

    移动是否发生：
        取决于对象是否有移动构造/移动赋值

    "T&& 只是给编译器一个信号：'这个对象可以被移动'，但真正的移动操作要靠移动构造/赋值函数来完成；如果没有这些函数，编译器会退回拷贝。"

3. std::move 做了什么
    它就是一个类型转换（cast），把传入的对象转成右值引用类型。

    为什么需要它？
        因为有名字的变量（如 a）永远是左值，即使它的类型是 int&&，表达式 a 本身也是左值；
        std::move(a) 把 a 从左值"强制标记"成右值，让编译器在重载决议时选到移动版本。

    template<typename T>
    constexpr typename std::remove_reference<T>::type&& move(T&& t) noexcept {
        return static_cast<typename std::remove_reference<T>::type&&>(t);
    }

4. moved-from 对象的状态
    "moved-from 对象必须处于'有效但未指定'状态：可以安全析构和重新赋值，但不能假设它的值是什么。对于基本类型，移动会退化成拷贝，值不变；对于自定义类型，要看移动函数怎么写，标准库类型通常会变成空，但不保证。"

    有效但未指定状态：可以访问成员函数，但不要依赖容器里的“旧数据语义”


5. 推荐写法

    #include <vector>
    #include <iostream>

    struct Widget {
        std::vector<int> data;
        
        Widget(std::vector<int> d) : data(std::move(d)) {}      ///内部使用move， 注意d为左值， 无论外部使用拷贝或移动
    };

    int main() {
        std::vector<int> v = {1, 2, 3};
        Widget w(v);                ///此处 左值 拷贝   整体一次拷贝 + 一次移动
        Widget w1(std::move(v));    ///此处 右值 移动   整体 两次移动
        std::cout << v.size();  // 输出什么？
    }


6. 三五零法则

    Rule of Three (c++ 98)
        如果你需要自定义以下任意一个：
            析构函数 ~T()
            拷贝构造函数 T(const T&)
            拷贝赋值运算符 T& operator=(const T&)
        那么你通常需要定义全部三个。

    Rule of Five (c++ 11)
        析构函数 ~T()
        拷贝构造 T(const T&)
        拷贝赋值 T& operator=(const T&)
        移动构造 T(T&&) ⬅️ 新增
        移动赋值 T& operator=(T&&) ⬅️ 新增

        原则：
        如果定义了任意一个，通常需要定义全部五个（或显式 = delete 不需要的）；
        移动版本要标记 noexcept（否则 STL 容器扩容时不会用移动，会退化成拷贝）

    Rule of Zero (现代c++推荐)
        尽量不写这五个函数，让编译器自动生成。
        用 RAII 类（如 std::unique_ptr、std::vector）管理资源；

    // ❌ 旧写法：需要手动管理资源
    class Widget {
        int* data;
    public:
        Widget(int val) : data(new int(val)) {}
        ~Widget() { delete data; }
        Widget(const Widget& w) : data(new int(*w.data)) {}
        Widget& operator=(const Widget& w) { 
            if (this != &w) {
                delete data;
                data = new int(*w.data);
            }
            return *this;
        }
        Widget(Widget&& w) noexcept : data(w.data) { w.data = nullptr; }
        Widget& operator=(Widget&& w) noexcept {
            if (this != &w) {
                delete data;
                data = w.data;
                w.data = nullptr;
            }
            return *this;
        }
    };

    // ✅ 现代写法：Rule of Zero
    class Widget {
        std::unique_ptr<int> data;
    public:
        Widget(int val) : data(std::make_unique<int>(val)) {}
        // 编译器自动生成移动构造/赋值（拷贝被 delete，因为 unique_ptr 不可拷贝）
    };


7. 完美转发
    什么是完美转发？
        完美转发是指在模板函数中，将参数原封不动地转发给另一个函数，保持参数的值类别（左值/右值）、const/volatile 属性不变。通过转发引用（T&&）+ 引用折叠 + std::forward<T> 实现。

    为什么需要完美转发？
        传统的按值或按引用传参无法同时处理左值和右值，或者会丢失值类别信息。完美转发让泛型代码能高效地转发任意类型的参数，避免不必要的拷贝或移动。

    std::forward 和 std::move的区别？
        std::move：无条件将参数转换为右值引用；
        std::forward：根据模板参数 T 的类型，有条件地转换：如果 T 是左值引用类型，保持左值；如果 T 是非引用类型，转为右值。

    // 完美转发的标准写法
    template<typename T>
    void wrapper(T&& arg) {
        target(std::forward<T>(arg));  // 保持 arg 的值类别
    }

    // 可变参数版本
    template<typename... Args>
    void wrapper(Args&&... args) {
        target(std::forward<Args>(args)...);
    }

    // 引用折叠规则记忆
    // T = int&  → T&& = int& && → int&
    // T = int   → T&& = int&&

8. 模板
    1. 语言层 -> 机制层 -> 技巧层
        模板实体种类， 参数分类， 基本实例化
        特化/重载， SFINAE， type traits
        concepts, fold expression, 完美转发， 高阶库设计

    2. 模板实体与基础语法 【函数、类、变量】
        函数模板： 
            函数模板和普通函数重载在重载决议中的优先级如何？
                .普通函数（含全特化）优先于函数模板；在多个函数模板之间，再用部分排序选出更特化者

        类模板： 
            为什么类模板通常要把定义都写在头文件里？
                模板在编译期按具体类型实例化，实例化发生在使用点；编译器在每个使用它的翻译单元都必须看到完整定义，所以通常把模板定义放在头文件，而不是像普通函数那样只在一个 .cpp 里定义后靠链接器去找。

        变量/别名模板： 
            变量模板的典型使用场景是什么？为什么不直接用宏或者 static const？
            a. type traits 的 _v 辅助形式
                template <class T>
                constexpr bool is_integral_v = std::is_integral<T>::value;
            b. 按类型变化的常量（比如 pi<T> 这种）
                template <class T>
                inline constexpr T pi_v = T(3.1415926535897932385);

    3. 模板参数体系与模板实参推导
        模板实参推导： 
            模板类型参数 推导规则？
            a. 形参是按值：T / const T 
                规则：
                    先把实参类型中的引用去掉；
                    再把顶层 const / volatile 忽略；
                    剩下的就是 T。
                    数组退化为指针
                    函数退化为函数指针
            b. 形参是“普通引用”：T& / T const&
                规则：
                    先把实参类型的引用去掉；
                    T 保留实参的 cv 限定（const / volatile）；
                    形参本身是引用，所以不会再丢 cv。

            c. 形参是 T&&：右值引用 vs 转发引用
                推导规则：
                    实参是右值：T 推导成“裸类型” U，形参类型为 U&&；
                    实参是左值：T 推导为 U&，形参类型为 U& &&，经引用折叠变成 U&。

    4. 实例化
        什么是实例化？ 怎么理解两阶段查找？
        a. 隐示实例化, 显示实例化
        b. 实例化就是编译器在使用点，把“模板蓝图 + 具体模板实参”组合起来，生成真正的函数/类定义的过程
        c. 第一阶段在看到模板定义时就要解析所有不依赖模板参数的名字，保证模板本身是合法的；第二阶段在为具体类型实例化时才解析依赖模板参数的名字，比如 T::type，这既能发现错误，也为 SFINAE 留出空间

        举一个‘如果不理解两阶段查找就会踩坑’的例子。
            非依赖名
            依赖名

        extern template

    5. 特化/重载、SFINAE + type traits 
        全特化，偏特化，魏特化(重载)
            函数模板为什么不能偏特化？那如果我想对某些类型做特殊处理怎么办？

        SFINAE 基础， 什么是 SFINAE？


    6. concepts
        泛型 lambda、模板 lambda
        concepts & requires

    7. 工程实践
        Policy-based design + traits-based design
        你会如何设计一个可插拔 allocator 的容器？用模板大致描述一下接口形态？


