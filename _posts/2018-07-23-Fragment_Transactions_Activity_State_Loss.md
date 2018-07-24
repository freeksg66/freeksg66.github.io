---
layout:     post
title:      译 Fragment Transactions Activity State Loss
subtitle:   慎用commitAllowingStateLoss()
date:       2018-07-23
author:     GL
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - Android
---

每次初始化Honeycomb时，有时会报如下错误：

    java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
        at android.support.v4.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1341)
        at android.support.v4.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:1352)
        at android.support.v4.app.BackStackRecord.commitInternal(BackStackRecord.java:595)
        at android.support.v4.app.BackStackRecord.commit(BackStackRecord.java:574)

这篇文章将解释<b>为什么</b>和<b>何时</b>会抛出这个异常，并且将总结几个建议，确保它们不会再crash在你的应用中。

## 为什么会抛出这个异常

这个异常抛出的原因是，因为你在activity的state已经保存了之后（这个被称为Activity state loss），尝试commit一个FragmentTransaction。然而在我们了解这实际意义的细节之前，让我们首先看一下当onSaveInstanceState()方法调用时它的内部发生了什么。正如我在最近的一篇文章<a href="https://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html">Binders & Death Recipients</a>中所说,Android应用程序在Android运行时环境中几乎无法控制自己的生命周期。Android操作系统为了释放内存有能力随时终止进程，并且background activities可能在不发出warn的情况下被杀掉。为了确保这种操作系统有时这种随意的对用户隐藏的行为，在activity销毁之前，framework层每个activity一次机会去调用onSaveInstanceState()方法存储它的state。当存储的state在后面恢复时，用户会感觉好像这些activities从后台切换到了前台，不管这个Activity是否曾经已经被系统杀死过。

当framework调用onSaveInstanceState()方法时，它向这个方法里传递了一个Bundle对象，该对象的作用是让Activity使用它来存储state，并且Activity在Bundle中记录了它的dialogs、fragments和views的state。当这个方法返回时，操作系统序列化了Bundle对象通过Binder接口传到System Server进程中，这个过程中的存储是安全的。当系统随后决定重新创建Activity时，System Server进程会发送同一个Bundle对象返回该应用，为了使用该对象恢复Activity的old state。

那么为什么会抛出这个错误呢？好吧，问题源于这样的事实：这些Bundle对象仅代表了onSaveInstanceState（）被调用时的一个Activity的快照。这就意味着当你在onSaveInstanceState（）方法之后调用FragmentTransaction#commit（）时，这个transaction将不会被记住，因为它将不会作为Activity的state的一部分被记录。从用户角度来看，这个transaction好像丢失了，导致了一个突发的UI state loss。为了保护用户的体验，Android回避了不惜一切代价的state loss，从而简单地抛出了一个IllegalStateException错误，无论它是否发生。

## 什么时候会抛出这个异常

如果你以前已经遇到了这个异常，以也许已经注意到这个异常的抛出随不同平台版本有略微的差别。举个例子，你可能发现较旧的设备倾向于不那么频繁地抛出异常，或者你的应用程序在使用support library时比使用官方的framework classes时更容易崩溃。这些轻微的不一致导致许多人认为support library是错误的并且不可信任。 然而，这些假设通常不正确。

存在这些轻微不一致的原因源于Honeycomb中对Activity生命周期的重大改变。在Honeycomb之前，activities在被stop之后才被认为是可被杀掉的，这意味着onSaveInstanceState（）方法现在将在onPause（）之前调用，而不是立即在onPause()之前调用。这些不同将被总结道下面的表格中：

<table>
    <tr>
        <td></td>
        <td><b>pre-Honeycomb</b></td>
        <td><b>post-Honeycomb</b></td>
    </tr>
    <tr>
        <td>Activities将在onPause()方法之前被杀掉？</td>
        <td>NO</td>
        <td>NO</td>
    </tr>
    <tr>
        <td>Activities将在onStop()方法之前被杀掉？</td>
        <td>YES</td>
        <td>NO</td>
    </tr>
    <tr>
        <td>onSaveInstanceState(Bundle)方法将确保在什么方法之前调用？</td>
        <td>onPause()</td>
        <td>onStop()</td>
    </tr>
</table>

由于对Activity生命周期进行了轻微更改，因此support library有时需要根据平台版本更改其行为。
例如，在Honeycomb设备及更高版本上，每次在onSaveInstanceState（）之后调用commit（）时都会抛出异常，以警告开发人员发生了状态丢失。但是，每次发生这种情况时抛出异常对于Honeycomb之前的设备来说过于严格，因为Honeycomb设备在Activity生命周期中更早地调用了onSaveInstanceState（）方法，因此更容易受到意外状态丢失的影响。Android团队被迫做出妥协：为了更好地与旧版本的平台进行互操作，旧设备将不得不忍受onPause（）和onStop（）之间可能导致的意外状态丢失。 support library在两个平台上的行为总结在下表中：

