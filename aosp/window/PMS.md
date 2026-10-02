# PMS的启动过程
```java
//创建pms对象
            mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                    domainVerificationService, mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF,
                    mOnlyCore);
        } finally {
            Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
        }

```

启动过程是整个核心
