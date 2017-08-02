# OkHttp和RxJava的网络框架下载和上传文件

## 下载文件

自己使用Retrofit和RxJava写了一个网络框架，但是只能发送网络请求，这次就是完善了一下这个框架，加了一个下载的功能。

一般我们下载文件都是直接通过Url来下载的，所以我们可能用不到Retrofit，直接使用OkHttp就可以了。

### DownloadManager

还是之前的习惯，新写了一个DownloadManager，依然是单例模式

```
public class DownLoadManager {

    private static DownLoadManager mDownLoadManager = null;
    private static OkHttpClient mOkHttpClient = null;

    private DownLoadManager() {
        // 初始化一个OkHttpClient
        mOkHttpClient = new OkHttpClient();
    }

    public synchronized static DownLoadManager getInstance() {
        if (mDownLoadManager == null) {
            mDownLoadManager = new DownLoadManager();
        }
        return mDownLoadManager;
    }

    public void downLoad(final String url, Subscriber<String> rxSubscriber) {
        Observable.create(new Observable.OnSubscribe<File>() {
            @Override
            public void call(final Subscriber<? super File> subscriber) {
                // 创建Request
                Request request = new Request.Builder().url(url).build();
                // 同步调用接口
                mOkHttpClient.newCall(request).enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        // 发生错误就回调onError
                        subscriber.onError(e);
                        // 结束
                        subscriber.onCompleted();
                    }

                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        // 接口返回数据，转换成InputStream
                        InputStream inputStream = response.body().byteStream();
                        byte[] buf = new byte[2048];
                        FileOutputStream fos = null;
                        // 保存的地址
                        String savePath = isExitDir("TEST");
                        int len = 0;
                        File file = new File(savePath,getNameFromUrl(url));
                        try {
                            fos = new FileOutputStream(file);
                            while ((len = inputStream.read(buf)) != -1) {
                                fos.write(buf,0,len);
                            }
                            fos.flush();
                        } catch (Exception e) {
                            file = null;
                        } finally {
                            try {
                                if (inputStream != null) {
                                    inputStream.close();
                                }
                            } catch (Exception e) {

                            }
                            try {
                                if (fos != null) {
                                    fos.close();
                                }
                            } catch (Exception e) {

                            }
                        }
                        // 生成了新的File，成功回调回去
                        subscriber.onNext(file);
                        // 结束
                        subscriber.onCompleted();
                    }
                });
            }
        }).map(new Func1<File, String>() {
            @Override
            public String call(File file) {
                // 转换一下，把File转换成路径
                return file.getAbsolutePath();
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(rxSubscriber);
    }

    // 通过Url获取文件名称
    private String getNameFromUrl(String url) {
        return url.substring(url.lastIndexOf("/") + 1);
    }

    // 保存文件的地址
    private String isExitDir(String saveDir) {
        File downloadFile = new File(Environment.getExternalStorageDirectory(),saveDir);
        if (!downloadFile.mkdirs()) {
            try {
                downloadFile.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
                downloadFile = null;
            }
        }
        if (downloadFile == null) {
            return null;
        }
        return downloadFile.getAbsolutePath();

    }
}

```

### 总结

这只是一个简单的下载工具，具体的情况还需要按照需求修改，不过能实现基本的下载文件。

## 上传文件

当然，只有下载文件不行，还需要上传文件。上传文件就不需要重新写Manager了，直接使用之前的那个网络框架就行，如果没有看过之前的框架，可以先看看这里[简单的网络框架](https://github.com/Yuelinghui/personNote/blob/master/使用Retrofit和RxJava搭建网络框架.md)

一般上传文件的时候我们都是直接调用接口，文件只是Body中的一个参数。Retrofit支持上传文件，所以我们直接用Retrofit框架就行

### 定义接口

public interface UploadFileService {

    @Multipart
    @POST("XXX/upload")
    Observable<Result<FeedbackInfo>> submitFeedback(@Part("content") String content
    , @Part("image_pic\";filename=\"user_feedback.jpg") RequestBody body);
}

要上传文件，肯定是用POST方法，要上传文件就要使用Multipart注解，然后参数使用@Part注解。

*注意：这里文件使用RequestBody类型的参数，前面@Part注解中的内容：分号前面是服务端定的文件的key值，然后后面是上传给服务端之后文件的名称*

### 接口调用

定义一个方法来调用上传文件的接口

```
public void upload(String content, File image, RxSubscriber<UploadInfo> subscriber) {

            UploadFileService uploadFileService = NetManager.getInstance().create(UploadFileService.class);

            // 把文件生成RequestBody，第一个参数是MediaType，第二个参数是File
            // 图片一般是“image/*”，jpg和png都可以
            RequestBody requestBody = RequestBody.create(MediaType.parse("image/jpg"), image);

            // 然后调用发送请求
            RxManager.getInstance().doSubscribe(uploadFileService.upload(content, requestBody), subscriber);
        }
    }
```

如果Retrofit没有按照我之前那么设置的话，可能你这个会报错，*注意：一定要给Retrofit加上.addConverterFactory(ScalarsConverterFactory.create())*，如果不加这个的话，那发送给服务端的第一个参数会有两个括号““content””，这样服务端解析key值可能就会出错。

加上这个就可以避免这种情况，具体原因待我看完源码分享出来。

### 总结

上传文件比下载文件看起来简单，但是写的时候一点不简单，因为我对于Retrofit的注解还不是太熟悉，都是一点点摸索出来的。
