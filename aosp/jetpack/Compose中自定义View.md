#compose
# Compose中的Canvas
DrawScope：
> 将DrawScope的坐标空间 向左向上平移，以及修改当前绘画区域的尺寸。这提供了一个回调，以便在修改后的坐标空间内发出更多的绘画指令。这个方法将DrawScope的宽度修改为相当于宽度-（左+右），以及修改高度-（顶+底）。
```
```