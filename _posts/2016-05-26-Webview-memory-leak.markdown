---
layout:     post
title:      "Webview 使用及内存泄露问题"
subtitle:   ""
date:       2016-05-26 
author:     "Qiby"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Android
    - 内存
---


# Webview 使用及内存泄露问题

1. 如果包含webview，在activity的onResume和onPause中，需要分别调用webview的onResume和onPause
2. 如果包含webview，当webview从parent上remove掉时，webview需要调用onPause; 当webview再次add到parent时，webview需要调用onResume;（否则webview会白屏）
3. webview内存泄露问题特别注意：
Activity退出时在onDestory中释放webview
mContainer.removeView(mWebView);
mWebView.setWebChromeClient(null);
mWebView.setDownloadListener(null);
mWebView.setWebViewClient(null);
mWebView.destroy();
顺序不要搞错，先remove Webview，后做webview的释放，否则会出现内存泄露
