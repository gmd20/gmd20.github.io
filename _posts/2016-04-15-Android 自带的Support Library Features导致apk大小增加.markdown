
Android 自带的Support Library Features （http://developer.android.com/tools/support-library/features.html），导致编译出来的APK里面很多 drawable使用的控件的图像。

这样编译出来的apk也许兼容性更好。但如果没用上的话，可以把它删掉，比如写个小app玩玩，界面都没有的，就没必要用着玩意了。
可以通过以下方法删掉，达到缩小apk的目的，修改后大小用1兆多变成20k了。

1.
--
把 build.gradle 改成这样
```text
pply plugin: 'com.android.application'

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.3"

    defaultConfig {
        applicationId "heath.lockscreen"
        minSdkVersion 21
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
}
```

2.
--

然后把  styles.xml 主题配置里面的对这样库的依赖换掉。

