> 编写:[kesenhoo](https://github.com/kesenhoo)

> 校对:

# Loading Large Bitmaps Efficiently(有效地加载大尺寸位图)
图片有不同的形状与大小。在大多数情况下它们的实际大小都比需要呈现出来的要大很多。例如，系统的Gallery程序会显示那些你使用设备camera拍摄的图片，但是那些图片的分辨率通常都比你的设备屏幕分辨率要高很多。

考虑到程序是在有限的内存下工作，理想情况是你只需要在内存中加载一个低分辨率的版本即可。这个低分辨率的版本应该是与你的UI大小所匹配的，这样才便于显示。一个高分辨率的图片不会提供任何可见的好处，却会占用宝贵的(precious)的内存资源，并且会在快速滑动图片时导致(incurs)附加的效率问。

这一课会介绍如何通过加载一个低版本的图片到内存中去decoding大的bitmaps，从而避免超出程序的内存限制。

## Read Bitmap Dimensions and Type(读取位图的尺寸与类型)
BitmapFactory 类提供了一些decode的方法 ([decodeByteArray()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeByteArray(byte[], int, int, android.graphics.BitmapFactory.Options), [decodeFile()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeFile(java.lang.String, android.graphics.BitmapFactory.Options), [decodeResource()](http://developer.android.com/reference/android/graphics/BitmapFactory.html#decodeResource(android.content.res.Resources, int, android.graphics.BitmapFactory.Options), etc.) 用来从不同的资源中创建一个Bitmap. 根据你的图片数据源来选择合适的decode方法. 那些方法在构造位图的时候会尝试分配内存，因此会容易导致OutOfMemory的异常。每一种decode方法都提供了通过[BitmapFactory.Options](http://developer.android.com/reference/android/graphics/BitmapFactory.Options.html) 来设置一些附加的标记来指定decode的选项。设置 [inJustDecodeBounds](http://developer.android.com/reference/android/graphics/BitmapFactory.Options.html#inJustDecodeBounds) 属性为true可以在decoding的时候避免内存的分配，它会返回一个null的bitmap，但是 outWidth, outHeight 与 outMimeType 还是可以获取。这个技术可以允许你在构造bitmap之前优先读图片的尺寸与类型。

```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

为了避免java.lang.OutOfMemory 的异常，我们在真正decode图片之前检查它的尺寸，除非你确定这个数据源提供了准确无误的图片且不会导致占用过多的内存。

## Load a Scaled Down Version into Memory(加载一个按比例缩小的版本到内存中)
通过上面的步骤我们已经知道了图片的尺寸，那些数据可以用来决定是应该加载整个图片到内存中还是加一个缩小的版本。下面有一些因素需要考虑：

* 评估加载完整图片所需要耗费的内存。
* 程序在加载这张图片时会涉及到其他内存需求。
* 呈现这张图片的组件的尺寸大小。
* 屏幕大小与当前设备的屏幕密度。

例如，如果把一个原图是`1024*768` pixel的图片显示到ImageView为`128*96` pixel的缩略图就没有必要把整张图片都加载到内存中。

为了告诉decoder去加载一个低版本的图片到内存，需要在你的BitmapFactory.Options 中设置 inSampleSize 为 true 。For example, 一个分辨率为2048x1536 的图片，如果设置 inSampleSize 为4，那么会产出一个大概为512x384的bitmap。加载这张小的图片仅仅使用大概0.75MB，如果是加载全图那么大概要花费12MB(前提都是bitmap的配置是 ARGB_8888). 下面有一段根据目标图片大小来计算Sample图片大小的Sample Code:
```java
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {
        if (width > height) {
            inSampleSize = Math.round((float)height / (float)reqHeight);
        } else {
            inSampleSize = Math.round((float)width / (float)reqWidth);
        }
    }
    return inSampleSize;
}
```
**Note:** 设置 inSampleSize 为2的幂对于decoder会更加的有效率，然而，如果你打算把调整过大小的图片Cache到磁盘上，设置为更加接近的合适大小则能够更加有效的节省缓存的空间.

为了使用这个方法，首先需要设置 inJustDecodeBounds 为 true, 把options的值传递过来，然后使用 inSampleSize 的值并设置 inJustDecodeBounds 为 false 来重新Decode一遍。

```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {

    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```

使用上面这个方法可以简单的加载一个任意大小的图片并显示为100*100 pixel的缩略图形式。像下面演示的一样:

```java
mImageView.setImageBitmap(
    decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```

你可以通过替换合适的BitmapFactory.decode* 方法来写一个类似的方法从其他的数据源进行decode bitmap。
