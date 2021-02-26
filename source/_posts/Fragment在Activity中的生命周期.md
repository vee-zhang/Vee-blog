---
title: Fragment在Activity中的生命周期
date: 2021-02-26 11:17:35
tags:  android,生命周期,fragment,activity
---

## 流程图

```yuml
// {type:activity}
// {generate:true}

(start)[activity启动]->(A: super.onCreate)->(A: setContentView)->(A: super.onStart)->(A: super.onResume)
(A: setContentView)->(F: onAttach)->(F: onCreate)->(F: onCreateView)->(F: onViewCreated)->(A: super.onStart)
(A: super.onStart)->(F: onActivityCreated)->(F: onStart)->(A: super.onResume)
(A: super.onResume)-><a>[之后]->(F: onResume)->|b|
(A: super.onResume)-><b>
<b>[进入后台]->(A: super.onPause)->(F: onPause)->(A: super.onStop)
(A: super.onPause)->(A: super.onStop)->(F: onStop)
(A: super.onStop)-><c>[恢复]->(A: super.onRestart)->(A: super.onStart)
<c>[销毁]->(A: super.onDestory)->(F: onDestoryView)->(F: onDestory)->(F: onDetach)->(end)

```

## 启动

```
D: FirstActivity's super.onCreate: 前
D: FirstActivity's super.onCreate: 后
D: FirstActivity's setContentView: 前

D: FirstFragment's onAttach: 
D: FirstFragment's onCreate: 
D: FirstFragment's onCreateView: 
D: FirstFragment's onViewCreated: 

D: FirstActivity's setContentView: 后
D: FirstActivity's super.onStart: 前

D: FirstFragment's onActivityCreated: 
D: FirstFragment's onStart: 

D: FirstActivity's super.onStart: 后
D: FirstActivity's super.onResume: 前
D: FirstActivity's super.onResume: 后
D: FirstFragment's onResume: 
```

fragment的`onAttach`，`onCreate`，`onCreateView`，`onViewCreated`是在Activity的`setContentView`中执行的。

想一下就知道，Activity的`setContentView`的目的是从xml中加载View，而我把fragment写在了xml里面。

>这里有个小插曲，事实证明，Activity的`super.onCreate`可以写在`setContentView`之后，但是xml中存在fragment时就会报错。

fragment的`onActivityCreated`、`onStart`是在Activity的`super.onStart`方法中执行。

fragment的`onResume`是在activity的`super.onResume`之后执行。

## 跳转

```
D: FirstActivity's super.onPause: 前
D: FirstFragment's onPause: 
D: FirstActivity's super.onPause: 后

D: SecondActivity's super.onCreate: 前
D: SecondActivity's super.onCreate: 后
D: SecondActivity's setContentView: 前
D: SecondFragment's onAttach: 
D: SecondFragment's onCreate: 
D: SecondFragment's onCreateView: 
D: SecondFragment's onViewCreated: 
D: SecondActivity's setContentView: 后
D: SecondActivity's super.onStart: 前
D: SecondFragment's onActivityCreated: 
D: SecondFragment's onStart: 
D: SecondActivity's super.onStart: 后
D: SecondActivity's super.onResume: 前
D: SecondActivity's super.onResume: 后
D: SecondFragment's onResume: 

D: FirstActivity's super.onStop: 前
D: FirstFragment's onStop: 
D: FirstActivity's super.onStop: 后
```

fragment的`onPause`是在Activity的`super.onPause`中执行。

当第二个含有fragment的Activity可见后，上一个Activity的生命周期才会继续。

然后上一个fragment的`onStop`在Activity的`onStop`中执行。

### 返回

```
D: SecondActivity's super.onPause: 前
D: SecondFragment's onPause: 
D: SecondActivity's super.onPause: 后

D: FirstActivity's super.onRestart: 前
D: FirstActivity's super.onRestart: 后

D: FirstActivity's super.onStart: 前
D: FirstFragment's onStart: 
D: FirstActivity's super.onStart: 后
D: FirstActivity's super.onResume: 前
D: FirstActivity's super.onResume: 后
D: FirstFragment's onResume: 

D: SecondActivity's super.onStop: 前
D: SecondFragment's onStop: 
D: SecondActivity's super.onStop: 后

D: SecondActivity's super.onDestroy: 前
D: SecondFragment's onDestroyView: 
D: SecondFragment's onDestroy: 
D: SecondFragment's onDetach: 
D: SecondActivity's super.onDestroy: 后

```

先执行第二个Activity的`onPause`。

再回调前一个Activity的`onRestart`。

fragment的`onStart`在Activity的`super.onStart`中执行。

fragment的`onResume`在Activity的`super.onResume`中执行。

fragment的`onStop`在Activity的`super.onStop`中执行。

fragment的`onDestroyView`、`onDestroy`、`onDetach`在Activity的`super.onDestroy`中执行。

**注意**    fragment的`onResume`是在Activity的`onResume`之后执行的！！！

## 退入后台

```
D: FirstActivity's super.onPause: 前
D: FirstFragment's onPause: 
D: FirstActivity's super.onPause: 后
D: FirstActivity's super.onStop: 前
D: FirstFragment's onStop: 
D: FirstActivity's super.onStop: 后
```

## 回到前台

```
D: FirstActivity's super.onRestart: 前
D: FirstActivity's super.onRestart: 后
D: FirstActivity's super.onStart: 前
D: FirstFragment's onStart: 
D: FirstActivity's super.onStart: 后
D: FirstActivity's super.onResume: 前
D: FirstActivity's super.onResume: 后
D: FirstFragment's onResume: 
```

## 退出

```
D: FirstActivity's super.onPause: 前
D: FirstFragment's onPause: 
D: FirstActivity's super.onPause: 后

D: FirstActivity's super.onStop: 前
D: FirstFragment's onStop: 
D: FirstActivity's super.onStop: 后

D: FirstActivity's super.onDestroy: 前
D: FirstFragment's onDestroyView: 
D: FirstFragment's onDestroy: 
D: FirstFragment's onDetach: 
D: FirstActivity's super.onDestroy: 后
```