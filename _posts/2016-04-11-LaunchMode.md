# # Activity启动模式-LaunchMode

标签（空格分隔）： LaunchMode

---
##四种启动模式
### Standard
**`标准模式`**,每当有一次Intent请求，就会创建一个新的Activity实例。
#####  1.Android 5.0 之前
* 同一应用内
	新生成的Activity，放入发送Intent者Task的栈顶。
```
TaskRecord{537925a8 #42 A com.zlq.lmt U 0}
      Run #3: ActivityRecord{538314d0 com.zlq.lmt/.StandardActivity}
      Run #2: ActivityRecord{5385a7c4 com.zlq.lmt/.StandardActivity}
      Run #1: ActivityRecord{53760908 com.zlq.lmt/.MainActivity}
```
* 跨应用启动
	 新生成的Activity，放入发送Intent者Task的栈的栈顶（尽管他们属于不同的程序，还是会放入调用者程序的栈内）。
```
TaskRecord{537df318 #52 A com.zlq.bbb U 0}
      Run #2: ActivityRecord{537a889c com.zlq.lmt/.StandardActivity}
      Run #1: ActivityRecord{537a4a5c com.zlq.bbb/.MainActivityB}
```
这时，我们打开任务管理器（最近任务按钮）。会发现最近任务中现实的应用名为B应用，展示的界面却是A应用的StandardActivity(因为其位于Task栈顶)。
![跨应用启动Standard Activity][1]
	 
#####	2.Android 5.0 之后
*  同一应用内
	 与Android 5.0之前保持一致
*  跨应用启动
    经检验与Android5.0之前保持一致。
    Android6.0上依然没改变。参考资料`深入讲解Android中Activity launchMode`内容或许有误 。

##### 3.使用场景
standard这种启动模式适合于撰写邮件Activity或者社交网络消息发布Activity。如果你想为每一个intent创建一个Activity处理，那么就是用standard这种模式。

### SingleTop

**`栈顶复用模式`**. SingleTop其实和Standard几乎一样，使用SingleTop的Activity也可以创建很多个实例。唯一不同的就是，如果调用的目标Activity已经位于调用者的Task的栈顶，则不创建新实例，而是使用当前的这个Activity实例，并调用这个实例的onNewIntent方法。 
在singleTop这种模式下，我们需要处理应用这个模式的Activity的onCreate和onNewIntent两个方法，确保逻辑正常。
```
  TaskRecord{537925a8 #42 A com.zlq.lmt U 0}
      Run #4: ActivityRecord{537e3114 com.zlq.lmt/.SingleTopActivity}
      Run #3: ActivityRecord{537dfe7c com.zlq.lmt/.StandardActivity}
      Run #2: ActivityRecord{53770808 com.zlq.lmt/.SingleTopActivity}
      Run #1: ActivityRecord{53760908 com.zlq.lmt/.MainActivity}
```
栈顶无法像Standard模式一样，同事存在两个，但是整个Task列表中间隔存在多个是可以的。

