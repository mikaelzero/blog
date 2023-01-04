![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/skHOtL.jpg)

1 App 层
2 Framework 层
3 硬件抽象层（ hal ：主要对接这一层）
4 驱动层
5 硬件层
http://androidxref.com/7.1.1_r6/xref/

### App 层（java 层的调用）

```java
public ImxOrientationImpl(Context context) {    newSensorData = false;    m_context = context;    // 实例化传感器管理者    mSensorMng = (SensorManager)m_context.getSystemService(Context.SENSOR_SERVICE);    orientationSensor = mSensorMng.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR);    phoneOrientationListener = new PhoneOrientationListener();    mSensorMng.registerListener(phoneOrientationListener, orientationSensor, mSensorMng.SENSOR_DELAY_FASTEST);}

class PhoneOrientationListener implements SensorEventListener {    @Override    public void onSensorChanged(SensorEvent event) {        // TODO Auto-generated method stub        synchronized (this) {            SensorManager.getQuaternionFromVector(phoneInWorldSpaceQuaternion, event.values);        }    }    @Override    public void onAccuracyChanged(Sensor sensor, int accuracy) {        // TODO Auto-generated method stub    }}

//AccSensor = mSensorMng.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);//MagSensor = mSensorMng.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD);//GyroSensor = mSensorMng.getDefaultSensor(Sensor.TYPE_GYROSCOPE);
```

Sensor 分为物理 sensor 和虚拟 sensor. 物理 sensor 对应具体的硬件, 虚拟 sensor 是软件层虚拟出来的.
sensor 源码：frameworks/base/core/java/android/hardware/Snesor.java

### Sensor 总体调用关系图

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/8AU3iX.jpg)

Sensor  框架分为三个层次, 客户度、服务端、HAL 层, 服务端负责从 HAL 读取数据, 并将数据写到管道中, 客户端通过管道读取服务端数据.
客户端主要类
　　 SensorManager.java
　　　　 SensorManager 定义为一个抽象类, 定义了一些主要的方法, 该类主要是应用层直接使用的类, 提供给应用层的接口
　　 SystemSensorManager.java
　　　　继承于 SensorManager, 客户端消息处理的实体, 应用程序通过获取其实例, 并注册监听接口, 获取 sensor 数据.
　　 sensorEventListener 接口
　　　　用于注册监听的接口
　　 sensorThread（5.0 以后废弃, 用 Looper 替代）
　　　　是 SystemSensorManager 的一个内部类, 开启一个新线程负责读取读取 sensor 数据, 当注册了 sensorEventListener 接口的时候才会启动线程
　　 android_hardware_SensorManager.cpp
　　　　负责与 java 层通信的 JNI 接口
　　 SensorManager.cpp
　　　　 sensor 在 Native 层的客户端, 负责与服务端 SensorService.cpp 的通信
　　 SenorEventQueue.cpp
　　　　消息队列

### Sensor service

服务端主要类
　　 SensorService.cpp
　　　　服务端数据处理中心
　　 SensorEventConnection
　　　　从 BnSensorEventConnection 继承来, 实现接口 ISensorEventConnection 的一些方法, ISensorEventConnection 在 SensorEventQueue 会保存一个指针, 指向调用服务接口创建的 SensorEventConnection 对象
　　 Bittube.cpp
　　　　在这个类中创建了管道, 用于服务端与客户端读写数据
　　 SensorDevice
　　　　负责与 HAL 读取数据
HAL 层
　　 Sensor.h 是 google 为 Sensor 定义的 Hal 接口, 单独提出去

客户端源码：
frameworks/base/core/java/android/hardware/SensorManager.java
frameworks/base/core/java/android/hardware/SystemSensorManager.java
frameworks/base/core/jni/android_hardware_SensorManager.cpp
frameworks/native/include/gui/SensorManager.h
frameworks/native/libs/gui/SensorManager.cpp
服务器端源码：
frameworks/native/services/sensorservice/SensorService.cpp
frameworks/native/services/sensorservice/ SensorEventConnection.cpp
frameworks/native/services/sensorservice/SensorDevice.cpp
frameworks/native/libs/gui/ Bittube.cpp

### 客户端读取数据

Sensormanager 的初始化
1 SystemSensorManager 的构造：nativeClassInit, nativeCreate, nativeGetSensorAtIndex
2 这几个函数实现在 android_hardware_SensorManager 中
3 在 nativeCreate 创建 SensorManager 类

