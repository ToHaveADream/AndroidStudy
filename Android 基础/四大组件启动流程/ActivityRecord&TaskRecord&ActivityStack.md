# ActivityRecord、TaskRecord、ActivityStack 以及Activity 启动模式详解

## 1、简介

来张简单的关系图：

![关系图](/picture/01.webp)

- 一个 `ActivityRecord` 对应一个 `Activity`, 保存了一个 Activity 的所有信息；但是一个 Activity 可能会有多个 ActivityRecord，因为 Activity 可以被多次启动，这个主要取决于其启动模式
- 一个 TaskRecord 由一个或者多个 ActivityRecord 组成，这就是我们所说的任务栈，具有后进先出的特点
- ActivityStack 则是用来管理 TaskRecord 的，包含了多个 TaskRecord

下面进入详细的代码分析：

## 代码分析

### 2.1 ActivityRecord

ActivityRecord,源码中的注释介绍：An entry in the history stack, representing an activity. 翻译：历史栈中的一个条目，代表一个 Activity。

```java
    frameworks/base/services/core/java/com/android/server/am/ActivityRecord.java
    
    final class ActivityRecord extends ConfigurationContainer implements AppWindowContainerListener {

        final ActivityManagerService service; // owner
        final IApplicationToken.Stub appToken; // window manager token
        AppWindowContainerController mWindowContainerController;
        final ActivityInfo info; // all about me
        final ApplicationInfo appInfo; // information about activity's app
        
        //省略其他成员变量
        
        //ActivityRecord所在的TaskRecord
        private TaskRecord task;        // the task this is in.
        
        //构造方法，需要传递大量信息
        ActivityRecord(ActivityManagerService _service, ProcessRecord _caller, int _launchedFromPid,
                       int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
                       ActivityInfo aInfo, Configuration _configuration,
                       com.android.server.am.ActivityRecord _resultTo, String _resultWho, int _reqCode,
                       boolean _componentSpecified, boolean _rootVoiceInteraction,
                       ActivityStackSupervisor supervisor, ActivityOptions options,
                       com.android.server.am.ActivityRecord sourceRecord) {
        
        }
    }
```

- 实际上，ActivityRecord 中存在着大量的成员变量，包含了一个 Activity 的所有信息。
- ActivityRecord 中的成员变量 task 表示其所在的 TaskRecord, 由此可以看出：ActivityRecord 与 TaskRecord 建立了联系

`startActivity` 时会创建一个 ActivityRecord:

```java
    frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java

    class ActivityStarter {
        private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
                                  String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
                                  IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                  IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
                                  String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
                                  ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
                                  com.android.server.am.ActivityRecord[] outActivity, TaskRecord inTask) {

            //其他代码略
            
            ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                    callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                    resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                    mSupervisor, options, sourceRecord);
                    
            //其他代码略
        }
    }
```

### 2.2 TaskRecord

`TaskRecord`,内部维护一个 `ArrayList<ActivityRecord>` 用来保存 `ActivityRecord`

```java
    frameworks/base/services/core/java/com/android/server/am/TaskRecord.java
    
    final class TaskRecord extends ConfigurationContainer implements TaskWindowContainerListener {
        final int taskId;       //任务ID
        final ArrayList<ActivityRecord> mActivities;   //使用一个ArrayList来保存所有的ActivityRecord
        private ActivityStack mStack;   //TaskRecord所在的ActivityStack
        
        //构造方法
        TaskRecord(ActivityManagerService service, int _taskId, ActivityInfo info, Intent _intent,
                   IVoiceInteractionSession _voiceSession, IVoiceInteractor _voiceInteractor, int type) {
            
        }
        
        //添加Activity到顶部
        void addActivityToTop(com.android.server.am.ActivityRecord r) {
            addActivityAtIndex(mActivities.size(), r);
        }
        
        //添加Activity到指定的索引位置
        void addActivityAtIndex(int index, ActivityRecord r) {
            //...

            r.setTask(this);//为ActivityRecord设置TaskRecord，就是这里建立的联系

            //...
            
            index = Math.min(size, index);
            mActivities.add(index, r);//添加到mActivities
            
            //...
        }

        //其他代码略
    }
```

- 可以看到 TaskRecord 中使用了一个 ArrayList 来保存所有的 ActivtyRecord
- 同样，TaskRecord 中的 mStack 表示其所在的 ActivityStack

