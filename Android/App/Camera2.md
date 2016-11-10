`CameraManager`：系统服务，通过 `CameraManager`去获得相机设备对象。

`CameraDevice`：提供描述相机硬件设备支持可用的和输出的参数，这些信息通过`CameraCharacteristics`获得，`CameraCharacteristics`又是从`getCameraCharacteristics(cameraId)`获得。

### 通过相机拍照

```Java
public void openCamera(String cameraId, CameraDevice.StateCallback callback, Handler handler)
```

- 使用`getCameraIdList()`来获得可用摄像设备的列表。一旦成功打开相机，`CameraDevice.StateCallback`中的`onOpened(CameraDevice)`将被调用。相机设备可以通过调用`createCaptureSession()`和`createCaptureRequest()`去设置。如果打开相机设备失败，那么设备回调的`onError`方法将被调用，后续调用相机设备将抛出一个`CameraAccessException`。

```Java
public abstract CaptureRequest.Builder createCaptureRequest(int templateType)
```

- 为请求拍照创建一个`CaptureRequest.Builder`，初始化目标用例的模板。为特定的相机设备选择最好的设置，不建议为不同的相机设备重用相同的请求。创建一个builder，根据需要进行设置。
```
TEMPLATE_PREVIEW
TEMPLATE_RECORD
TEMPLATE_STILL_CAPTURE
TEMPLATE_VIDEO_SNAPSHOT
TEMPLATE_MANUAL
...
```

```Java
public abstract void createCaptureSession(List<surface> outputs, CameraCaptureSession.StateCallback callback, Handler handler)
```

- 创建由相机设备的输出surface组成的拍照会话。会话确定了为每个拍照的请求的Output Surfaces。请求可以使用全部或部分的Output Surfaces。每个surface必须预先设置适当的大小和格式去匹配相机设备的可支持的大小和格式。一个目标surface可以从不同的类中获取。
```
SurfaceView
TextureView
MediaCodec
MediaRecorder
RenderScriptAllocation
ImageReader
```

- Request一旦创建可以交给活动的拍照会话`CameraCaptureSession`，进行拍照（capture ）或者连续拍照（captureBurst），预览setRepeatingRequest()或setRepeatingBurst()。



权限
```xml
<uses-permission android:name="android.permission.CAMERA/"/>
```

布局
```xml
<!--?xml version=1.0 encoding=utf-8?-->
<linearlayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_height="match_parent"
    android:layout_width="match_parent"
    android:orientation="vertical" >
 
    <Textureview
        android:id="@+id/textureview"
        android:layout_height="fill_parent/"
        android:layout_width="fill_parent">
    </Textureview>
</linearlayout>
```

核心代码

```Java
public class CameraFragment extends Fragment implements TextureView.SurfaceTextureListener {
    private TextureView mPreviewView;
    private Handler mHandler;
    private HandlerThread mThreadHandler;
    private Size mPreviewSize;
    private CaptureRequest.Builder mPreviewBuilder;
 
    public static CameraFragment newInstance() {
        return new CameraFragment();
    }
 
    @SuppressWarnings(ResourceType)
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View v = inflater.inflate(R.layout.camera_frag, null);
        initLooper();
        initUIAndListener(v);
        return v;
    }
    //很多过程都变成了异步的了，所以这里需要一个子线程的looper
    private void initLooper() {
        mThreadHandler = new HandlerThread(CAMERA2);
        mThreadHandler.start();
        mHandler = new Handler(mThreadHandler.getLooper());
    }
    //可以通过TextureView或者SurfaceView
    private void initUIAndListener(View v) {
        mPreviewView = (TextureView) v.findViewById(R.id.textureview);
        mPreviewView.setSurfaceTextureListener(this);
    }
 
    @SuppressWarnings(ResourceType)
    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        try {
            //获得CameraManager
            CameraManager cameraManager = (CameraManager) getActivity().getSystemService(Context.CAMERA_SERVICE);
            //获得属性
            CameraCharacteristics characteristics = cameraManager.getCameraCharacteristics(0);
            //支持的STREAM CONFIGURATION
            StreamConfigurationMap map = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
            //显示的size
            mPreviewSize = map.getOutputSizes(SurfaceTexture.class)[0];
            //打开相机
            cameraManager.openCamera(0, mCameraDeviceStateCallback, mHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }
 
    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height) {
    }
 
    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
        return false;
    }
 
    //TextureView.SurfaceTextureListener
    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {
    }
 
    private CameraDevice.StateCallback mCameraDeviceStateCallback = new CameraDevice.StateCallback() {
 
        @Override
        public void onOpened(CameraDevice camera) {
            try {
                startPreview(camera);
            } catch (CameraAccessException e) {
                e.printStackTrace();
            }
        }
 
        @Override
        public void onDisconnected(CameraDevice camera) {
        }
 
        @Override
        public void onError(CameraDevice camera, int error) {
        }
    };
    //开始预览，主要是camera.createCaptureSession这段代码很重要，创建会话
    private void startPreview(CameraDevice camera) throws CameraAccessException {
        SurfaceTexture texture = mPreviewView.getSurfaceTexture();
        texture.setDefaultBufferSize(mPreviewSize.getWidth(), mPreviewSize.getHeight());
        Surface surface = new Surface(texture);
        try {
            mPreviewBuilder = camera.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
        mPreviewBuilder.addTarget(surface);
        camera.createCaptureSession(Arrays.asList(surface), mSessionStateCallback, mHandler);
    }
 
    private CameraCaptureSession.StateCallback mSessionStateCallback = new CameraCaptureSession.StateCallback() {
 
        @Override
        public void onConfigured(CameraCaptureSession session) {
            try {
                updatePreview(session);
            } catch (CameraAccessException e) {
                e.printStackTrace();
            }
        }
 
        @Override
        public void onConfigureFailed(CameraCaptureSession session) {
        }
    };
 
    private void updatePreview(CameraCaptureSession session) throws CameraAccessException {
        session.setRepeatingRequest(mPreviewBuilder.build(), null, mHandler);
    }
}
```