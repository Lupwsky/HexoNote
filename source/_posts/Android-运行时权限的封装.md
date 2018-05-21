---
title: Android-运行时权限的封装
date: 2018-05-21 16:29:19
categories: Android
---

# 运行时权限工具类的封装

自己封装的运行时权限工具类，代码如下：

```java
@TargetApi(Build.VERSION_CODES.KITKAT_WATCH)
public class PermissionUtils {
    // 定义权限组，权限的申请在同一组的只需要申请其中一个就可以了，
    // 所以这里只需要定义其中一个就可以代表一个权限组
    // ...
}
```

<!-- more -->

```java
/**
* Created by lupwp on 2017/10/30.
*
* 权限管理工具类，本类主要用于危险权限组的权限申请
*/

@TargetApi(Build.VERSION_CODES.KITKAT_WATCH)
public class PermissionUtils {
    // 定义权限组，权限的申请在同一组的只需要申请其中一个就可以了，
    // 所以这里只需要定义其中一个就可以代表一个权限组
    // (1) 日历权限 - READ_CALENDAR、WRITE_CALENDAR
    public final static String CALENDAR = Manifest.permission.READ_CALENDAR;

    // (2) 相机权限 - CAMERA
    public final static String  CAMERA = Manifest.permission.CAMERA;

    // (3) 联系人权限 - READ_CONTACTS、WRITE_CONTACTS、GET_ACCOUNTS
    public final static String CONTACTS = Manifest.permission.READ_CONTACTS;

    // (4) 地理位置权限 - ACCESS_FINE_LOCATION、ACCESS_COARSE_LOCATION
    public final static String LOCATION = Manifest.permission.ACCESS_FINE_LOCATION;

    // (5) 录音权限 - RECORD_AUDIO
    public final static String AUDIO = Manifest.permission.RECORD_AUDIO;

    // (6) 电话权限 - READ_PHONE_STATE、CALL_PHONE、READ_CALL_LOG、
    //               WRITE_CALL_LOG、ADD_VOICEMAIL、USE_SIP、PROCESS_OUTGOING_CALLS
    public final static String PHONE = Manifest.permission.CALL_PHONE;

    // (7) SENSORS - BODY_SENSORS，这个权限API >= 20新增的
    public final static String SENSORS = Manifest.permission.BODY_SENSORS;

    // (8) 短信权限 - SEND_SMS、RECEIVE_SMS、READ_SMS、RECEIVE_WAP_PUSH、RECEIVE_MMS
    public final static String SMS = Manifest.permission.READ_SMS;

    // (9) 文件读写权限 - READ_EXTERNAL_STORAGE、WRITE_EXTERNAL_STORAGE
    public final static String STORAGE = Manifest.permission.WRITE_EXTERNAL_STORAGE;


    private Context context;
    private int requestCode;
    private List<String> requestPermissionList;

    private OnRequestResultListener listener;


    public PermissionUtils(Context context, int requestCode,
                          List<String> requestPermissionList, OnRequestResultListener listener) {
        this.context = context;
        this.requestCode = requestCode;
        this.requestPermissionList = requestPermissionList;
        this.listener = listener;
    }


    /**
     * 检测权限并发起权限请求
     *
     * 废弃原因：
     * 按照正常的逻辑，先调用context.checkSelfPermission方法去检测用户是否允许了这个权限，如果没有允许权限，
     * 加入权限到一个列表中，然后一次请求多个权限，但是在使用的时候，测试到魅族和部分小米手机调用context.checkSelfPermission总是返回true，
     * 魅族手是普通权限时全部开放的，所以返回true，为了更好的兼容，使用support-core-utils里面的PermissionChecker类来检测
     *
     */
    private boolean checkAndRequestPermissions() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            List<String> needRequestList = new ArrayList<>();
            for (String permission : requestPermissionList) {
                int result = PermissionChecker.checkSelfPermission(context, permission);
                if (result != PackageManager.PERMISSION_GRANTED) {
                    needRequestList.add(permission);
                }
            }

            if (needRequestList.size() > 0) {
                String[] params = needRequestList.toArray(new String[needRequestList.size()]);
                ActivityCompat.requestPermissions((Activity) context, params, requestCode);
                return false;
            } else {
                return true;
            }
        } else {
            return true;
        }
    }


    /**
     * 请求权限
     *
     * 注意的地方：
     * 在模拟器上调用了requestPermissions方法后会弹出权限确认对话框，这是正常的逻辑行为。
     * 但是在魅族和部分小米手机上只有在Activity的onRequestPermissionsResult方法里面执行和请求权限相关的服务才会弹出对话框，
     * 例如，这里请求了相机的权限，只有在onRequestPermissionsResult里面执行一个和相机权限相关的动作(如跳转到扫描页面)才会染出对话框，
     * 否则不会弹出对话框，这一点要注意。魅族手机已确认是这样的。
     *
     */
    public void doTask() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (checkAndRequestPermissions()) {
                listener.success();
            }
        }
    }


    /**
     * 检测用户是否同意了相关权限
     *
     * @param requestCode  请求码
     * @param permissions  权限集
     * @param grantResults 结果集
     */
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (this.requestCode != requestCode) return;
            for (int i = 0; i < grantResults.length; i++) {
                int result = grantResults[i];

                if (result != PackageManager.PERMISSION_GRANTED) {
                    if (((Activity) context).shouldShowRequestPermissionRationale(permissions[i])) {
                        listener.failed(false); // 用户点击了拒绝
                    } else {
                        listener.failed(true);  // 用户点击了不再询问
                    }
                    return;
                }
            }
        }
        listener.success();
    }



    public interface OnRequestResultListener {
        void success();
        void failed(boolean neverAsk);
    }
}
```

# 使用方法

```java
public class CommodityAdd extends BaseActivity {
    private PermissionUtils selectImagePermission;  // 权限辅助类

    // ...

    // 选择图片权限初始化
    selectImagePermission = new PermissionUtils(this, 10000,
        Arrays.asList(PermissionUtils.CAMERA, PermissionUtils.STORAGE),
        new PermissionUtils.OnRequestResultListener() {
            @Override
            public void success() {
                // do task
            }

            @Override
            public void failed(boolean neverAsk) {
                if (neverAsk) {
                    toast("需要读写文件和使用相机的权限，请在设置页面确认这两项权限都已经开启");
                }
            }
        });

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        selectImagePermission.onRequestPermissionsResult(requestCode, permissions, grantResults);
   }
}
```

最后在点击事件需要请求这一权限的地方执行 selectImagePermission.doTask() 即可调用成功。