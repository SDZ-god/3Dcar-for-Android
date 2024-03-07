# 3Dcar-for-Android
android {
    namespace = "com.example.mylibrary"
    compileSdk = 34
    ndkPath = "D:/tuanjie_edit/2022.3.2t6/Editor/Data/PlaybackEngines/AndroidPlayer/NDK"
    defaultConfig {
        minSdk = 24

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        consumerProguardFiles("consumer-rules.pro")
        ndk {
            // Skip deprecated ABIs. Only required when using NDK 16 or earlier.
            abiFilters.clear()
            abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86", "x86_64")
        }
    }
    splits {

        abi {

            isEnable = true

            reset()

            include("armeabi-v7a")

            isUniversalApk = true

        }

    }
    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}
___________________________________________________________________________________________________________________
package com.unity3d.player;

import android.app.Activity;
import android.content.Intent;
import android.content.res.Configuration;
import android.graphics.PixelFormat;
import android.os.Bundle;
import android.view.KeyEvent;
import android.view.LayoutInflater;
import android.view.MotionEvent;
import android.view.SurfaceView;
import android.view.View;
import android.view.ViewGroup;
import android.view.Window;
import android.view.WindowManager;
import android.os.Process;

public class UnityPlayerActivity extends Activity
{
    protected UnityPlayer mUnityPlayer; // don't change the name of this variable; referenced from native code
    private View mContentView;

    // Override this in your custom UnityPlayerActivity to tweak the command line arguments passed to the Unity Android Player
    // The command line arguments are passed as a string, separated by spaces
    // UnityPlayerActivity calls this from 'onCreate'
    // Supported: -force-gles20, -force-gles30, -force-gles31, -force-gles31aep, -force-gles32, -force-gles, -force-vulkan
    // See https://docs.unity3d.com/Manual/CommandLineArguments.html
    // @param cmdLine the current command line arguments, may be null
    // @return the modified command line string or null
    protected String updateUnityCommandLineArguments(String cmdLine)
    {
        return cmdLine;
    }

    // Setup activity layout
    @Override protected void onCreate(Bundle savedInstanceState)
    {
        requestWindowFeature(Window.FEATURE_NO_TITLE);
        super.onCreate(savedInstanceState);

        String cmdLine = updateUnityCommandLineArguments(getIntent().getStringExtra("unity"));
        getIntent().putExtra("unity", cmdLine);

        mUnityPlayer = new UnityPlayer(this, lifecycleEvents);
        View view = mUnityPlayer.getView();
        view.setAlpha(0);
        if (getLayoutId() > 0) {
            mContentView = LayoutInflater.from(this).inflate(getLayoutId(), null, false);
            setContentView(mContentView);
            int res_id = getResources().getIdentifier("layout_unity_content", "id", getPackageName());
            ViewGroup contentUnityLayout = mContentView.findViewById(res_id);
            if (contentUnityLayout != null) {
                contentUnityLayout.addView(mUnityPlayer);
            } else if (mContentView instanceof ViewGroup) {
                ((ViewGroup) mContentView).addView(mUnityPlayer);
            } else {
                mContentView = mUnityPlayer;
            }
            setContentView(mContentView);
        } else {
            mContentView = mUnityPlayer;
            setContentView(mUnityPlayer);
        }
        mUnityPlayer.requestFocus();
        init();
    }
    private IUnityPlayerLifecycleEvents lifecycleEvents = new IUnityPlayerLifecycleEvents() {
        @Override
        public void onUnityPlayerUnloaded() {
            UnityPlayerActivity.this.onUnityPlayerUnloaded();
        }

        @Override
        public void onUnityPlayerQuitted() {
            UnityPlayerActivity.this.onUnityPlayerQuitted();
        }
    };
    protected View getContentView() {
        return mContentView;
    }
    protected void sendUnityMessage(String s1, String s2, String s3) {
        UnityPlayer.UnitySendMessage(s1, s2, s3);

    }
    protected int getLayoutId() {
        return -1;
    }

    protected void init() {
    }
    // When Unity player unloaded move task to background
   public void onUnityPlayerUnloaded() {
        moveTaskToBack(true);
    }

    // Callback before Unity player process is killed
   public void onUnityPlayerQuitted() {
    }

    @Override protected void onNewIntent(Intent intent)
    {
        // To support deep linking, we need to make sure that the client can get access to
        // the last sent intent. The clients access this through a JNI api that allows them
        // to get the intent set on launch. To update that after launch we have to manually
        // replace the intent with the one caught here.
        setIntent(intent);
        mUnityPlayer.newIntent(intent);
    }

