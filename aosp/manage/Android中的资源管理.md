# 资源管理

## 资源管理框架

主要由于Resources和AssetManager两个类来实现的。
其中Resources可以根据ID来查找资源，而AssetManager类根据文件名来找找资源

字符串资源直接编译在resources.arsc文件中，而界面布局资源是在apk包里面对应的单独文件的

```sequence {theme="simple"}
ActivityThread->ActivityThread:performLaunchActivity
ActivityThread->ActivityThread:createBaseContextForActivity
ActivityThread->ContextImpl:createActivityContext
ContextImpl->ResourcesManager:createBaseActivityResources
ResourcesManager-->ContextImpl:Resources
ContextImpl-->ActivityThread:appContext
ActivityThread->Instrumentation:newActivity
```

```sequence {theme="simple"}
Activity->Activity: 1：getResources
Activity-->ContextImpl: 2:getResources
ContextImpl->ResourcesManager: 3:getResources
ResourcesManager->ResourcesManager: 4:createBaseActivityResources
ResourcesManager->ResourcesManager: 5: getOrCreateActivityResourcesStructLocked
ResourcesManager->ResourcesManager: 6: getOrCreateActivityResourcesStructLocked:ActivityResources(init)
ResourcesManager->ResourcesManager: 7: updateResourcesForActivity
ResourcesManager->ResourcesManager: 8: getOrCreateResources
```

