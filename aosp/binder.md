# AIDL(Android Interface Definition Language)

## 使用
1. 先建立一个android工程，用来充当跨进程通讯的服务端
2. 创建一个包名用来存放aidl文件

android studio为我们自动生成了一个代理类 Stub 

3. 创建服务端service
4. 在AndroidMenifest中声明service
5. 新建一个工程，充当客户端