# ViewDragHelper简单使用

## 简介

ViewDragHelper是在v4包里提供的类，专门用作控制ViewGroup内子View的滑动的。DrawerLayout内部就是使用它来实现页面的滑动的。

之前要在自定义View中实现子View的滑动需要复写onTouchEvent和onInterceptTouchEvent等方法，在onTouchEvent内计算拖动的距离，然后让子View运动，这些的写法非常麻烦。

ViewDragHelper封装了这些方法，使用起来非常方便。

## 使用

### 初始化

ViewDragHelper使用工厂方法初始化

```
 ViewDragHelper.create(ViewGroup forParent, float sensitivity, Callback cb)
```

这里的三个参数分别表示：
1. 想要让内部子View滑动的ViewGroup。这里需要传入ViewGroup，因为ViewDragHelper是提供给ViewGroup来控制子View的滑动的。
2. sensitivity，是控制敏感度的参数。这个数值越大相应滑动的敏感度越高。默认的敏感度是8dp。（`private static final int TOUCH_SLOP = 8;`）

```
helper.mTouchSlop = (int) (helper.mTouchSlop * (1 / sensitivity));
```
3. Callback，是我们需要实现的接口，来响应子View的滑动。

### ViewDragHelper.Callback

如果我们要实现子View的滑动，还需要实现Callback的方法，这里主要讲几个重要的方法：

1. tryCaptureView，用来控制哪个子View可以滑动

    ```
    @Override
    public boolean tryCaptureView(View child, int pointerId) {
        return mDragView == child;
    }
    ```
child就是我们现在拖动的子View，只有这个方法返回true时才能让子View滑动。当一个ViewGroup里有两个子View，一个可以拖动，一个不能拖动时，就根据这个方法来判断哪个可以拖动。

2. clampViewPositionHorizontal，clampViewPositionVertical，这两个方法就是来计算子View横向滑动和纵向滑动的距离的。默认返回的都是0，这时子View是滑动不了的，当返回一个大于0的值时，子View才能跟随手指滑动

    ```
    @Override
    public int clampViewPositionVertical(View child, int top, int dy) {
        return top;
    }

    @Override
    public int clampViewPositionHorizontal(View child, int left, int dx) {
        return left;
    }
    ```

3. onViewReleased，当手指松开子View时触发，可以通过这个方法来设置当手指松开时子View返回原位置。

    ```
     @Override
    public void onViewReleased(View releasedChild, float xvel, float yvel) {
        // 控制子View返回之前的位置
        ...
    }
    ```
4. getViewHorizontalDragRange，getViewVerticalDragRange，这两个方法通过字面的意思是控制子View的滑动距离的，其实就是定义滑动多少距离之后才算真正的滑动。如果我们没有实现这两个方法，默认返回的是0，这时，一个clickable的子View（比如Button）就没法响应onClick。**只有这两个方法返回大于0的时候这个子View才能响应onClick**。

    ```
    @Override
    public int getViewHorizontalDragRange(View child) {
        return mDragHelper.getTouchSlop();
    }
    @Override
    public int getViewVerticalDragRange(View child) {
        return mDragHelper.getTouchSlop();
    }
    ```
5. 还有其他一些方法，我们可以根据需要来添加
    ```
    public void onViewDragStateChanged(int state) {}
    
    public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {}
    
    public void onEdgeTouched(int edgeFlags, int pointerId) {}
    
    ...
    ```

### 使用步骤

1. 自定义ViewGroup（比如继承FrameLayout）
2. 初始化ViewDragHelper
3. 根据需要实现Callback里的方法
4. 覆写onInterceptTouchEvent和onTouchEvent，把event交给ViewDragHelper来实现

    ```  
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
            mDragHelper.cancel();
            return false;
        }
        return mDragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mDragHelper.processTouchEvent(event);
        return true;
    }
    ```
5. 覆写onLayout，当子View移动的时候会触发ViewGroup的onLayout，这时我们通过layout方法来改变子View的位置。如果我们没有实现这个方法，我们会发现当我们移动子View一段距离，手指停止的时候，子View还会返回。

    ```
    // 这里的mLeft是在clampViewPositionHorizontal记录下来的
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        mDragView.layout(mLeft, top, mLeft + mDragView.getMeasuredWidth(), bottom);
    }
    ```
OK，自定义的ViewGroup内部的子View可以跟随我们的手指移动了！这比我们之前自己写onToucheEvent方法简单方便了很多！
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzI2NDc4NzY1XX0=
-->