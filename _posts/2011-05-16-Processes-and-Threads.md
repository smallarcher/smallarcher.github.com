---
layout: post
title: Android笔记--Processes and Threads
category: android
---

# Processes and Threads
默认情况下，一个app的所有components在一个process下的同一个thread下运行（**main thread**）。当一个components启动时，如果app已经有一个process（其他components已经启动），这个components在这个process内启动，使用same thread。然而你可以arrange你的app在独立的process，并且你可以创建additional threads for any process

## Processes
**mainifest** 中为每一种components（**"activity, service, receiver, provider"**）提供***android:process*** 属性可以指定components要运行的所属的process。你可以控制每一个component运行在自己的process或者一些components share a process。你可以设置***android:process***不同的app的components运行在同一个process。

*<application>* 也有android:process属性，提供所有components的默认值。

Android可能在内存低时shutdown一个process。 自然隶属于这个process的components也destroyed。需要时再次启动。

### Process lifecycle
Android System place 每一个process into an ***importance hierarchy***，资源不足时lowest importance将被kill。
***importtance hierarchy***中有5个levels, 从高到低如下所示：

1.  **Foreground process**
    与用户当前操作相关的process
    *   正在交互的Activity（Activity's onResume()被调用）
    *   绑定在正在交互的Acitity的Service
    *   Service that running *in the foreground*---service调用startForeground()
    *   正在执行生命周期的回调的Service---（onCreate, onStart, onDestroy）
    *   正在执行onReceive的BroadcastReceiver
    通常只有少数**Foreground process**。只有当系统内存低到不能运行时，才会kill他们。通常这个时候，设备已经到了一个**memory paging state**（内存分页状态？），因此杀死一些**Foreground Process**以保证用户界面的响应。
2.  **Visible process**
    虽然没有Foreground components，但是仍然可以影响用户可见的process。
    *   onPsuse被调用的Activity
    *   绑定到visible or foreground activity的Service
    visible process通常也很重要，除非只有这么做才能保证foreground process正常运行时，才这么做。
3.  **Service process**
    由startService启动的，并且不属于前面2种的。 尽管service processes 不直接与用户界面相关，但是他们通常做着用户关心的事情，如后台音乐播放或者网络数据下载。除非为了保证前两种的正常运行才会kill他。
4.  **Background process**
    用户不可见的进程（onStop被调用）。这类进程和用户体验无直接联系，系统随时kill以reclaim memory。通常有很多**Background process**，因此他们在一个LRU（least recently used）队列。如果activity正确的实现了他的生命周期，并保存它当前的状态，kill这类进程不会对用户体验有影响，因为当用户回到这个activity时，会恢复他之前的状态。详见Actitiies文档关于保存恢复状态的描述。
5.  **Emtpy process**
    没有任何active application component的进程。这种进程的存在只有一个原因：为增快下次启动component的时间。例如，一个content provider A 正在为process B服务，或者一个service A 绑定到process B，A总是比B低优先级。
由于运行中的service进程总是比后台的activity进程优先级高，因此一个activity中的长时间的操作最好是起一个service。比如，一个上传图片到网络的activity，最好是起一个service来执行上传，这样即使用户离开actitivy，上传工作还能在后台继续。Service可以保证不敢activity发生什么，至少有一个service process的优先级。
**broadcast receivers** should employ services而不是put time-consuming operations in a thread也是同样的原因。

## Threads
当app启动，系统创建一个thread来执行，称作***main***。 这个线程很重要，负责用户界面的事件调度，包括绘制事件。因此，main thread有时被称作***UI thread***

系统并不会为每一个compnent的instance创建一个独立的thread。同一个进程下的所有的components在UI thread中被instantiated，对于各个component的系统调用也从UI thread dispatch。相应的，method that response to system callback（onKeyDown）或者生命周期的回调也在UI线程中运作。

例如，当用户touch a button，UI线程dispatch触摸事件给widget，这个widget设置pressed status，并post 一个invalidate request给event queue。然后，UI线程dequeues the request并且notify widget重绘。

当你的app为用户交互执行密集型的工作时，这个thread会表现很差。富国每一个UI时间，都要一个很长时间的操作（网络访问，数据库查询等），整个UI可能阻塞。这时，没有事件能被dispatch，包括绘制时间按。对用户来说就是程序没有响应。如果阻塞超过一定时间（现在是5秒）会出现程序无响应的对话框。

另外，UI toolkit ***不是线程安全的***。因此，不要从worker thread操作UI线程---你对用户界面的操作，必须全部在UI线程中。这里有2个原则。

