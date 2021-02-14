### ConstantValue

> 转载自：https://blog.csdn.net/weixin_43689480/article/details/96099841

## ConstantValue属性在类加载过程的准备阶段做的事情是什么

在编译时Javac将会为被static和final修改的常量生成ConstantValue属性(此时ConstantValue属性的值是多少，暂时不知道，)，在类加载的准备阶段虚拟机便会根据ConstantValue为常量设置相应的值（这个值是什么意思呢，比如我们在程序中定义final static int a = 100，那么这个a就是ConstantValue属性，然后在准备阶段中a的值就会变成100），这就是ConstantValue属性在类加载过程的准备阶段做的事

## ConstantValue属性，且限于基本类型和String，为什么会这样呢

因为从常量池中只能引用到基本类型和String类型的字面量，所以就只能限于基本类型和String