    // Quit Unity
    @Override protected void onDestroy ()
    {
        mUnityPlayer.destroy();
        super.onDestroy();
    }

    // If the activity is in multi window mode or resizing the activity is allowed we will use
    // onStart/onStop (the visibility callbacks) to determine when to pause/resume.
    // Otherwise it will be done in onPause/onResume as Unity has done historically to preserve
    // existing behavior.
    @Override protected void onStop()
    {
        super.onStop();

        if (!MultiWindowSupport.getAllowResizableWindow(this))
            return;

        mUnityPlayer.pause();
    }

    @Override protected void onStart()
    {
        super.onStart();

        if (!MultiWindowSupport.getAllowResizableWindow(this))
            return;

        mUnityPlayer.resume();
    }

    // Pause Unity
    @Override protected void onPause()
    {
        super.onPause();

        MultiWindowSupport.saveMultiWindowMode(this);

        if (MultiWindowSupport.getAllowResizableWindow(this))
            return;

        mUnityPlayer.pause();
    }

    // Resume Unity
    @Override protected void onResume()
    {
        super.onResume();

        if (MultiWindowSupport.getAllowResizableWindow(this) && !MultiWindowSupport.isMultiWindowModeChangedToTrue(this))
            return;

        mUnityPlayer.resume();
    }

    // Low Memory Unity
    @Override public void onLowMemory()
    {
        super.onLowMemory();
        mUnityPlayer.lowMemory();
    }

    // Trim Memory Unity
    @Override public void onTrimMemory(int level)
    {
        super.onTrimMemory(level);
        if (level == TRIM_MEMORY_RUNNING_CRITICAL)
        {
            mUnityPlayer.lowMemory();
        }
    }

    // This ensures the layout will be correct.
    @Override public void onConfigurationChanged(Configuration newConfig)
    {
        super.onConfigurationChanged(newConfig);
        mUnityPlayer.configurationChanged(newConfig);
    }

    // Notify Unity of the focus change.
    @Override public void onWindowFocusChanged(boolean hasFocus)
    {
        super.onWindowFocusChanged(hasFocus);
        mUnityPlayer.windowFocusChanged(hasFocus);
    }

    // For some reason the multiple keyevent type is not supported by the ndk.
    // Force event injection by overriding dispatchKeyEvent().
    @Override public boolean dispatchKeyEvent(KeyEvent event)
    {
        if (event.getAction() == KeyEvent.ACTION_MULTIPLE)
            return mUnityPlayer.injectEvent(event);
        return super.dispatchKeyEvent(event);
    }

    // Pass any events not handled by (unfocused) views straight to UnityPlayer
    @Override public boolean onKeyUp(int keyCode, KeyEvent event)     { return mUnityPlayer.onKeyUp(keyCode, event); }
    @Override public boolean onKeyDown(int keyCode, KeyEvent event)   { return mUnityPlayer.onKeyDown(keyCode, event); }
    @Override public boolean onTouchEvent(MotionEvent event)          { return mUnityPlayer.onTouchEvent(event); }
    @Override public boolean onGenericMotionEvent(MotionEvent event)  { return mUnityPlayer.onGenericMotionEvent(event); }
}
__________________________________________________________________________________________________________________________________
package com.example.mynewcar2;


import android.os.Bundle;
import android.view.View;
import android.widget.Button;

import com.unity3d.player.UnityPlayerActivity;

public class MainActivity extends UnityPlayerActivity {

    private View view;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }

    @Override
    protected int getLayoutId() {
        return R.layout.activity_main;
    }

    @Override
    protected void init() {
        view = getContentView();
        Button btn_xuanzhuan_l = view.findViewById(R.id.btn_xuanzhuan_l);
        Button btn_xuanzhuan_r = view.findViewById(R.id.btn_xuanzhuan_r);
        Button btn_cl = view.findViewById(R.id.btn_cl);
//        sendUnityMessage("CarRotate");
//        sendUnityMessage("AutoRotate");
//        sendUnityMessage("WheelRun");
        btn_xuanzhuan_l.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                sendUnityMessage("CarRotateLeft");
            }
        });
        btn_xuanzhuan_r.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                sendUnityMessage("CarRotateRight");
            }
        });
        btn_cl.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                sendUnityMessage("WheelRun");
            }
        });
    }

    private void sendUnityMessage(String s) {
        sendUnityMessage("Main Camera", s, "");
    }


}
