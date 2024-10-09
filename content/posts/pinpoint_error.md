---
title: "Pinpoint_error"
date: 2024-09-27T17:06:20+08:00
draft: false
---

# 背景
最近升级了公司框架，然后发现错误日志一直在刷。
```log
09-29 09:14:01.001 [o-32622-exec-64] WARN  .p.p.t.d.i.CoyoteOutputStreamInterceptor -- before. Caused:null
java.lang.ArrayIndexOutOfBoundsException: null
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.logResponseValue(DefaultDigiwinHttpBodyDataHolder.java:155) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext.logResponseValue(DefaultDigiwinHttpBodyContext.java:66) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.plugin.tomcat.digiwin.interceptor.CoyoteOutputStreamInterceptor.before(CoyoteOutputStreamInterceptor.java:38) ~[pinpoint-tomcat-plugin-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.interceptor.ExceptionHandleAroundInterceptor.before(ExceptionHandleAroundInterceptor.java:35) ~[?:2.5.1-p1]
	at org.apache.catalina.connector.CoyoteOutputStream.write(CoyoteOutputStream.java) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.springframework.util.FastByteArrayOutputStream.writeTo(FastByteArrayOutputStream.java:248) ~[spring-core-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.util.ContentCachingResponseWrapper.copyBodyToResponse(ContentCachingResponseWrapper.java:228) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.util.ContentCachingResponseWrapper.copyBodyToResponse(ContentCachingResponseWrapper.java:212) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at com.digiwin.gateway.filter.StandardHeaderFilter.doFilter(StandardHeaderFilter.java:66) ~[dwapiplatform-filter-5.2.0.1117.jar:5.2.0.1117]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:544) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:616) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1629) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_372]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_372]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.lang.Thread.run(Thread.java:750) ~[?:1.8.0_372]
09-29 09:14:01.001 [o-32622-exec-64] WARN  .p.p.t.d.i.CoyoteOutputStreamInterceptor -- before. Caused:null
java.lang.ArrayIndexOutOfBoundsException: null
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.logResponseValue(DefaultDigiwinHttpBodyDataHolder.java:155) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext.logResponseValue(DefaultDigiwinHttpBodyContext.java:66) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.plugin.tomcat.digiwin.interceptor.CoyoteOutputStreamInterceptor.before(CoyoteOutputStreamInterceptor.java:38) ~[pinpoint-tomcat-plugin-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.interceptor.ExceptionHandleAroundInterceptor.before(ExceptionHandleAroundInterceptor.java:35) ~[?:2.5.1-p1]
	at org.apache.catalina.connector.CoyoteOutputStream.write(CoyoteOutputStream.java) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.springframework.util.FastByteArrayOutputStream.writeTo(FastByteArrayOutputStream.java:251) ~[spring-core-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.util.ContentCachingResponseWrapper.copyBodyToResponse(ContentCachingResponseWrapper.java:228) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.util.ContentCachingResponseWrapper.copyBodyToResponse(ContentCachingResponseWrapper.java:212) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at com.digiwin.gateway.filter.StandardHeaderFilter.doFilter(StandardHeaderFilter.java:66) ~[dwapiplatform-filter-5.2.0.1117.jar:5.2.0.1117]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) ~[spring-web-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:544) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:616) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1629) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_372]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_372]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.lang.Thread.run(Thread.java:750) ~[?:1.8.0_372]
```

```
09-29 09:12:55.055 [o-32622-exec-32] ERROR n.p.p.c.DefaultDigiwinHttpBodyDataHolder -- exact  body error code  exception:
com.google.gson.JsonSyntaxException: com.google.gson.stream.MalformedJsonException: Use JsonReader.setLenient(true) to accept malformed JSON at line 1 column 241 path $
	at com.google.gson.Gson.assertFullConsumption(Gson.java:1148) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1138) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1047) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:982) ~[gson-2.10.1.jar:?]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.close(DefaultDigiwinHttpBodyDataHolder.java:177) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext.close(DefaultDigiwinHttpBodyContext.java:48) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.plugin.request.ServletRequestListener.destroyed(ServletRequestListener.java:192) ~[?:2.5.1-p1]
	at com.navercorp.pinpoint.plugin.tomcat.javax.interceptor.StandardHostValveInvokeInterceptor.after(StandardHostValveInvokeInterceptor.java:152) ~[pinpoint-tomcat-plugin-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.interceptor.ExceptionHandleAroundInterceptor.after(ExceptionHandleAroundInterceptor.java:44) ~[?:2.5.1-p1]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:196) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:616) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1629) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_372]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_372]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.lang.Thread.run(Thread.java:750) ~[?:1.8.0_372]
Caused by: com.google.gson.stream.MalformedJsonException: Use JsonReader.setLenient(true) to accept malformed JSON at line 1 column 241 path $
	at com.google.gson.stream.JsonReader.syntaxError(JsonReader.java:1659) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.checkLenient(JsonReader.java:1465) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.doPeek(JsonReader.java:551) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.peek(JsonReader.java:433) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.assertFullConsumption(Gson.java:1144) ~[gson-2.10.1.jar:?]
	... 21 more

```

# 调查过程
因为 pinpoint jar 包只放在了容器里，所以本地复现麻烦些，所以准备用 arthas 定位下原因。

反编译一下代码
```
jad com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder
```

