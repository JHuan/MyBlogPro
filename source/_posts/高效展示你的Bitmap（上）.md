layout: hexo
title: 【Android training】高效展示你的Bitmap（上）
date: 2016-2-3 16:02:33
tags: Android training 翻译
---

参考文献：
[Android Training : Displaying Bitmaps Efficiently](http://developer.android.com/training/displaying-bitmaps/index.html)

前言:Bitmap无疑是内存大户之一，稍不谨慎，虚拟机抛出个OOM（OutofMemoryError）会让你很抓狂。OOM的定位和调试是个头痛的问题，会有专门的章节讲解这一问题，在此之前，要使用正确的体位来把玩我们的bitmap！ o(￣ε￣*)

## 高效加载大图
现实中，咱们的安卓设备可能会接触到各种形状和大小不同的图片。在一些情况中图片大小大于咱们的手机屏幕，而且在大多数情况下我们不需要这么大的图片展示给用户看，那么就很有必要采取必要的措施来避免直接加载这么大的图片。

### 读取图片尺寸和类型
[BitmapFactory](http://developer.android.com/reference/android/graphics/BitmapFactory.html) 类提供了一系列解析bitmap的方法 (decodeByteArray(), decodeFile(), decodeResource(), etc.)。这些方法是非常容易导致OOM的！你可以通过BitmapFactory.Options来改变加载选项，设置inJustDecodeBounds 选项为true 可以有效避免解析bitmap时的内存分配，来达到只读取图片信息而不是为了显示图片，阔以避免不必要的内存分配！

```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

### 加载个适配当前需求大小的图片到内存中去吧

好，通过上一步知道了图片的尺寸，我们开始可以决定加载一个怎样的图片了，是原图就可以？还是要做个压缩后的图片（原文是subsampled version卧槽好专业）？别急，先来考虑下以下因素：

+ 估算下原图加载所需分配的内存大小
+ 应用内适合分多少内存给这张图片
+ 目标view的尺寸
+ 屏幕尺寸和屏幕密度

如果仔细评估过还是加载个压缩后的图片比较好，那可以通过BitmapFactory.Options的inSampleSize可以设置采样率，以下是个栗子：
```
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {

        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }

    return inSampleSize;
}
```

> 注意：采用2的n次幂是因为解码器会用一个向下取整2的n次幂的常量值，[inSampleSize][1]相关文档解释了这一点。




  [1]: http://developer.android.com/reference/android/graphics/BitmapFactory.Options.html#inSampleSize
  
## 在非UI线程里处理bitmap

很显然，在UI线程里处理bitmap可是会被老板骂的╮(╯▽╰)╭，读取图片、图像处理涉及到磁盘访问以及图像处理算法复杂度，这些都是耗时操作，一不小心就引起ANR的哦~so，来看看使用什么体位解决这个问题！

### 用AsyncTask吧！
[AsyncTask](http://developer.android.com/reference/android/os/AsyncTask.html)为开发者提供了处理后台耗时操作很便利的解决方案，以下就是个栗子：
```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    private final WeakReference<ImageView> imageViewReference;
    private int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference<ImageView>(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```

对ImageView的弱引用可以保证AsyncTask不会阻挠ImageView的垃圾回收（不然就有可能内存泄露啦），由于不能保证后台任务完成时ImageView还在，所以在onPostExecute时要检查ImageView还在不。启动这个异步任务灰常简单，只要以下代码：
```
public void loadBitmap(int resId, ImageView imageView) {
    BitmapWorkerTask task = new BitmapWorkerTask(imageView);
    task.execute(resId);
}
```

### 处理同步问题

但是，一些常见的View（譬如ListView）就这么用AsyncTask会有一系列的问题。ListView在滚动时，为了节省内存，会回收掉子View的内存。so，如果你没想太多，ListView的每一个Item直接上AsyncTask加载图片，可能导致A   子View加载图片的时候用户滚动ListView，A   子View被回收了并且其被回收的内存恰好分配给了B子View，同时加载图片异步任务还在继续，正要显示时所持有的内存引用恰好指向了B 子View，那么B子View就会错位显示A子View该显示的图片啦。更糟糕的是异步任务可是没办法保证按顺序显示图片的。

还好有一篇博客[ Multithreading for Performance ](http://android-developers.blogspot.com/2010/07/multithreading-for-performance.html)提出了解决方案，该类继承自Drawbale类，存放一个指向BitmapWorkerTask的弱引用，当异步任务完成时图像就会显示。

```
static class AsyncDrawable extends BitmapDrawable {
    private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap,
            BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference =
            new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return bitmapWorkerTaskReference.get();
    }
}
```

我们可以用这种姿势调用：

```
public void loadBitmap(int resId, ImageView imageView) {
    if (cancelPotentialWork(resId, imageView)) {
        final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        final AsyncDrawable asyncDrawable =
                new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
        imageView.setImageDrawable(asyncDrawable);
        task.execute(resId);
    }
}
```

这个cancelPotentialWork方法主要检查这个imageView是否已经和一个已经在执行的异步任务绑定了，如果是的话，会尝试取消这个任务

```
public static boolean cancelPotentialWork(int data, ImageView imageView) {
    final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

    if (bitmapWorkerTask != null) {
        final int bitmapData = bitmapWorkerTask.data;
        // If bitmapData is not yet set or it differs from the new data
        if (bitmapData == 0 || bitmapData != data) {
            // Cancel previous task
            bitmapWorkerTask.cancel(true);
        } else {
            // The same work is already in progress
            return false;
        }
    }
    // No task associated with the ImageView, or an existing task was cancelled
    return true;
    
   
}

 private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
   if (imageView != null) {
       final Drawable drawable = imageView.getDrawable();
       if (drawable instanceof AsyncDrawable) {
           final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
           return asyncDrawable.getBitmapWorkerTask();
       }
    }
    return null;
}
```

最后一步就是更新BitmapWorkerTask 里的onPostExecute()，加上任务是否取消以及当前任务是否匹配当前ImageView：

```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            final BitmapWorkerTask bitmapWorkerTask =
                    getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask && imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
```

以上所说的解决方案适用于ListView,GridView等会回收子View的组件。只需简单的调用loadBitmap，就可以放心的加载图片啦！

