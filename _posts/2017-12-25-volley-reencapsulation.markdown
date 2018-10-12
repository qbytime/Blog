---
layout:     post
title:      "Volley Request 二次封装"
subtitle:   ""
date:       2017-12-25
author:     "Qibenyu"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - Android
---

# Volley Request 二次封装
乐视日历网络请求框架使用的Volley框架，但是遗留代码的封装耦合太紧，不好用，每一个请求需要单独创建一个新的类，所以我重新做了一下封装。

Volley的Request非常易于扩展，所以二次封装思路是继承Request，使用Builder模式构建请求参数。


```
public class LefengObjectRequest<T> extends Request<T> {

    private static final String TAG = "LefengObjectRequest";

    private Response.Listener mListener;

    private Map<String, String> mHeaders;

    private Map<String, Object> mParamters;

    private String mApi;

    private String mBody;

    private Class<T> clazz;

    private Builder mBuilder;

    private LefengObjectRequest(int method, String url, Response.ErrorListener listener) {
        super(method, url, listener);
    }

    public LefengObjectRequest(Builder builder) {
        this(builder.method, builder.url, builder.errorListener);
        this.mListener = builder.listener;
        this.mApi = builder.api;
        this.mHeaders = builder.headers;
        this.mParamters = builder.paramters;
        this.mBody = builder.body;
        this.clazz = builder.clazz;
        this.mBuilder = builder;
    }

    @Override
    protected Response<T> parseNetworkResponse(NetworkResponse networkResponse) {
        String parsed;
        try {
            parsed = new String(networkResponse.data, HttpHeaderParser.parseCharset(networkResponse.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(networkResponse.data);
        }
        Log.d(TAG, "parseNetworkResponse: " + parsed + ",url = " + getUrl());

        T t = null;
        try {
            t = JSON.parseObject(parsed, clazz);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return Response.success(t, HttpHeaderParser.parseCacheHeaders(networkResponse));


    }

    @Override
    protected void deliverResponse(T t) {
        try {
            mListener.onResponse(t);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public Map<String, String> getHeaders() throws AuthFailureError {
        return mHeaders;

    }

    public Builder getBuilder() {
        return mBuilder;
    }

    public static class Builder<T> {

        String baseUrl;

        int method;

        String api;

        String url;

        String body;

        Class clazz;

        Map<String, String> headers = new HashMap<>();

        Map<String, Object> paramters = new HashMap<>();

        ArrayList<Interceptor> interceptors = new ArrayList<>(0);

        Response.Listener<T> listener;

        Response.ErrorListener errorListener;

        String accessKey;

        String secretKey;


        public Builder() {
            this.method = Method.GET;
        }

        public Builder baseUrl(String url) {
            if (url == null) throw new NullPointerException("url == null");
            this.baseUrl = url;
            return this;
        }

        public Builder api(String api) {
            this.api = api;
            return this;
        }

        public Builder addHeader(String name, String value) {
            headers.put(name, value);
            return this;
        }

        public Builder addInterceptor(Interceptor interceptor) {
            if (interceptor == null) throw new IllegalArgumentException("intercept == null");
            interceptors.add(interceptor);
            return this;
        }

        public Builder method(int method) {
            this.method = method;
            return this;
        }

        public Builder setClass(Class clazz) {
            this.clazz = clazz;
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder addParameter(String name, Object value) {
            paramters.put(name, value);
            return this;
        }

        public Builder setListener(Response.Listener listener) {
            this.listener = listener;
            return this;
        }

        public Builder setErrorListener(Response.ErrorListener errorListener) {
            this.errorListener = errorListener;
            return this;
        }

        public Builder setAccessKey(String ak) {
            this.accessKey = ak;
            return this;
        }

        public Builder setSecretKey(String sk) {
            this.secretKey = sk;
            return this;
        }

        public LefengObjectRequest build() {
            if (TextUtils.isEmpty(baseUrl)) throw new IllegalStateException("url == null");

            if (TextUtils.isEmpty(api)) throw new IllegalStateException("interface not set");

            buildUrlWithParamters();

            for (Interceptor interceptor : interceptors) {
                interceptor.intercept(this);
            }

            return new LefengObjectRequest<T>(this);
        }

        private void buildUrlWithParamters() {
            url = baseUrl + api;
            String paramString = "";
            if (!paramters.isEmpty()) {
                SortedSet<String> set = new TreeSet<String>();
                for (String key : paramters.keySet()) {
                    Object value = paramters.get(key);
                    set.add(key + "=" + value);
                }
                paramString = LeSignature.join(set, "&");
            }
            url = url + "?" + paramString;
        }

    }

}

```
其中借鉴OkHttp的Interceptor来进行参数过滤，并且增加乐视的请求签名
```
public interface Interceptor {

    void intercept(LefengObjectRequest.Builder builder);
}
```

最终Request创建为
```
Request request = new LefengObjectRequest.Builder<Scratchable>()
                .baseUrl(TOP_BETA_SCRATCH_URL)
                .api(API_SCRATCH_AD)
                .method(Request.Method.GET)
                .setListener(listener)
                .setErrorListener(errorListener)
                .addParameter("im", Util.getImei())
                .addInterceptor(new CalendarSignInterceptor())
                .setClass(Scratchable.class)
                .build();
        CalendarApplication.getInstance().getRequestQueue().add(request);
```