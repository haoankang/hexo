---
title: IO二三事2——NIO
date: 2019-11-11 20:29:25
categories:
tags: I/O
---
## 一些基础
1. 什么是NIO.
   >NIO(Non-blocking I/O,也被称为New I/O),是一种同步非阻塞I/O模型(select/poll)，也是I/O多路复用的基础(epoll)，
   常用于解决高并发与大量连接的I/O问题；
2. NIO的核心概念.
    >NIO的核心是Buffer、Channel和selector；
    >* 缓冲区Buffer是一个数组对象，NIO库中，所有数据都是用缓冲区处理的；因此熟悉缓冲区的api和特性
    至关重要；
    >* 通道Channel是双向的，网络数据通过Channel读取和写入，分为两大类：用于网络读写的SelectableChannel和用于文件操作的FileChannel；
    >* 多路复用器Selector，提供选择已就绪任务的能力；多路复用器的底层系统调用有select/poll/epoll，
    epoll的优点：没有最大1024的连接句柄限制，事件驱动方式，mmap映射内存更高效；

3. NIO服务端的简单示例.
    ```java
    public class NioServerBetter {
    
        private String ip;
        private int port;
    
        private AtomicBoolean flag = new AtomicBoolean(true);
    
        public void setFlag(Boolean flg) {
            this.flag.set(flg);
        }
    
        public NioServerBetter(String ip, int port){
            this.ip = ip;
            this.port = port;
        }
    
        void run(){
            try {
                ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
                serverSocketChannel.configureBlocking(false);
                serverSocketChannel.bind(new InetSocketAddress(port));
                Selector selector = Selector.open();
                serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
    
                while(flag.get()){
                    selector.select(1000);
    
                    Set<SelectionKey> selectionKeySet = selector.selectedKeys();
                    Iterator<SelectionKey> iterator = selectionKeySet.iterator();
                    while(iterator.hasNext()){
                        SelectionKey selectionKey = iterator.next();
                        iterator.remove();
                        handleKey(selectionKey,selector);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
    
        }
    
        void handleKey(SelectionKey selectionKey,Selector selector) throws IOException {
            if(selectionKey.isValid()){
                if(selectionKey.isAcceptable()){
                    System.out.println("server accept key: "+selectionKey.toString());
                    ServerSocketChannel serverSocketChannel = (ServerSocketChannel) selectionKey.channel();
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector, SelectionKey.OP_READ);
                }else if(selectionKey.isReadable()){
                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                    System.out.println("server read ready: "+socketChannel);
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    
                    while(socketChannel.read(byteBuffer)>0){
                        byteBuffer.flip();
                        System.out.println(new String(byteBuffer.array(),0,byteBuffer.limit(), StandardCharsets.UTF_8));
                    }
                    socketChannel.register(selector, SelectionKey.OP_WRITE);
                }else if(selectionKey.isWritable()){
                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                    System.out.println("server write ready: "+socketChannel);
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    byteBuffer.put("服务端反馈信息".getBytes(StandardCharsets.UTF_8));
                    byteBuffer.flip();
                    socketChannel.write(byteBuffer);
                    socketChannel.register(selector, SelectionKey.OP_READ);
                }
            }
        }
    
        public static void main(String[] args) {
            new NioServerBetter("localhost", 9931).run();
        }
    }
    ```

4. NIO客户端的简单示例.
    ```java
    public class NioClientBetter {
    
        private int port;
    
        private volatile boolean flag = true;
    
        public NioClientBetter(int port){
            this.port = port;
        }
    
        public void setFlag(boolean flag) {
            this.flag = flag;
        }
    
        public void execute(){
            try {
                SocketChannel socketChannel = SocketChannel.open();
                socketChannel.configureBlocking(false);
                Selector selector = Selector.open();
                if(socketChannel.connect(new InetSocketAddress(port))){
                    socketChannel.finishConnect();
                    socketChannel.register(selector, SelectionKey.OP_WRITE);
                }else{
                    socketChannel.register(selector, SelectionKey.OP_CONNECT);
                }
    
                while(flag){
                    selector.select(1000);
                    Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    Iterator<SelectionKey> iterator = selectionKeys.iterator();
                    SelectionKey selectionKey = null;
                    while(iterator.hasNext()){
                        selectionKey = iterator.next();
                        iterator.remove();
                        handleClient(selectionKey, selector);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
    
        }
    
        void handleClient(SelectionKey selectionKey, Selector selector) throws IOException {
            if(selectionKey.isValid()){
                if(selectionKey.isConnectable()){
                    System.out.println("client connect key: "+selectionKey.toString());
                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                    socketChannel.finishConnect();
                    socketChannel.register(selector, SelectionKey.OP_WRITE);
                }else if(selectionKey.isWritable()){
                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                    System.out.println("client write ready: "+socketChannel);
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    byteBuffer.put("只是第一个测试.".getBytes(StandardCharsets.UTF_8));
                    byteBuffer.flip();
                    socketChannel.write(byteBuffer);
                    socketChannel.register(selector, SelectionKey.OP_READ);
                }else if(selectionKey.isReadable()){
                    SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                    System.out.println("client read ready: "+socketChannel);
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    while(socketChannel.read(byteBuffer)>0){
                        byteBuffer.flip();
                        System.out.println(new String(byteBuffer.array(),0,byteBuffer.limit(), StandardCharsets.UTF_8));
                    }
                    socketChannel.register(selector, SelectionKey.OP_WRITE);
    
                    try {
                        TimeUnit.SECONDS.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    flag = false;
                }
            }
        }
    
        public static void main(String[] args) {
            new NioClientBetter(9931).execute();
        }
    }
    ```
5. 一些说明.
    >* channel使用selector监听时，一定要配置成非阻塞的；
    >* 从示例可以看出，服务端和客户端代码逻辑类似，核心逻辑就是把channel注册到selector，且不同的channel注册到不同的selector
    会返回SelectionKey，是唯一的，以此来定位channel；
    >* 每次事件处理后需要重新注册到selector上；
    >* 注意遍历侦听的SelectionKey时，要把处理过的key及时清除，否则会重复消费；此处注意处理的逻辑不要新开线程处理，
    会导致消息还没处理完又被下次侦听接收引发错误；

6. NIO的缺点.
    >* NIO的类库和API繁杂，写出高质量的NIO程序需要掌握大量技能；
    >* 可靠性能力补齐工作量和难度较大，如断连重连、网络闪断、半包读写、失败缓存等；
    >* Java NIO的一些自身bug；
    