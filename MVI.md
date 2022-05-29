MVI与MVVM相似，借鉴了前端框架的思想，更加强调数据的单向流动和唯一数据源。
![[Pasted image 20220521163804.png]]
1. Model：与MVVM中的Model不同的是，MVI的Model主要指UI状态（State）。例如页面加载状态、控件位置等都是一种UI状态。
2. View：与其他MVX中的View一致，可能是一个Activity或者任意UI承载单元。MVI中的View通过订阅Model的变化实现界面刷新。
3. Intent：此Intent不是Activity的Intent，用户的任何操作都被包装成Intent后发送给Model层进行数据请求。
