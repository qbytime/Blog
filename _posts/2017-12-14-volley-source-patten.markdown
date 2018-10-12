---
layout:     post
title:      "Volley 代码设计原则和设计模式"
subtitle:   ""
date:       2017-12-14
author:     "Qiby"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Android
    - Volley
---


# Volley 代码设计原则和设计模式

接手日历大概不到4个月的时间，整个项目完全面向过程不说，编码还自带混淆。自己编码功力尚浅，深怕坠入其中。故最近在研究Google推出的Volley框架。虽然已经2017年了,现在最流行的网络框架是OkHttp+Retrofit,但是日历中使用的是Volley, 阅读Volley源码也是感觉收货颇丰.借着Volley我也想整理一些写代码的规范和Volley框架的适用场景

Volley框架代码简洁，体积小，扩展性极强。因为内部只有4个线程进行网络操作，采用阻塞方式等待请求，所以避免了开启请求时创建线程的开销，和没有任务的期间回收线程的开销，于此同时因为只有4个网络请求线程和阻塞的方式造成Volley适合数据量不大但是通信频繁的场景

### 1.OCP(Open-Close Principle)开闭原则

开闭原则宗旨是对扩展开放open，对修改关闭close。

如何实现？
	1. 抽象化是关键
	2. 对可变性的封装原则(Principle of Encapsulation of Variation EVP)。
	3. 对可能的拓展预留接口

在Volley中，开闭原则体现得比较好的是Request类族的设计。在开发C/S应用时，服务器返回Response的数据格式多种多样，有字符串类型、xml、json等。而解析服务器返回的Response的原始数据类型则是通过Request类来实现的，这样就使得Request类对于服务器返回的数据格式有良好的扩展性，即Request的可变性太大。

```
public class StringRequest extends Request<String> {

    public StringRequest(int method, String url, Listener<String> listener,
            ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }
    public StringRequest(String url, Listener<String> listener, ErrorListener errorListener) {
        this(Method.GET, url, listener, errorListener);
    }

    @Override
    protected void deliverResponse(String response) {
        Response.Listener<String> listener;
        synchronized (mLock) {
            listener = mListener;
        }
        if (listener != null) {
            listener.onResponse(response);
        }
    }

    @Override
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }
}
```
StringRequest继承与Request，把response.data的chararray数据转化为String。那么如果向服务器请求Image怎么办？
```
public class ImageRequest extends Request<Bitmap> {
    public static final int DEFAULT_IMAGE_TIMEOUT_MS = 1000;
    public static final int DEFAULT_IMAGE_MAX_RETRIES = 2;
    public static final float DEFAULT_IMAGE_BACKOFF_MULT = 2f;

    private final Object mLock = new Object();
    private Response.Listener<Bitmap> mListener;
    private final Config mDecodeConfig;
    private final int mMaxWidth;
    private final int mMaxHeight;
    private final ScaleType mScaleType;
    private static final Object sDecodeLock = new Object();


    public ImageRequest(String url, Response.Listener<Bitmap> listener, int maxWidth, int maxHeight,
            ScaleType scaleType, Config decodeConfig, Response.ErrorListener errorListener) {
        super(Method.GET, url, errorListener);
        setRetryPolicy(new DefaultRetryPolicy(DEFAULT_IMAGE_TIMEOUT_MS, DEFAULT_IMAGE_MAX_RETRIES,
                DEFAULT_IMAGE_BACKOFF_MULT));
        mListener = listener;
        mDecodeConfig = decodeConfig;
        mMaxWidth = maxWidth;
        mMaxHeight = maxHeight;
        mScaleType = scaleType;
    }

    @Override
    public Priority getPriority() {
        return Priority.LOW;
    }

    @Override
    protected Response<Bitmap> parseNetworkResponse(NetworkResponse response) {
        // Serialize all decode on a global lock to reduce concurrent heap usage.
        synchronized (sDecodeLock) {
            try {
                return doParse(response);
            } catch (OutOfMemoryError e) {
                VolleyLog.e("Caught OOM for %d byte image, url=%s", response.data.length, getUrl());
                return Response.error(new ParseError(e));
            }
        }
    }

    private Response<Bitmap> doParse(NetworkResponse response) {
        byte[] data = response.data;
        BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
        Bitmap bitmap = null;
        if (mMaxWidth == 0 && mMaxHeight == 0) {
            decodeOptions.inPreferredConfig = mDecodeConfig;
            bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
        } else {
            // If we have to resize this image, first get the natural bounds.
            decodeOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
            int actualWidth = decodeOptions.outWidth;
            int actualHeight = decodeOptions.outHeight;

            // Then compute the dimensions we would ideally like to decode to.
            int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,
                    actualWidth, actualHeight, mScaleType);
            int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,
                    actualHeight, actualWidth, mScaleType);

            // Decode to the nearest power of two scaling factor.
            decodeOptions.inJustDecodeBounds = false;
            // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
            // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
            decodeOptions.inSampleSize =
                findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
            Bitmap tempBitmap =
                BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

            // If necessary, scale down to the maximal acceptable size.
            if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth ||
                    tempBitmap.getHeight() > desiredHeight)) {
                bitmap = Bitmap.createScaledBitmap(tempBitmap,
                        desiredWidth, desiredHeight, true);
                tempBitmap.recycle();
            } else {
                bitmap = tempBitmap;
            }
        }

        if (bitmap == null) {
            return Response.error(new ParseError(response));
        } else {
            return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
        }
    }
}
```
只要集成Request，并且重写parseNetworkResponse，并封装成Response返回。这样通过扩展的形式来应对软件的变化或者说用户需求的多样性，即避免了破坏原有系统，又保证了软件系统的可扩展性。