```java
[arthas@78]$ jad com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder

ClassLoader:                                                                                                                                      
+-ParallelClassLoader@2093176254{name='pinpoint.agent'}                                                                                           

Location:                                                                                                                                         
/agent_pinpoint/lib/pinpoint-profiler-2.5.1-p1.jar                                                                                                

        /*
         * Decompiled with CFR.
         * 
         * Could not load the following classes:
         *  com.navercorp.pinpoint.bootstrap.config.ProfilerConfig
         *  com.navercorp.pinpoint.bootstrap.context.Trace
         *  com.navercorp.pinpoint.bootstrap.context.TraceId
         *  com.navercorp.pinpoint.bootstrap.logging.PLogger
         *  com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory
         *  com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyDataHolder
         *  com.navercorp.pinpoint.common.util.StringUtils
         *  com.navercorp.pinpoint.profiler.context.Binder
         *  com.navercorp.pinpoint.profiler.context.DigiwinHttpBody
         *  com.navercorp.pinpoint.profiler.context.DigiwinRequestHttpBody
         *  com.navercorp.pinpoint.profiler.context.DigiwinRequestHttpBody$RequestTypEnum
         *  com.navercorp.pinpoint.profiler.context.DigiwinResponseHttpBody
         *  com.navercorp.pinpoint.profiler.context.Reference
         *  com.navercorp.pinpoint.profiler.context.digiwin.dto.DigiwinBusinessExceptionDto
         *  com.navercorp.pinpoint.profiler.context.module.DigiwinHttpBodyDataSender
         *  com.navercorp.pinpoint.profiler.sender.DataSender
         */
        package com.navercorp.pinpoint.profiler.context;
        
        import com.google.gson.Gson;
        import com.google.inject.Inject;
        import com.navercorp.pinpoint.bootstrap.config.ProfilerConfig;
        import com.navercorp.pinpoint.bootstrap.context.Trace;
        import com.navercorp.pinpoint.bootstrap.context.TraceId;
        import com.navercorp.pinpoint.bootstrap.logging.PLogger;
        import com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory;
        import com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyDataHolder;
        import com.navercorp.pinpoint.common.util.StringUtils;
        import com.navercorp.pinpoint.profiler.context.Binder;
        import com.navercorp.pinpoint.profiler.context.DigiwinHttpBody;
        import com.navercorp.pinpoint.profiler.context.DigiwinRequestHttpBody;
        import com.navercorp.pinpoint.profiler.context.DigiwinResponseHttpBody;
        import com.navercorp.pinpoint.profiler.context.Reference;
        import com.navercorp.pinpoint.profiler.context.digiwin.dto.DigiwinBusinessExceptionDto;
        import com.navercorp.pinpoint.profiler.context.module.DigiwinHttpBodyDataSender;
        import com.navercorp.pinpoint.profiler.sender.DataSender;
        import java.util.HashMap;
        import java.util.Map;
        import javax.inject.Named;
        
        public class DefaultDigiwinHttpBodyDataHolder
        implements DigiwinHttpBodyDataHolder {
            private final PLogger logger = PLoggerFactory.getLogger(this.getClass());
            private DataSender digiwinHttpBodyGrpcDataSender;
            private Binder<String> digiwinRequestPathThreadBinder;
            ThreadLocal<Long> digiwinResponseBodySizeThradLocal = new ThreadLocal();
            private final ProfilerConfig profilerConfig;
            Binder<Trace> traceBinder;
            private final String applicationName;
            private Binder<Integer> diwinRequestTypeThreadBinder;
            private Binder<Long> diwinRequestBodySizeThreadBinder;
            private final Binder<Boolean> digiwinEaiExceptionThreadBinder;
            private final Map<String, Binder> binderMap = new HashMap<String, Binder>();
            private final Binder<byte[]> bodyValueBinder;
            private final int MAX_LEGTH;
            private final Gson GSON = new Gson();
        
            public void logRequestPath(String path) {
/* 86*/         Reference pathReference = this.digiwinRequestPathThreadBinder.get();
/* 87*/         pathReference.set((Object)path);
            }
        
            public boolean logDigiwinHttpRequestBody(TraceId traceId, String applicationName) {
                String binderPath = (String)this.digiwinRequestPathThreadBinder.get().get();
                String realPath = StringUtils.isEmpty((String)binderPath) ? "" : binderPath;
                DigiwinRequestHttpBody digiwinHttpRequestBody = new DigiwinRequestHttpBody(traceId, traceId.getSpanId(), ((Long)this.diwinRequestBodySizeThreadBinder.get().get()).longValue(), applicationName, realPath, DigiwinRequestHttpBody.RequestTypEnum.valueToEnum((int)((Integer)this.diwinRequestTypeThreadBinder.get().get())));
/*101*/         return this.digiwinHttpBodyGrpcDataSender.send((Object)digiwinHttpRequestBody);
            }
        
            public boolean logDigiwinHttpResponseBody(TraceId traceId, long bodySize, String applicationName) {
                String binderPath = (String)this.digiwinRequestPathThreadBinder.get().get();
                String realPath = StringUtils.isEmpty((String)binderPath) ? "" : binderPath;
                DigiwinHttpBody digiwinHttpResponseBody = new DigiwinHttpBody(traceId, traceId.getSpanId(), bodySize, applicationName, realPath);
/*112*/         return this.digiwinHttpBodyGrpcDataSender.send((Object)digiwinHttpResponseBody);
            }
        
            public boolean logResponseSize(long size) {
                Long oldSize = this.digiwinResponseBodySizeThradLocal.get();
/*119*/         oldSize = null == oldSize ? Long.valueOf(0L + size) : Long.valueOf(oldSize + size);
/*124*/         this.digiwinResponseBodySizeThradLocal.set(oldSize);
/*125*/         return true;
            }
        
            public void logRequestType(Integer type) {
/*130*/         this.diwinRequestTypeThreadBinder.get().set((Object)type);
            }
        
            public void logRequestBodySize(Long requestBodySize) {
/*135*/         this.diwinRequestBodySizeThreadBinder.get().set((Object)requestBodySize);
            }
        
            public void logResponseValue(byte[] b, int off, int len) {
                Long currentSize;
                byte[] bytes = (byte[])this.bodyValueBinder.get().get();
/*141*/         if (bytes == null) {
/*142*/             bytes = new byte[this.MAX_LEGTH];
/*144*/             this.bodyValueBinder.get().set((Object)bytes);
                }
/*147*/         if ((currentSize = this.digiwinResponseBodySizeThradLocal.get()) == null) {
/*150*/             currentSize = 0L;
/*151*/             this.digiwinResponseBodySizeThradLocal.set(currentSize);
                }
/*154*/         if (currentSize < (long)this.MAX_LEGTH || currentSize + (long)len < (long)this.MAX_LEGTH) {
/*155*/             System.arraycopy(b, off, bytes, Math.toIntExact(currentSize), len);
                }
            }
        
            public void setFiled(String filedName, Object value) {
                Binder binder = this.binderMap.get(filedName);
/*200*/         if (null != binder) {
/*201*/             binder.get().set(value);
                }
            }
        
            public <T> T getFiled(String filedName, Class<T> type) {
                Binder binder = this.binderMap.get(filedName);
/*209*/         if (null != binder) {
/*210*/             return (T)binder.get().get();
                }
/*213*/         return null;
            }
        
            @Inject
            public DefaultDigiwinHttpBodyDataHolder(@DigiwinHttpBodyDataSender DataSender digiwinHttpBodyGrpcDataSender, @Named(value="digiwinRequestPathThreadBinder") Binder<String> digiwinRequestPathThreadBinder, Binder<Trace> traceBinder, ProfilerConfig profilerConfig, @Named(value="diwinRequestTypeThreadBinder") Binder<Integer> diwinRequestTypeThreadBinder, @Named(value="diwinRequestBodySizeThreadBinder") Binder<Long> diwinRequestBodySizeThreadBinder, @Named(value="digiwinEaiExceptionThreadBinder") Binder<Boolean> digiwinEaiExceptionThreadBinder, @Named(value="diwinResponseBodyValueThreadBinder") Binder<byte[]> bodyValueBinder) {
/* 60*/         this.digiwinHttpBodyGrpcDataSender = digiwinHttpBodyGrpcDataSender;
/* 61*/         this.digiwinRequestPathThreadBinder = digiwinRequestPathThreadBinder;
/* 62*/         this.diwinRequestTypeThreadBinder = diwinRequestTypeThreadBinder;
/* 63*/         this.diwinRequestBodySizeThreadBinder = diwinRequestBodySizeThreadBinder;
/* 64*/         this.traceBinder = traceBinder;
/* 65*/         this.profilerConfig = profilerConfig;
/* 66*/         this.digiwinEaiExceptionThreadBinder = digiwinEaiExceptionThreadBinder;
/* 67*/         this.applicationName = profilerConfig.readString("pinpoint.applicationName", "unknown_applicationName");
/* 68*/         this.bodyValueBinder = bodyValueBinder;
/* 71*/         this.MAX_LEGTH = profilerConfig.readInt("digiwin.profiler.error.response.max.length", 5120);
/* 74*/         this.binderMap.put("digiwinRequestPathThreadBinder", digiwinRequestPathThreadBinder);
/* 75*/         this.binderMap.put("diwinRequestTypeThreadBinder", diwinRequestTypeThreadBinder);
/* 77*/         this.binderMap.put("diwinRequestBodySizeThreadBinder", diwinRequestBodySizeThreadBinder);
/* 78*/         this.binderMap.put("digiwinEaiExceptionThreadBinder", digiwinEaiExceptionThreadBinder);
/* 79*/         this.binderMap.put("diwinResponseBodyValueThreadBinder", bodyValueBinder);
            }
        
            public void close(int statusCode) {
                String binderPath = (String)this.digiwinRequestPathThreadBinder.get().get();
                String realPath = StringUtils.isEmpty((String)binderPath) ? "" : binderPath;
/*164*/         Long bodySize = this.digiwinResponseBodySizeThradLocal.get();
                Trace trace = (Trace)this.traceBinder.get().get();
/*167*/         if (null == trace) {
/*168*/             return;
                }
/*172*/         byte[] bytes = (byte[])this.bodyValueBinder.get().get();
/*173*/         String errApp = null;
/*174*/         if (bytes != null) {
                    try {
                        String body = new String(bytes, 0, Math.toIntExact(bodySize));
/*177*/                 DigiwinBusinessExceptionDto digiwinBusinessExceptionDto = this.GSON.fromJson(body, DigiwinBusinessExceptionDto.class);
/*179*/                 if (statusCode != 200 && statusCode != 404 && digiwinBusinessExceptionDto.vailed()) {
/*180*/                     errApp = digiwinBusinessExceptionDto.extractErrorAppCode();
                        }
                    }
                    catch (Throwable e) {
/*183*/                 this.logger.error("exact  body error code  exception:", e);
                    }
                }
/*187*/         TraceId traceId = trace.getTraceId();
/*188*/         this.logDigiwinHttpRequestBody(traceId, this.applicationName);
                DigiwinResponseHttpBody digiwinHttpResponseBody = new DigiwinResponseHttpBody(traceId, traceId.getSpanId(), null == bodySize ? 0L : bodySize, this.applicationName, realPath, statusCode, System.currentTimeMillis() - trace.getStartTime(), errApp);
/*191*/         this.digiwinHttpBodyGrpcDataSender.send((Object)digiwinHttpResponseBody);
/*192*/         this.digiwinResponseBodySizeThradLocal.remove();
/*193*/         this.digiwinRequestPathThreadBinder.get().clear();
/*194*/         this.digiwinEaiExceptionThreadBinder.get().clear();
            }
        }
```

