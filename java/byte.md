|类型|大小|
|:--:|:--:|
|byte(字节)|8 bit(位)|
|char|2 byte|
|short|1 byte|
|int|2 byte|
|long|8 byte|
|float|4 byte|
|double|8 byte|


关于boolean占几个字节，java规范中，没有明确指出boolean的大小。在《Java虚拟机规范》给出了4个字节，和boolean数组1个字节的定义，具体还要看虚拟机实现是否按照规范来，所以1个字节、4个字节都是有可能的