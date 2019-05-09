### `ByteBuffer` 与 `ByteBuf` 的区别

+ `ByteBuffer` 是 `JDK NIO` 类库的，`ByteBuf` 是 `Netty` 类库的

+ `ByteBuffer` 长度固定，不支持动态拓展，`ByteBuf` 支持动态拓展，但是有最大长度参数限制

+ `ByteBuffer` 只有一个标识位置的指针 `position`，读写时需要手动调用 `flip()` 和 `rewind()`等操作，`ByteBuf` 使用两个标识位置指针（`readerIndex`、`writerIndex`），读写时不需要手动做什么操作