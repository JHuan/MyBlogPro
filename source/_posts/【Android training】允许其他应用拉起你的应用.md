layout: hexo
title: 【Android training】引导到另一个APP
date: 2015-11-5 20:30:22
tags: Android training 翻译

---

原文地址：[Allowing Other Apps to Start Your Activity](http://developer.android.com/training/basics/intents/filters.html)

为了让别的应用能拉起你的应用，你需要在你的AndroidMainfest.xml中对应的`<activity>`组件里添加`<intent-filter>`。
当你的应用安装在设备时，系统会识别你的intent filter然后添加到内置的目录里。当APP使用隐式Intent调用startActivity()或者startActivityForResult（）时，系统会找到哪些应用能接收这个Intent。

## 添加一个Intent Filter

为了合适地定义你的应用中的Activity所能处理的Intent,每个添加的Intent Filter都需要尽可能地具体。
一个标准的Intent包含以下元素：

### Action
将要执行的动作名称，规定在你的`<action>`标签中
### Data
描述所需传递的数据，规定在你的`<data>`标签中。可以用多个属性规定这个元素，比如说MIME类型，uri前缀，uri scheme，或者是这些属性的组合或者重复。
### Category
提供一种附加的方法来标识activity如何处理这个Intent，通常和用户的手势位置从哪儿开始有关。系统支持多种不同的分类，但是大部分很少用的到。**但是，所有隐式Intent默认被定义成CATEGORY_DEFAULT**

举个栗子，这是一个能接收ACTION_SEND动作，文字或者图像作为数据的Activity：
```
<activity android:name="ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
        <data android:mimeType="image/*"/>
    </intent-filter>
</activity>
```
每个Intent只能确定一个Action和一种data类型，但`<intent-filter>`可以含有多个`<action>`, `<category>`, 和 `<data>`。

> 注意：为了能接收隐式Intent，你必须将CATEGORY_DEFAULT这个category加入intent filter中。因为隐式Intent默认分类就是CATEGORY_DEFAULT

## 在你的Activity中处理传递过来的Intent

为了决定你的Activity如何处理不同的Action，你可以从Intent中获取到Action进行处理。
当你的Activity启动时，可以调用getIntent（）获取到启动Activity的Intent。
举个栗子：
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    setContentView(R.layout.main);

    // Get the intent that started this activity
    Intent intent = getIntent();
    Uri data = intent.getData();

    // Figure out what to do based on the intent type
    if (intent.getType().indexOf("image/") != -1) {
        // Handle intents with image data ...
    } else if (intent.getType().equals("text/plain")) {
        // Handle intents with text ...
    }
}
```

## 返回一个结果
如果你想将执行结果返回给拉起你的Actvity，只要简单的调用setResult（），指定返回码和Intent。当操作结束需要返回到原来的Activity时，需要调用finish()结束你的Activity。
举个栗子：
```
// Create intent to deliver some kind of result data
Intent result = new Intent("com.example.RESULT_ACTION", Uri.parse("content://result_uri");
setResult(Activity.RESULT_OK, result);
finish()
```
你必须指定一个返回码。通常都是RESULT_OK 或者 RESULT_CANCELED。你还可以用Intent带上额外的数据。

>注意：返回结果默认返回码是RESULT_CANCELED，因此，如果用户在完成动作且你设置返回结果之前按下返回键，原来的Acitvity会接受到“cancel”的结果




