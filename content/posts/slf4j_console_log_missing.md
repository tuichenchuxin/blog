---
title: "Slf4j 控制台日志丢失问题调查"
date: 2024-10-30T17:41:42+08:00
draft: false
---

今天本地调试项目发现长日志始终打不出来，然后 debug 发现, flush 之后，原来打出来一半的日志就会消失

```java
protected void directEncodeEvent(final LogEvent event) {
    getLayout().encode(event, manager);
    if (this.immediateFlush || event.isEndOfBatch()) {
        manager.flush();
    }
}
```

然后我发现超过 2048 之后，会分批输出，奇怪的事为啥输出一半，后一半继续输出之后，就消失了呢。
```java
static void encodeText(final CharsetEncoder charsetEncoder, final CharBuffer charBuf, final ByteBuffer byteBuf,
        final StringBuilder text, final ByteBufferDestination destination)
        throws CharacterCodingException {
    charsetEncoder.reset();
    if (text.length() > charBuf.capacity()) {
        encodeChunkedText(charsetEncoder, charBuf, byteBuf, text, destination);
        return;
    }
    charBuf.clear();
    text.getChars(0, text.length(), charBuf.array(), charBuf.arrayOffset());
    charBuf.limit(text.length());
    final CoderResult result = charsetEncoder.encode(charBuf, byteBuf, true);
    writeEncodedText(charsetEncoder, charBuf, byteBuf, destination, result);
}
```

最后发现这里有个折叠标志，原来超过之后 idea 自动折叠了。。哈哈一个小乌龙

![](/img/QQ_1730281535349.png)
