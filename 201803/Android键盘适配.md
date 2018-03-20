# Android软键盘分析

&emtp;&emtp;通过当前的上下文，获取系统级别的 Context.INPUT_METHOD_SERVICE 服务，拿到管理软键盘的控制类。通过该对象调用其成员方法来对软键盘进行操作。

```
InputMethodManager imm = (InputMethodManager) getSystemService(Context.INPUT_METHOD_SERVICE);

```
## InputMethodManager.java（IMM）：
输入法框架(IMF)的系统中心 API ，决定应用程序和当前输入法的相互影响关系，通过上述代码获得实例。
被描述为一个管理其他部分相互作用的系统中心类，对外客户端的一个API，而InputManager实现了一个运行用户生成文本的特殊的交互模型，系统绑定当前正在使用的输入法，引起他被创建并运行，并管理软键盘，值得注意的是同事只有一个输入法在运行，服务的对象通常是TextView或者其子类。

##显示输入法流程
* InputMethodMangerService 负责管理系统的所有输入法，包括输入法 service(IMS) 加载及切换。
* 程序获得焦点是，就会通过 InputMethodManager 向 InputMethodMangerService 发出请求绑定自己到当前输入法上。
* 当程序的某个需要输入法的 view 比如 EditView 获得焦点时，就会通过 InputMethodManager 向 InputMethodManagerService 请求显示输入法，而这时 InputMethodManagerService 收到请求后，会将请求的 EditText 的数据通信接口发送给当前输入法，并请求显示输入法。输入法收到请求后，就显示自己的 UI dialog,同时保存目标 view 的数据结构，当用户实现输入后，直接通过 view 的数据通信接口将字符传递到对应的 View 。而这个数据传递的过程是通过IPC机制进行通讯的

##安全性问题
本质是不受约束地驱动UI闭馆监视用户输入，必须小心限制他们的选择和相互作用，当然系统已经帮我做了，主要是以下几点
* 只允许系统访问经BIND_INPUT_METHOD权限许可访问IME的InputMethod接口。通过绑定到要求这个权限的服务来强制实现这一点。
* 一个IME永远不能在屏幕关闭的时与InputConnection交互，
* IMF中可能有许多客户进程，但是同一时间只有一个是激活的，未激活客户端不能与IMF核心交互通过小叔机制实现。
## InputMethodMangerService.java（IMS）源码解读
控制的软键盘的实际类是InputMethodMangerService.java文件里面，我们来看看大概是怎么个回事，此类是在源码中的 /frameworks/base/services/core/java/com/android/server/InputMethodMangerService.java，源码依托于Android 6.0。
在开头写明了，此类是系统服务类，并且是私有的类型
    ```
   / **
     * This class provides a system service that manages input methods.
    */
    
    ```
    


###显示软键盘

这里面有三个方法：

```
  public boolean showSoftInput(View view, int flags) {
        throw new RuntimeException("Stub!");
    }

    public boolean showSoftInput(View view, int flags, ResultReceiver resultReceiver) {
        throw new RuntimeException("Stub!");
    }
```
从源码中我们可以得到两个方法都会调用到

