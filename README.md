
## **前言**


在上一篇随笔中，我们探讨了如何实现一套自定义通信协议，其中涉及到的粘包和拆包处理最初是完全自定义实现的，后来则改为了继承 `ByteToMessageDecoder` 来简化处理。


本篇将重点讨论这两种实现方式在缓存管理上的主要区别，并深入分析其中的不同之处以及值得借鉴的经验和技巧。


## 代码回顾


### 1）完全自定义实现


无缓存的情况


* 反复从ByteBuf中提取完整的消息
* 剩余的残缺消息写入缓存（会进行数据拷贝）


有缓存的情况


* 将新收到的数据接入缓存
* 反复从缓存中提取完整消息
* 释放缓存内读取过的数据（会进行数据移动，导致拷贝）




```
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    private static final int HEADER_LENGTH = 4; //消息头部长度
    private ByteBuf buffer = Unpooled.buffer(1024); //缓存残缺消息

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf income = (ByteBuf) msg;

        //上一次有缓存存在，则本数据包不是消息头开头，
        if(buffer.readableBytes() > 0) {
            //进行必要的扩容，下面的readBytes不会自动扩容
            buffer.ensureWritable(income.readableBytes()); 
            income.readBytes(buffer, income.readableBytes());

            readMsgFromBuffer(buffer);

            //剩下一点残缺消息
            if(buffer.readableBytes() > 0) {
                //保留剩下的数据，重置读索引为0
                System.out.println("缓存剩余字节："+buffer.readableBytes());
                buffer.discardReadBytes();
            } else { //刚刚好，则清空数据
                buffer.clear();
            }
        } else {
            readMsgFromBuffer(income);

            //剩下的数据全部写入缓存
            if (income.readableBytes() >0) {
                System.out.println("剩余字节:"+income.readableBytes());
                income.readBytes(buffer, income.readableBytes());
            }
        }

    }

    //从字节数组中读取完整的消息
    private void readMsgFromBuffer(ByteBuf byteBuf) {
        //剩余可读消息是否包含一个消息头
        while(byteBuf.readableBytes() >= HEADER_LENGTH) {
            byteBuf.markReaderIndex(); //由于可能读不到完整的消息，所以读之前先标记索引位置，方便重置
            //读取消息头
            byte[] headerBytes = new byte[4];
            byteBuf.readBytes(headerBytes);
            //获取类型
            int type = headerBytes[0] & 0xFF;
            //获取消息体长度
            int bodyLength = ((headerBytes[1] & 0xFF) << 16) |
                    ((headerBytes[2] & 0xFF) << 8) |
                    (headerBytes[3] & 0xFF);

            //不包含请求体
            if (byteBuf.readableBytes() < bodyLength) {
                byteBuf.resetReaderIndex(); //重置读索引到当前消息头位置
                break;
            }

            // 完整消息体已经接收，处理消息
            byte[] body = new byte[bodyLength];
            byteBuf.readBytes(body);
            //System.out.println("type:"+type+"||length:"+bodyLength+"||body:"+new String(body, CharsetUtil.UTF_8));
            if(type == 1) {
                try {
                    HelloRequest request = HelloRequest.parseFrom(body);
                    System.out.println("收到消息:"+request.toString());
                } catch (Exception e) {
                    System.out.println("解析失败："+new String(body, CharsetUtil.UTF_8));
                }
            } else {
                System.out.println("消息类型未知："+type);
            }

        }
    }

    ....
}
```


### 2）继承ByteToMessageDecoder的实现


使用ByteToMessageDecoder后，数据的解码变得更加简化。只需检查缓冲区是否有足够的数据来提取一个/多个完整的消息。


如果数据不足，解码过程就会结束，无需额外管理缓存。




