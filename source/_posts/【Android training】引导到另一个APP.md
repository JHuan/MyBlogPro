layout: hexo
title: 【Android training】引导到另一个APP
date: 2015-10-30 20:12:22
tags: Android training 翻译
---

Android最重要的一个特性就是可以通过action引导到其他APP里去了。比如说你想在你的APP中将一些地址显示在地图上，你无需专门搞个activity显示地图，你可以用带有地址信息的Intent发起请求，Android系统就会拉起一个APP去做到这点。

## 用隐式Intent
隐式Intent无需你指定具体的Class，把指定的action(比如view, edit, send, or get)，再把数据带上就欧啦。

以下是一些例子：
```
//打电话
Uri number = Uri.parse("tel:5551234");
Intent callIntent = new Intent(Intent.ACTION_DIAL, number);
//用地图
Uri location = Uri.parse("geo:0,0?q=1600+Amphitheatre+Parkway,+Mountain+View,+California");
Uri location = Uri.parse("geo:37.422219,-122.08364?z=14"); 
Intent mapIntent = new Intent(Intent.ACTION_VIEW, location);
//浏览网页
Uri webpage = Uri.parse("http://www.android.com");
Intent webIntent = new Intent(Intent.ACTION_VIEW, webpage);
```

## 看看有没应用可以接受俺们的Intent

尽管Android系统保证肯定有个内置应用（比如电话，电邮，日历等）会接受俺们的Intent,但你始终应该在使用这个Intent之前要确认下。

>注意：如果你作死使用了一个根本没有应用能接受的Intent，你的APP会Crash的！(ノಠ益ಠ)ノ彡┻━┻

调用queryIntentActivities()就可以获得能接受你的Intent的应用列表，如果返回的不是空列表你就能安心使用这个Intent啦，\(^o^)/YES!
```
PackageManager packageManager = getPackageManager();
List activities = packageManager.queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
boolean isIntentSafe = activities.size() > 0;
```
如果isIntent是true，那就至少有一个应用能接受你的Intent啦

##用你的Intent拉起一个Activity

当你创建好Intent塞进了数据之后，便可以调用startActivity了。如果系统识别到有多个APP可以处理你的Intent,系统会展示一个对话框供用户选择哪个APP打开，如果只有一个APP能处理你的Intent，那就会立刻被打开。

![pic 1](http://developer.android.com/images/training/basics/intents-choice.png "pic1:供你选择拉起的APP对话框长这样~")

以下是个完整的例子，演示如何创建一个可以拉起地图应用的intent，然后退出应用拉起地图：

```
// Build the intent
Uri location = Uri.parse("geo:0,0?q=1600+Amphitheatre+Parkway,+Mountain+View,+California");
Intent mapIntent = new Intent(Intent.ACTION_VIEW, location);

// Verify it resolves
PackageManager packageManager = getPackageManager();
List<ResolveInfo> activities = packageManager.queryIntentActivities(mapIntent, 0);
boolean isIntentSafe = activities.size() > 0;

// Start an activity if it's safe
if (isIntentSafe) {
    startActivity(mapIntent);
}
```
![pic 2](http://developer.android.com/images/training/basics/intent-chooser.png)

## 显示一个应用选择对话框

可以注意得到当你使用startActivity（）将你的Intent传递到另一个app并且不止一个app可以接收你的Intent时，用户可以通过一个应用选择对话框来决定使用哪个应用，并且能够选择哪个应用作为默认打开方式。如果场景是浏览网页或者拍照那还是个不错的选择。

但是，有些场景可能用户更需要每次都使用不同的app作为打开方式，比如说“分享”。这时你应该强制显示应用选择对话框让用户每次打开时选择使用哪个APP。




```
Intent intent = new Intent(Intent.ACTION_SEND);
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show chooser
Intent chooser = Intent.createChooser(intent, title);

// Verify the intent will resolve to at least one activity
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```




 
 