<table>
    <tr>
        <td></td>
        <td><b>pre-Honeycomb</b></td>
        <td><b>post-Honeycomb</b></td>
    </tr>
    <tr>
        <td>在onPause()方法之前调用onPause()</td>
        <td>OK</td>
        <td>OK</td>
    </tr>
    <tr>
        <td>在onPause()方法和onStop()方法之间调用commit()方法</td>
        <td>STATE LOSS</td>
        <td>OK</td>
    </tr>
    <tr>
        <td>在onStop()方法之后调用commit()方法</td>
        <td>EXCEPTION</td>
        <td>EXCEPTION</td>
    </tr>
</table>

## 怎样避免这个异常

一旦了解了实际发生的情况，避免Activity state丢失变得更加容易。 如果你已经在帖子中做到这一点，希望你能更好地了解support library的工作原理以及为什么在应用程序中避免状态丢失非常重要。 但是，如果你在这篇文章中提到了快速修复，那么在你的应用程序中使用FragmentTransactions时，这里有一些建议可以保留在你的脑海中：

* <b>在Activity生命周期方法中committing transactions时要小心。</b>大多数应用程序只会在第一次调用onCreate（）或者响应用户输入时commit transactions，并且永远不会遇到任何问题。但是，当你的transactions开始冒险进入其他Activity生命周期方法时，例如onActivityResult（），onStart（）和onResume（），事情会变得有点棘手。例如，你不应在FragmentActivity＃onResume（）方法中commit transactions，因为在某些情况下可以在activity state恢复之前调用该方法（有关更多信息，请<a href="http://developer.android.com/reference/android/support/v4/app/FragmentActivity.html#onResume()">参阅文档</a>）。如果你的应用程序需要在除onCreate（）之外的Activity生命周期方法中commit transactions，请在FragmentActivity＃onResumeFragments（）或Activity＃onPostResume（）中执行此操作。保证在Activity恢复到其原始state后调用这两种方法，从而避免state丢失的可能性。 （作为如何完成此操作的示例，请查看我对此<a href="http://stackoverflow.com/q/16265733/844882">StackOverflow问题的答案</a>，以获取有关如何提交FragmentTransactions以响应对Activity＃onActivityResult（）方法的调用的一些想法）。在这些方法中执行事务的问题是，在调用它们时，它们不知道Activity生命周期的当前状态。 例如，请考虑以下事件序列：

1. 一个Activity执行了一个AsyncTask。
2. 用户按了Home键，造成activity的onSaveInstanceState()和onStop()方法被调用。
3. AsyncTask完成了并且onPostExecute()方法被调用了，不知道该activity已被停止。
4. 在onPostExecute()方法中一个FragmentTransaction被commit了，就会抛出这个异常。

通常，在这些情况下避免异常的最佳方法是简单地避免在异步回调方法中一起commit transactions。 Google工程师似乎也同意这一观点。 根据Android开发者小组的<a href="https://groups.google.com/d/msg/android-developers/dXZZjhRjkMk/QybqCW5ukDwJ">这篇文章</a>，Android团队认为，在异步回调方法中提交FragmentTransactions可能导致UI的主要变化对用户体验不利。 如果你的应用程序需要在这些回调方法中执行transaction并且没有简单的方法来保证在onSaveInstanceState（）之后不会调用回调，则可能不得不求助于使用commitAllowingStateLoss（）并处理可能的state丢失发生。 （另请参阅这两个StackOverflow帖子以获取其他提示，<a href="http://stackoverflow.com/q/8040280/844882">此处</a>和<a href="http://stackoverflow.com/q/7992496/844882">此处</a>）。

* <b>避免在异步的回调方法中执行transactions。</b>这个结论通常使用类似这种方法，比如AsyncTask#onPostExecute()和LoaderManager.LoaderCallbacks#onLoadFinished()。
* <b>使用commitAllowingStateLoss()方法只作为最后的手段。</b>调用commit（）和commitAllowingStateLoss（）之间的唯一区别是，如果发生state丢失，后者不会抛出异常。 通常你不想使用此方法，因为它意味着可能会发生state丢失。 当然，更好的解决方案是编写应用程序，以便在保存活动状态之前保证调用commit（），因为这将带来更好的用户体验。 除非无法避免state丢失的可能性，否则不应使用commitAllowingStateLoss（）。



<b></b>

{% highlight java %}
{% endhighlight %}