### SingleTask
**`栈内复用模式`**.使用singleTask启动模式的Activity在一个应用Task中只会存在一个实例。如果这个实例已经存在，intent就会通过onNewIntent传递到这个Activity，即多次调用不会创建新实例。否则新的Activity实例被创建。
情况包含以下几种：
##### 同一程序内：
1. 任务栈不存在, 初次启动SingleTask实例, 会创建任务栈和实例.
	Google在singleTask的[文档](http://developer.android.com/guide/components/tasks-and-back-stack.html)有这样一段描述：
 >The system creates a new task and instantiates the activity at the root of the new task.
 
	意思为 系统会创建一个新的Task，并创建Activity实例放入这个新的Task的底部。然而实际并非如此，在我的例子中，singleTask Activity并创建并放入了调用者所在的Task，而不是放入新的Task:
```
TaskRecord{5378ff88 #44 A com.zlq.lmt U 0}
      Run #3: ActivityRecord{537e2ff0 com.zlq.lmt/.SingleTaskActivity}
      Run #2: ActivityRecord{537de2ec com.zlq.lmt/.StandardActivity}
      Run #1: ActivityRecord{53788be0 com.zlq.lmt/.MainActivity}
```

怎样才能符合文档中所描述的情况呢？那就是 `taskAffinity`属性和singleTask启动模式配合使用.
```xml
	<activity
	    android:name=".SingleTaskWithTaskAffinityActivity"
	    android:label="SingleTaskWithTaskAffinityActivity"
	    android:launchMode="singleTask"
	    android:taskAffinity="com.zlq.new">
	</activity>
	
```
此时再执行同样的操作，栈内的情况：
```
TaskRecord{53778428 #45 A com.zlq.new U 0}
      Run #3: ActivityRecord{537db410 com.zlq.lmt/.SingleTaskWithTaskAffinityActivity}
TaskRecord{5378ff88 #44 A com.zlq.lmt U 0}
      Run #2: ActivityRecord{53760908 com.zlq.lmt/.StandardActivity}
      Run #1: ActivityRecord{53788be0 com.zlq.lmt/.MainActivity}
```

	其实，**把启动模式设置为singleTask，framework在启动该activity时只会把它标示为可在一个新任务中启动，至于是否在一个新任务中启动，还要受其他条件的限制。**使用`taskAffinity`属性会指定新的Activity所属栈，可与SingleTask配合使用, 对Standard模式无效.新任务栈是com.zlq.new.
2. 任务栈存在, 初次启动SingleTask实例, Task栈中不存在singleTask Activity的实例。那么就需要创建这个Activity的实例，并且将这个实例放入和调用者相同的Task中并位于栈顶。与Standard模式相同.
3.  任务栈相同,如果singleTask Activity实例已然存在,再次启动SingleTask实例, 那么在Activity回退栈中，所有位于该Activity上面的Activity实例都将被销毁掉（销毁过程会调用Activity生命周期回调），这样使得singleTask Activity实例位于栈顶（具有clearTop的效果）。与此同时，Intent会通过onNewIntent传递到这个SingleTask Activity实例。 并清除其上面实例, 具有clearTop的效果.最终，singleTask Activity实例会位于栈顶。
4.  任务栈不同, 再次启动SingleTask实例, 会导致任务栈切换, 后台置于前台.

##### 跨应用之间：
1.  任务栈不存在, 初次启动SingleTask实例, 会创建一个新的任务栈，然后创建SingleTask Activity的实例，将其放入新的Task中。Task变化如下。
从
```
TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #0: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
```
变为：
```
TaskRecord{5c70a93 #17 A=com.zlq.lmt U=0 sz=1}
        Run #1: ActivityRecord{4cd8b0f u0 com.zlq.lmt/.SingleTaskActivity t17}
TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #0: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
```
最近任务变化：
![此处输入图片的描述][2]
变为：
![此处输入图片的描述][3]
2. 任务栈存在, 初次启动SingleTask实例, Task栈中不存在singleTask Activity的实例。
如果singleTask Activity所在的应用进程存在，但是singleTask Activity实例不存在，那么从别的应用启动这个Activity，新的Activity实例会被创建，并放入到所属进程所在的Task中，并位于栈顶位置。
从
```
TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #1: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=1}
        Run #0: ActivityRecord{f0eba63 u0 com.zlq.lmt/.MainActivity t18}
```
↓变为↓：
```
TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #2: ActivityRecord{73d091c u0 com.zlq.lmt/.SingleTaskActivity t18}
TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #1: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #0: ActivityRecord{f0eba63 u0 com.zlq.lmt/.MainActivity t18}
```
3. 如果singleTask Activity实例存在，从其他程序被启动，那么这个Activity所在的Task会被移到顶部，并且在这个Task中，位于singleTask Activity实例之上的所有Activity将会被正常销毁掉。如果我们按返回键，那么我们首先会回退到这个Task中的其他Activity，直到当前Task的Activity回退栈为空时，才会返回到调用者的Task。
从
```
Running activities (most recent first):
      TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #4: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
      TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=4}
        Run #3: ActivityRecord{fa7aae9 u0 com.zlq.lmt/.StandardActivity t18}
        Run #2: ActivityRecord{dfd9a3b u0 com.zlq.lmt/.StandardActivity t18}
        Run #1: ActivityRecord{660ce3c u0 com.zlq.lmt/.SingleTaskActivity t18}
        Run #0: ActivityRecord{f0eba63 u0 com.zlq.lmt/.MainActivity t18}
```
↓变为↓：
```
Running activities (most recent first):
      TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #2: ActivityRecord{660ce3c u0 com.zlq.lmt/.SingleTaskActivity t18}
      TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #1: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
      TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #0: ActivityRecord{f0eba63 u0 com.zlq.lmt/.MainActivity t18}
```
可以看到，TASK ID为`#18` 的任务栈已经从原来的`4`个变为最终的`1+1`个。


#####  使用场景：
该模式的使用场景多类似于邮件客户端的收件箱或者社交应用的时间线Activity。上述两种场景需要对应的Activity只保持一个实例即可，但是也要谨慎使用这种模式，因为它可以在用户未感知的情况下销毁掉其他Activity。


### SingleInstance
**`单实例模式`**启动时, 系统会为其创造一个单独的任务栈, 以后每次使用, 都会使用这个单例, 直到其被销毁, 属于真正的单例模式.singleTask差不多，唯一不同的就是存放singleInstance Activity实例的Task只能存放一个该模式的Activity实例，不能有任何其他的Activity。
虽然是两个task，但是在系统的任务管理器中，却始终显示一个，即位于顶部的Task中。

## 相关知识点
### 查看当前任务栈:
    adb shell dumpsys activity | sed -n -e '/Stack #/p' -e '/Running activities/,/Run #0/p'
输出的结果如：
```
Running activities (most recent first):
      TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #2: ActivityRecord{660ce3c u0 com.zlq.lmt/.SingleTaskActivity t18}
      TaskRecord{5bf28 #16 A=com.zlq.bbb U=0 sz=1}
        Run #1: ActivityRecord{f4a1b15 u0 com.zlq.bbb/.MainActivityB t16}
      TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}
        Run #0: ActivityRecord{f0eba63 u0 com.zlq.lmt/.MainActivity t18}
```
*拿`TaskRecord{65dfdf0 #18 A=com.zlq.lmt U=0 sz=2}`为例*
以TaskRecord开头的（如）为一组 TaskRecord记录，#18为Task的ID，A=包名，sz为该Task的Activity数量。
以Run #开头的为一个ActivityRecord记录。其中也包含了包名、类名、TASK ID等信息。

>其中以Task ID为一个任务栈的唯一标识，ID相同的TaskRecord属于同一个任务栈（可以理解为同一应用）。
### 对startActivityForResult的影响:

startActivityForResult 不同于 startActivity, 在使用 startActivityForResult 时不管LaunchMode设置为哪种模式，都会在调用者Task栈中新建实例以正确地返回数据。在栈中的展现形式均与Standard相同（可生成多份连续的实例）。
1. SingleTop,当其使用startActivityForResult时表现和Standard启动模式时完全相同
2. SingleTask，当不定义 taskAffinity 属性时使用startActivityForResult和Standard启动模式时表现完全相同。当定义了taskAffinity 属性后，变现将和下面第3条表现一致。
3. SingleInstance，无法通过startActivity创建自己(无论当前所属哪个栈)。startActivityForResult 随意在当前栈（传入者所在栈）新建实例。
```
 Running activities (most recent first):
      TaskRecord{ef9f20b #33 A=com.zlq.lmt U=0 sz=4}
        Run #3: ActivityRecord{34f60b1 u0 com.zlq.lmt/.SingleTaskActivity t33} *
        Run #2: ActivityRecord{914547d u0 com.zlq.lmt/.SingleTaskActivity t33} *
        Run #1: ActivityRecord{a1f3e09 u0 com.zlq.lmt/.SingleTaskActivity t33}
        Run #0: ActivityRecord{f7db1c u0 com.zlq.lmt/.MainActivity t33}
```

是面代码片段中，加了`*`标的表示使用startActivity无法建立，是 使用`startActivityForResult` 建立的Activity。
由此可知, 因为startActivityForResult需要返回值, 会保留实例, 部分覆盖单例效果.

>注意: 4.x版本通过startActivityForResult启动singleTask, 无法正常获取返回值, [参考](http://stackoverflow.com/questions/8960072/onactivityresult-with-launchmode-singletask).
5.x以上版本修复此问题, 考虑兼容性, 不推荐使用startActivityForResult和singleTask.
###碎碎念
以上均为参考资料和自己实践验证所得结果。大家可去下载我的验证代码:[GitHub][4]
###参考资料链接
[深入讲解Android中Activity launchMode][5]
[分析 Activity 的启动模式][6]
[Android中Activity四种启动模式和taskAffinity属性详解][7]


  [1]: http://7xkrut.com1.z1.glb.clouddn.com/5901D7F1-186C-41A5-B71B-87E04833F29D.png
  [2]: http://7xkrut.com1.z1.glb.clouddn.com/single_task_1_apps.png
  [3]: http://7xkrut.com1.z1.glb.clouddn.com/single_task_2_apps.png
  [4]: https://github.com/StormGens/LaunchModeTest
  [5]: http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/index.html
  [6]: http://www.wangchenlong.org/2016/02/23/1603/234-activity-launch-mode/
  [7]: http://blog.csdn.net/zhangjg_blog/article/details/10923643