```

    @Override
    public boolean showSoftInput(IInputMethodClient client, int flags,
            ResultReceiver resultReceiver) {
        if (!calledFromValidUser()) {
            return false;
        }
        int uid = Binder.getCallingUid();
        long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mMethodMap) {
                if (mCurClient == null || client == null
                        || mCurClient.client.asBinder() != client.asBinder()) {
                    try {
                        if (!mIWindowManager.inputMethodClientHasFocus(client)) {
                            Slog.w(TAG, "Ignoring showSoftInput of uid " + uid + ": " + client);
                            return false;
                        }
                    } catch (RemoteException e) {
                        return false;
                    }
                }

                if (DEBUG) Slog.v(TAG, "Client requesting input be shown");
                return showCurrentInputLocked(flags, resultReceiver);
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }

      boolean showCurrentInputLocked(int flags, ResultReceiver resultReceiver) {
        mShowRequested = true;
        if ((flags&InputMethodManager.SHOW_IMPLICIT) == 0) {
            mShowExplicitlyRequested = true;
        }
        if ((flags&InputMethodManager.SHOW_FORCED) != 0) {
            mShowExplicitlyRequested = true;
            mShowForced = true;
        }

        if (!mSystemReady) {
            return false;
        }

        boolean res = false;
        if (mCurMethod != null) {
            if (DEBUG) Slog.d(TAG, "showCurrentInputLocked: mCurToken=" + mCurToken);
            executeOrSendMessage(mCurMethod, mCaller.obtainMessageIOO(
                    MSG_SHOW_SOFT_INPUT, getImeShowFlags(), mCurMethod,
                    resultReceiver));
            mInputShown = true;
            if (mHaveConnection && !mVisibleBound) {
                bindCurrentInputMethodService(
                        mCurIntent, mVisibleConnection, Context.BIND_AUTO_CREATE
                                | Context.BIND_TREAT_LIKE_ACTIVITY
                                | Context.BIND_FOREGROUND_SERVICE);
                mVisibleBound = true;
            }
            res = true;
        } else if (mHaveConnection && SystemClock.uptimeMillis()
                >= (mLastBindTime+TIME_TO_RECONNECT)) {
      
            EventLog.writeEvent(EventLogTags.IMF_FORCE_RECONNECT_IME, mCurMethodId,
                    SystemClock.uptimeMillis()-mLastBindTime,1);
            Slog.w(TAG, "Force disconnect/connect to the IME in showCurrentInputLocked()");
            mContext.unbindService(this);
            bindCurrentInputMethodService(mCurIntent, this, Context.BIND_AUTO_CREATE
                    | Context.BIND_NOT_VISIBLE);
        } else {
            if (DEBUG) {
                Slog.d(TAG, "Can't show input: connection = " + mHaveConnection + ", time = "
                        + ((mLastBindTime+TIME_TO_RECONNECT) - SystemClock.uptimeMillis()));
            }
        }

        return res;
    }

```

通过源码中我们可以得到实际调用的是 showCurrentInputLocked 函数，而还有有意思的地方是在此方法中， flag 暂时是没有实际作用的, flags 与参数进行相与运算，并且对 mShowExplicitlyRequested 和 mShowForced 进行赋值，但是这三个值在当前方法中均没有使用。也就是说传入 flag 的值均不会影响软键盘的显示。但却影响软键盘的隐藏，下文就会讲到。

###隐藏软键盘的底层实现：

```
 @Override
    public boolean hideSoftInput(IInputMethodClient client, int flags,
            ResultReceiver resultReceiver) {
        if (!calledFromValidUser()) {
            return false;
        }
        int uid = Binder.getCallingUid();
        long ident = Binder.clearCallingIdentity();
        try {
            synchronized (mMethodMap) {
                if (mCurClient == null || client == null
                        || mCurClient.client.asBinder() != client.asBinder()) {
                    try {
                        // We need to check if this is the current client with
                        // focus in the window manager, to allow this call to
                        // be made before input is started in it.
                        if (!mIWindowManager.inputMethodClientHasFocus(client)) {
                            if (DEBUG) Slog.w(TAG, "Ignoring hideSoftInput of uid "+ uid + ": " + client);
                            return false;
                        }
                    } catch (RemoteException e) {
                        return false;
                    }
                }

                if (DEBUG) Slog.v(TAG, "Client requesting input be hidden");
                return hideCurrentInputLocked(flags, resultReceiver);
            }
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }

    boolean hideCurrentInputLocked(int flags, ResultReceiver resultReceiver) {
        if ((flags&InputMethodManager.HIDE_IMPLICIT_ONLY) != 0
                && (mShowExplicitlyRequested || mShowForced)) {
            if (DEBUG) Slog.v(TAG, "Not hiding: explicit show not cancelled by non-explicit hide");
            return false;
        }
        if (mShowForced && (flags&InputMethodManager.HIDE_NOT_ALWAYS) != 0) {
            if (DEBUG) Slog.v(TAG, "Not hiding: forced show not cancelled by not-always hide");
            return false;
        }
        final boolean shouldHideSoftInput = (mCurMethod != null) && (mInputShown || (mImeWindowVis & InputMethodService.IME_ACTIVE) != 0);
        boolean res;
        if (shouldHideSoftInput) {

            executeOrSendMessage(mCurMethod, mCaller.obtainMessageOO(
                    MSG_HIDE_SOFT_INPUT, mCurMethod, resultReceiver));
            res = true;
        } else {
            res = false;
        }
        if (mHaveConnection && mVisibleBound) {
            mContext.unbindService(mVisibleConnection);
            mVisibleBound = false;
        }
        mInputShown = false;
        mShowRequested = false;
        mShowExplicitlyRequested = false;
        mShowForced = false;
        return res;
    }
```

