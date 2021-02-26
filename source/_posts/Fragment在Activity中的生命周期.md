---
title: Fragment在Activity中的生命周期
date: 2021-02-26 11:17:35
tags:  android,生命周期,fragment,activity
---

## 启动

```
D: FirstActivity's super.onCreate: 前
D: FirstFragment's onAttach: 
D: FirstFragment's onCreate: 
D: FirstFragment's onCreateView: 
D: FirstFragment's onViewCreated: 
D: FirstActivity's super.onCreate: 后
D: FirstActivity's super.onStart: 前
D: FirstFragment's onActivityCreated: 
D: FirstFragment's onStart: 
D: FirstActivity's super.onStart: 后
D: FirstActivity's super.onResume: 前
D: FirstActivity's super.onResume: 后
D: FirstFragment's onResume: 

```

## 跳转

```
D: FirstFragment's onPause: 
D: SecondActivity's super.onCreate: 前
D: SecondFragment's onAttach: 
D: SecondFragment's onCreate: 
D: SecondFragment's onCreateView: 
D: SecondFragment's onViewCreated: 
D: SecondActivity's super.onCreate: 后
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

### 返回

```
D: SecondFragment's onPause: 
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