### 单一职责原则(Single Responsibility Principle SRP)
  所谓单一职责原则就是一个类只负责一个职责，只有一个引起变化的原因。

  如果一个类承担的职责过多，就等于把这些职责耦合在一起，一个职责的变化会削弱或抑制这个类完成其他职责的能力，这个耦合会导致脆弱的设计。

软件设计真正要做的许多内容，就是发现职责并把这些职责相互分离；如果能够想到多于一个动机去改变一个类，那么这个类就具有多于一个职责，就应该考虑类的分离。

在Volley中，`Network.java` 具有单一职责原则
```
/**
 * An interface for performing requests.
 */
public interface Network {
    NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```
注解写的很清晰，此接口是为了请求网络数据抽象出来的接口，接口中只干了一件事情，进行网络请求。也是此接口可以扩展Volley框架的底层请求

### 里氏替换原则（Liskov Substitution Principle LSP）

里氏替换原则是面向对象设计的基本原则之一。子类通过实现了父类接口，能够替父类的使用地方！通过这个原则，我们客户端在使用父类接口的时候，通过子类实现！
意思就是说我们依赖父类接口，在客户端声明一个父类接口，通过其子类来实现
这个时候就要求子类必须能够替换父类所出现的任何地方，这样做的好处就是，在根据新要求扩展父类接口的新子类的时候而不影响当前客户端的使用！

`Volley.newRequestQueue()` 是Volley中最基础的API，`BasicNetwork`构造函数接受以`HttpStack`为基类的两个子类作为参数`HurlStack`和`HttpClientStack`。这两个参数用来适配网络请求API，当sdk版本大于9时使用`HttpUrlConnection`，小于9时使用`HttpClient`。
```
public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
        BasicNetwork network;
        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                network = new BasicNetwork(new HurlStack());
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                // At some point in the future we'll move our minSdkVersion past Froyo and can
                // delete this fallback (along with all Apache HTTP code).
                String userAgent = "volley/0";
                try {
                    String packageName = context.getPackageName();
                    PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
                    userAgent = packageName + "/" + info.versionCode;
                } catch (NameNotFoundException e) {
                }

                network = new BasicNetwork(
                        new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
            }
        } else {
            network = new BasicNetwork(stack);
        }

        return newRequestQueue(context, network);
    }
```
### 依赖倒置原则（Dependence Inversion Principle DIP ）
所谓依赖倒置原则就是要依赖于抽象，不要依赖于具体。简单的说就是对抽象进行编程，不要对实现进行编程，这样就降低了客户与实现模块间的耦合。

面向过程的开发，上层调用下层，上层依赖于下层，当下层剧烈变化时，上层也要跟着变化，这就会导致模块的复用性降低而且大大提高了开发的成本。

面向对象的开发很好的解决了这个问题，一般的情况下抽象的变化概率很小，让用户程序依赖于抽象，实现的细节也依赖于抽象。即使实现细节不断变化，只要抽象不变，客户程序就不需要变化。这大大降低了客户程序域实现细节的耦合度。

```
public class RequestQueue {
	private final Network mNetwork;
}
```
依赖于`Network`接口，而不是依赖于子类。
