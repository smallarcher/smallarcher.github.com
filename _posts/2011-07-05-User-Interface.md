---
layout: post
title: Android笔记--User Interface
category: android
---

# User Interface
包括View和ViewGroup，分别是"widget"和"layout"的父类

## View Hierarchy
![View Hierarchy](http://developer.android.com/images/viewgroup.png)

## Layout

## Widgets

## UI Events
To be informed of UI events, 有2中方法：

* Define an event listener and register it with the View.

     View.onXXXListener;
     View.setOnXXXListener();
* Override an existing callback method for the View.

    如果你要实现一个自己的View，并且希望监听特定的时间时，你可以处理onTouchEvent，onTrackballEvent，onKeyDown。这让你能够为自定义的View中对每一个事件定义自己的行为，并决定是否要分发给其他的子View。再次强调，这些都是View的回调，所以你之有在建立自定义View的时候才重写这些。

## Menus
通常通过MENU键来呼出菜单。然而，你也可以自己增加上下文菜单，它通过用户的长按来触发。

Menus也在View hierachy的结构中，但是你不需要自己在结构中定义。你只要在Activity中定义onCreateOptionsMenu或者onCreateContextMenu并定义菜单的内容。Android会在适当的时候创建它。

Menus也自己处理事件。所以你不需要自己注册event listener。当菜单的item被选择时，onOptionsItemSelected或者onContextItemSelected会被回调。

就像layout一样，你可以选择在XML文件中定义菜单。
详见: [Creating Menus](http://developer.android.com/guide/topics/ui/menus.html)

## Advanced Topics
### Adapters
运行时的数据绑定。 AdapterView是一个ViewGroup的实现，负责子View的初始化和数据绑定。

### Styles and Themes
Style是一组格式化属性，你可以应用到单个元素。 例如，制定特定View的字号，颜色等。
Themes是一组格式化属性，你可以应用到app的所有的activities，或者单个的activity。例如，定义特殊的窗体颜色、背景，菜单的字号和颜色。