`startActivity()` 时也会创建一个 `TaskRecord`:

```java
    frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java
    
    class ActivityStarter {

        private int setTaskFromReuseOrCreateNewTask(TaskRecord taskToAffiliate, int preferredLaunchStackId, ActivityStack topStack) {
            mTargetStack = computeStackFocus(mStartActivity, true, mLaunchBounds, mLaunchFlags, mOptions);

            if (mReuseTask == null) {
                //创建一个createTaskRecord，实际上是调用ActivityStack里面的createTaskRecord（）方法，ActivityStack下面会讲到
                final TaskRecord task = mTargetStack.createTaskRecord(
                        mSupervisor.getNextTaskIdForUserLocked(mStartActivity.userId),
                        mNewTaskInfo != null ? mNewTaskInfo : mStartActivity.info,
                        mNewTaskIntent != null ? mNewTaskIntent : mIntent, mVoiceSession,
                        mVoiceInteractor, !mLaunchTaskBehind /* toTop */, mStartActivity.mActivityType);

                //其他代码略
            }
        }
    }
```

### 2.3 ActivityStack

`ActivityStack`,内部维护了一个 `ArrayList<TaskRecord>`,用来管理 `TaskRecord`.

```java
    frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
    
    class ActivityStack<T extends StackWindowController> extends ConfigurationContainer implements StackWindowListener {

        private final ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();//使用一个ArrayList来保存TaskRecord

        final int mStackId;

        protected final ActivityStackSupervisor mStackSupervisor;//持有一个ActivityStackSupervisor，所有的运行中的ActivityStacks都通过它来进行管理
        
        //构造方法
        ActivityStack(ActivityStackSupervisor.ActivityDisplay display, int stackId,
                      ActivityStackSupervisor supervisor, RecentTasks recentTasks, boolean onTop) {

        }
        
        TaskRecord createTaskRecord(int taskId, ActivityInfo info, Intent intent,
                                    IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                    boolean toTop, int type) {
                                    
            //创建一个task
            TaskRecord task = new TaskRecord(mService, taskId, info, intent, voiceSession, voiceInteractor, type);
            
            //将task添加到ActivityStack中去
            addTask(task, toTop, "createTaskRecord");

            //其他代码略

            return task;
        }
        
        //添加Task
        void addTask(final TaskRecord task, final boolean toTop, String reason) {

            addTask(task, toTop ? MAX_VALUE : 0, true /* schedulePictureInPictureModeChange */, reason);

            //其他代码略
        }

        //添加Task到指定位置
        void addTask(final TaskRecord task, int position, boolean schedulePictureInPictureModeChange,
                     String reason) {
            mTaskHistory.remove(task);//若存在，先移除
            
            //...
            
            mTaskHistory.add(position, task);//添加task到mTaskHistory
            task.setStack(this);//为TaskRecord设置ActivityStack

            //...
        }
        
        //其他代码略
    }
```

- 可以看到 `ActivityStack` 使用了一个 `ArrayList` 来保存 `TaskRecord`
- 另外，`ActivityStack` 中还持有 `ActivityStackSupervisor` 对象，这个是用来管理 `ActivityStacks`的

`ActivityStack` 是由 `ActivityStackSupervisor` 来创建的，实际 `ActivityStackSupervisor` 就是用来管理 `ActivityStack` 的，继续看下面的 `ActivityStackSupervisor` 分析。

### 2.4 ActivityStackSupervisor

`ActivityStackSupervisor`,顾名思义，就是用来管理 `ActivityStack` 的。

```java
    frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
    
    public class ActivityStackSupervisor extends ConfigurationContainer implements DisplayListener {

        ActivityStack mHomeStack;//管理的是Launcher相关的任务

        ActivityStack mFocusedStack;//管理非Launcher相关的任务
        
        //创建ActivityStack
        ActivityStack createStack(int stackId, ActivityStackSupervisor.ActivityDisplay display, boolean onTop) {
            switch (stackId) {
                case PINNED_STACK_ID:
                    //PinnedActivityStack是ActivityStack的子类
                    return new PinnedActivityStack(display, stackId, this, mRecentTasks, onTop);
                default:
                    //创建一个ActivityStack
                    return new ActivityStack(display, stackId, this, mRecentTasks, onTop);
            }
        }
    }
```