看起来就进行了数组拷贝。
那么推测原因可能是 byte[] 不完整

继续顺着堆栈来看

```
jad com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext
```
```java
[arthas@78]$ jad com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext


ClassLoader:                                                                                                                                      
+-ParallelClassLoader@2093176254{name='pinpoint.agent'}                                                                                           

Location:                                                                                                                                         
/agent_pinpoint/lib/pinpoint-profiler-2.5.1-p1.jar                                                                                                

       /*
        * Decompiled with CFR.
        * 
        * Could not load the following classes:
        *  com.navercorp.pinpoint.bootstrap.logging.PLogger
        *  com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory
        *  com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyContext
        *  com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyDataHolder
        */
       package com.navercorp.pinpoint.profiler.context;
       
       import com.google.inject.Inject;
       import com.navercorp.pinpoint.bootstrap.logging.PLogger;
       import com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory;
       import com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyContext;
       import com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyDataHolder;
       
       public class DefaultDigiwinHttpBodyContext
       implements DigiwinHttpBodyContext {
           private final PLogger logger = PLoggerFactory.getLogger(this.getClass());
           DigiwinHttpBodyDataHolder digiwinHttpBodyDataHolder;
       
           public void logRequestType(Integer type) {
/*32*/         this.digiwinHttpBodyDataHolder.logRequestType(type);
           }
       
           public void logRequestBodySize(Long requestBodySize) {
/*37*/         this.digiwinHttpBodyDataHolder.logRequestBodySize(requestBodySize);
           }
       
           public void logResponseValue(byte[] b, int off, int len) {
/*66*/         this.digiwinHttpBodyDataHolder.logResponseValue(b, off, len);
           }
       
           public void setFiled(String filedName, Object value) {
/*56*/         this.digiwinHttpBodyDataHolder.setFiled(filedName, value);
           }
       
           public <T> T getFiled(String filedName, Class<T> type) {
/*61*/         return (T)this.digiwinHttpBodyDataHolder.getFiled(filedName, type);
           }
       
           public DigiwinHttpBodyDataHolder getDigiwinHttpBodyDataHolder() {
/*22*/         return this.digiwinHttpBodyDataHolder;
           }
       
           public void logResponseBodySize(long size) {
/*42*/         this.digiwinHttpBodyDataHolder.logResponseSize(size);
           }
       
           @Inject
           public DefaultDigiwinHttpBodyContext(DigiwinHttpBodyDataHolder digiwinHttpBodyDataHolder) {
/*17*/         this.digiwinHttpBodyDataHolder = digiwinHttpBodyDataHolder;
           }
       
           public void init() {
           }
       
           public void close(int statusCode) {
               try {
/*48*/             this.digiwinHttpBodyDataHolder.close(statusCode);
               }
               catch (Throwable e) {
/*50*/             this.logger.error("DefaultDigiwinHttpBodyContext  close error", e);
               }
           }
       }

```

