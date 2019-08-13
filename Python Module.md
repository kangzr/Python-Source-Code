**import任何模块都会加入到`sys.modules` dict中**

globals() 和 locals()返回全局和局部命名空间里的名字(函数内部使用，这返回函数内的相关名字)，return type是dict.

python内部reload机制只会更新或添加符号，不会删除符号

