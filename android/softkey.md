# 软键盘处理

## 收起与弹出

> 这部分处理与对应的 EditText 绑定，在键盘弹起时对应的 EditText 会自动获取焦点，收起时会失去焦点

```java
// 关闭键盘
fun closeKeyboard(et: EditText, activity: Activity) {
    val imm = activity.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.hideSoftInputFromWindow(et.windowToken, 0)
    et.clearFocus()
    if (activity.currentFocus != null) {
        imm.hideSoftInputFromWindow(activity.currentFocus!!.windowToken, 0)
    }
}
// 打开键盘
fun openKeyboard(et: EditText, ctx: Context) {
    et.isFocusable = true
    et.isFocusableInTouchMode = true
    et.requestFocus()
    val imm = ctx.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.toggleSoftInput(InputMethodManager.SHOW_FORCED,
            InputMethodManager.HIDE_IMPLICIT_ONLY)
}
```

## 高度变化监听

> 主要思路：**使用一个 PopupWindow 覆盖在当前界面上，它会在键盘弹出收起时高度自动发生变化。同时添加 OnGlobalLayoutListener 回调，当高度发生变化时会收到回调**

这里面有一点可以优化的地方：

1. 可以自动跟踪 Activity 的生命周期，当其关闭时自动调用 close\(\)

```java
public class KeyboardHeightProvider extends PopupWindow {
    // 回调
    private Function1<Integer, Unit> observer;

    private int keyboardLandscapeHeight;

    private int keyboardPortraitHeight;

    // PopupWindow 中显示的 View
    private View popupView;

    // 界面中的 content View
    private View parentView;

    private Activity activity;

    public static boolean isOpen;
    private final OnGlobalLayoutListener listener = new OnGlobalLayoutListener() {
        @Override
        public void onGlobalLayout() {
            if (popupView != null) {
                handleOnGlobalLayout();
            }
        }
    };

    public KeyboardHeightProvider(Activity activity) {
        super(activity);
        if (activity == null) {
            return;
        }
        this.activity = activity;

        LayoutInflater inflater = (LayoutInflater) activity.getSystemService(Activity.LAYOUT_INFLATER_SERVICE);
        // 添加一个长宽都 match_parent 的 View，给 PopupWindow 用
        this.popupView = new Space(activity);
        this.popupView.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
        setContentView(popupView);

        setSoftInputMode(LayoutParams.SOFT_INPUT_ADJUST_RESIZE | LayoutParams.SOFT_INPUT_STATE_ALWAYS_VISIBLE);
        setInputMethodMode(PopupWindow.INPUT_METHOD_NEEDED);

        parentView = activity.findViewById(android.R.id.content);

        setWidth(0);
        setHeight(LayoutParams.MATCH_PARENT);

        popupView.getViewTreeObserver().addOnGlobalLayoutListener(listener);
    }


    // 开始显示
    public void start() {
        if (parentView == null) {
            return;
        }
        if (activity == null || activity.isFinishing() || activity.isDestroyed()) {
            return;
        }
        try {
            if (!isShowing() && parentView.getWindowToken() != null) {
                setBackgroundDrawable(new ColorDrawable(0));
                showAtLocation(parentView, Gravity.NO_GRAVITY, 0, 0);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // Activity 退出时关闭 PopupWindow
    public void close() {
        this.observer = null;
        dismiss();
        popupView.getViewTreeObserver().removeOnGlobalLayoutListener(listener);
    }

    public KeyboardHeightProvider setKeyboardHeightObserver(Function1<Integer, Unit> observer) {
        this.observer = observer;
        return this;
    }

    private void handleOnGlobalLayout() {
        if (activity == null || parentView == null || popupView == null) {
            return;
        }
        Point screenSize = new Point();
        activity.getWindowManager().getDefaultDisplay().getSize(screenSize);

        Rect rect = new Rect();
        popupView.getWindowVisibleDisplayFrame(rect);

        int orientation = getScreenOrientation();

        int keyboardHeight = parentView.getHeight() - (rect.bottom - rect.top);
        isOpen = keyboardHeight > 0;
        if (keyboardHeight == 0) {
            notifyKeyboardHeightChanged(0, orientation);
        } else if (orientation == Configuration.ORIENTATION_PORTRAIT) {
            this.keyboardPortraitHeight = keyboardHeight;
            notifyKeyboardHeightChanged(keyboardPortraitHeight, orientation);
        } else {
            this.keyboardLandscapeHeight = keyboardHeight;
            notifyKeyboardHeightChanged(keyboardLandscapeHeight, orientation);
        }
    }

    private int getScreenOrientation() {
        return activity.getResources().getConfiguration().orientation;
    }

    private void notifyKeyboardHeightChanged(int height, int orientation) {
        if (observer != null) {
            observer.invoke(height);
        }
    }
}
```

