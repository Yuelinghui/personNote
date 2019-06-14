# Glide图片加载框架的简单使用

## Glide的特点

* 使用简单
* 可配置度高，自适应程度高
* 支持常见图片格式：Jpg png gif webp
* 支持多种数据源：网络、本地、资源、Assets 等
* 高效缓存策略：支持Memory和Disk图片缓存 默认Bitmap格式采用RGB_565内存使用至少减少一半
* 生命周期集成：根据Activity/Fragment生命周期自动管理请求
* 高效处理Bitmap：使用Bitmap Pool使Bitmap复用，主动调用recycle回收需要回收的Bitmap，减小系统回收压力

## 加载图片

```
Glide.wiht(context).load(url).placeholder(R.drawable.holder).into(imageView);
```

## 加载Gif

```
// 显示gif静态图片
Glide.with(context).load(url).asBitmap();
// 显示gif动态图片
Glide.with(context).load(url).asGif();
```

## 本地视频快照

```
Glide.with(context).load("视频地址");
```
可以把视频解码为一张图片

## 对缩略图支持

```
Glide.with(context).load(url).thumbnail(0.1f).into(imageView);
```
加载view的十分之一尺寸的缩略图，然后再加载全图

## 生命周期的集成

同时将Activity/Fragment作为with()参数的好处是：图片加载会和Activity/Fragment的生命周期保持一致，请求会在onStop的时候自动暂停，在onStart的时候重新启动,gif的动画也会在onStop的时候停止，以免在后台消耗电量。

## 转码

Glide的.toBytes()和.transcode()方法允许在后台获取、解码和转换一个图片，你可以将一张图片转换成更多有用的图片格式

```
Glide.with(context).load(“/user/profile/photo/path”)
.asBitmap().toBytes().centerCrop()
.into(new SimpleTarget<byte[]>(250,250){
   @Override
   public void onResourceReady(byte[] data, GlideAnimation anim) {
     // 这里可以处理转码结果
   }
});
```

## 动画

支持cross fades和View的属性动画

```
.animate(ViewPropertyAnimation.Animator)
```
## 设置缓存策略

```
.diskCacheStrategy(DiskCacheStrategy.ALL)
```
* all:缓存源资源和转换后的资源
* none:不作任何磁盘缓存
* source:缓存源资源
* result：缓存转换后的资源

## 缓存的动态清理

```
//清理磁盘缓存 需要在子线程中执行
Glide.get(this).clearDiskCache();
//清理内存缓存  可以在UI主线程中进行
Glide.get(this).clearMemory();
```

## 自定义图片处理

当我们下载完图片的时候，可以自定义处理，比如设置成圆的，或者带圆角

```
Glide.with(this).load(imageUrl).transform(new GlideRoundTransform(this)).into(imageView);
```

Glide自带有CropCircleTransformation（处理成圆形），RoundedCornersTransformation（带圆角）等，我们也可以自定义类（继承Transformation）

## 设置加载的尺寸

```
.override(width, height)
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg2MzY1MDU3N119
-->