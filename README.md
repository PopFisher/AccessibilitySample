# AccessibilitySample
本文主要介绍辅助功能的使用

1. 辅助功能基本原理
2. 辅助功能基本配置和框架搭建
3. 辅助功能实战解析

## 辅助功能基本原理
&emsp;&emsp;辅助功能（AccessibilityService）其实是一个Android系统提供给的一种服务，本身是继承Service类的。这个服务提供了增强的用户界面，旨在帮助残障人士或者可能暂时无法与设备充分交互的人们。

&emsp;&emsp;从开发者的角度看，其实就是提供两种功能：查找界面元素，实现模拟点击。实现一个辅助功能服务要求继承AccessibilityService类并实现它的抽象方法。自定义一个服务类AccessibilitySampleService（这个命名可以随意），继承系统的AccessibilityService并覆写onAccessibilityEvent和onInterrupt方法。编写好服务类之后，在系统配置文件（AndroidManifest.xml）中注册服务。完成前面两个步骤就完成了基本发辅助功能服务注册与配置，具体的功能实现需要在onAccessibilityEvent中完成，根据onAccessibilityEvent回调方法传递过来的AccessibilityEvent对象可以对事件进行过滤，结合AccessibilitySampleService本身提供的查找节点与模拟点击相关的接口即可实现权限节点的查找与点击。
![](/docpic/accessibility.png "辅助功能类")

## 辅助功能基本配置和框架搭建
### 创建自定义辅助功能服务类
    import android.accessibilityservice.AccessibilityService;
	import android.view.accessibility.AccessibilityEvent;
	
	import com.accessibility.utils.AccessibilityLog;
	public class AccessibilitySampleService extends AccessibilityService {

	    @Override
	    protected void onServiceConnected() {
	        super.onServiceConnected();
	    }
	
	    @Override
	    public void onAccessibilityEvent(AccessibilityEvent event) {
	        // 此方法是在主线程中回调过来的，所以消息是阻塞执行的
	        // 获取包名
	        String pkgName = event.getPackageName().toString();
	        int eventType = event.getEventType();
			// AccessibilityOperator封装了辅助功能的界面查找与模拟点击事件等操作
	        AccessibilityOperator.getInstance().updateEvent(this, event);
	        AccessibilityLog.printLog("eventType: " + eventType + " pkgName: " + pkgName);
	        switch (eventType) {
	            case AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED:
	                break;
	        }
	    }
	
	    @Override
	    public void onInterrupt() {
	
	    }
	}
### 注册辅助功能服务
	// 注册辅助功能服务
	<service android:name=".AccessibilitySampleService"
		android:label="@string/accessibility_tip"
		android:exported="true"
		android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
		android:process=":BackgroundService">
		<intent-filter>
			<action android:name="android.accessibilityservice.AccessibilityService" />
		</intent-filter>
		// 通过xml文件完成辅助功能相关配置，也可以在onServiceConnected中动态配置
		<meta-data
			android:name="android.accessibilityservice"
			android:resource="@xml/accessibility_config"/>
	</service>

上面android:label="@string/accessibility_tip"是配置此辅助功能服务在系统辅助功能页面里面显示的名字。

**accessibility_config文件内容如下：**

	<?xml version="1.0" encoding="utf-8"?>
	<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
	    android:accessibilityEventTypes="typeAllMask"
	    android:accessibilityFeedbackType="feedbackGeneric"
	    android:canRetrieveWindowContent="true"
	    android:description="@string/accessibility_desc"
	    android:notificationTimeout="100" />

### 跳转到系统辅助功能页面，开启辅助功能服务
&emsp;&emsp;完成上面配置之后，辅助功能服务就注册成功了，在系统辅助功能页面就能找到这个服务，但是默认是关闭的，也就是说，这个服务要开始为我们服务，还需要去系统界面开启那个开关。下面是跳转到辅助功能页面的代码，跳转过去之后，手动点击开关按钮。开关打开之后，这个辅助功能服务就开始工作了，系统开始回调onAccessibilityEvent方法。我们可以在onAccessibilityEvent方法中处理查找节点与点击操作。

	public class OpenAccessibilitySettingHelper {
	    private static final String ACTION = "action";
	    private static final String ACTION_START_ACCESSIBILITY_SETTING = "action_start_accessibility_setting";
	
	    public static void jumpToSettingPage(Context context) {
	        try {
	            Intent intent = new Intent(context,  AccessibilityOpenHelperActivity.class);
	            intent.putExtra(ACTION, ACTION_START_ACCESSIBILITY_SETTING);
	            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	            context.startActivity(intent);
	        } catch (Exception ignore) {}
	    }
	}

**下图是小米手机开启辅助功能的界面**

![](/docpic/sys_accessibility_page.jpg "小米上叫做无障碍")
## 辅助功能实战解析

### 实现界面自动点击操作，动画有点模糊，将就看吧

![](/docpic/accessibility_op.gif "操作演示")

### 界面节点查找与模拟点击
&emsp;&emsp;AccessibilityOperator封装了辅助功能的界面查找与模拟点击事件等操作，下面介绍几个关键的技术点。

**界面节点查找操作**

&emsp;&emsp;AccessibilityNodeInfo提供两种查找View节点的方法


**1. 根据View的ID精确查找，但是要求SDK_INT >= 18才能用**

	 /**
     * 根据View的ID搜索符合条件的节点,精确搜索方式;
     * 这个只适用于自己写的界面，因为ID可能重复
     * api要求18及以上
     * @param viewId
     */
    public List<AccessibilityNodeInfo> findNodesById(String viewId) {
        AccessibilityNodeInfo nodeInfo = getRootNodeInfo();
        if (nodeInfo != null) {
            if (Build.VERSION.SDK_INT >= 18) {
                return nodeInfo.findAccessibilityNodeInfosByViewId(viewId);
            }
        }
        return null;
    }

**2. 根据View的Text文本进行模糊查找**

	/**
     * 根据Text搜索所有符合条件的节点, 模糊搜索方式
     */
    public List<AccessibilityNodeInfo> findNodesByText(String text) {
        AccessibilityNodeInfo nodeInfo = getRootNodeInfo();
        if (nodeInfo != null) {
           return nodeInfo.findAccessibilityNodeInfosByText(text);
        }
        return null;
    }

**模拟界面操作**
**普通的View事件模拟（ACTION_CLICK）**

	private boolean performClick(List<AccessibilityNodeInfo> nodeInfos) {
        if (nodeInfos != null && !nodeInfos.isEmpty()) {
            AccessibilityNodeInfo node;
            for (int i = 0; i < nodeInfos.size(); i++) {
                node = nodeInfos.get(i);
                // 获得点击View的类型
                AccessibilityLog.printLog("View类型：" + node.getClassName());
                // 进行模拟点击
                if (node.isEnabled()) {
                    return node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                }
            }
        }
        return false;
    }

**全局事件模拟（返回键：AccessibilityService.GLOBAL_ACTION_BACK）**

	public boolean clickBackKey() {
        return performGlobalAction(AccessibilityService.GLOBAL_ACTION_BACK);
    }

    private boolean performGlobalAction(int action) {
        return mAccessibilityService.performGlobalAction(action);
    }