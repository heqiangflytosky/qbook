---
title: React Native-Image控件
categories: React Native
comments: true
tags: [React Native]
description: 介绍React Native中Image控件的基本使用
date: 2017-01-13 10:00:00
---

## 基本用法

### 加载本地图片

```
<Image source={require('./img/baidu.png')}/>
```

### 加载App内资源图片

```
<Image source={{uri: 'ic_launcher'}} style={{width: 140, height: 140}} />
```

### 加载网络图片

```
<Image source={{uri:'http://172.17.137.68/heqiang/2.jpg'}} style={{width: 200, height: 200}}/>
```

资源图片和网络图片必须声明图片寬高，否则不显示。

### 适配不同平台

有时我们希望在不同平台之间用不同的图片    
比如baidu.android.png，baidu.ios.png，代码中只需要写baidu.png，便可以适配android和ios平台    
baidu@2x.png，baidu@3x.png还可以适配不同分辨率的机型。如果没有图片恰好满足屏幕分辨率，则会自动选中最接近的一个图片。这点是和Android中是类似的。    

### 代码

```javascript
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
  Image
} from 'react-native';

class HelloWorldAppp extends Component{
  render() {
    console.log("render()");
    return (
      <View>
        <Text style={styles.title_text}>本地图片</Text>
        <Image source={require('./img/baidu.png')}/>
        <Text style={styles.title_text}>资源图片</Text>
        <Image source={{uri: 'ic_launcher'}} style={{width: 140, height: 140}} />
        <Text style={styles.title_text}>网络图片</Text>
        <Image source={{uri:'http://*******.jpg'}} style={{width: 200, height: 200}}/>
    );
  }
}

AppRegistry.registerComponent('AwesomeProject', () => HelloWorldAppp);

const styles = StyleSheet.create({
  title_text:{
    fontSize:18,
  }

});
```

### 效果图


<img src="/images/react-native-image/image1.png" width="270" height="480"/>

效果图如上如。    





