# handler方法
Handler是Android消息机制的上层接口，通过它可以轻松地将一个任务切换到Handler所在的线程中去执行，
该线程既可以是主线程，也可以是子线程，要看构造Handler时使用的构造方法中传入的Looper位于哪里；

## MessageQueue
MessageQueue是一个消息队列，先进先出，内容为Message；

### Message
Message内保存了信息内容，可以为message设置callback，在处理完该条消息的时候执行；
```
Message message = Message.obtain();
message.what = 1;
message.obj = "A";
message.callback = new Runnable(){Override Run() {...} };
```

## Looper
Looper是一个消息处理工具,每个looper内部持有一个messagequeue，looper启动后会不停从queue
中dequeue出message分发给持有它的handler；
```
Lopper.prepare();   // 检查本地线程是否有looper，没有创建新的，有抛异常
Looper.loop();     // 转起来
```
主线程自带一个looper不需要手动创建；

## Handler
handler是一个处理工具，Handler在创建的时候传入这个线程的唯一looper，当有消息传进来的时候，
（也就是当有一个线程想要另一个线程执行任务时），通过这个handler的post（传一个runnable）或
sendMessage（传一个message）方法，将其传入messageUqeue；当looper loop到这个消息时，再将其分发
给这个handler来处理；

- 一个looper可以被多个handler持有；
- 在发起消息的线程内会持有目标线程的handler，因此可以通过这个handler来塞消息进这个handler的queue，
消息内会持有这个handler，因此loop出时会确认由这个handler来处理；
- 消息取出来给到对应handler来处理，传message可以通过message case来执行要执行的，或者直接
post了一个Runnable里面的run方法保存了要执行的，message都带一个callback，用来dispatch这个
message后来执行；因此是两种让目标线程干活的方式；
```
// 源码：post方法最后会把runnable包装成个带callback的空message
public final boolean post(Runnable r)
  {
    return sendMessageDelayed(getPostMessage(r), 0);
  }
public final boolean sendMessageDelayed(Message msg, long delayMillis)
  {
    if (delayMillis < 0) {
    delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
  }

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
一个例子看：
```
public class MainActivity extends AppCompatActivity {

    Handler mainHandler,workHandler;
    HandlerThread mHandlerThread;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 主线程handler
        mainHandler = new Handler();
        // 创建工作线程，自带looper
        mHandlerThread = new HandlerThread("handlerThread");
        mHandlerThread.start();
        // 传入looper。创建工作线程handler
        workHandler = new Handler(mHandlerThread.getLooper()){
            @Override
            public void handleMessage(Message msg)
            {   //设置了两种消息处理操作,通过msg来进行识别
                switch(msg.what){
                    case 1:
                        ... // 要做的操作
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run () {
                                text.setText("第一次执行");
                            }
                        });
                        break;
                    case 2:
                        ... //要执行的操作
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run () {
                                text.setText("第二次执行");
                            }
                        });
                        break;
                    default: break;
                }
            }
        };

        /**
         * 步骤④：使用工作线程Handler向工作线程的消息队列发送消息
         * 在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
         */
        button1 = (Button) findViewById(R.id.button1);
        button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Message msg = Message.obtain();
                msg.what = 1; //消息的标识
                msg.obj = "A"; // 消息的存放
                // 通过Handler发送消息到其绑定的消息队列
                workHandler.sendMessage(msg);
            }
        });

        button2 = (Button) findViewById(R.id.button2);
        button2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Message msg = Message.obtain();
                msg.what = 2;
                msg.obj = "B";
                workHandler.sendMessage(msg);
            }
        });

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mHandlerThread.quit(); // 退出消息循环
        workHandler.removeCallbacks(null); // 防止Handler内存泄露 清空消息队列
    }
}
```
上边的例子通过向工作线程塞消息让工作线程操作，完成后工作线程给主线程handler塞消息，更新对应UI；
- 塞消息要通过对方的handler（持有了对应现成的looper）；

## HandlerThread
继承Thread类，创建线程后线程里直接持有一个looper，可以直接用来创建handler和处理消息；

## 参考文档
[handler/thread/handlerThread](https://blog.csdn.net/weixin_41101173/article/details/79687313)
[handler的post/sendMessage区别](https://cloud.tencent.com/developer/article/1727098)
