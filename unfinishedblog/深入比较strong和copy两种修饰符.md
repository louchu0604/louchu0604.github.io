strong和copy是property的两个基本修饰符，在oc中，一般情况下用来修饰对象。
strong：property的默认修饰符。
copy：一般用来修饰拥有可变对象类型的不可变对象类型。比如NSString NSDictionary NSArray.
在学习oc的过程中，大多数的资料里都提到了以上两点，却没有详细说明这么做的原因。
本文旨在oc底层实现的基础上探讨以上两点的原因和合理性。

第一部分：先举很简单的例子来比较两者的区别

第二部分：从源码的角度来探讨问题


part one ：strong对象的底层实现
首先来看一段伪代码：
id obj= [[nsobject allo] init];
使用clang命令将以上代码转换成c++格式
clang -rewrite-objc 
<!-- 这里讲述strong对象生成的过程 弄清主要步骤 -->
<!-- z -->

part two：copy对象的底层实现

<!-- 这里讲copy -->
