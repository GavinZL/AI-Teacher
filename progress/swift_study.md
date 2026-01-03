
1. 属性包装器 (Property Wrapper)
    告诉框架： 这个数据是谁拥有？ 谁负责存？ 谁会因为它的变化而刷新UI？

    @State      当前view拥有， 跟随view生命周期， 适用简单局部状态：开关、计数器、文本输入 
        【小范围 + 值类型 + 当前 View 自己用】
    @Binding    别人拥有，当前只是引用；跟随view生命周期；适用 子view需要修改父view的状态
        【子 View 需要读写父 View 的某个 @State】
    @ObserverdObject 外部传进来的对象，view不负责创建/持有； 跟随view生命周期； 适用view观察一个已有的模型对象，如列表item、外部manager
        【对象是外面给我的，我只负责监听和刷新 UI，不负责它活多久】
    @StateObject 当前view创建并持有对象； 跟随view的生命周期，且创建一次； 这个view自己管理的‘引用类型模型’，如页面ViewModel
        【View 自己 new 的对象 + 想保留状态】
    @Environment 环境统一提供；跟随环境，系统上层提供的设置：colorScheme、dismiss
        【系统或上层统一注入、我只读不直接管理生命周期】
    @EnvironmentObject 环境统一提供的ObservedObject， 全局/跨多层view， 全局共享的数据， 如用户Session、Setting
        1. 在上层用 .environmentObject(session) 注入
        2. 在任意子 View 中声明，@EnvironmentObject var session: SessionManager
        【全局/多页面共享的 ObservableObject + 不想层层传递】


