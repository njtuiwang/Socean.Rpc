# Socean.Rpc
 
简介：
Socean.RPC是一个高效的rpc框架，框架特点是稳定和高效，在普通PC上测试，长连接模式下每秒处理请求数14w左右，常规请求处理时间0.04毫秒

本框架性能可满足部分unity3D或IM服务器端的需求，普通电脑上测试，可支持10000长连接，且每秒10个请求（每100毫秒1个请求 ），如果在性能好的服务器上，20000长连接且每秒20个请求应该是没问题的
  
框架特点:
高性能、稳定、支持异步、资源占用很小
  
  使用简介
  -------------------------------------------------------------------
  一、常规用法之EasyProxy
 
  EasyProxy是Socean.RPC的一个动态代理实现，特点是性能高、稳定性好、使用简便
  
  服务端 :
  
  1.定义序列化器和消息处理器
    
    public class RpcSeralizer : Socean.Rpc.DynamicProxy.IBinarySerializer
    {
        public object Deserialize(byte[] contentBytes, Type type)
        {
            var content = Encoding.UTF8.GetString(contentBytes);
            return Newtonsoft.Json.JsonConvert.DeserializeObject(content, type);
        }

        public byte[] Serialize(object obj)
        {
            var content = Newtonsoft.Json.JsonConvert.SerializeObject(obj);
            return Encoding.UTF8.GetBytes(content);
        }
    }
    
    public class CustomMessageProcessor : Socean.Rpc.DynamicProxy.EasyProxyMessageProcessor
    {
        public override void Init(IServiceHost serviceHost)
        {
            serviceHost.RegisterServices(Assembly.GetExecutingAssembly(), new RpcSerializer());
        }
    }
    
 2.定义服务
 
    public class Book
    {
        public string Name { get; set; }

        public double Price { get; set; }
    }

    [RpcService(ServiceName="IBookService")]  //如果不设置ServiceName,默认为类名
    public class BookService                  //此处可继承IBookService，也可不继承，但两者的ServiceName必须一致
    {
        public bool RegisterForSale(Book book)
        {
            Console.WriteLine("RegisterForSale,bookName:{0},bookPrice:{1}", book.Name, book.Price);
            return true;
        }

        public void AddStock(string bookName, int count)
        {
            Console.WriteLine("AddStock,bookName:{0},count:{1}", bookName, count);
        }
    }
    
 3.启动服务
 
    var server = new RpcServer();
    server.Bind(IPAddress.Parse("127.0.0.1"), 11111);
    server.Start<CustomMessageProcessor>();
 
 
 客户端：
 
  1.定义序列化器
  
    public class RpcSeralizer : Socean.Rpc.DynamicProxy.IBinarySerializer
    {
        public object Deserialize(byte[] contentBytes, Type type)
        {
            var content = Encoding.UTF8.GetString(contentBytes);
            return Newtonsoft.Json.JsonConvert.DeserializeObject(content, type);
        }

        public byte[] Serialize(object obj)
        {
            var content = Newtonsoft.Json.JsonConvert.SerializeObject(obj);
            return Encoding.UTF8.GetBytes(content);
        }
    }
    
   2.定义服务接口 
   
    [RpcProxy(ServiceName = "IBookService")]   //如果不设置ServiceName,默认为接口名
    public interface IBookService
    {
        bool RegisterForSale(Book book);

        void AddStock(string bookName, int count);
    }

    public class Book
    {
        public string Name { get; set; }

        public double Price { get; set; }
    }
    
   3.生成代理服务
   
     var bookServiceProxy = EasyProxyGenerator<IBookService>.Create(IPAddress.Parse("127.0.0.1"), 11111, new RpcSerializer());
   
   4.执行函数
   
     bookServiceProxy.RegisterForSale(new Book { Name = "相对论", Price = 108.88 });
     bookServiceProxy.AddStock("相对论", 1000);
 
 
 其他用法
    
  想实现日志或鉴权等功能，可以使用ServiceFilter
    
  1.定义日志记录ServiceFilter
    
    public class LogFilter : IServiceFilter
    {       
        public void Do(ServiceContext context, FilterChain filterChain)
        {
            Console.WriteLine("before request");

            filterChain.DoNext(context);

            Console.WriteLine("after request");
        }
    }
    
  2.注册日志记录Filter
    
    public class CustomMessageProcessor : Socean.Rpc.DynamicProxy.EasyProxyMessageProcessor
    {
        public override void Init(IServiceHost serviceHost)
        {
            serviceHost.RegisterServices(Assembly.GetExecutingAssembly(), new RpcSerializer());
            
            serviceHost.RegisterFilter(new LogFilter());
        }
    }
        
        
 -------------------------------------------------------------------
  二、常规用法之底层函数
  
  如果想重新封装一套动态代理，或追求更高的收发效率，可使用底层函数
  
  服务端 :

  1.定义实体
  
    public class Book
    {
        public string Name { get; set; }
        
        public double Price { get; set; }
    } 
 
  2.定义消息处理器
 
     public class DefaultMessageProcessor : IMessageProcessor
     {
          public void Init()
          {          
          }

          public async Task<ResponseBase> Process(Socean.Rpc.Core.Message.FrameData frameData)
          {
              var title = Encoding.UTF8.GetString(frameData.TitleBytes);
              if (title == "/books/namechange")
              {
                  var content = Encoding.UTF8.GetString(frameData.ContentBytes);                 
                  var book = Newtonsoft.Json.JsonConvert.DeserializeObject<Book>(content);
                  book.Name = "new name";

                  var responseContent = Newtonsoft.Json.JsonConvert.SerializeObject(book);
                  return new BytesResponse(Encoding.UTF8.GetBytes(responseContent));
              }           

              return new ErrorResponse((byte)ResponseCode.SERVICE_NOT_FOUND);
          }
      }


  3.启动服务
  
    var server = new RpcServer();
    server.Bind(IPAddress.Any, 11111);       
    server.Start<DefaultMessageProcessor>();  
  
  -------------------------------------------------------------------

  客户端:
  
  1.定义实体
  
    public class Book
    {
        public string Name { get; set; }
        
        public double Price { get; set; }
    }
 
 
  2.执行同步调用
  
    public Book ChangeBookName(Book book)
    {
        using (var rpcClient = new FastRpcClient(IPAddress.Parse("127.0.0.1"), 11111))
        {
            var requestContent = JsonConvert.SerializeObject(book);
            var response = rpcClient.Query(Encoding.UTF8.GetBytes("/books/namechange"), Encoding.UTF8.GetBytes(requestContent));
            var content = Encoding.UTF8.GetString(response.ContentBytes);
            return JsonConvert.DeserializeObject<Book>(content);
        }
    }
    
  3.执行异步调用
  
    public async Task<Book> ChangeBookName(Book book)
    {
        using (var rpcClient = new FastRpcClient(IPAddress.Parse("127.0.0.1"), 11111))
        {
            var requestContent = JsonConvert.SerializeObject(book);
            var response = await rpcClient.QueryAsync(Encoding.UTF8.GetBytes("/books/namechange"), Encoding.UTF8.GetBytes(requestContent));
            var content = Encoding.UTF8.GetString(response.ContentBytes);
            return JsonConvert.DeserializeObject<Book>(content);
        }
    }
    
  -------------------------------------------------------------------
  
  其他：
     
  1.NetworkSettings类可修改连接超时时间等参数  
  2.因Unity3D下无法使用Emit，所以EasyProxy不可用，请移除DynamicProxy空间下的文件即可
  
