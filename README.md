#springboot-mqrpc

通过spring +RabbitMQ+hessian的Rpc远程调用来理解RPC调用原理

1.对Rpc的简单阐述

对RPC通俗的理解就是，调用远程服务和调用本地服务一样透明化无感知。使用过dubbo和motan的同学肯定有这种感觉。实现RPC调用过程，无非解决两个问题：

1.数据的传输：这里使用RabbitMQ来收发消息，保证消息的可靠性

2.请求和响应数据的序列化和反序列化：采用Hessian

如果有自己的序列化方案，还得确定传输的消息体结构，这里不做考虑

2.远程调用过程

首先：消费者和生产者spring容器初始化的时候，会根据配置的的api在RabbitMQ上建立相应的队列,消费者会监听相关队列

1）生产者（client）调用以本地调用方式调用服务；

2）client 接收到调用后通过Hessian将方法、参数等组装成能够进行网络传输的消息体；

3）client 通过代理类，执行invoke方法，统一将消息发送到MQ监听的服务端；

4）server 收到消息后通过Hessian进行解码；

5）server 根据解码结果调用本地的服务；

6）本地服务执行并将结果返回给server ；

7）server 将返回结果通过Hessian打包发送至消费方；

8）client 接收到消息，并进行解码；

9）生产者得到最终结果。

3.测试代码及图例
```
    /**
     * 变态暴力测试
     * @param args
     */
    public static void main(String[] args) {
        GenericXmlApplicationContext context = new GenericXmlApplicationContext(
                "classpath:/applicationContext-client.xml");
        Service service = (Service) context.getBean("myService");
        new ClientTest().exec(service);
    }
    public void exec(Service service){
        ExecutorService executorService= Executors.newFixedThreadPool(30);
        for(int i=0;i<=30;i++){
            executorService.submit(new Task(service));
        }
    }
    private class Task implements Callable {
        private Service service;
        public Task(Service service){
            this.service=service;
        }
        @Override
        public Object call() throws Exception {
            for(int i=0;i<=100000;i++){
                System.out.println("servicEcho当前线程："+Thread.currentThread().getName()+"| 线程任务数"+i+"| 输出："+service.echo("Hello AMQP!"));
                System.out.println("serviceStudent当前线程："+Thread.currentThread().getName()+"| 线程任务数"+i+"| 输出："+service.getStudent(null).getName());

            }
            return null;
        }
    }
```
![输入图片说明](http://git.oschina.net/uploads/images/2016/1209/224812_d065d835_492218.png "在这里输入图片标题")
![输入图片说明](http://git.oschina.net/uploads/images/2016/1209/224822_878ad0d7_492218.png "在这里输入图片标题")
![输入图片说明](http://git.oschina.net/uploads/images/2016/1209/224833_86035221_492218.png "在这里输入图片标题")

3.最后的最后的
欢迎光顾，作者博客：http://www.kailing.pub/