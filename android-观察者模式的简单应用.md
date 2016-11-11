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

Android系统提供给了我们观察者模式中观察者的接口，其实这是Java的。

```
public abstract interface java.util.Observer {
    public abstract void update(Observable arg0, Object arg1);
}

```

####Verifiable接口实现

我们需要定义我们的输入框，让它来实现Verifiable这个接口。

```
public class CustomEdit extends EditText implements Verifiable {
    // 观察它的对象
    private Observer mVerifyObserver = null;
    public CustomEdit(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        initView();
    }
    public CustomEdit(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }
    public CustomEdit(Context context) {
        this(context, null);
    }

    // 初始化控件，给控件设置一个TextChangedListener
    private void initView() {
        addTextChangedListener(mTextWatcher);
    }
    ...
}
```

我们在初始化EditText的时候给它设置了一个TextWatcher，这样当输入框内的文字变动时我们可以监听到。

```
// TextWatcher
private TextWatcher mTextWatcher = new TextWatcher() {

    @Override
    public void afterTextChanged(Editable s) {
        if (mVerifyObserver != null) {
            mVerifyObserver.update(null, null);
        }
    }
    ...
};
```

在TextWatcher监听中，当文字发生完变化的时候我们调用观察者的update()方法。

然后我们看EditText是怎么实现Verify接口的

```
...

@Override
public void addObserver(Observer obj) {
    mVerifyObserver = obj;
}

@Override
public boolean verify() {
    if (!TextUtils.isEmpty(getText())) {
        return true;
    }
    return false;
}

@Override
public boolean isBlank() {
    return TextUtils.isEmpty(getText());
}

...
```

当EditText添加观察者时(addObserver(Observer))，我们记住这个观察者对象。判断是否完成校验(verify())，当有输入文字的时候为true，没有输入文字的时候为false。

####Observer接口实现

我们使用的Button实现Observer接口

```
public class CustomButton extends Button implements Observer {
    // 需要观察的对象
    private LinkedHashSet<Verifiable> mVerifiers = new LinkedHashSet<Verifiable>();

    ...

    // 添加被观察对象
    public void observer(Verifiable verifier) {
        // 当有新的被观察对象添加的时候，按钮应该不可用
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

    // 解除被观察对象
    public void removeObserver(Verifiable verifier) {
        if (verifier != null) {
            mVerifiers.remove(verifier);
            update(null, null);
        }
    }

    // 清除被观察对象
    public void clearObserver() {
        if (!ListUtil.isEmpty(mVerifiers)) {
            mVerifiers.clear();
            update(null, null);
        }
    }

    @Override
    public void update(Observable observable, Object data) {
       for (Verifiable verifier : mVerifiers) {
            if (verifier.isBlank()) {
                CustomButton.this.setEnabled(false);
                return;
               }
            CustomButton.this.setEnabled(true);
        }
    }
    ...
}
```

点击的Button已经定义好了，它就是一个观察者了，使用的时候可以指定Button观察的对象（CustomButton.observer(Verify verify)）。

####使用方法

```
// 从布局文件里拿到EditTextview和Button
CustomEditText editText = (CustomEditText)findViewById(R.id.edit);

CustomButton button = (CustomButton)findViewById(R.id.btn);

// 把Button和EditText绑定
button.observer(editText);
```

这样之后，当输入框内没有文字的时候，按钮是不可点击的，我们还可以通过selector来设置Button不可用时的颜色。当输入框内输入文字，按钮就可以变得可以点击。