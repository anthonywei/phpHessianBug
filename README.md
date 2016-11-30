# phpHessianBug
Type Stirng Expected Hessian bug report

PHP hessian Type String Expected bug解决方案

hessian协议对于long string类型传输有以下定义
(1) 块大小为1到31个byte时： 
用一个byte表示长度(一个byte最多表示31的长度)，后面跟具体数据。 

(2) 块大小为32到1023个byte时： 
用两个byte表示长度，后面跟具体数据，因只需要高位byte的4个bit位，加低位byte就能够表示1023的长度，为了不浪费，高位byte的另外4个bit位被压缩用于flag。 

(3) 块大小为1023到32k-1个byte时： 
用’s’前缀标识为小块，进行块读取。 

实际上在我们使用的版本也是类似，只是并非‘s’这类标识的，对数据进行分析发现是这样的
R?{data1}R?{data2}....0?{dataN}/R?{dataN}
其中R是数据块标识，?表示数据块的大小（16位），最后一段数据如果大于1023则用R标识，如果小于1023则用0标识

这里的32K指的是32个字，并非32个byte，因为hessian是utf8的字节，所以实际上32K的字大约是35K左右的字节（因为中文）

实测的时候，有个频道的接口，返回数据大约是70.08K左右，所以前面有2个数据块，后面有小于1023的数据块，如下
R?{data1}R?{data2}0?{data3}

hessian的php库代码首先读取数据data1，再读取data2都没有问题，但是在读取数据data3的时候，首先读取的前面的0，这个时候php做了一个判断

if(!$code)
    $code = $this->read();

意思是如果$code判断命中则读取下一个，问题就在于php的语言在判断字符串'0'的时候，也会命中，即!$code为真，于是这'0'就被忽略去了，继续读下一个字符，下一个字符的值报错，于是就一路错下去

解决方案：
php代码改为
if(!isset($code))
    $code = $this->read();​