1.  Do not block UI the thread
2.  Do not access Android UI toolkit from outside the UI thread

### Worker threads
一下是一个例子，一个click listenter下载一个图片，显示在ImageView中

{% highlight java %}

    public void onClick(View v) {
        new Thread(new Runnable() {
            public void run() {
                final Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");
                mImageView.post(new Runnable() {
                    public void run() {
                        mImageView.setImageBitmap(bitmap);
                    }
                });
            }
        }).start();
    }
    
{% endhighlight %}

看起来work fine，因为他创建了一个新的线程去处理网络操作。然而，他违反了第二条规则：不要在UI线程外访问UI toolkit，可能会引发不确定的意外的行为，并且很难跟踪。

因此，android提供几个从其他线程访问UI线程的方法。

* Actitity.runOnUiThread(Runnable)
* View.post(Runnable)
* View.postDelayed(Runnable, long)

上面的例子可以通过View.post修正

{% highlight java %}

    public void onClick(View v) {
        new Thread(new Runnable() {
            public void run() {
                final Bitmap bitmap = loadImageFromNetwork("http://example.com/image.png");
                mImageView.post(new Runnable() {
                    public void run() {
                        mImageView.setImageBitmap(bitmap);
                    }
                });
            }
        }).start();
    }
    
{% endhighlight %}

现在的实现才是线程安全的。
然而，当操作的复杂度增长，这种代码可读性会很差，很难维护。worker线程处理更复杂的交互时，你可以考虑使用[***Handler***](http://developer.android.com/reference/android/os/Handler.html)来处理UI线程分发的消息。但是，最佳解决方案可能还是集成***AsyncTask***类，它简化了需要与UI交互的worker线程的执行。

#### Using AsyncTask
***AsynTask***提供了与界面的异步交互。他在worker线程提供一个阻塞的操作，然后在UI线程publish结果，你不要自己来处理threads或者handlers。
使用方法：集成AsyncTask实现doInbackground回调函数，它运行在一个后台运行的线程池。为了更新UI，你需要实现onPostExecute，它取得doInBackground的结果并在UI线程中处理。 你只需要通过调用在UI线程调用execute即可。

{% highlight java %}

    public void onClick(View v) {
        new DownloadImageTask().execute("http://example.com/image.png");
    }
    
    private class DownloadImageTask extends AsyncTask<String, Void, Bitmap> {
        /** The system calls this to perform work in a worker thread and
          * delivers it the parameters given to AsyncTask.execute() */
        protected Bitmap doInBackground(String... urls) {
            return loadImageFromNetwork(urls[0]);
        }
        
        /** The system calls this to perform work in the UI thread and delivers
          * the result from doInBackground() */
        protected void onPostExecute(Bitmap result) {
            mImageView.setImageBitmap(result);
        }
    }
    
{% endhighlight %}
   
现在UI安全，并且代码简洁。AsyncTask的工作方式：
* 你需要制定参数的类型，处理的值的类型，和最终返回值的类型，通过模板
* doInBackground自动在worker线程执行
* onPreExecute， onPostExecute， onProgressUpdate在UI线程中被调用
* doInBackground的返回被发送给onPostExecute
* 你在可以doInBackground中调用publishProgress来在UI线程中执行onProgressUpdate
* 你可以从任何线程中取消任务的执行

***Caution***：你可能遇到的一个问题就是，当使用worker线程的时候，由于运行时的配置更改（屏幕方向旋转），worker线程意外的重启。要了解如何让你的任务persist，和当activity destroy的时候如何取消task，查看[Shelves](http://code.google.com/p/shelves/)源码。

### Thread-safe methods
有时候，你的methods可能被多个线程调用，应此需要保证线程安全。
对于可以被远程调用的方法，如bound service中的方法，这尤其重要。调用一个实现在IBinder中的方法时，如果调用者与IBinder在同一个进程，方法在调用者的线程中执行，但是，如果调用者和IBinder不在同一个进程中，方法在IBinder的进程中的某个线程中执行，这个线程是系统从线程池中选择的。由于Service可以有多个客户端，多个线程来同时执行IBinder中的方法，因此此时必须要保证线程安全。

Content provider同样也是从线程池中取线程来执行，因此也要保证线程安全。

## Interprocess Comunication
android提供进程间交互，称作RPCs：一个方法被一个app component调用，但是在另外一个进程中执行，并返回结果给调用者。
要实现进程间通讯，app必须绑定一个service。详见[Services](http://developer.android.com/guide/topics/fundamentals/services.html)手册。

