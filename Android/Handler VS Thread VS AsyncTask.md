## Handler VS Thread VS AsyncTask
If you look at the source code of AsyncTask and Handler, you will see their code is written purely in Java. (Of course, there are some exceptions, but that is not an important point.)

So there is no magic in AsyncTask or Handler. They just make your job easier as a developer.

For example: If Program A calls method A(), method A() could run in a different thread with Program A.You can easily verify(验证) it using:

Thread t = Thread.currentThread();    
int id = t.getId();

Why you should use a new thread? You can google for it. Many many reasons.

**So, what is the difference between Thread, AsyncTask, and Handler?**

AsyncTask and Handler are written in Java (internally they use a Thread), so **everything you can do with Handler or AsyncTask, you can achieve using a Thread too.**

What can Handler and AsyncTask really help you with?

The most obvious reason is communication between the caller thread and the worker thread. (**Caller Thread:** A thread which calls the Worker Thread to perform some task. A Caller Thread does not necessarily have to be the UI thread). Of course, you can communicate between two threads in other ways, but there are many disadvantages (and dangers) due to thread safety issues.

That is why you should use Handler and AsyncTask. They do most of the work for you, you just need to know what methods to override.

**The difference between Handler and AsyncTask is:** Use AsyncTask when Caller thread is a UI Thread. This is what android document says:

*AsyncTask enables proper and easy use of the UI thread. This class allows to perform background operations and publish results on the UI thread without having to manipulate threads and/or handlers*

I want to emphasize on two points:

1) Easy use of the UI thread (so, use when caller thread is UI Thread).

2) No need to manipulate handlers. (means: You can use Handler instead of AsyncTask, but AsyncTask is an easier option).

There are many things in this post I haven't said yet, for example: what is UI Thread, or why it's easier. You must know some method behind each kind and use it, you will completely understand why..

@: when you read the Android document, you will see:

Handler allows you to send and process Message and Runnable objects associated with a thread's MessageQueue
They may seem strange at first. Just understand that each thread has each message queue (like a to do list), and the thread will take each message and do it until the message queue is empty (just like you finish your work and go to bed). So, when Handler communicates, it just gives a message to caller thread and it will wait to process. Complicated? Just remember that Handler can communicate with the caller thread in a safe way.