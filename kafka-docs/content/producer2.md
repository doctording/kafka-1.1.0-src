# 如何处理粘包/拆包问题？

## Kafka处理粘包和拆包

### 粘包和拆包原因

1. 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去，将会发生粘包；
2. 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包；
3. 要发送的数据大于TCP发送缓冲区剩余空间大小，将会发生拆包；
4. 待发送数据大于MSS（最大报文长度），TCP在传输前将进行拆包。即TCP报文长度-TCP头部长度>MSS。

### NetworkReceive

NetworkReceive，网络接收到的消息的对象：`接收的字节大小` 加上 `具体大小的内容`

```java
/**
 * A size delimited Receive that consists of a 4 byte network-ordered size N followed by N bytes of content
 */
public class NetworkReceive implements Receive {
    
    // Need a method to read from ReadableByteChannel because BlockingChannel requires read with timeout
    // See: http://stackoverflow.com/questions/2866557/timeout-for-socketchannel-doesnt-work
    // This can go away after we get rid of BlockingChannel
    @Deprecated
    public long readFromReadableChannel(ReadableByteChannel channel) throws IOException {
        int read = 0;
        // size 初始值4字节：this.size = ByteBuffer.allocate(4);
        if (size.hasRemaining()) {
            int bytesRead = channel.read(size);
            if (bytesRead < 0)
                throw new EOFException();
            read += bytesRead;
            // 先读取size
            if (!size.hasRemaining()) {
                size.rewind();
                int receiveSize = size.getInt();
                if (receiveSize < 0)
                    throw new InvalidReceiveException("Invalid receive (size = " + receiveSize + ")");
                if (maxSize != UNLIMITED && receiveSize > maxSize)
                    throw new InvalidReceiveException("Invalid receive (size = " + receiveSize + " larger than " + maxSize + ")");
                requestedBufferSize = receiveSize; //may be 0 for some payloads (SASL)
                if (receiveSize == 0) {
                    buffer = EMPTY_BUFFER;
                }
            }
        }
        if (buffer == null && requestedBufferSize != -1) { //we know the size we want but havent been able to allocate it yet
            // 再分配内存
            buffer = memoryPool.tryAllocate(requestedBufferSize);
            if (buffer == null)
                log.trace("Broker low on memory - could not allocate buffer of size {} for source {}", requestedBufferSize, source);
        }
        if (buffer != null) {
            int bytesRead = channel.read(buffer);
            if (bytesRead < 0)
                throw new EOFException();
            read += bytesRead;
        }

        return read;
    }

}
```

* 什么时候读完？

```java
@Override
public boolean complete() {
    return !size.hasRemaining() && buffer != null && !buffer.hasRemaining();
}
```

读不到，并且buffer里面也都被读完了，那么就表示网络消息都读取完成了