继续朝上面查找

```
jad com.navercorp.pinpoint.plugin.tomcat.digiwin.interceptor.CoyoteOutputStreamInterceptor
```
```java
[arthas@78]$ jad com.navercorp.pinpoint.plugin.tomcat.digiwin.interceptor.CoyoteOutputStreamInterceptor

ClassLoader:                                                                                                                                      
+-sun.misc.Launcher$AppClassLoader@18b4aac2                                                                                                       
  +-sun.misc.Launcher$ExtClassLoader@498a612d                                                                                                     

Location:                                                                                                                                         
/agent_pinpoint/plugin/pinpoint-tomcat-plugin-2.5.1-p1.jar                                                                                        

       /*
        * Decompiled with CFR.
        * 
        * Could not load the following classes:
        *  com.navercorp.pinpoint.bootstrap.context.TraceContext
        *  com.navercorp.pinpoint.bootstrap.interceptor.AroundInterceptor
        *  com.navercorp.pinpoint.bootstrap.logging.PLogger
        *  com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory
        *  com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyContext
        */
       package com.navercorp.pinpoint.plugin.tomcat.digiwin.interceptor;
       
       import com.navercorp.pinpoint.bootstrap.context.TraceContext;
       import com.navercorp.pinpoint.bootstrap.interceptor.AroundInterceptor;
       import com.navercorp.pinpoint.bootstrap.logging.PLogger;
       import com.navercorp.pinpoint.bootstrap.logging.PLoggerFactory;
       import com.navercorp.pinpoint.bootstrap.plugin.http.digiwin.DigiwinHttpBodyContext;
       
       public class CoyoteOutputStreamInterceptor
       implements AroundInterceptor {
           private final PLogger logger = PLoggerFactory.getLogger(this.getClass());
           private final TraceContext traceContext;
       
           public CoyoteOutputStreamInterceptor(TraceContext traceContext) {
/*14*/         this.traceContext = traceContext;
           }
       
           public void before(Object target, Object[] args) {
/*20*/         int len = 0;
               try {
                   byte[] b;
/*22*/             DigiwinHttpBodyContext digiwinHttpBodyContext = this.traceContext.getDigiwinHttpBodyContext();
/*23*/             if (args.length == 1) {
                       // empty if block
                   }
/*25*/             if (args.length == 1 && args[0].getClass().isArray()) {
/*26*/                 b = (byte[])args[0];
/*27*/                 len = b.length;
/*28*/                 digiwinHttpBodyContext.logResponseValue(b, 0, len);
                   }
/*31*/             if (args.length == 3) {
/*33*/                 b = (byte[])args[0];
/*34*/                 int off = (Integer)args[1];
/*35*/                 len = (Integer)args[2];
/*38*/                 digiwinHttpBodyContext.logResponseValue(b, off, len);
                   }
/*41*/             if (null != digiwinHttpBodyContext) {
/*42*/                 digiwinHttpBodyContext.logResponseBodySize((long)len);
                   }
               }
               catch (Throwable t) {
/*46*/             this.logger.warn("before. Caused:{}", (Object)t.getMessage(), (Object)t);
               }
           }
       
           public void after(Object target, Object[] args, Object result, Throwable throwable) {
           }
       }
```

