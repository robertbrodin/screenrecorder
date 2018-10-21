# Screenrecorder 
-> Forked from https://github.com/chinmoyp/screenrecorder (with a few edits)
#
**App Info:**
The program allows the user to record the ENTIRE screen while inside of the app. The recorder does not record when outside of the app. MediaProjection and MediaProjectionManager (android.media.projection.*) are the two classes main classes implemented. 

*All of these variables are mandatory for the MediaProjection to work. The process does not change for implementing this in Augmented Reality. The only difference is putting the code you are copying in onCreate() after super.OnCreate() and setContentView, and before the rest of the AR Code.*

## How to use:
The code must be copied into the Activity that is will recorded in. (See https://github.com/robertbrodin/screenrecorder/blob/master/app/src/main/java/org/risingtide/screenrecorder/MainActivity.java for comments)
**Necessary classes to import:**
```
// Need to import all of these Classes to make the Screen recording work.
import android.os.Environment;
import android.content.Context;
import android.content.Intent;
import android.hardware.display.DisplayManager;
import android.hardware.display.VirtualDisplay;
import android.media.MediaRecorder;
import android.media.projection.MediaProjection;
import android.media.projection.MediaProjectionManager;
import android.media.projection.MediaProjection.Callback;
import android.util.DisplayMetrics;
import android.util.Log;
import android.view.View;
import android.widget.Toast;
import android.widget.ToggleButton;
import android.media.projection.*;
import java.io.File;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
```

**Necessary variables to include after class constructor:**
Include all of these variables below the class declaration in the file you are copying this to.


```
// Default constructor for the class
public class MainActivity extends AppCompatActivity {

    // Remove TAG in an augmented reality file (AR file should already has a default TAG).
    private static final String TAG = "MainActivity";

    // DO NOT CHANGE PERMISSION CODE (messes up onActivityResult() method)
    private static final int PERMISSION_CODE = 1;
    private int mScreenDensity;
    private MediaProjectionManager mProjectionManager;

    // DISPLAY_WIDTH and DISPLAY_HEIGHT are set default to 480 by 640, but are changed relative to the size of the phone being used in the onCreate() method.
    private static int DISPLAY_WIDTH = 480;
    private static int DISPLAY_HEIGHT = 640;

    private MediaProjection mMediaProjection;
    private VirtualDisplay mVirtualDisplay;
    private MediaProjection.Callback mMediaProjectionCallback;
    private ToggleButton mToggleButton;
    private MediaRecorder mMediaRecorder;
```
**Code needed after onCreate() method:**
Copy the code into the onCreate() method. This will work with AR.
```
 DisplayMetrics metrics = new DisplayMetrics();
        getWindowManager().getDefaultDisplay().getMetrics(metrics);
        mScreenDensity = metrics.densityDpi;

        DISPLAY_HEIGHT = metrics.heightPixels;
        DISPLAY_WIDTH = metrics.widthPixels;

        // inits the recorder (settings for recording)
        initRecorder();
        prepareRecorder();

        // Creates a MediaProjectionManager object in the context of the app.
        mProjectionManager = (MediaProjectionManager) getSystemService
                (Context.MEDIA_PROJECTION_SERVICE);

        // Finds button with ID toggle. This can be changed to any button, you will just need to cast it to (RadioButton) or (Button) instead of (ToggleButton).
        mToggleButton = (ToggleButton) findViewById(R.id.toggle);

        // Creates an event listener to see when the button is clicked.
        mToggleButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                onToggleScreenShare(v);
            }
        });

        mMediaProjectionCallback = new MediaProjectionCallback();
    }

    // onDestroy makes sure the recording stops (failsafe if .stop() doesn't work the first time).
    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mMediaProjection != null) {
            mMediaProjection.stop();
            mMediaProjection = null;
        }
```
**Lastly, copy the code below outside of the onCreate() method. This code includes all methods needed to record:**
```
// onActivityResult() checks permissions and starts the recording using createVirtualDisplay().
    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode != PERMISSION_CODE) {
            Log.e(TAG, "Unknown request code: " + requestCode);
            return;
        }
        if (resultCode != RESULT_OK) {
            Toast.makeText(this,
                    "Screen Cast Permission Denied", Toast.LENGTH_SHORT).show();
            mToggleButton.setChecked(false);
            return;
        }
        mMediaProjection = mProjectionManager.getMediaProjection(resultCode, data);
        mMediaProjection.registerCallback(mMediaProjectionCallback, null);
        mVirtualDisplay = createVirtualDisplay();
        mMediaRecorder.start();
    }

    // Toggles Screen recording on and off.
    public void onToggleScreenShare(View view) {
        if (((ToggleButton) view).isChecked()) {
            shareScreen();
        } else {
            mMediaRecorder.stop();
            mMediaRecorder.reset();
            Log.v(TAG, "Recording Stopped");
            stopScreenSharing();
        }
    }

    // Starts screen recording.
    private void shareScreen() {
        if (mMediaProjection == null) {
            startActivityForResult(mProjectionManager.createScreenCaptureIntent(), PERMISSION_CODE);
            return;
        }
        mVirtualDisplay = createVirtualDisplay();
        mMediaRecorder.start();
    }

    // Stops screen recording.
    private void stopScreenSharing() {
        if (mVirtualDisplay == null) {
            return;
        }
        mVirtualDisplay.release();
        //mMediaRecorder.release();
    }

    // Creates a VirtualDisplay object using the parameters set at the start.
    private VirtualDisplay createVirtualDisplay() {
        return mMediaProjection.createVirtualDisplay("MainActivity",
                DISPLAY_WIDTH, DISPLAY_HEIGHT, mScreenDensity,
                DisplayManager.VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR,
                mMediaRecorder.getSurface(), null /*Callbacks*/, null /*Handler*/);
    }

    // Checks if mToggleButton is clicked. Will have to change mToggleButton to the new button variable if a mToggleButton is not being used.
    private class MediaProjectionCallback extends MediaProjection.Callback {
        @Override
        public void onStop() {
            if (mToggleButton.isChecked()) {
                mToggleButton.setChecked(false);
                mMediaRecorder.stop();
                mMediaRecorder.reset();
                Log.v(TAG, "Recording Stopped");
            }
            mMediaProjection = null;
            stopScreenSharing();
            Log.i(TAG, "MediaProjection Stopped");
        }
    }

    // Prepares recorder (checks for exceptions)
    private void prepareRecorder() {
        try {
            mMediaRecorder.prepare();
        } catch (IllegalStateException | IOException e) {
            e.printStackTrace();
            finish();
        }
    }

    // Used to create the desired filePath that the file will be saved in. By default it goes to /sdcard/recordings, but could be changed to anything.
    public String getFilePath() {
        // Can change the folder which the video is saved into.
        final String directory = Environment.getExternalStorageDirectory() + File.separator + "Recordings";
        if (!Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())) {
            // Toast is just a fancy class that helps print custom error messages.
            Toast.makeText(this, "Failed to get External Storage", Toast.LENGTH_SHORT).show();
            return null;
        }
        // Using final because the folder variable should NEVER be changed.
        final File folder = new File(directory);
        boolean success = true;

        // If the folder doesn't exist, it is created using mkdir().
        if (!folder.exists()) {
            success = folder.mkdir();
        }
        String filePath;

        // Once the folder exists (if it didn't already), the filePath is set to the directory + capture_date.mp4. Capture can be changed quite easily.
        if (success) {
            String videoName = ("capture_" + getCurSysDate() + ".mp4");
            filePath = directory + File.separator + videoName;
        } else {
            Toast.makeText(this, "Failed to create Recordings directory", Toast.LENGTH_SHORT).show();
            return null;
        }
        return filePath;
    }

    // Gets the date (relative to the device). Will be used for the file name in getFilePath().
    public String getCurSysDate() {
        return new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss").format(new Date());
    }

    // Used for Codec and recording settings!
    private void initRecorder() {
        int YOUR_REQUEST_CODE = 200; // could be something else..
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            ActivityCompat.requestPermissions(this, new String[]{android.Manifest.permission.RECORD_AUDIO, android.Manifest.permission.WRITE_EXTERNAL_STORAGE},
                    YOUR_REQUEST_CODE);


            if (mMediaRecorder == null) {
                mMediaRecorder = new MediaRecorder();
                mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
                mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.SURFACE);
                mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4);
                mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264);
                mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
                // Bitrate is set relative to the screen, so it is just the width of the device * the height of the device.
                mMediaRecorder.setVideoEncodingBitRate(DISPLAY_WIDTH * DISPLAY_HEIGHT);
                // Framerate crashed at 60 when testing.
                mMediaRecorder.setVideoFrameRate(30);
                // Sets video size relative to the phone.
                mMediaRecorder.setVideoSize(DISPLAY_WIDTH, DISPLAY_HEIGHT);
                // Sets file path using the getFilePath() method.
                mMediaRecorder.setOutputFile(getFilePath());
            }
        }
    }
    ```
