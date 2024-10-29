---
date: 2023-10-30 22:52:48
tag:
 - Android
---

# 【探索与避坑】自定义Toast：为什么在Activity finish时调用会显示不出来？

## 问题起源
- 我们的项目早期开发时，使用[Toasty](https://github.com/GrenderG/Toasty)这个开源库实现带状态的Toast，并进行一层封装，通过Handler post到主线程的消息队列执行，达到调用时只要传入字符串这一个参数的效果，使用良好，代码如下：
```kotlin
fun toastInfo(message: String) {
    Handler(Looper.getMainLooper()).post {
        Toasty.info(MyApplication.instance, message).show()
    }
}
```
- 经过若干年的迭代，安卓版本升级，targetSdkVersion也提升到了30+。近期在测试时，猛然发现一些地方的Toast居然没有显示出来。于是开始了问题排除和溯源。

## 初步探索
- 首先，并不是所有地方的Toast都不能正常弹出。于是我开始查找代码，看不正常的地方有什么特殊之处。然后发现，不能正常弹出的地方都是这样写的：
```kotlin
toastInfo("Toast内容")
finish()
```
- 尝试把finish去掉，Toast显示了；把finish移到调用Toast前，Toast也显示了。好吧，把所有这样调用的地方修改一下，问题解决！
- 但是这样并没有找到问题的根本原因，要改的地方比较多且易漏，而且后续其他的开发者也可能犯同样的错误。加之旧版本的安卓系统上并没有遇到这个问题，且Toasty库常年没有更新，难道是这个库和新的安卓有兼容问题？
- 那就先做个排除法。把封装的toastInfo方法中Toasty的那行调用直接改成原生Toast：```
Toast.makeText(MyApplication.instance, message, Toast.LENGTH_SHORT).show()```,编译运行，Toast正常显示。看来Toasty这个库有问题。它是怎么做实现的呢？

## 查看源码
- 查阅Toasty的实现代码，调用info()之后返回的是一个标准的Toast，这个方法又调用custom方法，进行自定义布局。我把这个方法贴在下面：
```java
public static Toast custom(@NonNull Context context, @NonNull CharSequence message, Drawable icon,
                           @ColorInt int tintColor, @ColorInt int textColor, int duration,
                           boolean withIcon, boolean shouldTint) {
    final Toast currentToast = Toast.makeText(context, "", duration);
    final View toastLayout = ((LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE))
            .inflate(R.layout.toast_layout, null);
    final LinearLayout toastRoot = toastLayout.findViewById(R.id.toast_root);
    final ImageView toastIcon = toastLayout.findViewById(R.id.toast_icon);
    final TextView toastTextView = toastLayout.findViewById(R.id.toast_text);
    Drawable drawableFrame;

    if (shouldTint)
        drawableFrame = ToastyUtils.tint9PatchDrawableFrame(context, tintColor);
    else
        drawableFrame = ToastyUtils.getDrawable(context, R.drawable.toast_frame);
    ToastyUtils.setBackground(toastLayout, drawableFrame);

    if (withIcon) {
        if (icon == null)
            throw new IllegalArgumentException("Avoid passing 'icon' as null if 'withIcon' is set to true");
        if (isRTL && Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1)
            toastRoot.setLayoutDirection(View.LAYOUT_DIRECTION_RTL);
        ToastyUtils.setBackground(toastIcon, tintIcon ? ToastyUtils.tintIcon(icon, textColor) : icon);
    } else {
        toastIcon.setVisibility(View.GONE);
    }

    toastTextView.setText(message);
    toastTextView.setTextColor(textColor);
    toastTextView.setTypeface(currentTypeface);
    toastTextView.setTextSize(TypedValue.COMPLEX_UNIT_SP, textSize);

    currentToast.setView(toastLayout);

    if (!allowQueue) {
        if (lastToast != null)
            lastToast.cancel();
        lastToast = currentToast;
    }

    // Make sure to use default values for non-specified ones.
    currentToast.setGravity(
            toastGravity == -1 ? currentToast.getGravity() : toastGravity,
            xOffset == -1 ? currentToast.getXOffset() : xOffset,
            yOffset == -1 ? currentToast.getYOffset() : yOffset
    );

    return currentToast;
}
```
- 重点来了：第33行setView是一个Deprecated的方法：
> Custom toast views are deprecated. Apps can create a standard text toast with the makeText(Context, CharSequence, int) method, or use a Snackbar when in the foreground. Starting from Android Build.VERSION_CODES.R, apps targeting API level Build.VERSION_CODES.R or higher that are in the background will not have custom toast views displayed.
- 说自定义的Toast在安卓11+，targetAPI30+，且APP在background状态时，则不会显示。可是我的APP明明在前台啊，只不过正好在finish Activity；但和标准的Toast的主要区别也就这一处了。好吧，那就去看看谷歌是怎么改动这里的代码的吧。

- 上[cs.android.com](https://cs.android.com)看看谷歌的实现。Toast类的show方法的代码是这样的：
```java
public void show() {
    if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
        checkState(mNextView != null || mText != null, "You must either set a text or a view");
    } else {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }
    }

    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;
    final int displayId = mContext.getDisplayId();

    try {
        if (Compatibility.isChangeEnabled(CHANGE_TEXT_TOASTS_IN_THE_SYSTEM)) {
            if (mNextView != null) {
                // It's a custom toast
                service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
            } else {
                // It's a text toast
                ITransientNotificationCallback callback =
                        new CallbackBinder(mCallbacks, mHandler);
                service.enqueueTextToast(pkg, mToken, mText, mDuration, displayId, callback);
            }
        } else {
            service.enqueueToast(pkg, mToken, tn, mDuration, displayId);
        }
    } catch (RemoteException e) {
        // Empty
    }
}
```
- 看来enqueueToast是重点，跟进去看一下：
```java
        @Override
        public void enqueueToast(String pkg, IBinder token, ITransientNotification callback,
                int duration, boolean isUiContext, int displayId) {
            enqueueToast(pkg, token, /* text= */ null, callback, duration, isUiContext, displayId,
                    /* textCallback= */ null);
        }

        private void enqueueToast(String pkg, IBinder token, @Nullable CharSequence text,
                @Nullable ITransientNotification callback, int duration, boolean isUiContext,
                int displayId, @Nullable ITransientNotificationCallback textCallback) {
            if (DBG) {
                Slog.i(TAG, "enqueueToast pkg=" + pkg + " token=" + token + " duration=" + duration
                        + " isUiContext=" + isUiContext + " displayId=" + displayId);
            }

            if (pkg == null || (text == null && callback == null)
                    || (text != null && callback != null) || token == null) {
                Slog.e(TAG, "Not enqueuing toast. pkg=" + pkg + " text=" + text + " callback="
                        + " token=" + token);
                return;
            }

            final int callingUid = Binder.getCallingUid();
            if (!isUiContext && displayId == Display.DEFAULT_DISPLAY
                    && mUm.isVisibleBackgroundUsersSupported()) {
                // When the caller is a visible background user using a non-UI context (like the
                // application context), the Toast must be displayed in the display the user was
                // started visible on.
                int userId = UserHandle.getUserId(callingUid);
                int userDisplayId = mUmInternal.getMainDisplayAssignedToUser(userId);
                if (displayId != userDisplayId) {
                    if (DBG) {
                        Slogf.d(TAG, "Changing display id from %d to %d on user %d", displayId,
                                userDisplayId, userId);
                    }
                    displayId = userDisplayId;
                }
            }

            checkCallerIsSameApp(pkg);
            final boolean isSystemToast = isCallerIsSystemOrSystemUi()
                    || PackageManagerService.PLATFORM_PACKAGE_NAME.equals(pkg);
            boolean isAppRenderedToast = (callback != null);
            if (!checkCanEnqueueToast(pkg, callingUid, displayId, isAppRenderedToast,
                    isSystemToast)) {
                return;
            }

            synchronized (mToastQueue) {
                int callingPid = Binder.getCallingPid();
                final long callingId = Binder.clearCallingIdentity();
                try {
                    ToastRecord record;
                    int index = indexOfToastLocked(pkg, token);
                    // If it's already in the queue, we update it in place, we don't
                    // move it to the end of the queue.
                    if (index >= 0) {
                        record = mToastQueue.get(index);
                        record.update(duration);
                    } else {
                        // Limit the number of toasts that any given package can enqueue.
                        // Prevents DOS attacks and deals with leaks.
                        int count = 0;
                        final int N = mToastQueue.size();
                        for (int i = 0; i < N; i++) {
                            final ToastRecord r = mToastQueue.get(i);
                            if (r.pkg.equals(pkg)) {
                                count++;
                                if (count >= MAX_PACKAGE_TOASTS) {
                                    Slog.e(TAG, "Package has already queued " + count
                                            + " toasts. Not showing more. Package=" + pkg);
                                    return;
                                }
                            }
                        }

                        Binder windowToken = new Binder();
                        mWindowManagerInternal.addWindowToken(windowToken, TYPE_TOAST, displayId,
                                null /* options */);
                        record = getToastRecord(callingUid, callingPid, pkg, isSystemToast, token,
                                text, callback, duration, windowToken, displayId, textCallback);
                        mToastQueue.add(record);
                        index = mToastQueue.size() - 1;
                        keepProcessAliveForToastIfNeededLocked(callingPid);
                    }
                    // If it's at index 0, it's the current toast.  It doesn't matter if it's
                    // new or just been updated, show it.
                    // If the callback fails, this will remove it from the list, so don't
                    // assume that it's valid after this.
                    if (index == 0) {
                        showNextToastLocked(false);
                    }
                } finally {
                    Binder.restoreCallingIdentity(callingId);
                }
            }
        }
```
- 44行的checkCanEnqueueToast进去看一下：
```java
        private boolean checkCanEnqueueToast(String pkg, int callingUid, int displayId,
                boolean isAppRenderedToast, boolean isSystemToast) {
            final boolean isPackageSuspended = isPackagePaused(pkg);
            final boolean notificationsDisabledForPackage = !areNotificationsEnabledForPackage(pkg,
                    callingUid);

            final boolean appIsForeground;
            final long callingIdentity = Binder.clearCallingIdentity();
            try {
                appIsForeground = mActivityManager.getUidImportance(callingUid)
                        == IMPORTANCE_FOREGROUND;
            } finally {
                Binder.restoreCallingIdentity(callingIdentity);
            }

            if (!isSystemToast && ((notificationsDisabledForPackage && !appIsForeground)
                    || isPackageSuspended)) {
                Slog.e(TAG, "Suppressing toast from package " + pkg
                        + (isPackageSuspended ? " due to package suspended."
                        : " by user request."));
                return false;
            }

            if (blockToast(callingUid, isSystemToast, isAppRenderedToast,
                    isPackageInForegroundForToast(callingUid))) {
                Slog.w(TAG, "Blocking custom toast from package " + pkg
                        + " due to package not in the foreground at time the toast was posted");
                return false;
            }

            int userId = UserHandle.getUserId(callingUid);
            if (!isSystemToast && !mUmInternal.isUserVisible(userId, displayId)) {
                Slog.e(TAG, "Suppressing toast from package " + pkg + "/" + callingUid + " as user "
                        + userId + " is not visible on display " + displayId);
                return false;
            }

            return true;
        }
```
- 这些情况都会导致Toast不显示，怎么查看系统输出的日志呢？上网查找后得到答案：```adb logcat -b system```。终于捕获到了踪迹：```W NotificationService: Blocking custom toast from package com.my.package.name due to package not in the foreground at time the toast was posted```
- 好吧，还真是说不在foreground导致的。25行isPackageInForegroundForToast进去看一下：
```java
/**
     * Implementation note: Our definition of foreground for toasts is an implementation matter
     * and should strike a balance between functionality and anti-abuse effectiveness. We
     * currently worry about the following cases:
     * <ol>
     *     <li>App with fullscreen activity: Allow toasts
     *     <li>App behind translucent activity from other app: Block toasts
     *     <li>App in multi-window: Allow toasts
     *     <li>App with expanded bubble: Allow toasts
     *     <li>App posting toasts on onCreate(), onStart(), onResume(): Allow toasts
     *     <li>App posting toasts on onPause(), onStop(), onDestroy(): Block toasts
     * </ol>
     * Checking if the UID has any resumed activities satisfy use-cases above.
     *
     * <p>Checking if {@code mActivityManager.getUidImportance(callingUid) ==
     * IMPORTANCE_FOREGROUND} does not work because it considers the app in foreground if it has
     * any visible activities, failing case 2 in list above.
     */
    private boolean isPackageInForegroundForToast(int callingUid) {
        return mAtm.hasResumedActivity(callingUid);
    }
```
- 看来问题找到了。谷歌在这里所说的background并不是我想当然的以为APP不在前台，而是存在ResumedActivity。而我调用Toast同时finish的代码正好命中最后一条```App posting toasts on onPause(), onStop(), onDestroy(): Block toasts```，导致了Toast不展示。

## 解决问题
问题找到，解决的思路也就有了。
- 方案1：用纯文字Toast，不使用setView出来的Toast。对显示效果影响太大，不采用。
- 方案2：改变调用次序。如前所述缺点，不采用。
- 方案3：换用其他活跃的Toast框架。查询Github后，发现现在支持自定义布局的Toast框架大多数是自定义的布局甚至悬浮窗实现（这里的框架代码没有细看，可能有不准确），前者不能跨Activity展示，那我调用的同时finish Activity也是不能正常展示的；后者还要请求悬浮窗权限，颇有杀鸡用牛刀的感觉，因此还是不采用。
- 最后我使用的方案4：将post改成postDelayed，设置一个小延时，这样等到Activity结束后Toast出来（应该不会有Activity在100ms都结束不了的吧，那怕是性能有问题了），避免了被系统判中，改动小，对用户感知的影响不明显。改动后的代码如下：
```kotlin
fun toastInfo(message: String) {
    Handler(Looper.getMainLooper()).postDelayed({
        Toasty.info(MyApplication.instance, message).show()
    }, 100)
}
```

## 后话
- 这是在已有的老项目中，为了最小的改动而提出的解决方案。如果是新的APP，那就干脆别用Toast的setView去搞什么自定义Toast了，毕竟都已经被谷歌Deprecated了，别再去踩坑了。
- 文章仅为个人理解，如果有不准确之处或者更好的办法，欢迎提出。