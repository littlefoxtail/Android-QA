# 进程创建流程

进程：每个App在启动前必须先创建一个进程，该进程由`Zygote`fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。

线程：线程对应用开发者来说非常熟悉，线程没有自己独立的地址空间，而是在其所在进程之间资源共享。

- system_server进程：是用于管理整个Java framework层，包含ActivityManager，PowerManager等各种系统服务;
- Zygote进程：是Android系统的首个Java进程，Zygote是所有Java进程的父进程，包括 system_server进程以及所有的App进程都是Zygote的子进程，注意这里说的是子进程，而非子线程。

1. App发起进程：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
2. system_server进程：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
3. zygote进程：在执行ZygoteInit.main()后便进入runSelectLoop()循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；
4. 新进程：执行handleChildProc方法，最后调用ActivityThread.main()方法。

## system_server发起请求

### Process.start

```java
public class Process {
    /**
     * Start a new process.
     *
     * <p>If processes are enabled, a new process is created and the
     * static main() function of a <var>processClass</var> is executed there.
     * The process will continue running after this function returns.
     *
     * <p>If processes are not enabled, a new thread in the caller's
     * process is created and main() of <var>processclass</var> called there.
     *
     * <p>The niceName parameter, if not an empty string, is a custom name to
     * give to the process instead of using processClass.  This allows you to
     * make easily identifyable processes even if you are using the same base
     * <var>processClass</var> to start them.
     *
     * When invokeWith is not null, the process will be started as a fresh app
     * and not a zygote fork. Note that this is only allowed for uid 0 or when
     * runtimeFlags contains DEBUG_ENABLE_DEBUGGER.
     *
     */
    public static final ProcessStartResult start(..) {
        return ZYGOTE_PROCESS.start(..);
    }
}
```

```java
public class ZygoteProcess {
    /**
     * Starts a new process via the zygote mechanism.
     */
    private Process.ProcessStartResult startViaZygote(..)
                                                      throws ZygoteStartFailedEx {
        ArrayList<String> argsForZygote = new ArrayList<String>();

        // --runtime-args, --setuid=, --setgid=,
        // and --setgroups= must go first
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
            argsForZygote.add("--mount-external-default");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
            argsForZygote.add("--mount-external-read");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
            argsForZygote.add("--mount-external-write");
        }
        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

        // --setgroups is a comma-separated list
        if (gids != null && gids.length > 0) {
            StringBuilder sb = new StringBuilder();
            sb.append("--setgroups=");

            int sz = gids.length;
            for (int i = 0; i < sz; i++) {
                if (i != 0) {
                    sb.append(',');
                }
                sb.append(gids[i]);
            }

            argsForZygote.add(sb.toString());
        }

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }

        if (seInfo != null) {
            argsForZygote.add("--seinfo=" + seInfo);
        }

        if (instructionSet != null) {
            argsForZygote.add("--instruction-set=" + instructionSet);
        }

        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }

        if (invokeWith != null) {
            argsForZygote.add("--invoke-with");
            argsForZygote.add(invokeWith);
        }

        if (startChildZygote) {
            argsForZygote.add("--start-child-zygote");
        }

        argsForZygote.add(processClass);

        if (extraArgs != null) {
            for (String arg : extraArgs) {
                argsForZygote.add(arg);
            }
        }

        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                              useBlastulaPool,
                                              argsForZygote);
        }
    }
}
```

该过程主要生成`argsForZygote`数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数

### zygoteSendArgsAndGetResult

```java
private static Process.ProcessStartResult zygoteSendArgsAndGetResult(..)
            throws ZygoteStartFailedEx {
        // Throw early if any of the arguments are malformed. This means we can
        // avoid writing a partial response to the zygote.
        for (String arg : args) {
            if (arg.indexOf('\n') >= 0) {
                throw new ZygoteStartFailedEx("embedded newlines not allowed");
            }
        }

        /*
         * See com.android.internal.os.ZygoteArguments.parseArgs()
         * Presently the wire format to the zygote process is:
         * a) a count of arguments (argc, in essence)
         * b) a number of newline-separated argument strings equal to count
         *
         * After the zygote process reads these it will write the pid of
         * the child or -1 on failure, followed by boolean to
         * indicate whether a wrapper process was used.
         */
        String msgStr = Integer.toString(args.size()) + "\n"
                        + String.join("\n", args) + "\n";

        // Should there be a timeout on this?
        Process.ProcessStartResult result = new Process.ProcessStartResult();

        // TODO (chriswailes): Move branch body into separate function.
        if (useBlastulaPool && Zygote.BLASTULA_POOL_ENABLED && isValidBlastulaCommand(args)) {
            LocalSocket blastulaSessionSocket = null;

            try {
                blastulaSessionSocket = zygoteState.getBlastulaSessionSocket();

                final BufferedWriter blastulaWriter =
                        new BufferedWriter(
                                new OutputStreamWriter(blastulaSessionSocket.getOutputStream()),
                                Zygote.SOCKET_BUFFER_SIZE);
                final DataInputStream blastulaReader =
                        new DataInputStream(blastulaSessionSocket.getInputStream());

                blastulaWriter.write(msgStr);
                blastulaWriter.flush();
                // 等地啊socket服务端（即zygote）返回新创建的进程pid；
                result.pid = blastulaReader.readInt();
                // Blastulas can't be used to spawn processes that need wrappers.
                result.usingWrapper = false;

                if (result.pid < 0) {
                    throw new ZygoteStartFailedEx("Blastula specialization failed");
                }

                return result;
            } catch (IOException ex) {
                // If there was an IOException using the blastula pool we will log the error and
                // attempt to start the process through the Zygote.
                Log.e(LOG_TAG, "IO Exception while communicating with blastula pool - "
                               + ex.toString());
            } finally {
                try {
                    blastulaSessionSocket.close();
                } catch (IOException ex) {
                    Log.e(LOG_TAG, "Failed to close blastula session socket: " + ex.getMessage());
                }
            }
        }

        try {
            final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
            final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;

            zygoteWriter.write(msgStr);
            zygoteWriter.flush();

            // Always read the entire result from the input stream to avoid leaving
            // bytes in the stream for future process starts to accidentally stumble
            // upon.
            result.pid = zygoteInputStream.readInt();
            result.usingWrapper = zygoteInputStream.readBoolean();
        } catch (IOException ex) {
            zygoteState.close();
            Log.e(LOG_TAG, "IO Exception while communicating with Zygote - "
                    + ex.toString());
            throw new ZygoteStartFailedEx(ex);
        }

        if (result.pid < 0) {
            throw new ZygoteStartFailedEx("fork() failed");
        }

        return result;
    }
```