```
public class MessageDecoder extends ByteToMessageDecoder {
    private static final int HEADER_LENGTH = 4; //消息头部长度

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List out) throws Exception {
        // 检查是否足够的字节来读取一个消息头
        while (in.readableBytes() >= HEADER_LENGTH) {
            in.markReaderIndex(); // 标记当前读取位置，便于重置

            // 读取消息头部
            byte[] headerBytes = new byte[4];
            in.readBytes(headerBytes);

            // 获取类型
            int type = headerBytes[0] & 0xFF;
            // 获取消息体长度
            int bodyLength = ((headerBytes[1] & 0xFF) << 16) |
                    ((headerBytes[2] & 0xFF) << 8) |
                    (headerBytes[3] & 0xFF);

            // 检查缓冲区中的数据是否足够读取整个消息体
            if (in.readableBytes() < bodyLength) {
                in.resetReaderIndex(); // 重置读指针，等待更多数据
                break;
            }

            // 读取消息体
            byte[] body = new byte[bodyLength];
            in.readBytes(body);

            // 处理消息
            try {
                Object msg = null;
                if(type == 1) {
                    msg = HelloRequest.parseFrom(body);
                } else if(type == 2) {
                    msg = HelloResponse.parseFrom(body);
                } else {
                    System.out.println("未知消息："+new String(body, CharsetUtil.UTF_8));
                }
                if(Objects.nonNull(msg)) {
                    out.add(msg);
                }

            } catch (Exception e) {
                System.out.println("解析失败: " + new String(body, CharsetUtil.UTF_8));
            }
        }
    }
}
```


## ByteToMessageDecoder源码


### 核心属性




```
    //缓存
    private ByteBuf cumulation;
    //累加器（用于拼接缓存和新到数据）
    private Cumulator cumulator = MERGE_CUMULATOR;
   
    //X次channelRead之后，释放已读数据
    private int discardAfterReads = 16;
    //累计channelRead次数（每次释放完会重置）
    private int numReads;
```


### 处理流程


1\.新到数据存放到缓冲区（使用累加器Cumulator进行数据合并）


2\.循环调用子类的decode方法，读取消息存入List，直到数据不足


3\.遍历List，依次传递给下一个处理器


### 累加器


提供2种累加器实现，MERGE\_CUMULATOR和COMPOSITE\_CUMULATOR


**1）MERGE\_CUMULATOR（默认实现）**


缓存存在的时候，直接进行数据拷贝，与缓存数据进行整合。


下面的代码可以看到，如果缓冲区空间不够，则会进行扩容操作。


跟自定义实现中的"buffer.ensureWritable(income.readableBytes())"一致。


整体思路跟自定义实现差不多，不过它多考虑了两种情况


* 数据被共享：共享数据会被其他使用者影响，需排除影响
* 数据只读：只读空间无法被写入，而缓冲区是需要写入新数据的




```
    public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
        //cumulation是上一次的缓存，in是新到的数据
        @Override
        public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
            try {
                final ByteBuf buffer;
                if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                    || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
                    // Expand cumulation (by replace it) when either there is not more room in the buffer
                    // or if the refCnt is greater then 1 which may happen when the user use slice().retain() or
                    // duplicate().retain() or if its read-only.
                    //
                    // See:
                    // - https://github.com/netty/netty/issues/2327
                    // - https://github.com/netty/netty/issues/1764
                    buffer = expandCumulation(alloc, cumulation, in.readableBytes());
                } else {
                    buffer = cumulation;
                }
                //新到数据写入缓存
                buffer.writeBytes(in);
                return buffer;
            } finally {
                // We must release in in all cases as otherwise it may produce a leak if writeBytes(...) throw
                // for whatever release (for example because of OutOfMemoryError)
                in.release();
            }
        }
    };
```


**2）COMPOSITE\_CUMULATOR**


 上面的处理，新到数据与缓存的合并是通过数据拷贝。而下面这种方式，则是使用组合（数据没有移动，只是提供一个整合后的视图）




```
  public static final Cumulator COMPOSITE_CUMULATOR = new Cumulator() {
        @Override
        public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
            ByteBuf buffer;
            try {
                if (cumulation.refCnt() > 1) {
                    // Expand cumulation (by replace it) when the refCnt is greater then 1 which may happen when the
                    // user use slice().retain() or duplicate().retain().
                    //
                    // See:
                    // - https://github.com/netty/netty/issues/2327
                    // - https://github.com/netty/netty/issues/1764
                    buffer = expandCumulation(alloc, cumulation, in.readableBytes());
                    buffer.writeBytes(in);
                } else {
                    CompositeByteBuf composite;
                    if (cumulation instanceof CompositeByteBuf) {
                        //上一次缓存已经是组合对象
                        composite = (CompositeByteBuf) cumulation;
                    } else {
                        composite = alloc.compositeBuffer(Integer.MAX_VALUE);
                        //缓存加入组合
                        composite.addComponent(true, cumulation);
                    }
                    //新到数据加入组合
                    composite.addComponent(true, in);
                    in = null;
                    buffer = composite;
                }
                return buffer;
            } finally {
                //由于使用组合方式，数据还在原来的地方。不能直接释放
                if (in != null) {
                    // We must release if the ownership was not transferred as otherwise it may produce a leak if
                    // writeBytes(...) throw for whatever release (for example because of OutOfMemoryError).
                    in.release();
                }
            }
        }
    };
```