- `ActivityStackSupervisor` 内部有两个不同的 `ActivityStack` 对象：`mHomeTask`、`mFocusedStack`, 用来管理不同的任务
- `ActivityStackSupervisor` 内部包含了创建 `ActivityStack` 对象的方法

AMS 初始化时会创建一个 `ActivityStackSupervisor` 对象

### 2.5 总结

![](/picture/02.webp)

## 场景分析

下面通过启动 Activity 的代码来分析一下：

### 3.1 桌面

首先，我们看下处于桌面时的状态，运行命令：

```
adb shell dumpsys activitys
```

结果如下：

```java
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
Display #0 (activities from top to bottom):
  Stack #0:
  
  //中间省略其他...
  
    Task id #102
    
  //中间省略其他...
  
      TaskRecord{446ae9e #102 I=com.google.android.apps.nexuslauncher/.NexusLauncherActivity U=0 StackId=0 sz=1}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.google.android.apps.nexuslauncher/.NexusLauncherActivity }
        Hist #0: ActivityRecord{54fa22 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t102}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.google.android.apps.nexuslauncher/.NexusLauncherActivity }
          ProcessRecord{19c7c43 2203:com.google.android.apps.nexuslauncher/u0a22}

    Running activities (most recent first):
      TaskRecord{446ae9e #102 I=com.google.android.apps.nexuslauncher/.NexusLauncherActivity U=0 StackId=0 sz=1}
        Run #0: ActivityRecord{54fa22 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t102}

    mResumedActivity: ActivityRecord{54fa22 u0 com.google.android.apps.nexuslauncher/.NexusLauncherActivity t102}
    
//省略其他
```

这里的`Stack #0`就是`ActivityStackSupervisor`中的`mHomeStack`，`mHomeStack`管理的是Launcher相关的任务

![](/picture/03.webp)

### 3.2 从桌面启动一个 Activity

> 从桌面启动一个 App，然后运行上面的命令。

从桌面点击图标启动一个 `Activity`,可以看到，会多了一个 `Stack#1` ,这个 `Stack#1` 就是 `ActivityStackSupervisor` 中的 `mFocusedTask`, 其负责管理的是非 Launcher 相关的任务。同时也会创建一个新的 ActivityRecord 和 TaskRecord, ActivityRecord 放到 TaskRecord 中，TaskRecord 则放到 mFocusedStack 中。

![](/picture/04.webp)

### 3.3 默认模式从 A 启动 B

然后，我们从 Activity 中启动一个 BActivity, 可以看到会创建一个新的 ActivityRecord 然后放到已有的 TaskRecord 栈顶

![](/picture/05.webp)

### 3.4 从 A 启动 B 创建新栈

如果我们想启动的 BActivity 也在一个新的栈中呢？我们可以用 `singleInstance` 的方式来启动 BActivity。这种方式会创建一个新的 ActivityRecord 和 TaskRecord,把 ActivityRecord 放到新的 TaskRecord 中去。

![](/picture/06.webp)

## 启动模式

### 4.1 standard

现在有个A Activity,我们在A上面启动B，再然后在B上面启动A，其过程如图所示

![](/picture/07.webp)

### 5.2 SingleTop

> 如果要启动的 Activity 已经在栈顶，则不会重新创建 Activity，只会调用该 Activity的 onNewIntent 方法。如果要启动的 Activity的 Activity 不在栈顶，则会重新创建该 Activity的实例

现在有个A Activity,我们在A以`standerd`模式上面启动B，然后在B上面以`singleTop`模式启动A，其过程如图所示，这里会新创建一个A实例：

![](/picture/08.webp)

如果在B上面以`singleTop`模式启动B的话，则不会重新创建B，只会调用`onNewIntent()`方法，其过程如图所示：

![](/picture/09.webp)

### 5.3 singleTask

> 如果要启动的Activity已经存在于它想要归属的栈中，那么不会创建该Activity实例，将栈中位于该Activity上的所有的Activity出栈，同时该Activity的`onNewIntent()`方法会被调用。
>  如果要启动的Activity不存在于它想要归属的栈中，并且该栈存在，则会创建该Activity的实例。
>  如果要启动的Activity想要归属的栈不存在，则首先要创建一个新栈，然后创建该Activity实例并压入到新栈中。

比如：现在有个A Activity，我们在A以`standerd`模式上面启动B，然后在B上面以`singleTask`模式启动A，其过程如图所示：

![](/picture/10.webp)

### 5.4 singleInstance

