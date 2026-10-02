# Toast

```java
public Toast(..) {
    mTN = new TN(context.getPackageName(), looper);

    private static class TN {
        if (looper == null) {
            looper = Looper.myLooper();
            if (looper == null) {
                throw new RuntimeException(..);
            }
        }

        mHandler = new Handler(looper, null) {
            @override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case SHOW: {
                        IBinder token = (IBinder) msg.obj;
                        handleShow(token);
                        break;
                    }
                    case HIDE: {
                        handleHide();
                        mNextView = null;
                        break;
                    }
                }
            }
        }
    }
}
```