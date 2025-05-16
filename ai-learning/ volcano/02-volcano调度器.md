

# 插件
插件主要实现了3个函数：Name， OnSessionOpen， OnSessionClose
OnSessionOpen在会话开始时执行一些操作，并注册一些关于调度细节的函数。
OnSessionClose在会话结束时清理一些资源。
下面写一些volcano现在有的插件