查看fsimage和editslog：
	这两种文件是经过序列化的，费文本的，不能直接查看的，Hadoop
X中，hdfs提供了查看这两种文件的工具。
命令hdfs oiv用于将fsimage文件转换成其他格式的，如文本文件，xml文件

hdfs ovi
必须参数
	-i -inputfile<arg> 输入Fsimage文件
	-o -outputFile<arg>输出转换后的文件，如果存在，则会覆盖
可选参数
	-p -processor <arg> 将fsimage文件转换成哪种格式，默认为ls
	-h -help