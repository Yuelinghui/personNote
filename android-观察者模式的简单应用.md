##Android中观察者模式的简单应用

###应用需求分析

在Android开发中经常会遇到输入信息然后点击下一步提交信息，但是有些信息是必填的，如果这个信息不填，下一步的按钮是置灰的，不能点击的。这时候我们就可以使用观察者模式，按钮观察必填信息的输入框，输入框没有输入时，按钮置灰不可点击，当输入框有输入时，按钮可以点击。当然关于信息的正确校验不适合放在这里，比较方便的是点击下一步的时候在`onClick`事件中校验。

###应用

####接口定义

先定义一个可校验的接口

```
public interface Verifiable {
 // 校验
 boolean verify();
 // 是否为空白
 boolean isBlank();
 // 添加观察者
 void addObserver(Observer obj);
}
```

Android中提供了观察者模式的接口Observer。我们使用的Button实现这个接口

```
public class CustomButton extends Button implements Observer {
    // 需要观察的对象
    private LinkedHashSet<Verifiable> mVerifiers = new LinkedHashSet<Verifiable>();

    public CustomButton(Context context) {
        this(context, null);
    }

    public CustomButton(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CustomButton(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
    }

    // 添加观察者
    public void observer(Verifiable verifier) {
        if (this.isEnabled()) {
            this.setEnabled(false);
        }

        // 校验：如果当前verifier属于view 但是 不可显示则不做监听
        if ((verifier instanceof View)
        && ((View) verifier).getVisibility() != View.VISIBLE) {
            update(null, null);
            return;
        }

        if (verifier != null && !mVerifiers.contains(verifier)) {
            mVerifiers.add(verifier);
            verifier.addObserver(this);
        }
        update(null, null);
    }

    // 解除观察者
    public void removeObserver(Verifiable verifier) {
        if (verifier != null) {
            mVerifiers.remove(verifier);
            if (ListUtil.isEmpty(mVerifiers)) {
                mAutoPerformClick = false;
            }
            this.update(null, null);
        }
    }

    // 清除观察者
    public void clearObserver() {
        if (!ListUtil.isEmpty(mVerifiers)) {
            mVerifiers.clear();
            mAutoPerformClick = false;
            this.update(null, null);
        }
    }



    @Override
    public void update(Observable observable, Object data) {
        if (mAutoPerformClick) {
            if (!ListUtil.isEmpty(mVerifiers)) {
                if (isVerify()) {
                    this.postDelayed(new Runnable() {

                        @Override
                        public void run() {
                            performClick();
                        }
                    }, PERFORM_DELAY_TIME);
                }
            }
        } else {
            for (Verifiable verifier : mVerifiers) {
                if (verifier.isBlank()) {
                    CustomButton.this.setEnabled(false);
                    return;
                }
            }
            CustomButton.this.setEnabled(true);
        }
    }

    // 判断是否已经通过校验
    private boolean isVerify() {
        for (Verifiable verifier : mVerifiers) {
            if (!verifier.verify()) {
                return false;
            }
        }
        return true;
    }



 /**

 * 设置自动执行下一步按钮

 *

 * @param auto

 */

 public void setAutoPerformClick(boolean autoPerformClick) {

 mAutoPerformClick = autoPerformClick;

 }



 /**

 * 获得是否自动执行下一步

 *

 * @return mAutoPerformClick

 */

 public boolean isAutoPerformClick() {

 return mAutoPerformClick;

 }



 public int getVerifiersSize() {

 if (ListUtil.isEmpty(mVerifiers)) {

 return 0;

 }

 return mVerifiers.size();

 }

}

```