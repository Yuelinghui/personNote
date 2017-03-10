#Bitmap所占内存以及内存计算方法

Android中一张图片（BitMap）占用的内存主要和以下几个因数有关：图片长度，图片宽度，单位像素占用的字节数。 一张图片（BitMap）占用的内存=图片长度\*图片宽度\*单位像素占用的字节数 注：图片长度和图片宽度的单位是像素。

创建一个BitMap时，其单位像素占用的字节数由其参数BitmapFactory.Options的inPreferredConfig变量决定。 

##Bitmap.Config类

Bitmap.Config类是个枚举类型，它可以是以下几个值：

* Bitmap.Config.ALPHA_8：此时图片只有alpha值，没有RGB值，
一个像素占用一个字节。**这种格式的图片，看起来质量太差，已经不推荐使用。**

* Bitmap.Config.ARGB_4444：一个像素占用2个字节，alpha(A)值，Red（R）值，Green(G)值，Blue（B）值各占4个bites，共16bites,即2个字节

* Bitmap.Config.ARGB_8888：一个像素占用4个字节，alpha(A)值，Red（R）值，Green(G)值，Blue（B）值各占8个bites，共32bites,即4个字节。
这是一种高质量的图片格式，电脑上普通采用的格式。它也是Android手机上一个BitMap的默认格式。

* Bitmap.Config.RGB_565：一个像素占用2个字节，没有alpha(A)值，即不支持透明和半透明，Red（R）值占5个bites，Green(G)值占6个bites，Blue（B）值占5个bites,共16bites,即2个字节。
对于没有透明和半透明颜色的图片来说，该格式的图片能够达到比较的呈现效果，相对于ARGB_8888来说也能减少一半的内存开销。因此它是一个不错的选择。另外我们通过android.content.res.Resources来取得一个张图片时，它也是以该格式来构建BitMap的。

从Android4.0开始，该选项无效。即使设置为该值，系统任然会采用 ARGB_8888来构造图片
