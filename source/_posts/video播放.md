# VideoView

> 项目需要做一个简单的播放视频功能demo,后期会换成公司自己的组件,所以就没考虑使用第三方库了,直接上系统的VideoView,在这里记录下操作;
>顺便吐槽下:一直都听说简书编辑器好用,第一次使用,有点失望,markdown跟效果分栏竟然不能同步滚动,也不支持[TOC],没有个目录实在很不习惯,表情图标也不能插入,代码区块加空行经常都识别不了啊,就不能让我简单滴从笔记中直接粘贴md文本吗... ==!

[Demo项目下载](https://github.com/lucid-lynxz/BlogSamples/tree/master/VideoViewDemo)
[自己封装了一个播放器](https://github.com/lucid-lynxz/ZZVideoPlayer)

## 资源
1. [Android三种播放视频的方式](http://itindex.net/detail/35521-android-%E6%92%AD%E6%94%BE-%E8%A7%86%E9%A2%91)
2. [Android播放器框架分析之AwesomePlayer](http://www.mikewootc.com/wiki/android/mid/mediaplayer_awesome.html)
3. [音频与视频播放](http://www.bkjia.com/Androidjc/1110895.html) 讲的player类,比较全
4. [Android视频播放器实现小窗口和全屏状态切换](http://blog.csdn.net/u010072711/article/details/51517170)


> 视频播放原理:
系统会首先确定视频的格式，然后得到视频的编码..然后对编码进行解码，得到一帧一帧的图像，最后在画布上进行迅速更新,显然需要在独立的线程中完成,这时就需要使用surfaceView了


[android 支持的编码格式](http://developer.android.com/intl/zh-cn/guide/appendix/media-formats.html#core) 
![android_video_support_format.png](https://user-gold-cdn.xitu.io/2017/12/27/16096ad67f6d3007?w=994&h=606&f=png&s=44312)

## 基本使用
```java
VideoView mVv = (VideoView) findViewById(R.id.vv);
//添加播放控制条,还是自定义好点
mVv.setMediaController(new MediaController(this));

//设置视频源播放res/raw中的文件,文件名小写字母,格式: 3gp,mp4等,flv的不一定支持;
Uri rawUri = Uri.parse("android.resource://" + getPackageName() + "/" + R.raw.shuai_dan_ge);
mVv.setVideoURI(rawUri);

// 播放在线视频
mVideoUri = Uri.parse("http://****/abc.mp4");
mVv.setVideoPath(mVideoUri.toString());

mVv.start();
mVv.requestFocus();

/*
其他方法:
mVv.pause();
mVv.stop();
mVv.resume();
mVv.setOnPreparedListener(this);
mVv.setOnErrorListener(this);
mVv.setOnCompletionListener(this);**

Error信息处理 :
经常会碰到视频编码格式不支持的情况,这里还是处理一下,若不想弹出提示框就返回true;
http://developer.android.com/intl/zh-cn/reference/android/media/MediaPlayer.OnErrorListener.html

@Override 
public boolean onError(MediaPlayer mp, int what, int extra) { 
   if(what==MediaPlayer.MEDIA_ERROR_SERVER_DIED){ 
        Log.v(TAG,"Media Error,Server Died"+extra); 
   }else if(what==MediaPlayer.MEDIA_ERROR_UNKNOWN){ 
        Log.v(TAG,"Media Error,Error Unknown "+extra); 
   } 
return true; 
} 
*/
```
### 错误信息
[QCMediaPlayer.java](https://github.com/fallowu/slim_hardware_qcom_media/blob/master/QCMediaPlayer/com/qualcomm/qcmedia/QCMediaPlayer.java)
```shell
//常见错误: "无法播放此视频" -我测试的是:红米1s电信版4.4.4无法播放,但在三星s6（5.1.1）上就可以播放
//播放源:http://27.152.191.198/c12.e.99.com/b/p/67/c4ff9f6535ac41a598bb05bf5b05b185/c4ff9f6535ac41a598bb05bf5b05b185.v.854.480.f4v
MediaPlayer-JNI: QCMediaPlayer mediaplayer NOT present
MediaPlayer: Unable to create media player
MediaPlayer: Couldn't open file on client side, trying server side
MediaPlayer: error (1, -2147483648)
MediaPlayer: Error (1,-2147483648)
```
[有人说](http://stackoverflow.com/questions/24501086/why-mediaplayer-throws-not-present-error-when-creating-instance-of-it) 用下面的方式可以处理该异常,但我是使用系统封装好的控件,这个操作不到吧? 先记录下:
```java
MediaPlayer player = MediaPlayer.create(this, Uri.parse(sound_file_path));
MediaPlayer player = MediaPlayer.create(this, soundRedId, loop);
```

## 全屏播放 - 横竖屏切换

* `androidmanifest.xml` 中依然还是定义竖屏,并定义一个切换横纵屏按钮 `btnChange` :
```xml
<activity
    android:name="lynxz.org.video.VideoActivity"
    android:configChanges="keyboard|orientation|screenSize"
    android:screenOrientation="portrait"
    android:theme="@style/Theme.AppCompat.Light.NoActionBar"/>
```

* 布局:需要在 `VidioView` 外层套一个容器,比如:
```xml
    <RelativeLayout
        android:id="@+id/rl_vv"
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:background="@android:color/black"
        android:minHeight="200dp"
        android:visibility="visible">

        <VideoView
            android:id="@+id/vv"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"/>
    </RelativeLayout>
```

  这么做是为了在切换屏幕方向的时候对 `rl_vv` 进行拉伸,而内部的 `VideoView` 会依据视频尺寸重新计算宽高,我们看看其 `onMeasure()` 源码就明了了,但若是直接具体指定了view的宽高,则视频会被拉伸:
```java
//VideoView.java 
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int width = getDefaultSize(mVideoWidth, widthMeasureSpec);
    int height = getDefaultSize(mVideoHeight, heightMeasureSpec);
    ......
    setMeasuredDimension(width, height);
}
```


* 按钮监听,手动切换
```java
btnSwitch.setOnClickListener(View -> {
    if (getRequestedOrientation() == ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE) {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
    } else {
        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
    }
});
```
 设置VideoView布局尺寸
```java
@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
    if (mVv == null) {
        return;
    }
    if (this.getResources().getConfiguration().orientation == Configuration.ORIENTATION_LANDSCAPE){//横屏
        getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
        getWindow().getDecorView().invalidate();
        float height = DensityUtil.getWidthInPx(this);
        float width = DensityUtil.getHeightInPx(this);
        mRlVv.getLayoutParams().height = (int) width;
        mRlVv.getLayoutParams().width = (int) height;
    } else {
        final WindowManager.LayoutParams attrs = getWindow().getAttributes();
        attrs.flags &= (~WindowManager.LayoutParams.FLAG_FULLSCREEN);
        getWindow().setAttributes(attrs);
        getWindow().clearFlags(WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS);
        float width = DensityUtil.getWidthInPx(this);
        float height = DensityUtil.dip2px(this, 200.f);
        mRlVv.getLayoutParams().height = (int) height;
        mRlVv.getLayoutParams().width = (int) width;
    }
}
```

 自定义工具类
```java
//DensityUtil.java
public static final float getHeightInPx(Context context) {
	final float height = context.getResources().getDisplayMetrics().heightPixels;
	return height;
}
public static final float getWidthInPx(Context context) {
	final float width = context.getResources().getDisplayMetrics().widthPixels;
	return width;
}
```
另外,如果是将播放器放于fragment中进行横竖屏切换,则需要在onCreateView中`setRetainInstance(true);`,这样旋转后,才不会重新创建从头开始播放;

## 获取第一帧的内容作为封面
[参考文章](http://www.cnblogs.com/yydcdut/p/4222913.html)
```java
@TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
private void createVideoThumbnail() {
    Observable<Bitmap> observable = Observable.create(new Observable.OnSubscribe<Bitmap>() {
        @Override
        public void call(Subscriber<? super Bitmap> subscriber) {
            Bitmap bitmap = null;
            MediaMetadataRetriever retriever = new MediaMetadataRetriever();
            int kind = MediaStore.Video.Thumbnails.MINI_KIND;
            if (Build.VERSION.SDK_INT >= 14) {
                retriever.setDataSource(mVideoUrl, new HashMap<String, String>());
            } else {
                retriever.setDataSource(mVideoUrl);
            }
            bitmap = retriever.getFrameAtTime();
            subscriber.onNext(bitmap);
            retriever.release();
        }
    });

observable.observeOn(AndroidSchedulers.mainThread())
        .subscribeOn(Schedulers.io())
        .subscribe(new Action1<Bitmap>() {
            @Override
            public void call(Bitmap bitmap) {
                //设置封面
                mYourVideoPlayerContainer.setBackgroundDrawable(new BitmapDrawable(bitmap));
            }
        });
}
```

## 滑动改变屏幕亮度/音量
* 权限申请
```xml
<uses-permission android:name="android.permission.WRITE_SETTINGS"/>
<uses-permission android:name="android.permission.VIBRATE"/> //按需申请
```

* 修改亮度方法
```java
/*设置当前屏幕亮度值 0--255，并使之生效*/
private void setScreenBrightness(float value) {
    WindowManager.LayoutParams lp = getWindow().getAttributes();
    lp.screenBrightness = lp.screenBrightness + value / 255.0f;
    Vibrator vibrator;
    if (lp.screenBrightness > 1) {
        lp.screenBrightness = 1;
        //              vibrator = (Vibrator) getSystemService(VIBRATOR_SERVICE);
        //              long[] pattern = {10, 200}; // OFF/ON/OFF/ON...
        //              vibrator.vibrate(pattern, -1);
    } else if (lp.screenBrightness < 0.2) {
        lp.screenBrightness = (float) 0.2;
        //              vibrator = (Vibrator) getSystemService(VIBRATOR_SERVICE);
        //              long[] pattern = {10, 200}; // OFF/ON/OFF/ON...
        //              vibrator.vibrate(pattern, -1);
    }
    getWindow().setAttributes(lp);

    // 保存设置的屏幕亮度值
    // Settings.System.putInt(getContentResolver(), Settings.System.SCREEN_BRIGHTNESS, (int) value);
}
```

* 设置屏幕亮度模式方法 (自动/手动)
```java
// value 可取值: Settings.System.SCREEN_BRIGHTNESS_MODE_AUTOMATIC / SCREEN_BRIGHTNESS_MODE_MANUAL
private void setScreenMode(int value) {
    Settings.System.putInt(getContentResolver(), Settings.System.SCREEN_BRIGHTNESS_MODE, value);
}
```

* 监听播放区域
```java
mGestureDetector = new GestureDetector(this, mGestureListener);
vv.setOnTouchListener(this); 
@Override
public boolean onTouch(View v, MotionEvent event) {
    return mGestureDetector.onTouchEvent(event);
}
```

* onScroll的时候动态改变亮度
`onDown()` / `onScroll()` 返回true
```java
private android.view.GestureDetector.OnGestureListener mGestureListener = new GestureDetector.OnGestureListener() {
    @Override
    public boolean onDown(MotionEvent e) {
        return true;
    }

    @Override
    public void onShowPress(MotionEvent e) {
    }

    @Override
    public boolean onSingleTapUp(MotionEvent e) {
        return false;
    }

    @Override
    public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
        final double FLING_MIN_VELOCITY = 0.5;
        final double FLING_MIN_DISTANCE = 0.5;

        if (e1.getY() - e2.getY() > FLING_MIN_DISTANCE
                && Math.abs(distanceY) > FLING_MIN_VELOCITY) {
            setScreenBrightness(20);
        }
        if (e1.getY() - e2.getY() < FLING_MIN_DISTANCE
                && Math.abs(distanceY) > FLING_MIN_VELOCITY) {
            setScreenBrightness(-20);
        }
        return true;
    }

    @Override
    public void onLongPress(MotionEvent e) {
    }

    @Override
    public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
        return true;
    }
};
```

### 滑动修改音量
修改上方的 `onScroll()` 方法,调用以下操作
```java
    private void setVoiceVolume(boolean volumeUp) {
        //  设置音量绝对值的话,我在小米上突破不了限制,最大音量15,但是设置到10的时候就没法再增加了,最后使用系统的音量控制才可以
        //        int currentVolume = mAudioManager.getStreamVolume(AudioManager.STREAM_MUSIC);
        //        int maxVolume = mAudioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
        //        int flag = volumeUp ? 1 : -1;
        //        currentVolume += flag * 1;
        //        if (currentVolume >= maxVolume) {
        //            currentVolume = maxVolume;
        //        } else if (currentVolume <= 1) {
        //            currentVolume = 1;
        //        }
        //        Log.i(TAG, "setVoiceVolume currentVolume = " + currentVolume + " ,maxVolume = " + maxVolume);
        //        mAudioManager.setStreamVolume(AudioManager.STREAM_MUSIC, currentVolume, 0);

        //降低音量，调出系统音量控制
        if (volumeUp) {
            mAudioManager.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_RAISE,
                    AudioManager.FX_FOCUS_NAVIGATION_UP);
        } else {//增加音量，调出系统音量控制
            mAudioManager.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_LOWER,
                    AudioManager.FX_FOCUS_NAVIGATION_UP);
        }
    }
```

* 在页面关闭时可考虑恢复亮度/音量初始值
* 在onTouch的时候对触点进行判断,区分是修改音量或是改变亮度

## 需要处理的问题
### 拖动进度条,手动seekTo后,进度会跳动
断点跟踪后发现是native方法的问题,各大视频播放平台的客户端,比较普遍存在,暂无法处理:
![优酷客户端时间跳变](https://user-gold-cdn.xitu.io/2017/12/27/16096ad6ded691ba?w=462&h=273&f=gif&s=1162341)

找到些资源:
1. [关于Android VideoView seekTo不准确的解决方案](http://www.jianshu.com/p/f51b2febcfd2)
2. [视频关键帧提取](http://blog.csdn.net/qingyuanluofeng/article/details/45375647)
第一个提到的关键帧问题,我找了个视频测试了下,seekTo到固定的时间点,则跳变的位置也固定;

### 暂停/恢复 页面时,视频重新加载
现象: 在视频播放时,使页面 `onPause()` ,之后再恢复,则 `videoView` 会重新开始播放,临时的处理方案是在 `onPause()` 的时候记录当前播放进度位置,在 `onResume()` 的时候拖动到该进度位置,但是该方案仍会有黑屏现象,代码如下:
```java
int mPlayingPos = 0;

@Override
protected void onPause() {
    mPlayingPos = mVideoView.getCurrentPosition(); //先获取再stopPlay(),原因自己看源码
    mVideoView.stopPlayback();
    super.onPause();
}

@Override
protected void onResume() {
    if (mPlayingPos > 0) {
        //此处为更好的用户体验,可添加一个progressBar,有些客户端会在这个过程中隐藏底下控制栏,这方法也不错
        mVideoView.start();
        mVideoView.seekTo(mPlayingPos);
        mPlayingPos = 0;
    }
    super.onResume();
}
```

找到些可能相关的文章,链接已失效,快照如下(还得去看看 `surfaceView` 啊 ~ ~# ):
另一篇类似的: [android开发常见问题](http://www.boyunjian.com/do/article/snapshot.do?uid=3453185252600635258) 问题7,也指明是 `surfaceview` 的原因,之所以是黑色的见后面的解释:
> Activity 调用的顺序是 onPause() -> onStop()
> SurfaceView 调用了 surfaceDestroyed() 方法 
> 然后再切回程序 
> Activity 调用的顺序是 onRestart() -> onStart() -> onResume ()
> SurfaceView` 调用了 surfaceChanged() -> surfaceCreated() 方法 
> 按挂断键或锁定屏幕 
> Activity 只调用 onPause() 方法 
> 解锁后 Activity 调用 onResume() 方法 
> SurfaceView 什么方法都不调用 

### 网络变化/切换应用后恢复播放
播放过程中,假如只缓冲了一部分视频,则当播放完缓冲部分后,会抛出1004异常,即使此时网络连接已经恢复,控件也不会自动继续缓冲:
`MediaPlayer: error (1, -1004)` 
源码注释: `File or network related operation errors`
同时,由于SurfaceView在页面onStop()时会destroy,比如播放时,用户按下home键或切换到其他应用页面再返回时,视频播放停止,此时需要重新加载视频并播放到上次停止的位置;

另外,有测试发现在三星G9200手机上,报 `1004` 这个错的时候会弹出错误提示框,然后卡死重启...
对比了下日志:
```java
// API23 MediaPlayer.java
@Override
public void handleMessage(Message msg) {
    if (mMediaPlayer.mNativeContext == 0) {
        Log.w(TAG, "mediaplayer went away with unhandled events");
        return;
    }
    switch(msg.what) {
        case MEDIA_ERROR:
        Log.e(TAG, "Error (" + msg.arg1 + "," + msg.arg2 + ")");
        boolean error_was_handled = false;
        if (mOnErrorListener != null) {
            error_was_handled = mOnErrorListener.onError(mMediaPlayer, msg.arg1, msg.arg2);
        }
        if (mOnCompletionListener != null && ! error_was_handled) {
            mOnCompletionListener.onCompletion(mMediaPlayer);
        }
        stayAwake(false);
        return;
    default:
        Log.e(TAG, "Unknown message type " + msg.what);
        return;
    }
}
```

```java
//API23 VideoView.java
public void setOnErrorListener(OnErrorListener l){
    mOnErrorListener = l;
}


private MediaPlayer.OnErrorListener mErrorListener =
    new MediaPlayer.OnErrorListener() {
    public boolean onError(MediaPlayer mp, int framework_err, int impl_err) {
    ......

        /* If an error handler has been supplied, use it and finish. */
        if (mOnErrorListener != null) { //如果这里没有处理,则每次发生异常都会弹出提示框,可能造成崩溃
            if (mOnErrorListener.onError(mMediaPlayer, framework_err, impl_err)) {
                return true;
            }
        }

        /* Otherwise, pop up an error dialog so the user knows that
            * something bad has happened. Only try and pop up the dialog
            * if we're attached to a window. When we're going away and no
            * longer have a window, don't bother showing the user an error.
            */
        if (getWindowToken() != null) {
            Resources r = mContext.getResources();
            int messageId;

            if (framework_err == MediaPlayer.MEDIA_ERROR_NOT_VALID_FOR_PROGRESSIVE_PLAYBACK) {
                messageId = com.android.internal.R.string.VideoView_error_text_invalid_progressive_playback;
            } else {
                messageId = com.android.internal.R.string.VideoView_error_text_unknown;
            }

            // 弹出错误提示框
            new AlertDialog.Builder(mContext)
                    .setMessage(messageId)
                    .setPositiveButton(com.android.internal.R.string.VideoView_error_button,
                            new DialogInterface.OnClickListener() {
                                public void onClick(DialogInterface dialog, int whichButton) {
                                    /* If we get here, there is no onError listener, so
                                        * at least inform them that the video is over.
                                        */
                                    if (mOnCompletionListener != null) {
                                        mOnCompletionListener.onCompletion(mMediaPlayer);
                                    }
                                }
                            })
                    .setCancelable(false)
                    .show();
        }
        return true;
    }
};

```

因此需要对网络变化进行监听:
```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

```java
@Override
protected void onResume() {
    super.onResume();
    mNetworkState = NetworkHelper.getNetworkType(this);
    //播放网络视频时,需要检测判断网络状态变化
    if (SCHEME_HTTP.equalsIgnoreCase(mVideoUri.getScheme()) && mNetworkState == 0) {
        MessageUtils.showAlertDialog(this, "提示", getResources().getString(R.string.network_error), null);
    } else {
        if (mPlayingPos > 0) {
            mVv.start();
            mVv.seekTo(mPlayingPos);
            mPlayingPos = 0;
        }
    }
}

/**
 * 监听网络变化,用于重新缓冲
 */
private void registerNetworkReceiver() {
    if (mNetworkReceiver == null) {
        mNetworkReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String action = intent.getAction();
                if (SCHEME_HTTP.equalsIgnoreCase(mVideoUri.getScheme())
                            && action.equalsIgnoreCase(ConnectivityManager.CONNECTIVITY_ACTION)) {
                    doWhenNetworkChange();
                }
            }
        };
    }
    registerReceiver(mNetworkReceiver, new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION));
}


/**
 * 网络播放
 */
public void doWhenNetworkChange() {
    mNetworkState = NetworkHelper.getNetworkType(this);
    //保存当前已缓存长度
    int bufferPercentage = mVv.getBufferPercentage();
    mLastLoadLength = bufferPercentage * mVv.getDuration() / 100;
    //这里需要判断 0
    int currentPosition = mVv.getCurrentPosition();
    if (currentPosition > 0) {
        mPlayingPos = currentPosition;
    }
    debugLog(bufferPercentage + " 网络变化 ... " + mNetworkState + " 缓存长度 " + mLastLoadLength + " -- " + currentPosition);

    if (mNetworkState == NetworkHelper.NETWORK_TYPE_INVALID && bufferPercentage < 100) {
        // 监听当前播放位置,在达到缓冲长度前自动停止
        if (mCheckPlayingProgressTimer == null) {
            mCheckPlayingProgressTimer = new Timer();
        }
        mCheckPlayingProgressTimer.schedule(new TimerTask() {
            @Override
            public void run() {
                if (mPlayingPos >= mLastLoadLength - deltaTime) {
                    mVv.pause();
                }
            }
        }, 0, 1000);//每秒检测一次
    } else {
        restartPlayVideo();
    }
}

private void restartPlayVideo() {
    //todo 添加 progressBar 体验好点
    if (mCheckPlayingProgressTimer != null) {
        mCheckPlayingProgressTimer.cancel();
        mCheckPlayingProgressTimer = null;
    }
    mVv.setVideoURI(mVideoUri);
    mVv.start();
    mVv.seekTo(mPlayingPos);

    mLastLoadLength = -1;
    mPlayingPos = 0;
}

@Override
protected void onPause() {
    mPlayingPos = mVv.getCurrentPosition();
    mVv.pause();
    super.onPause();
}

@Override
protected void onStop() {
    mVv.stopPlayback();
    mLastLoadLength = 0;
    debugLog("onResume " + mPlayingPos + " -- " + mLastLoadLength);
    super.onStop();
}

@Override
protected void onDestroy() {
    super.onDestroy();
    if (mCheckPlayingProgressTimer != null) {
        mCheckPlayingProgressTimer.cancel();
        mCheckPlayingProgressTimer = null;
    }
    ......
    unregisterNetworkReceiver();
}
```




### seekbar变化超出缓冲长度
使用系统提供的控件 `mVv.setMediaController(new MediaController(this));` 的话,在断网时,仍可以拖动超出缓冲长度的范围,会报错,这个还是得自定义才能控制可拖动位置,不再赘述;
```shell
MediaPlayer: Attempt to perform seekTo in wrong state: mPlayer=0x7f7ebbf5c0, mCurrentState=0
MediaPlayer: Error (1,-1004)
```

```java
//API 23 MediaController.java
private final OnSeekBarChangeListener mSeekListener = new OnSeekBarChangeListener() {
    @Override
    public void onProgressChanged(SeekBar bar, int progress, boolean fromuser) {
        if (!fromuser) {
            // We're not interested in programmatically generated changes to
            // the progress bar's position.
            return;
        }

        //这里就只是设定mediaplayer的播放位置而已
        long duration = mPlayer.getDuration();
        long newposition = (duration * progress) / 1000L;
        mPlayer.seekTo( (int) newposition);
        if (mCurrentTime != null)
            mCurrentTime.setText(stringForTime( (int) newposition));
    }
}
```

当断网后,用户拖动超出缓冲区长度的话mediaplayer报错,此时再次点击VideoView区域,不会触发显示控制条,真是各种不方便啊,还是建议自己写一个控制条;


# SurfaceView

## 资源
1. [SurfaceView 源码分析及使用](http://tech.youzan.com/surfaceview-sourcecode/) 
 这篇讲到了 `SurfaceView` 会显示黑色区域的原因:
> SurfaceView 的 draw 和 dispatchDraw 方法中看到，SurfaceView 中，windownType变量被初始化为WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA，所以在创建绘制这个 View 的过程中整个 Canvas 会被涂成黑色
2. [浮层视频效果,在另外一个Window使用SurfaceView无法正常显示的问题排查与解决](http://blog.csdn.net/Hello__Zero/article/details/42389197)

## surfaceView黑屏
1. 无内容时,默认会绘制黑色背景图
```java
//SurfaceView.java
int mWindowType = WindowManager.LayoutParams.TYPE_APPLICATION_MEDIA;
public void draw(Canvas canvas) {
    if (mWindowType != WindowManager.LayoutParams.TYPE_APPLICATION_PANEL) {
        // draw() is not called when SKIP_DRAW is set
        if ((mPrivateFlags & PFLAG_SKIP_DRAW) == 0) {
            // punch a whole in the view-hierarchy below us
            canvas.drawColor(0, PorterDuff.Mode.CLEAR);
        }
    }
    super.draw(canvas);
}
```
感谢[@尚弟很忙哒](http://www.jianshu.com/users/ce0ae357d9bb) 的提醒, 设置页面主题为透明(`android:theme="@android:style/Theme.Translucent"`)时,在初始缓冲阶段,VideoView区域会变成透明:
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="org.lynxz.androiddemos.VideoViewActivity">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:background="@android:color/holo_blue_bright">

        <VideoView
            android:id="@+id/vv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"/>
    </RelativeLayout>
</RelativeLayout>
```
在SurfaceView类开头有酱紫的一段注释,大概能解释为什么缓冲时候视频区域会透明了:
```quota
The surface is Z ordered so that it is behind the window holding its SurfaceView; 
the SurfaceView punches a hole in its window to allow its surface to be displayed.
```
因此处理方案就变成将SurfaceView挪到上层即可:
```java
mVv.setZOrderOnTop(true);
```
不过挪动之后就可以设置VideoView的背景,此时才不会遮盖实际的视频绘图了,xml中指定吧,这里省略,不过如果VideoView区域还有其他控件的话,会被遮盖,所以最后我就没设定zorderOnTop了,而是直接在xml中指定VideoView的背景色,然后在onPrepare回调的时候,去掉背景即可(按需延时,或者在有播放进度,要更新进度条的时候进行去掉背景操作都ok,不然可能会有一瞬间的透明):
```java
mVv.setBackgroundColor(Color.TRANSPARENT);
```
之前是打算像网上说的给VideoView的holder添加一个callback,(`mVv.getHolder().addCallback(new SurfaceHolder.Callback() {...}`) ,在 `surfaceCreated()` 的时候获取canvas并手动绘制背景色,但是 `holder.lockCanvas()` 一直返回 `null` ,log信息提示:
```log
E/SurfaceHolder: Exception locking surface
                           java.lang.IllegalArgumentException
                           at android.view.Surface.nativeLockCanvas(Native Method)
                            .......
```
看到native我暂时就没招了,打住,老实用变通方法吧;

2. 手机 "菜单键" 导致应用被stop,虽然此时看起来可见
`SurfaceView.java`   的注释: 在调用菜单键的时候虽然页面貌似可见,但实际已经调用了onStop()方法了,而surface在window不可见时会销毁:
>The Surface will be created for you while the SurfaceView's window is
 visible; you should implement {@link SurfaceHolder.Callback#surfaceCreated}
 and {@link SurfaceHolder.Callback#surfaceDestroyed} to discover when the
 Surface is created and destroyed as the window is shown and hidden.

 ![ 按下菜单键后返回](https://user-gold-cdn.xitu.io/2017/12/27/16096ad6e9b31984?w=356&h=645&f=gif&s=1471630)

3. VideoView无法播放f4v格式(三星s6可以播放,红米1s(4.4.4)播放失败)....
以后能力够了可以参考下这篇 :     
    * [Android平台Stagefright中增加flv/f4v支持及相关原理介绍](http://blog.csdn.net/bonderwu/article/details/6261798)
    * [Stagefright功能扩展](http://www.iaeej.com/xxydzgc/ch/reader/create_pdf.aspx?file_no=20130512&year_id=2013&quarter_id=5&falg=1) 这篇论文前半部分有关于多媒体框架调用的介绍
