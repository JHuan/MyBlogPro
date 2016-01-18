layout: hexo
title: 【Android training】文件共享
date: 2015-10-30 20:12:22
tags: Android training 翻译
---
原文地址：[Sharing Files](http://developer.android.com/training/secure-file-sharing/index.html).


为了能安全的将一个文件提供给另一个APP，你需要配置好你的APP，给文件提供一个安全的句柄。Android 组件 [FileProvider](http://developer.android.com/reference/android/support/v4/content/FileProvider.html) 基于XML配置，为文件生成内容URI。

>  Tips: [FileProvider](http://developer.android.com/reference/android/support/v4/content/FileProvider.html) 已经作为 [v4 Support Library](http://developer.android.com/tools/support-library/features.html#v4)的一部分了哟~

## 声明一个FileProvider

定义一个FileProvider需要在你的mainfest中添加一个入口。这个入口声明了生成内容URI的authorities以及XML文件的名字。这个XML文件确定了你的APP能分享的目录。
以下是个栗子：
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp">
    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>
</manifest>
```

在这个栗子中，[android:authorities](http://developer.android.com/guide/topics/manifest/provider-element.html#auth)属性规定了URL所属的authorities,用于FileProvider生成content uri。meta-data 子元素指向了你想分享出去的XML文件，android:resource指定这个文件的路径和名称（并不包括.xml后缀名哟）

## 指定要分享的目录
当你把FileProvider加入到你的app的Mainfest中后，你需要指定你所要分享的文件所属目录。需要在你的项目目录下 res/xml/创建filepaths.xml 。在这个文件中通过增加XML元素表示你的目录。以下又是一颗栗子：
```
<paths>
    <files-path path="images/" name="myimages" />
</paths>
```
在这个栗子中，files-path 标签共享了你的应用内部存储里的files/. path属性表明分享files/目录下的子目录images/。name属性让FileFileProvider在生成的content URI加上由name指定的路径分隔符。详情可以看看[FileProvider](http://developer.android.com/reference/android/support/v4/content/FileProvider.html) 为何可以这么做。
`<paths>`元素可以有多个子元素，确定多个需要共享的文件夹。除了`<files-path>`标签之外，还可以用`<external-path>`元素共享外部存储文件夹，并且还可以用`<cache-path>`分享内部存储的缓存文件夹。

> 注意： 只能用XML文件的形式来制定你要分享到文件夹，你不能通过java代码的形式动态加一个文件夹

现在呢，你就有一个完整的FileProvider，它能帮你生成你应用内部存储中的files/目录及其子目录下文件的content URI。
再举个栗子，如果你按照上面的代码片段定义了一个FileProvider，然后你请求一个default_image.jpg的URI，FileProvider就会返回如下URI：
`content://com.example.myapp.fileprovider/myimages/default_image.jpg`

## 接收文件分享请求
为了能接收请求方APP的文件请求并响应对应的文件content uri，你应该提供一个文件选择Activity。这样的话请求方APP可以通过startActivityForResult启动这个Activity，传递一个带ACTION_PICK的action的Intent，该Activity接收并处理完毕要嗝屁时，就可以通过setResult反馈uri。

## 创建一个文件选择的Activity

直接上代码说明：
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    ...
        <application>
        ...
            <activity
                android:name=".FileSelectActivity"
                android:label="@"File Selector" >
                <intent-filter>
                    <action
                        android:name="android.intent.action.PICK"/>
                        //指定OPENABLE和DEFAULT两种类型
                    <category
                        android:name="android.intent.category.DEFAULT"/>
                    <category
                        android:name="android.intent.category.OPENABLE"/>
                        //通过mimeType制定所需数据类型
                    <data android:mimeType="text/plain"/>
                    <data android:mimeType="image/*"/>
                </intent-filter>
            </activity>
```

## 访问分享的文件

接收方APP会用Intent向请求方发送共享文件的uri，这个Intent将会被传递到请求方的onActivityResult()里。当请求方拿到了文件的cotent uri, 可以通过[FileDescriptor](http://developer.android.com/reference/java/io/FileDescriptor.html)访问共享文件。
在这个过程中，文件访问安全性是得到保证滴。由于content uri不包含目录路径，请求方不能获取到接收方其他的文件。
以下是个栗子~
```
  /*
     * When the Activity of the app that hosts files sets a result and calls
     * finish(), this method is invoked. The returned Intent contains the
     * content URI of a selected file. The result code indicates if the
     * selection worked or not.
     */
    @Override
    public void onActivityResult(int requestCode, int resultCode,
            Intent returnIntent) {
        // If the selection didn't work
        if (resultCode != RESULT_OK) {
            // Exit without doing anything else
            return;
        } else {
            // Get the file's content URI from the incoming Intent
            Uri returnUri = returnIntent.getData();
            /*
             * Try to open the file for "read" access using the
             * returned URI. If the file isn't found, write to the
             * error log and return.
             */
            try {
                /*
                 * Get the content resolver instance for this context, and use it
                 * to get a ParcelFileDescriptor for the file.
                 */
                mInputPFD = getContentResolver().openFileDescriptor(returnUri, "r");
            } catch (FileNotFoundException e) {
                e.printStackTrace();
                Log.e("MainActivity", "File not found.");
                return;
            }
            // Get a regular file descriptor for the file
            FileDescriptor fd = mInputPFD.getFileDescriptor();
            ...
        }
    }
```

## 获取共享文件的MIME格式
一个文件的数据格式可以表示请求方APP该如何处理该文件内容。我们可以通过调用ContentResolver.getType()来获取共享文件的数据类型，这个方法反馈该文件的MIME类型。**FileProvider默认用文件的后缀名来判断文件所属数据类型**。
以下是个栗子：
```
    /*
     * Get the file's content URI from the incoming Intent, then
     * get the file's MIME type
     */
    Uri returnUri = returnIntent.getData();
    String mimeType = getContentResolver().getType(returnUri);
```

## 获取共享文件的名字和大小
FileProvider 有着query() 接口的默认实现，返回一个带着文件大小和名字的Curosr。默认返回两列：

- DISPLAY_NAME
    文件名，返回值与File.getName()相同
- SIZE
 文件大小，返回值和File.length()相同

以下就是个获取的栗子：

```
 /*
     * Get the file's content URI from the incoming Intent,
     * then query the server app to get the file's display name
     * and size.
     */
    Uri returnUri = returnIntent.getData();
    Cursor returnCursor =
            getContentResolver().query(returnUri, null, null, null, null);
    /*
     * Get the column indexes of the data in the Cursor,
     * move to the first row in the Cursor, get the data,
     * and display it.
     */
    int nameIndex = returnCursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
    int sizeIndex = returnCursor.getColumnIndex(OpenableColumns.SIZE);
    returnCursor.moveToFirst();
    TextView nameView = (TextView) findViewById(R.id.filename_text);
    TextView sizeView = (TextView) findViewById(R.id.filesize_text);
    nameView.setText(returnCursor.getString(nameIndex));
    sizeView.setText(Long.toString(returnCursor.getLong(sizeIndex)));
```










