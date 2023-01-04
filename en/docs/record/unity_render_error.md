### call to OpenGL ES API with no current context (logged once per thread)

关闭多线程渲染

### 获取 android 的 class 的时候卡住

场景：unity 中获取的类的在 base 的 server 中

因为 server 部分的代码禁止访问

### 从 android 获取到的 textureId 无法显示出来

场景：
在系统中拦截 startActivity, 启动 activity 时创建对应的 virtualDisplay, 相同包名则或者原有的 displayID 进行显示, 然后把该 display 对应的 texture 给到 unity 让 unity 进行显示, 比如显示在一个面板上

1. 把 unity 初始化 gl 写在了 client 端, server 直接获取到的 surface 为 null, 处理方案改成了 textureID 在 unity 创建, server 先拿固定的 id 创建 surface
2. 还是为空, 猜测创建 surface 要和创建 textureid 的线程为同一个, 准备把创建 virtualDisplay 移动到 client 端处理试试看；先创建一个 display, 根据 bilibili 包名创建, 然后运行自己的 app, 直接拿这个 display
3. 思路不对, 不应该放在两个进程处理, 试试放在非 server 端一起处理看看
4. 都在同一个进程还是不行, 试试直接使用 app 的 surfaceview 看看能否正常显示（surfaceview 可以正常显示）
5. 在工模下, 测试 unity 能否接收 virtualDisplay(一开始不行, 后面成功后, 主要区别在于, unity 的项目一个有 vr 环境一个没有)
6. 测试了在 client 端的 Activity 创建 virtualDisplay, 生效.
7. 排查下, 带有 vr 环境的 unity 项目, 删除了 vr 环境是否还能正常运行
8. 已删除 vr 环境, 并且新建了空的场景, 生效, 之前不生效的原因暂时未查到, 可能和一些 unity 的配置相关,修改了配置中的 normal map encoding 的值为 xyz, 之前用的是 2021 版本的 unity 编辑器, 老版本没有其他选项