这里应该可以推测是 args[1] args[2] 的参数和 0 不匹配
```java
 /*
         * Decompiled with CFR.
         * 
         * Could not load the following classes:
         *  com.navercorp.pinpoint.bootstrap.interceptor.AroundInterceptor
         *  com.navercorp.pinpoint.bootstrap.interceptor.Interceptor
         *  com.navercorp.pinpoint.bootstrap.interceptor.registry.InterceptorRegistry
         */
        package org.apache.catalina.connector;
        
        import com.navercorp.pinpoint.bootstrap.interceptor.AroundInterceptor;
        import com.navercorp.pinpoint.bootstrap.interceptor.Interceptor;
        import com.navercorp.pinpoint.bootstrap.interceptor.registry.InterceptorRegistry;
        import java.io.IOException;
        import java.nio.ByteBuffer;
        import javax.servlet.ServletOutputStream;
        import javax.servlet.WriteListener;
        import org.apache.catalina.connector.OutputBuffer;
        import org.apache.tomcat.util.res.StringManager;
        
        public class CoyoteOutputStream
        extends ServletOutputStream {
            protected static final StringManager sm = StringManager.getManager(CoyoteOutputStream.class);
            protected OutputBuffer ob;
        
            protected CoyoteOutputStream(OutputBuffer ob) {
/* 47*/         this.ob = ob;
            }
        
            protected Object clone() throws CloneNotSupportedException {
                throw new CloneNotSupportedException();
            }
        
            void clear() {
/* 70*/         this.ob = null;
            }
        
            /*
             * WARNING - void declaration
             */
            @Override
            public void write(int n) throws IOException {
                Interceptor _$PINPOINT$_interceptor = InterceptorRegistry.getInterceptor((int)2344);
                Object _$PINPOINT$_result = null;
                Throwable _$PINPOINT$_throwable2 = null;
                Object[] _$PINPOINT$_args = new Object[]{new Integer(n)};
                ((AroundInterceptor)_$PINPOINT$_interceptor).before((Object)this, _$PINPOINT$_args);
                try {
                    void i;
/* 79*/             boolean nonBlocking = this.checkNonBlockingWrite();
/* 80*/             this.ob.writeByte((int)i);
/* 81*/             if (nonBlocking) {
/* 82*/                 this.checkRegisterForWrite();
                    }
/* 84*/             _$PINPOINT$_result = null;
                    _$PINPOINT$_throwable2 = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    return;
                }
                catch (Throwable _$PINPOINT$_throwable2) {
                    _$PINPOINT$_result = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    throw _$PINPOINT$_throwable2;
                }
            }
        
            /*
             * WARNING - void declaration
             */
            @Override
            public void write(byte[] byArray) throws IOException {
                Interceptor _$PINPOINT$_interceptor = InterceptorRegistry.getInterceptor((int)2345);
                Object _$PINPOINT$_result = null;
                Throwable _$PINPOINT$_throwable2 = null;
                Object[] _$PINPOINT$_args = new Object[]{byArray};
                ((AroundInterceptor)_$PINPOINT$_interceptor).before((Object)this, _$PINPOINT$_args);
                try {
                    void b;
/* 89*/             this.write((byte[])b, 0, ((void)b).length);
/* 90*/             _$PINPOINT$_result = null;
                    _$PINPOINT$_throwable2 = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    return;
                }
                catch (Throwable _$PINPOINT$_throwable2) {
                    _$PINPOINT$_result = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    throw _$PINPOINT$_throwable2;
                }
            }
        
            /*
             * WARNING - void declaration
             */
            @Override
            public void write(byte[] byArray, int n, int n2) throws IOException {
                Interceptor _$PINPOINT$_interceptor = InterceptorRegistry.getInterceptor((int)2346);
                Object _$PINPOINT$_result = null;
                Throwable _$PINPOINT$_throwable2 = null;
                Object[] _$PINPOINT$_args = new Object[]{byArray, new Integer(n), new Integer(n2)};
                ((AroundInterceptor)_$PINPOINT$_interceptor).before((Object)this, _$PINPOINT$_args);
                try {
                    void len;
                    void off;
                    void b;
/* 95*/             boolean nonBlocking = this.checkNonBlockingWrite();
/* 96*/             this.ob.write((byte[])b, (int)off, (int)len);
/* 97*/             if (nonBlocking) {
/* 98*/                 this.checkRegisterForWrite();
                    }
/*100*/             _$PINPOINT$_result = null;
                    _$PINPOINT$_throwable2 = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    return;
                }
                catch (Throwable _$PINPOINT$_throwable2) {
                    _$PINPOINT$_result = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    throw _$PINPOINT$_throwable2;
                }
            }
        
            /*
             * WARNING - void declaration
             */
            public void write(ByteBuffer byteBuffer) throws IOException {
                Interceptor _$PINPOINT$_interceptor = InterceptorRegistry.getInterceptor((int)2347);
                Object _$PINPOINT$_result = null;
                Throwable _$PINPOINT$_throwable2 = null;
                Object[] _$PINPOINT$_args = new Object[]{byteBuffer};
                ((AroundInterceptor)_$PINPOINT$_interceptor).before((Object)this, _$PINPOINT$_args);
                try {
                    void from;
/*104*/             boolean nonBlocking = this.checkNonBlockingWrite();
/*105*/             this.ob.write((ByteBuffer)from);
/*106*/             if (nonBlocking) {
/*107*/                 this.checkRegisterForWrite();
                    }
/*109*/             _$PINPOINT$_result = null;
                    _$PINPOINT$_throwable2 = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    return;
                }
                catch (Throwable _$PINPOINT$_throwable2) {
                    _$PINPOINT$_result = null;
                    ((AroundInterceptor)_$PINPOINT$_interceptor).after((Object)this, _$PINPOINT$_args, _$PINPOINT$_result, _$PINPOINT$_throwable2);
                    throw _$PINPOINT$_throwable2;
                }
            }
        
            @Override
            public void flush() throws IOException {
/*117*/         boolean nonBlocking = this.checkNonBlockingWrite();
/*118*/         this.ob.flush();
/*119*/         if (nonBlocking) {
/*120*/             this.checkRegisterForWrite();
                }
            }
        
            private boolean checkNonBlockingWrite() {
                boolean nonBlocking;
/*134*/         boolean bl = nonBlocking = !this.ob.isBlocking();
/*135*/         if (nonBlocking && !this.ob.isReady()) {
                    throw new IllegalStateException(sm.getString("coyoteOutputStream.nbNotready"));
                }
/*138*/         return nonBlocking;
            }
        
            private void checkRegisterForWrite() {
/*151*/         this.ob.checkRegisterForWrite();
            }
        
            @Override
            public void close() throws IOException {
/*157*/         this.ob.close();
            }
        
            @Override
            public boolean isReady() {
/*162*/         return this.ob.isReady();
            }
        
            @Override
            public void setWriteListener(WriteListener listener) {
/*168*/         this.ob.setWriteListener(listener);
            }
        }
```

