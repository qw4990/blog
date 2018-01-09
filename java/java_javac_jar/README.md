整理下 java, javac, jar 的手动用法;

## javac

用于编译java文件;

* 编译单个文件: 
  * javac com/bytedance/Hello.java;
* 编译多个文件: 
  * javac com/bytedance/Hello.java com.bytedance/Utils.java;
  * javac zyj\*.java;
  * javac \*.java;
* 编译带依赖的文件:
  * javac -cp /Users/zhangyuanjia/.m2/repository/com/bytedance/commons/0.0.4/commons-0.0.4.jar com/bytedance/Hello.java
* 指定输出文件夹:
  * javac -d classes com/bytedance/Hello.java;
* 更多: [http://www.codejava.net/java-core/tools/using-javac-command](http://www.codejava.net/java-core/tools/using-javac-command);

## java

用于运行class文件或者jar包;

* 直接运行:
  * java Hello;
  * java com.bytedance.Hello;
* 指定依赖:
  * java -cp /Users/zhangyuanjia/.m2/repository/com/bytedance/commons/0.0.4/commons-0.0.4.jar:. com.bytedance.Hello; 注意加上":.";
* 传递参数:
  * java Hello "zyj";
* 运行jar包:
  * java -jar MyApp.jar "code java";
* 更多: [http://www.codejava.net/java-core/tools/examples-of-using-java-command;](http://www.codejava.net/java-core/tools/examples-of-using-java-command)

## jar

用于打包java文件;

[http://www.codejava.net/java-core/tools/using-jar-command-examples](http://www.codejava.net/java-core/tools/using-jar-command-examples);



