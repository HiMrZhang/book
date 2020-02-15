# Java指令-Djava.ext.dirs的陷阱

这两天由于现场需求，需要把ES索引写入插件改造成带安全认证的方式，加密算法是由java.security相关类实现的。在DEMO中测试了若干次都没问题，但是放到生成环境中就是报NoSuchAlgorithmException的错误，相关错误如下：

        这个问题困扰了好久，经过多方面的排查，其中细节不一一描述，最终发现是生产环境下的启动指令属性-Djava.ext.dirs出现了问题。-Djava.ext.dirs会覆盖Java本身的ext设置，java.ext.dirs指定的目录由ExtClassLoader加载器加载，如果您的程序没有指定该系统属性，那么该加载器默认加载$JAVA\_HOME/jre/lib/ext目录下的所有jar文件。但如果你手动指定系统属性且忘了把$JAVA\_HOME/jre/lib/ext路径给加上，那么ExtClassLoader不会去加载$JAVA\_HOME/lib/ext下面的jar文件，这意味着你将失去一些功能，例如java自带的加解密算法实现。

       OK问题分析到这儿，什么原因已经很明朗，解决方案也很简单，只需在改路径后面补上ext的路径即可！比如：

       -Djava.ext.dirs=./plugin:$JAVA\_HOME/jre/lib/ext。windows环境下运行程序，应该用分号替代冒号来分隔。

————————————————

版权声明：本文为CSDN博主「cyony」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。

原文链接：https://blog.csdn.net/cyony/article/details/74375251