一直找上去都没有发现问题，那么写入本身为什么会报错呢？

查看报错的代码入参

```java
[arthas@78]$ watch com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder logResponseValue "{params}" -e -x 2
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 234 ms, listenerId: 4
method=com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.logResponseValue location=AtExceptionExit
ts=2024-09-29 10:11:17.615; [cost=0.155178ms] result=@ArrayList[
    @Object[][
        @byte[][isEmpty=false;size=8192],
        @Integer[0],
        @Integer[6131],
    ],
]
method=com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.logResponseValue location=AtExceptionExit
ts=2024-09-29 10:11:18.614; [cost=0.056122ms] result=@ArrayList[
    @Object[][
        @byte[][isEmpty=false;size=8192],
        @Integer[0],
        @Integer[6159],
    ],
]
```

```log
method=com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.logResponseValue location=AtExceptionExit
ts=2024-09-29 10:21:54.738; [cost=0.066177ms] result=@ArrayList[
    @Object[][
        @byte[][isEmpty=false;size=8192],
        @Integer[0],
        @Integer[6170],
    ],
    @DefaultDigiwinHttpBodyDataHolder[
        logger=@Log4j2PLoggerAdapter[com.navercorp.pinpoint.profiler.logging.Log4j2PLoggerAdapter@1ba88f5b],
        digiwinHttpBodyGrpcDataSender=@DigiwinHttpBodyGrpcDataSender[DigiwinHttpBodyGrpcDataSender{name='SpanGrpcDataSender', host='60.204.141.105', port=9998} com.navercorp.pinpoint.profiler.sender.DigiwinHttpBodyGrpcDataSender@6b8fe2b0],
        digiwinRequestPathThreadBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@5b0a671f],
        digiwinResponseBodySizeThradLocal=@ThreadLocal[java.lang.ThreadLocal@557df459],
        profilerConfig=@DefaultProfilerConfig[DefaultProfilerConfig{profileEnable='true', activeProfile=release, logDirMaxBackupSize=5, staticResourceCleanup=false, jdbcSqlCacheSize=1024, traceSqlBindValue=true, maxSqlBindValueSize=1024, httpStatusCodeErrors=HttpStatusCodeErrors{errors=[5xx]}, injectionModuleFactoryClazzName='null', applicationNamespace=''}],
        traceBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@a707bfa],
        applicationName=@String[Ali_PaaS_datamap],
        diwinRequestTypeThreadBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@1e7e9266],
        diwinRequestBodySizeThreadBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@16c43c29],
        digiwinEaiExceptionThreadBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@2ca1cd94],
        binderMap=@HashMap[isEmpty=false;size=5],
        bodyValueBinder=@ThreadLocalBinder[com.navercorp.pinpoint.profiler.context.ThreadLocalBinder@2507cc51],
        MAX_LEGTH=@Integer[5125],
        GSON=@Gson[{serializeNulls:false,factories:[Factory[typeHierarchy=com.google.gson.JsonElement,adapter=com.google.gson.internal.bind.TypeAdapters$28@4f008735], com.google.gson.internal.bind.ObjectTypeAdapter$1@7892cf88, com.google.gson.internal.Excluder@5a7d6063, Factory[type=java.lang.String,adapter=com.google.gson.internal.bind.TypeAdapters$15@41034a39], Factory[type=java.lang.Integer+int,adapter=com.google.gson.internal.bind.TypeAdapters$7@af15403], Factory[type=java.lang.Boolean+boolean,adapter=com.google.gson.internal.bind.TypeAdapters$3@64f1245a], Factory[type=java.lang.Byte+byte,adapter=com.google.gson.internal.bind.TypeAdapters$5@553e2edf], Factory[type=java.lang.Short+short,adapter=com.google.gson.internal.bind.TypeAdapters$6@e37e4c3], Factory[type=java.lang.Long+long,adapter=com.google.gson.internal.bind.TypeAdapters$11@e66e84a], Factory[type=java.lang.Double+double,adapter=com.google.gson.Gson$1@6358de28], Factory[type=java.lang.Float+float,adapter=com.google.gson.Gson$2@47344c4a], com.google.gson.internal.bind.NumberTypeAdapter$1@7614a993, Factory[type=java.util.concurrent.atomic.AtomicInteger,adapter=com.google.gson.TypeAdapter$1@4f893ba3], Factory[type=java.util.concurrent.atomic.AtomicBoolean,adapter=com.google.gson.TypeAdapter$1@6a8515e4], Factory[type=java.util.concurrent.atomic.AtomicLong,adapter=com.google.gson.TypeAdapter$1@f3652e2], Factory[type=java.util.concurrent.atomic.AtomicLongArray,adapter=com.google.gson.TypeAdapter$1@5cbd2aa6], Factory[type=java.util.concurrent.atomic.AtomicIntegerArray,adapter=com.google.gson.TypeAdapter$1@5a6f56b1], Factory[type=java.lang.Character+char,adapter=com.google.gson.internal.bind.TypeAdapters$14@1807522d], Factory[type=java.lang.StringBuilder,adapter=com.google.gson.internal.bind.TypeAdapters$19@259ba28d], Factory[type=java.lang.StringBuffer,adapter=com.google.gson.internal.bind.TypeAdapters$20@10c64482], Factory[type=java.math.BigDecimal,adapter=com.google.gson.internal.bind.TypeAdapters$16@30bc42f5], Factory[type=java.math.BigInteger,adapter=com.google.gson.internal.bind.TypeAdapters$17@1288f8f5], Factory[type=com.google.gson.internal.LazilyParsedNumber,adapter=com.google.gson.internal.bind.TypeAdapters$18@6a7fbfe3], Factory[type=java.net.URL,adapter=com.google.gson.internal.bind.TypeAdapters$21@48bf047a], Factory[type=java.net.URI,adapter=com.google.gson.internal.bind.TypeAdapters$22@1a464c72], Factory[type=java.util.UUID,adapter=com.google.gson.internal.bind.TypeAdapters$24@4f5c509f], Factory[type=java.util.Currency,adapter=com.google.gson.TypeAdapter$1@584b7e30], Factory[type=java.util.Locale,adapter=com.google.gson.internal.bind.TypeAdapters$27@68672d97], Factory[typeHierarchy=java.net.InetAddress,adapter=com.google.gson.internal.bind.TypeAdapters$23@406ad935], Factory[type=java.util.BitSet,adapter=com.google.gson.TypeAdapter$1@6d1fa10f], com.google.gson.internal.bind.DateTypeAdapter$1@6e26f862, Factory[type=java.util.Calendar+java.util.GregorianCalendar,adapter=com.google.gson.internal.bind.TypeAdapters$26@3f7c6ba4], com.google.gson.internal.sql.SqlTimeTypeAdapter$1@29c9be0f, com.google.gson.internal.sql.SqlDateTypeAdapter$1@39ce75e0, com.google.gson.internal.sql.SqlTimestampTypeAdapter$1@60727f53, com.google.gson.internal.bind.ArrayTypeAdapter$1@3c8f19b2, Factory[type=java.lang.Class,adapter=com.google.gson.TypeAdapter$1@58d2f1ea], com.google.gson.internal.bind.CollectionTypeAdapterFactory@12f9fe65, com.google.gson.internal.bind.MapTypeAdapterFactory@6182838e, com.google.gson.internal.bind.JsonAdapterAnnotationTypeAdapterFactory@25044485, com.google.gson.internal.bind.TypeAdapters$29@6a16ac6e, com.google.gson.internal.bind.ReflectiveTypeAdapterFactory@37416d9f],instanceCreators:{}}],
    ],
]
[arthas@78]$ 
```