### 主要方法——channelRead


在上述的自定义实现中，每次从缓冲区读取完数据，会释放掉已读数据，防止缓存数据无限增长。


buffer.discardReadBytes();


而这里做了优化，累积16次读取后，才会进行释放。（channelReadComplete的时候也会触发）


这样做的好处，就是可以减少数据拷贝的次数。（discard操作会把已读数据清空，重置读索引，然后把剩余数据往前挪）




```
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //仅处理ByteBuf，其他消息直接传给下一个Handler
        if (msg instanceof ByteBuf) {
            CodecOutputList out = CodecOutputList.newInstance();
            try {
                ByteBuf data = (ByteBuf) msg;
               
                first = cumulation == null;
                //缓冲区为空，直接赋值
                if (first) {
                    cumulation = data;
                } else {
                    //使用累加器进行数据合并
                    cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
                }
                //调用子类实现，从缓冲区中解析消息
                callDecode(ctx, cumulation, out);
            } catch (DecoderException e) {
                throw e;
            } catch (Exception e) {
                throw new DecoderException(e);
            } finally {
                if (cumulation != null && !cumulation.isReadable()) {
                    //缓冲区数据刚好读完，清空缓冲区，清空已读次数
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                } else if (++ numReads >= discardAfterReads) {
                    // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                    // See https://github.com/netty/netty/issues/4275
                    //已读数达到限定次数（默认16），释放已读数据
                    numReads = 0;
                    discardSomeReadBytes();
                }

                int size = out.size();
                //是不是没解析到消息
                decodeWasNull = !out.insertSinceRecycled();
                //将解析出来的消息逐个传个下一个Handler
                fireChannelRead(ctx, out, size);
                //清空List，下次再用
                out.recycle();
            }
        } else {
            //直接丢给下一个Handler
            ctx.fireChannelRead(msg);
        }
    }
```


### 主要方法——callDecode


这里主要通过检查List结果集和数据读取情况，来判断要不要结束解码循环。




```
    protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List out) {
        try {
            while (in.isReadable()) {
                //先读取List大小
                int outSize = out.size();
                //有数据，则先传给下一个Handler
                if (outSize > 0) {
                    fireChannelRead(ctx, out, outSize);
                    out.clear();

                    // Check if this handler was removed before continuing with decoding.
                    // If it was removed, it is not safe to continue to operate on the buffer.
                    //
                    // See:
                    // - https://github.com/netty/netty/issues/4635
                    if (ctx.isRemoved()) {
                        break;
                    }
                    outSize = 0;
                }

                //开始之前，先记录可读数据量
                int oldInputLength = in.readableBytes();
                //调用子类decode方法
                decodeRemovalReentryProtection(ctx, in, out);

                // Check if this handler was removed before continuing the loop.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See https://github.com/netty/netty/issues/1664
                if (ctx.isRemoved()) {
                    break;
                }

                //查看子类是否解析出数据
                if (outSize == out.size()) {
                    //数据没被动过，说明没有可解析的数据，直接break
                    if (oldInputLength == in.readableBytes()) {
                        break;
                    } else { //数据有被动过，但还没解析出数据，继续执行
                        continue;
                    }
                }
 
                //List内有新数据，但是数据没有被读过，说明子类实现有问题，报错
                if (oldInputLength == in.readableBytes()) {
                    throw new DecoderException(
                            StringUtil.simpleClassName(getClass()) +
                                    ".decode() did not read anything but decoded a message.");
                }
                //如果只解析一次，则直接结束
                if (isSingleDecode()) {
                    break;
                }
            }
        } catch (DecoderException e) {
            throw e;
        } catch (Exception cause) {
            throw new DecoderException(cause);
        }
    }
```


## 总结


核心内容并无太大差异，但 Netty 提供的抽象类在实现上考虑了更多细节，并经过社区的不断演进，功能变得更加稳定和完善。


因此，推荐继承 `ByteToMessageDecoder` 来实现解码。


 本博客参考[veee加速器](https://blog.liuyunzhuge.com)。转载请注明出处！
