# 使用Retrofit和RxJava搭建网络框架

## Android Studio导入框架

从maven导入需要依赖的包
```
com.squareup.okhttp3:okhttp:3.7.0
com.google.code.gson:gson:2.8.0
com.squareup.retrofit2:retrofit:2.3.0
com.squareup.retrofit2:converter-gson:2.3.0
com.squareup.retrofit2:adapter-rxjava:2.3.0
com.squareup.retrofit2:converter-scalars:2.3.0
io.reactivex:rxandroid:1.2.1
io.reactivex:rxjava:1.2.1
```

## 定义NetManager

NetManager使用单例模式，这个主要是为了定义接口

```
public class NetManager {
    private static NetManager mNetManager = null;
    private static Retrofit mRetrofit = null;
    private static OkHttpClient mOkHttpClient;

    private NetManager() {
        // 定义OkHttpClient
        mOkHttpClient = new OkHttpClient.Builder()
                // 这是为了上传数据时在Header添加cookie，后面会讲
                .addInterceptor(new AddCookiesInterceptor())
                // 这是为了服务端下发数据时获取cookie，后面讲
                .addInterceptor(new ReceivedCookiesInterceptor())
                .build();

        mRetrofit = new Retrofit.Builder()
                // 这是保证图文一起上传
                .addConverterFactory(ScalarsConverterFactory.create())
                // 返回参数使用Gson解析
                .addConverterFactory(GsonConverterFactory.create())
                // 使用RxJava作为适配
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                // 添加定义好的OkHttpClient
                .client(mOkHttpClient)
                // 这是调用接口的Url
                .baseUrl(UrlApi.getCurrentUrl())
                .build();
    }

    public synchronized static NetManager getInstance() {
        if (mNetManager == null) {
            mNetManager = new NetManager();
        }
        return mNetManager;
    }

    public <T> T create(Class<T> service) {
        return mRetrofit.create(service);
    }
}
```

*注意：这个Retrofit对于动态URL的支持不是很好，因为Url都是在初始化时定义好的，这个后期会继续研究*

## RxManager定义

RxManager也是使用单例模式，这个主要是为了链接网络并解析数据

```
public class RxManager {

    private static RxManager mRxManager = null;

    private RxManager() {

    }

    public synchronized static RxManager getInstance() {
        if (mRxManager == null) {
            mRxManager = new RxManager();
        }
        return mRxManager;
    }

    /**
    * 发送数据
    **/
    public <T> Subscription doSubscribe(final Observable<Result<T>> observable, final Subscriber<T> subscriber) {
        // 网络连接成功的Action
        Action1<Result<T>> onSuccessAction = new Action1<Result<T>>() {
            @Override
            public void call(Result<T> result) {
                if (result == null || result.getResultMeta() == null) {
                    subscriber.onError(new Throwable("没有数据返回"));
                }
                if (result.getResultMeta().getResultStatus() != 200) {
                    subscriber.onError(new Throwable(result.getResultMeta().getResultMessage()));
                }
                if (result.getResultMeta().getResultStatus() == 200) {
                    subscriber.onNext(result.getResultData());
                }

                subscriber.onCompleted();
            }
        };

        // 网络链接错误的Action
        Action1<Throwable> onErrorAction = new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                subscriber.onError(throwable);
            }
        };

        return observable.subscribeOn(Schedulers.io()) // 网络链接在IO线程
                .observeOn(AndroidSchedulers.mainThread()) // 回调在主线程
                .subscribe(onSuccessAction,onErrorAction); // 成功和失败转换的Action
    }

```

因为接口返回数据的格式和服务端已经定好了，所以Result就按照之前定好的写

```
public class Result<T> implements Serializable{

    @SerializedName("meta") // 这个注解的具体作用大家可以自行百度
    private Meta resultMeta;

    @SerializedName("response")
    private T resultData;

    public Meta getResultMeta() {
        return resultMeta;
    }

    public void setResultMeta(Meta resultMeta) {
        this.resultMeta = resultMeta;
    }

    public T getResultData() {
        return resultData;
    }

    public void setResultData(T resultData) {
        this.resultData = resultData;
    }

    @Override
    public String toString() {
        return "Result{" +
                "resultMeta=" + resultMeta +
                ", resultData=" + resultData +
                '}';
    }
}
```
resultMeta其实大家可以忽略的，里面就是一个status，表示网络链接状态和一个message，表示返回的信息，真正返回的对于我们有用的结构体是resultData，这里使用泛型。

## 使用Retrofit注解定义接口

