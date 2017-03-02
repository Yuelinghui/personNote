#实现静态图片高斯模糊方法

##工具类：

```
public class GaussBlurUtil {
    public static Bitmap doBlur(Context context,Bitmap sentBitmap, int radius) {//Let's create an empty bitmap with the same size of the bitmap we want to blur
        Bitmap outBitmap = Bitmap.createBitmap(sentBitmap.getWidth(), sentBitmap.getHeight(), Bitmap.Config.ARGB_8888);

        //Instantiate a new Renderscript
        RenderScript rs = RenderScript.create(context);

        //Create an Intrinsic Blur Script using the Renderscript
        ScriptIntrinsicBlur blurScript = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));

        //Create the Allocations (in/out) with the Renderscript and the in/out bitmaps
        Allocation allIn = Allocation.createFromBitmap(rs, sentBitmap);
        Allocation allOut = Allocation.createFromBitmap(rs, outBitmap);

        //Set the radius of the blur: 0 < radius <= 25
        blurScript.setRadius(radius);

        //Perform the Renderscript
        blurScript.setInput(allIn);
        blurScript.forEach(allOut);

        //Copy the final bitmap created by the out Allocation to the outBitmap
        allOut.copyTo(outBitmap);

        //recycle the original bitmap
        sentBitmap.recycle();

        //After finishing everything, we destroy the Renderscript.
        rs.destroy();

        return outBitmap;

    }
}
```

将图片高斯模糊是一个比较耗时的操作，所以最好放到AsyncTask里使用

```
private class AsycBlurBitmap extends AsyncTask<File, Void, Bitmap> {

        @Override
        protected Bitmap doInBackground(File... params) {
            if (params == null || params.length == 0) {
                return null;
            }
            // 拿到要模糊的图片
            Bitmap bitmap = BitmapFactory.decodeFile(params[0].getAbsolutePath());
            // 返回高斯模糊之后的图片
            return GaussBlurUtil.doBlur(getContext(), bitmap, 20);
        }

        @Override
        protected void onPostExecute(Bitmap bitmap) {
            if (bitmap != null) {
                // ImageView显示模糊之后的图片
                imageView.setImageBitmap(bitmap);
            }
        }
    }
```

这里传入的参数是File，也可以直接传入Bitmap，这个就看当时能够拿到的数据了。

**这里要注意传入的模糊半径必须`0<radius<=25`**