最终调用的也是 hideCurrentInputLocked 方法，在这个方法里面，我们传入的 flag 又有了作用。这里的判断用到了隐藏软键盘时使用的 flags，以及 mShowExplicitlyRequested 和 mShowForced 的值，如果不满足条件就直接返回了。可以看到当 flags 为 HIDE_IMPLICIT_ONLY 时，如果 mShowExplicitlyRequested 和 mShowForced 任意一个为 true，都会返回 false。当 flags 为 HIDE_NOT_ALWAYS 时，如果 mShowForced 为 true ，也会返回 false ，当 flags 为0时，两个 if 条件都不满足。这就解释了显示软键盘时使用的 flags 影响的是后面隐藏软键盘是否成功，以及隐藏软键盘时使用0作为 flags 总是能够隐藏软键盘。



```
   /**
     * 动态隐藏软键盘
     */
    public static void hideSoftInput(Activity activity) {
        View view = activity.getWindow().peekDecorView();
        if (view != null) {
            InputMethodManager inputManger = (InputMethodManager) activity
                    .getSystemService(Context.INPUT_METHOD_SERVICE);
            inputManger.hideSoftInputFromWindow(view.getWindowToken(), 0);
        }
    }

    /**
     * 动态隐藏软键盘
     */
    public static void hideSoftInput(Context context, EditText edit) {
        edit.requestFocus();
        if (context != null) {
            InputMethodManager inputManger = (InputMethodManager) context
                    .getSystemService(Context.INPUT_METHOD_SERVICE);
            inputManger.hideSoftInputFromWindow(edit.getWindowToken(), 0);
        }
    }

    /**
     * 动态显示软键盘
     */
    public static void showSoftInput(final Context context, final EditText edit) {
        if (context == null) {
            return;
        }
        edit.setFocusable(true);
        edit.setFocusableInTouchMode(true);
        edit.postDelayed(new Runnable() {
            @Override
            public void run() {
                edit.requestFocus();
                InputMethodManager inputManger = (InputMethodManager) context
                        .getSystemService(Context.INPUT_METHOD_SERVICE);
                inputManger.showSoftInput(edit, 0);
            }
        }, 200);
    }

    /**
     * 切换键盘显示与否状态
     */
    public static void toggleSoftInput(Context context, EditText edit) {
        edit.setFocusable(true);
        edit.setFocusableInTouchMode(true);
        edit.requestFocus();
        InputMethodManager inputManager = (InputMethodManager) context
                .getSystemService(Context.INPUT_METHOD_SERVICE);
        inputManager.toggleSoftInput(InputMethodManager.SHOW_FORCED, 0);
    }

    /**
     * 监听键盘显示隐藏
     * @param activity
     * @param rootView mRootLayout为xml布局文件中的顶层布局（根布局）
     * @param listener
     */
    public static void keyboardStateWatcher(Activity activity, View rootView, SoftKeyboardStateWatcher.SoftKeyboardStateListener listener) {
        new SoftKeyboardStateWatcher(rootView, activity).addSoftKeyboardStateListener(listener);
}
```