所以报错的原因应该是 currentSize + len 超过了 b 数组的长度了。
也就是说 b 和 currentSize 应该不匹配
```
System.arraycopy(b, off, bytes, Math.toIntExact(currentSize), len);
```

那么也就是要看
digiwinResponseBodySizeThradLocal 是在哪里设置值，以及他的值是如何流转的。

那么得下载这个包 pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1 再看。

# 从另一个方向调查下


```
09-29 09:12:55.055 [o-32622-exec-32] ERROR n.p.p.c.DefaultDigiwinHttpBodyDataHolder -- exact  body error code  exception:
com.google.gson.JsonSyntaxException: com.google.gson.stream.MalformedJsonException: Use JsonReader.setLenient(true) to accept malformed JSON at line 1 column 241 path $
	at com.google.gson.Gson.assertFullConsumption(Gson.java:1148) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1138) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1047) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:982) ~[gson-2.10.1.jar:?]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.close(DefaultDigiwinHttpBodyDataHolder.java:177) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext.close(DefaultDigiwinHttpBodyContext.java:48) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.plugin.request.ServletRequestListener.destroyed(ServletRequestListener.java:192) ~[?:2.5.1-p1]
	at com.navercorp.pinpoint.plugin.tomcat.javax.interceptor.StandardHostValveInvokeInterceptor.after(StandardHostValveInvokeInterceptor.java:152) ~[pinpoint-tomcat-plugin-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.interceptor.ExceptionHandleAroundInterceptor.after(ExceptionHandleAroundInterceptor.java:44) ~[?:2.5.1-p1]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:196) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:616) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1629) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_372]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_372]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.lang.Thread.run(Thread.java:750) ~[?:1.8.0_372]
Caused by: com.google.gson.stream.MalformedJsonException: Use JsonReader.setLenient(true) to accept malformed JSON at line 1 column 241 path $
	at com.google.gson.stream.JsonReader.syntaxError(JsonReader.java:1659) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.checkLenient(JsonReader.java:1465) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.doPeek(JsonReader.java:551) ~[gson-2.10.1.jar:?]
	at com.google.gson.stream.JsonReader.peek(JsonReader.java:433) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.assertFullConsumption(Gson.java:1144) ~[gson-2.10.1.jar:?]
	... 21 more

```

