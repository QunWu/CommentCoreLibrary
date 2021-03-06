# Instance Shadowing 双重存在
为了实现沙箱与外界的联通，我们会建立每一个需要的实例在沙箱外的影子示例（Shadow Instance）。在对
沙箱内的实例进行属性改写（Property Modification）和方法调用（Method Invocation）时，该操作
则会在影子实例上重复执行。由于影子实例无法与沙箱内实例主动发起通讯，影子实例的更改不能同步到沙箱内：

    Original Instance                         Shadow Instance
       (原始实例)           -------通讯------>     (影子实例)       <====> [DOM]
        [沙箱内]                                   [沙箱外] 

## 强绑定与克隆 Strong Binding & Cloning 
鉴于这种架构设计，所有实现影子实例的类，都会与影子进行强绑定。也就是说，在沙箱内克隆影子实例产生出的
克隆实例（Cloned Instance）依然会和原本的影子实例绑定。这样以来，就很可能出现不同步导致严重的
BUG。

    Original Instance                         Shadow Instance
       (原始实例)           -------通讯------>     (影子实例)      
        [沙箱内]                   |               [沙箱外] 
                                  |
      Cloned Instance             |
       (克隆实例)           --------
        [沙箱内]
        
如果对 Cloned Instance 执行操作（更改属性或调用方法）改变了状态，由于影子实例无法通告原始实例，
则会行程如下的链接状态：

    Original Instance                         Shadow Instance*
       (原始实例)           -------通讯------>     (影子实例*)      
        [沙箱内]                   |               [沙箱外] 
                                  |
      Cloned Instance*            |
       (克隆实例*)          --------
        [沙箱内]
        
这时虽然属性进行了更改，在 Original Instance下读取属性并不能读取到正确的属性信息。如果这时再对
原始实例进行操作则会产生出如下状态：

    Original Instance#                        Shadow Instance*#
       (原始实例#)          -------通讯------>     (影子实例*#)      
        [沙箱内]                   |               [沙箱外] 
                                  |
      Cloned Instance*            |
       (克隆实例*)          --------
        [沙箱内]
        
这时无论是原始实例还是克隆实例都无法正确的表达在沙箱外的影子实例的真实状况。由于操作并未被记录，所以
试图还原出外部的实际状况將近乎不可能。

## 保持安全干净的操作 Safety Precautions
在进行 `clone()` 操作时，请务必确保你克隆的对象没有强绑定。有关具体哪些对象有强绑定哪些没有，请
参考相关的各个子分类。一般来说，Display系统，Player系统和Tween系统产生的元件大都有强绑定。一些
例子比如，`DisplayObject`，`Sound`，`Tween` 和 `Util.interval/Timer` 定时器等都有强绑定。

保证安全访问的一些好的建议方法：

1. 对于克隆的组件只进行读取操作，或者避免克隆整个对象。大多数情况下使用引用和传递原来的对象就足够了
2. 如果克隆不可避，请尽早销毁克隆前的原始对象，或者不再使用它。不过有关重制属性，可以采取 
`a.prop = a.prop`来确保前段显示和 a 对象分离时的原有数值一样。
3. 如果不可销毁对象，则请在架构时确保你知道进行更改会引发不同步问题的可能性，并绕开危险的设计方法。

## 非同步调用设计原因 Design Justification
采取上述设计的原因在于沙箱。为了保证安全执行代码，我们采取了沙箱机制，这样的优点在于可以移植到更加
广泛的平台，如NodeWebkit甚至是C/C++系列的软件。你只需引入JS运行时（如V8），然后提供Native的
OOAPI接口，就可以完全自如的执行代码弹幕，同时还可以控制危险操作。

同时我们为了兼容性，选择了忽略更改同步性的策略。这种设计在牺牲了可控性的代价下，提高了效率并且保证了
兼容性。代价是可能会产生赛跑状况（Race Condition），但是由于在worker内是单线程的，所以这些危害
被大大降低了，目前主要只体现在clone下。更新属性和调用方法的流程如下：

更新属性：

1. 代码更新了属性，调用了对象的此属性字段的 setter 函数
2. setter函数更新了对象内部的缓存，同时把属性值打包传递到影子实例上
3. 沙箱中代码继续运行，影子实例获得到属性更改消息后则会执行相应操作

调用方法 ：

1. 调用了对象的方法，方法不返回时，直接把参数打包，同时如更改属性那样更改缓存，然后传递参数
2. 调用了对象的方法，有返回时，**先根据本地实例模拟一个返回结果**，递交函数到影子对象。如果影子
对象判定需要更新，则影子对象会派发更新消息，注意这些都是异步的，所以无法确定更新消息会在代码执行的
什么位置生效。


