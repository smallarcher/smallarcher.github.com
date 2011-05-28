---
layout: post
title: Android笔记--Activity
category: android
---

# Activities
* Creating an Activity
    * Implementing a user interface
    * Declaring the activity in the manifest
* Starting an Activity
    * Starting an Activity for Result
* Shutting Down an Activity
* Managing the Activity Lifecycle
    * Implementing the lifecycle callbacks
    * Saving activity state
    * Handling configuration changes
    * Coordinating activities


通常有一个activity被指定为main，第一次启动app时显示。每次一个新的activity启动，它就被压入stack中，并获得用户焦点，之前的activity就停止。，当***BACK***键被按时，从stack中弹出并destroy，之前的activity resume。
当activity由于一个新的activity启动而停止时，系统通过activity的回调通知。例如，当停止时，activity需要释放large objects，比如网络或者数据库链接。当resume时，重新获取，并resume actions that were interrupted。

## Creating an Activity
### Implementing a user interface
### Declaring the activity in the manifest
## Starting an Activity
### Starting an Activity for Result
## Shutting Down an Activity
可以通过调用**finish**结束activity。也可以通过调用**finishActivity**结束你之前启动的activity。
但是，尽量不要显示的结束activity，影响用户体验，系统会管理的。除非你确保你不想用户回到这个activity。
## Managing the Activity Lifecycle
activity本质上有3种状态

* Resumed
activity在前端获取用户焦点。也就是running
* Paused
另外一个activity在前端获取焦点，但是这个也还是visible。也就是说，另一个activity is visible在这个上面，并没有完全覆盖屏幕。这种activity completely avlie（保持在内存中，包括他所有状态和成员信息，并且attached to 窗口管理器），但是在内存极低的情况下也可能被kill
* Stopped
activity完全被遮盖，此时处于background。Also still allive（保持在内存中，包括有所状态和成员信息，但是not attached to 窗口管理器）。然而，此时，对用户已经不可见，当其他地方需要内存的时候，会被kill。

如果activity被paused或者stopped，系统可以调用finish或者直接kill进程。当activity重新打开时，重头开始。

### Implementing the lifecycle callbacks
***onCreate, onStart, onPause, onStop, onDestroy***
这些方法，定义了整个生命周期。

* ***entire lifetime***是从onCreate到onDestroy。在onCreate中设置全局状态（如布局），在onDestroy中释放资源。例如有一个线程在下载网络资源，可能在onCreate中启动，在onDestroy中停止。
* ***visible lifetime***在onStart和onStop之间。这时用户可以在屏幕上看到activity。例如，onStop在一个新的activity启动，并且这个不再可见时调用。在onStart和onStop之间，你可以维护你需要的资源。例如，你可以在onStart中注册一个BraodcastReceiver来监视屏幕的变化，在onStop的时候反注册掉。系统在其生命周期内可能多次调用onStart和onStop，因为activity之间的交互变化。
* ***foreground lifetime***在onResume和onPause之间。此时，activity在所有其他的之前，拥有用户的焦点。一个activity可能很频繁的出入***foreground***状态。例如，onPause在设备进入休眠或者一个对话框弹出时。因为经常发生，所以这两个方法的任务尽量简单，以免用户等待过久。

生命周期图：
![lifecycle of activity](http://developer.android.com/images/activity_lifecycle.png)

### Saving activity state
正常情况下，activity被暂停或者停止的时候，状态会保持，日后会恢复。

但是，当系统需要销毁一个activity来获取内存的时候，activity会被destroy，因此不能恢复到原来的状态。但是用户并不知道activity被destroy了，用户希望回到原始的状态。此时，在重建activity时就需要addtitional callback来保存和恢复状态。
保存状态的函数是onSaveInstanceState，系统在要销毁activity之前，会调用这个方法，给你一个Bundle来存储。然后当系统重建的时候会把这个Bundle交给onCreate。如果没有，则为null。

**NOTE:**系统并不保证在destroy之前一定会调用onSaveInstanceState，因为有时候并不需要这么做。例如，用户用BACK键退出。总是在onStop之前，也可能在onPause之前调用onSaveInstanceState。

然而，即使你不实现onSaveInstanceState，有些activity也会由于默认的实现恢复。一般，默认的实现，调用布局中每一个***View***的onSaveInstanceState，来保存信息。几乎所有的widget都实现了这个方法，以便在重建的时候自动恢复。比如**EditText**保存所有的用户输入，**Check Box**保存它的状态。你需要做的只是提供一个唯一的id给widget（android:id），如果没有ID，就不会保存。
尽管默认的实现保存了有用的信息，但是，你可能还是需要重写来保存附加的信息。比如，有时你需要保存成员变量的值来重建UI。但是记得重写的时候要先调用父类的onSaveInstanceState，以便保存UI状态。
**NOTE:**因为不保证一定会调用onSaveInstanceState，因此不要用它来存储persistent data。你应该在onPause中持久化数据，如写数据库。
测试恢复能力的最简单的方法就是旋转屏幕。

### Handling configuration changes
有些设备配置可能在运行时改变（屏幕旋转，键盘可用性，语言）。这个时候，android restart 正在运行的Activity（onDestroy， 然后立刻onCreate）。重启的行为让你能够自己控制对新配置的实行。
处理配置变更的最好的方式是，保存程序状态，onSaveInstanceState和onRestoreInstanceState。上一节讨论过。更详细的讨论在[Handling Runtime Changes](http://developer.android.com/guide/topics/resources/runtime-changes.html)

### Coordinating activities

当一个activity启动另一个，他们都经历lifecycle transitions。第一个pause，stop（但是，如果它仍然作为背景可见，他不stop），第二个启动。为了防止他们共享一些磁盘上的数据时出问题，理解第二个启动之前，第一个并没有完全stop很重要。事实上，第二个的启动和第一个的终止，有重叠。

当同一进程内的一个activity启动另外一个activity时的顺序是定义好的。例如，A启动B时：

1. A::onPause()
2. B::onCreate(), B::onStart(), B::onResume().  (B now has user focus.)
3. 如果A 不可见了，A::onStop().

知道这个顺序，你就可以管理你需要传递的信息了。例如，如果第一个结束时要写数据库给其他activity读，那么你应该在onPause中写，而不是onStop。