注册 eventListener（registerListenerImpl）
1 定义了消息泵 Looper, 若没有自定义 Looper, 则使用 mainLooper（创建该 manager 对象时的 looper）.
2 创建消息队列 sensorEventQueue：queue = new SensorEventQueue(listener, looper, this, fullClassName);
3 添加 sensor：queue.addSensor(sensor, delayUs, maxBatchReportLatencyUs）, 并调用 enableSensorLocked()接口对指定的 sensor 使能, 并设置采样时间.
4 SensorEventQueue()方法主要是调用了父类的 BaseEventQueue()方法, 通过 super()方法可以调用父类参数相同的方法.
5 BaseEventQueue()方法调用了 native 方法 nativeInitBaseEventQueue(), nativeInitSensorEventQueue()主要 new 一个 LooperCallBack 类型的对象 receiver, 然后他会调用 receiver 对象的 onFirstRef()方法来添加对 sensoreventQueue 事件的监听.
6 Looper 对文件描述符的监控与处理, 在 onFirstRef()中调用 mMessageQueue-> getLooper()->addFd 来添加要监听的事件, 同时指定回调函数处理事件, 当 sensor 有事件到来时, 会回调这里 LooperCallback 子类中重载的 handleEvent()来实现对可 I/O 事件的处理, 最后调用 dispatchSensorEvent()完成数据的存储同时向 app 的转发. 这里的 native 层的 handleEvent()需调用 Java 方法 dispatchSensorEvent(), 涉及到 JNI 编程的 C++和 Java 相互调用的知识, 这里 C++代码引用 Java 代码, 先通过 GetMethodID 获取方法 ID, 然后通过 CallVoidMethod 即可调用该方法. 当有数据到来时会调用 receiver 类的函数 handleEvent()来进行处理
7 dispatchSensorEvent()主要将数据传递给 APP, 调用 APP 的重载函数 onSensorChanged()来处理这条 event.

### 服务器端实现

1 启动 sensorservice
2 SensorService 创建完之后, 将会调用 SensorService::onFirstRef()方法, 在该方法中完成初始化工作.
2.1 首先获取 SensorDevice 实例, 在其构造函数中, 完成了对 Sensor 模块 HAL 的初始化：
这里主要做了三个工作：
调用 HAL 层的 hw_get_modele()方法, 加载 Sensor 模块 so 文件（hal 层, 我们这次对接的库）
调用 sensor.h 的 sensors_open 方法打开设备
调用 sensors_poll_device_t->activate()对 Sensor 模块使能
2.2 再看看 SensorService::onFirstRef()方法：
　　在这个方法中, 主要做了 4 件事情：
创建 SensorDevice 实例
获取 Sensor 列表
调用 SensorDevice.getSensorList(),获取 Sensor 模块所有传感器列表
为每个传感器注册监听器, HardwareSensor 实现了 SensorInterface 接口.
3 启动线程读取数据. 调用 run 方法启动新线程, 将调用 SensorService::threadLoop()方法.
在 while 循环中一直重复以下操作
3.1 调用 SensorDevice 的 poll 方法读取 HAL 层数据.
3.2 再调用 SensorEventConnection->sendEvents 将数据写到管道中. （客户端将监听该管道, 并读取数据）

### 客户端与服务端通信

![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/XmZ4Ii.jpg)

1 在图中可以看到有两个线程：
一个是服务端的一个线程, 这个线程负责源源不断的从 HAL 读取数据.
另一个是客户端的一个线程, 客户端线程负责从消息队列中读数据.

2 客户端可以创建多个消息队列, 一个消息队列对应有一个与服务器通信的连接接口.

3 服务端与客户端沟通的桥梁, 服务端读取到 HAL 层数据后, 会扫面有多少个与客户端连接的接口, 然后往每个接口的管道中写数据.

4 每一个连接接口都有对应的一个管道. 因为在实际使用中, 消息队列只会创建一个, 也就是说客户端与服务端之间的通信只有一个连接接口, 只有一个管道传数据.

### HAL 层传到 JAVA 层的数据的形式

数据是以一个结构体 sensors_event_t 的形式从 HAL 层传到 JNI 层.
Hardware/libhardware/include/hardware/Sensors.h：

在 JNI 层有一个 ASensorEvent 结构体与 sensors_event_t 向对应.
frameworks/native/include/android/sensor.h：

### Hal 层的库实现

1 Sensor 的库文件 sensors.msm8996.so .

2 三个与 HAL 对上层接口有关的几个结构体： /hardware/libhardware/include/hardware/hardware.h
struct hw_module_t;              //模块类型   
struct hw_module_methods_t;      //模块方法   
struct hw_device_t;  
2.1 一般来说, 在写 HAL 相关代码时都得包含这个 hardware.h 头文件.
2.2 每一个硬件模块都每必须有一个名为 HAL_MODULE_INFO_SYM 的数据结构变量, 它的第一个成员的类型必须为 hw_module_t.
2.3 硬件模块方法列表的定义, 这里只定义了一个 open 函数.
2.4 每一个设备数据结构的第一个成员函数必须是 hw_device_t 类型, 其次才是各个公共方法和属性.

3 sensorHal 对应的结构体：hardware/libhardware/include/hardware/sensors.h
struct sensors_module_t;              //模块类型   
struct hw_module_methods_t;      //模块方法   
struct sensors_poll_device_t;

4 实现以上 sensorHal 对应的结构体. （具体参看源码）
![](https://raw.githubusercontent.com/mikaelzero/ImageSource/main/uPic/7A3hz7.jpg)
