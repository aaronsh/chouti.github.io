---

layout: post
author: sam
title:  芯片批量校准系统
description: 批量化的数字传感器的校准系统
categories: [ 嵌入式开发, Python, QT, 服务器软件 ]
image: https://cdn-images-1.medium.com/max/11074/0*BwWQN0ol_F5dP4dj
featured: true
hidden: true

---

这是一套根据一家芯片生产公司的定制需求而开发的软件系统。它可以帮助芯片生产公司提高生产率，同时对每颗芯片生成记录，便于售后维护管理。

# 客户遇到的问题
## 芯片只能单颗手工校准，校准流程成为生产过程中的瓶颈。
## 芯片内的校准数据丢失，只能返厂进行手工校准，过程繁琐。

# 解决方案
由于校准过程的规范要求，单颗芯片的校准时间难以节省。所以此方案中，采用了同时对多颗芯片进行校准的方法，通过并行处理来提高生产率。 同时，为了防止校准人员的手动输入错误，此方案将参数的输入放置与后台管理中，校准人员仅需在校准软件中做简单选择即可。

此校准系统分为三个模块。第一个模块为校准仪器。它负责与芯片通信，收集校准用的各项数据，并将数据通过Usb发送到校准软件中。校准仪器的CPU采用STM32F1系列的单片机。在开发校准仪器的过程中，使用的开发语言为C语言。

第二个模块为校准软件。它由Python和Qt写成，运行在PC上，通过Usb与校准仪器进行通信。校准软件从校准后台服务器中获取当前的各种设置，并根据设置收集校准用的各项数据。它在收集完当前校准步骤中的数据后，通过网络，将数据保存到校准后台服务器中。它也会‘按照设定，计算校准参数，并将参数写入到芯片中，完成校准过程。

第三个模块为校准后台软件。它是基于Flask框架进行开发的。Flask框架是一款基于Python语言开发的，非常流行的，简洁轻巧的，扩展性强的Web微框架。校准后台软件在Flask框架的基础上添加了：用户权限管理、芯片型号管理、芯片校准数据记录、校准过程管理等功能。

# 方案的使用过程
  1. 将校准仪器通过usb连接计算机，在计算机上开启校准软件，选择当前校准的芯片型号，校准的内容。
  2. 将芯片插入治具的插槽中，将治具与校准仪器连接好。
  3. 待校准环境稳定后，校准人员点击开始，校准软件和校准仪器执行当前的校准内容。
  4. 校准完成后，根据校准软件的提示，校准人员将校准失败的芯片选出。校准通过的芯片会进入下一校准过程进行校准，直至校准通过。

# 方案的效果

此方案获得客户的极大满意。客户使用此方案，将产能从5K/Mon提高到了20K/Mon。

```kotlin
Handler().postDelayed({
   doSomething()
}, delay)
```

This API is handy and yet so sneaky. Don’t let yourself be fooled when using it on a view. To understand where the danger lies, we need to dig deeper into the `View` class.

---

# Handlers for Views

`View` in Android leverages the `Handler` API. It acts as a passthrough with an inner `Handler` and exposes both `post()` and `postDelayed()` functions.

Have a closer look at its implementation (at the time I’m writing the article):

<script src="https://gist.github.com/StephenVinouze/63ac5307d5f0ea4c9aa47aa76c7881cc.js" charset="utf-8"></script>

I want you to pay attention to the Javadoc comment. The `removeCallbacks` line hints at something about removing the callback.

Let’s see it in action:

```
myView.postDelayed({
    myView.animateSomething()
}, delay)
```

In this example, I want to animate my view when a given delay expires.

Once the delay expired, what tells me my view still exists? That it’s still visible to the user?

You see, views have lifecycles. They may be destroyed at any time, either by the system or because your user navigates inside your app.

Since you queued a message inside a `Looper`, the system will deliver it if you don’t tell it to do otherwise. You expose your app to the possibility of a crash with a `NullPointerException`.

You understand now how *fundamental* this little comment is. I’ve seen so many crashes due to this API because developers failed to handle lifecycles.

---

# What If I Want to Delay an Action on a View?

You could still use this API and extract the `Runnable` declared inside your `Handler`. You’ll need to remove the callback whenever it’s relevant in your code, as hinted in the method’s comment.

```kotlin
private val animateRunnable = Runnable {
    myView.animateSomething()
}

myView.postDelayed(animateRunnable, delay)

// Somewhere in your code
myView.removeCallback(animateRunnable)
```

I advocate banning these methods because they’re misused or inappropriate.

For instance, if you’re using RxJava, you won’t need them anymore. Making a stream with either a delay or a timer is trivial. And you can dispose of your stream with ease.

```kotlin
val disposable = Observable.timer(delay, TimeUnit.MILLISECONDS)
    .subscribe {
        myView.animateSomething()
    }

// Somewhere in your code
disposable.dispose()
```

---

# What If I Need to Wait for Another Frame?

So far, I’ve only mentioned `postDelayed()`. To me, `post()` is by far the most misused API I’ve ever seen when applied to a `View`.

Why use `post()` when there’s no delay attached to it? Remember `Handler` is queuing messages in a `Looper`. Posting will queue the message and deliver it to the next frame.

Usually, I’ve seen developers using this because the view was not laid out. Imagine you want to animate a view as soon as it appears. You need its position and/or its size, depending on which animation you intent to achieve.

You must wait for the view to be measured. So you delegate the animation to the next frame and cross your fingers that the view will be ready. With this solution, two things stand out:

1. You have no guarantee your view will be measured on the next frame.

1. This code looks patchy as heck.

Like `postDelayed()`, there are more reliable mechanisms. You should use `ViewTreeObserver` instead, in particular the two following callbacks:

- [OnPreDrawListener](https://developer.android.com/reference/android/view/ViewTreeObserver.OnPreDrawListener) notifies you when a view is about to be laid out. At this point, the view has been measured

- [OnGlobalLayoutListener](https://developer.android.com/reference/android/view/ViewTreeObserver.OnGlobalLayoutListener) triggers an event whenever your view state changes

You must be careful when using those methods. You’re likely to create memory leaks if you forget to remove the listener once you performed your action.

```kotlin
myView.viewTreeObserver.addOnPreDrawListener(object: ViewTreeObserver.OnPreDrawListener {
    override fun onPreDraw(): Boolean {
        myView.animateSomething()
        myView.viewTreeObserver.removeOnPreDrawListener(this)
        return true
    }
})

myView.viewTreeObserver.addOnGlobalLayoutListener(object: ViewTreeObserver.OnGlobalLayoutListener {
    override fun onPreDraw(): Boolean {
        myView.animateSomething()
        myView.viewTreeObserver.removeOnGlobalLayoutListener(this)
    }
})
```

Even better, use [KTX extensions](https://developer.android.com/kotlin/ktx), and let them take care of the boilerplate for you.

```kotlin
myView.doOnPreDraw {
    myView.animateSomething()
}

myView.doOnLayout {
    myView.animateSomething()
}
```

That’s all folks! I may sound harsh when advocating to ban this mechanism. At the very least, I encourage you to chase them out and ponder whether they’re suited to the task.