```java
public void close(int statusCode) {
                String binderPath = (String)this.digiwinRequestPathThreadBinder.get().get();
                String realPath = StringUtils.isEmpty((String)binderPath) ? "" : binderPath;
/*164*/         Long bodySize = this.digiwinResponseBodySizeThradLocal.get();
                Trace trace = (Trace)this.traceBinder.get().get();
/*167*/         if (null == trace) {
/*168*/             return;
                }
/*172*/         byte[] bytes = (byte[])this.bodyValueBinder.get().get();
/*173*/         String errApp = null;
/*174*/         if (bytes != null) {
                    try {
                        String body = new String(bytes, 0, Math.toIntExact(bodySize));
/*177*/                 DigiwinBusinessExceptionDto digiwinBusinessExceptionDto = this.GSON.fromJson(body, DigiwinBusinessExceptionDto.class);
/*179*/                 if (statusCode != 200 && statusCode != 404 && digiwinBusinessExceptionDto.vailed()) {
/*180*/                     errApp = digiwinBusinessExceptionDto.extractErrorAppCode();
                        }
                    }
                    catch (Throwable e) {
/*183*/                 this.logger.error("exact  body error code  exception:", e);
                    }
                }
/*187*/         TraceId traceId = trace.getTraceId();
/*188*/         this.logDigiwinHttpRequestBody(traceId, this.applicationName);
                DigiwinResponseHttpBody digiwinHttpResponseBody = new DigiwinResponseHttpBody(traceId, traceId.getSpanId(), null == bodySize ? 0L : bodySize, this.applicationName, realPath, statusCode, System.currentTimeMillis() - trace.getStartTime(), errApp);
/*191*/         this.digiwinHttpBodyGrpcDataSender.send((Object)digiwinHttpResponseBody);
/*192*/         this.digiwinResponseBodySizeThradLocal.remove();
/*193*/         this.digiwinRequestPathThreadBinder.get().clear();
/*194*/         this.digiwinEaiExceptionThreadBinder.get().clear();
            }
        }
```

```
09-29 09:10:59.059 [o-32622-exec-31] ERROR n.p.p.c.DefaultDigiwinHttpBodyDataHolder -- exact  body error code  exception:
com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was STRING at line 1 column 1 path $
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:397) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1227) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1137) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:1047) ~[gson-2.10.1.jar:?]
	at com.google.gson.Gson.fromJson(Gson.java:982) ~[gson-2.10.1.jar:?]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyDataHolder.close(DefaultDigiwinHttpBodyDataHolder.java:177) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.profiler.context.DefaultDigiwinHttpBodyContext.close(DefaultDigiwinHttpBodyContext.java:48) ~[pinpoint-profiler-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.plugin.request.ServletRequestListener.destroyed(ServletRequestListener.java:192) ~[?:2.5.1-p1]
	at com.navercorp.pinpoint.plugin.tomcat.javax.interceptor.StandardHostValveInvokeInterceptor.after(StandardHostValveInvokeInterceptor.java:152) ~[pinpoint-tomcat-plugin-2.5.1-p1.jar:2.5.1-p1]
	at com.navercorp.pinpoint.bootstrap.interceptor.ExceptionHandleAroundInterceptor.after(ExceptionHandleAroundInterceptor.java:44) ~[?:2.5.1-p1]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:196) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:616) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1629) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) ~[?:1.8.0_372]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) ~[?:1.8.0_372]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-8.5.64.jar:8.5.64]
	at java.lang.Thread.run(Thread.java:750) ~[?:1.8.0_372]
Caused by: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was STRING at line 1 column 1 path $
	at com.google.gson.stream.JsonReader.beginObject(JsonReader.java:393) ~[gson-2.10.1.jar:?]
	at com.google.gson.internal.bind.ReflectiveTypeAdapterFactory$Adapter.read(ReflectiveTypeAdapterFactory.java:386) ~[gson-2.10.1.jar:?]
	... 22 more
```

```
[arthas@84]$ watch com.google.gson.Gson fromJson "{params}" -e -x 2
method=com.google.gson.Gson.fromJson location=AtExceptionExit
ts=2024-09-29 11:19:30.190; [cost=0.122757ms] result=@ArrayList[
    @Object[][
        @String[Business":false,"performerType":"user","performerValue":"wangpan0920@digiwin.com","performerName":"PR","performerVariable":null,"companyId":null,"config":null,"lang":null,"_mergeRule":null}],"condition":null},"resCode":null,"to":["a26dcd0882439f99b6e74052198199ad"],"category":"PROCESS","config":{"groupField":"","supportPart":false,"supportSplit":false}},"profile":{"tenantName":"智驱中台工作台","tenantSid":593420788953664,"tenantId":"IntelligentDriveCenterWorkbench","userSid":1984610499,"userName":"集成账号","userId":"integration"},"uuid":"","status":200}],
        @TypeToken[com.navercorp.pinpoint.profiler.context.digiwin.dto.DigiwinBusinessExceptionDto],
    ],
]
Press Q or Ctrl+C to abort.
```

发现 json 都不太完整

看起来是 bytes 不完整

到这里基本上没啥头绪了。

# 只能下载 pinpoint 包到本地复现下了

# 最后
相关团队的同事帮忙修改了下问题