Retrofit定义接口一般都是用interface来定义，作为举例，拿首页的接口

```
public interface HomeService {

    @GET("XXX/homes/index/{page}")
    Observable<Result<HomeInfo>> homeList(@Path("page") int page);
}
```
这表明这是一个GET方法，@Path注解表示这个参数会放到url的路径里替换page。这个接口返回的实体是HomeInfo，这个就不写了。

## 发送请求

接口定义好了，就该发送请求并接收数据了。

```
public void queryList(int page, RxSubscriber<HomeInfo> subscriber) {

        HomeService homeService = NetManager.getInstance().create(HomeService.class);

        RxManager.getInstance().doSubscribe(homeService.homeList(page), subscriber);

    }
```

首先我们先通过NetManager生成homeService，然后调用RXManager的doSubscribe方法调用请求。这里有一个RxSubscribe，这是*自定义的流程回调处理器*

```
public abstract class RxSubscriber<T> extends Subscriber<T> {
    private static final String TAG = "RxSubscriber";

    public RxSubscriber() {
    }

    @Override
    public void onCompleted() {
       onFinish();
    }

    @Override
    public void onStart() {
        super.onStart();
        onNetStart();
    }

    @Override
    public void onError(Throwable e) {
        onFail(e.toString());
    }

    @Override
    public void onNext(T t) {
        onSuccess(t);
    }

    protected abstract void onNetStart();

    protected abstract void onSuccess(T t);

    protected abstract void onFail(String msg);

    protected abstract void onFinish();

}
```

*注意：因为Subscriber有onStart()方法，所以只能再定义一个onNetStart()方法来标识开始，或者可以不写，但是要是用的话，每次都需要手动去写*

在Activity里调用queryList方法

```
  queryList(0, new RxSubscriber<HomeInfo>() {

                    @Override
                    public void onNetStart() {
                        mSendTxt.setText("获取接口");
                    }

                    @Override
                    protected void onSuccess(HomeInfo result) {
                        if (result == null) {
                            return;
                        }
                        mResultTxt.setText(result.toString());
                    }

                    @Override
                    protected void onFail(String msg) {
                        mResultTxt.setText(msg);
                    }

                    @Override
                    protected void onFinish() {
                        mSendTxt.setText("接口完成");
                    }
                });
```

这样一个简单的网络框架就完成了。

## Header中Cookie的处理

服务端通过Header中的Cookie来判断具体是哪个用户，所以我们自定义了两个Interceptor来处理Cookie。

### 接收Cookie

刚开始用户没有登录，所以服务端没有传给我们Cookie，当我们登录之后，服务端在Header中放入Cookie，我们需要获取到并且保存，下次请求时带着这个Cookie。

```
public class ReceivedCookiesInterceptor implements Interceptor {

    private static final String KEY_COOKIE = "Set-Cookie";

    @Override
    public Response intercept(Chain chain) throws IOException {
        // 拿到服务端返回的Response
        Response originalResponse = chain.proceed(chain.request());
        // Header中有Cookie
        if (!TextUtils.isEmpty(originalResponse.header(KEY_COOKIE))) {
            HashSet<String> headerSet = new HashSet<>();
            // 把Header中的Cookie保存下来，因为Header中的Cookie可能有多个，所以都保存
            for (String header : originalResponse.headers(KEY_COOKIE)) {
                headerSet.add(header);
            }
            CookiesUtil.saveCookies(headerSet);
        }
        // 这个还是要继续传结果
        return originalResponse;
    }
}
```
CookieUtil就是一个工具类，把Cookie持久化保存在本地

### 发送带着Cookie

```
public class AddCookiesInterceptor implements Interceptor {

    private static final String KEY_COOKIES = "Cookie";

    @Override
    public Response intercept(Chain chain) throws IOException {

        Request.Builder builder = chain.request().newBuilder();
        if (CookiesUtil.getCookies() != null) {
            for (String header : CookiesUtil.getCookies()) {
                builder.addHeader(KEY_COOKIES, header);
            }
        }
        String url = chain.request().url().toString();
        return chain.proceed(builder.build());
    }
}
```

发送请求时到这里我们把Cookie添加到Header中

## 总结

Retrofit是最近比较火的网络框架，其实就是在OkHttp外面又包了一层，方便我们写网络请求，主要是注意那些注解的使用，后面我会继续研究，然后解析一下源码。
RxJava之前已经写过了，使用它是为了方便子线程调用网络，返回结果返回主线程。就不用自己写Handler和线程池了。