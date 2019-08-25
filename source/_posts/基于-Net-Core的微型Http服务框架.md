---
title: 基于.Net Core的微型Http服务框架
date: 2019-08-07 19:43:00
tags: 网络编程 
---
.Net Core是由微软公司开发的一套开源的，具有跨平台能力（Windows，Linux，Mac OSX）的应用程序开发框架。本人最近用.net写了一套简单的Http服务框架，并集成了MySql数据库，写此博文作为分享，也为了防止自己遗忘。本文适合了解Http协议的原理，以及面向对象程序设计各种概念的读者。
## 服务的组织架构
一个Http服务器会负责多个Http服务，即一个Http服务器存在多个接口，这些接口的业务应互不干扰。而每个接口的业务都涉及数据的处理、数据库的读写等操作。因此考虑将每个服务分为控制层(Controller)和数据层(Model)。控制层负责服务逻辑的控制，数据层负责数据的操作。同时，服务因该具有高度的可拓展性，即新服务的加入应该是便捷的。因此应该有一个服务管理器，能够动态添加或删除服务。为了方便，我们将每个服务都称之为应用（Application）。
在这种架构中，所有Http请求应携带应用标识符，以方便服务端区分请求的是哪一个应用。服务端接收Http请求后，通过标识符将请求参数下发给相应的应用，应用处理结束后，将要返回的数据通知给Http服务器，然后服务器回应数据，这样便完成了一次Http请求的处理。
综合以上分析，所有的应用都应继承自以下接口：
``` cs
interface IApplication
{
    string Name { get; }
    HttpResponseArgs Handle(HttpArgs args);
}
```

<!--more-->
其中，Name属性是唯一的，即这个应用的名称，用来区分不同的应用；Handle为该Application的处理逻辑。HttpArgs是Http请求参数，HttpResponseArgs是Http响应参数，将请求参数传递给对应的Application后，要求其返回一个HttpResponseArgs，用来对请求进行回应。
这样一来，不同的应用即为不同的类，他们都实现IApplication这个接口。因此，添加一个应用，只需添加一个该应用的实例即可。
``` cs
class Application
    {
        static Dictionary<string, Type> appTypes = new Dictionary<string, Type>();
        static string appKey = "app";
        public static bool LoadApp(Type appType)
        {
            if (!typeof(IApplication).IsAssignableFrom(appType)) return false;
            var construct = appType.GetConstructor(new Type[0]);
            var app = (IApplication)construct.Invoke(null);
            if (appTypes.ContainsKey(app.Name)) return false;
            appTypes.Add(app.Name, appType);
            return true;
        }
        public static bool UnloadApp(string appName)
        {
            if (appTypes.ContainsKey(appName))
                appTypes.Remove(appName);
            else
                return false;
            return true;
        }
        public static void SetAppKeyString(string keystr = "app")
        {
            appKey = keystr;
        }
        public static HttpResponseArgs Handle(HttpArgs args)
        {
            if (!appTypes.ContainsKey(args.GetArgValue(appKey))) return null;
            var construct = appTypes[args.GetArgValue(appKey)].GetConstructor(new Type[0]);
            var app = (IApplication)construct.Invoke(null);
            return app.Handle(args);
        }
    }
```


appTypes字典用于存放应用类型，与应用的Name相对应。LoadApp方法用来加载一个应用，对应用进行去重判断以后获取一个该应用的实例，并添加到字典中；UnloadApp方法用来卸载一个应用，与LoadApp相对应。SetAppKeyString用来设置http请求参数中区别不同app的参数键，默认为"app"。Handle用于接收Http请求参数后向对应的app进行下发，获取到app处理后的返回参数后返回给Http服务器。
## 数据操作层
服务端的每个app都会涉及到数据操作。不同的app的数据会有很大的区别，但是很多数据操作，比如对数据库的增删查改，都是类似的。因此考虑创建一个基类，其中包含通用的数据处理和数据库操作方法，然后所有应用的数据层都继承自这个类。（下列代码均为具体实现，不必仔细阅读）
``` cs
class BaseModel
    {
        /// <summary>
        /// 数据表名
        /// </summary>
        protected virtual string TableName { get; }
        /// <summary>
        /// 判断数据表中是否存在指定数据
        /// </summary>
        /// <param name="limit">查找条件</param>
        /// <returns></returns>
        public bool Find(Dictionary<string, object> limit)
        {
            var conn = DataBase.GetConnection();
            conn.Open();
            var findstr = string.Format("select * from {0} where", TableName);
            StringBuilder sb = new StringBuilder(findstr);
            foreach (var kv in limit)
            {
                var key = kv.Key;
                if ((!"<>=".Contains(key.TrimEnd(' ').Last())) && (!key.ToUpper().Contains(" LIKE"))) key += '=';
                sb.AppendFormat(" {0}'{1}' and", key, kv.Value);
            }
            sb.Remove(sb.Length - 3, 3);
            findstr = sb.ToString();
            MySqlCommand cmd = new MySqlCommand(findstr, conn);
            MySqlDataReader reader = cmd.ExecuteReader();
            var result = false;
            if (reader.Read()) result = true;
            conn.Close();
            return result;
        }
        /// <summary>
        /// 判断数据表中是否存在指定数据
        /// </summary>
        /// <param name="key">查找键</param>
        /// <param name="value">查找值</param>
        /// <returns></returns>
        public bool Find(string key, string value)
        {
            var conn = DataBase.GetConnection();
            conn.Open();
            var findstr = string.Format("select * from {0} where {1}='{2}'", TableName, key, value);
            MySqlCommand cmd = new MySqlCommand(findstr, conn);
            MySqlDataReader reader = cmd.ExecuteReader();
            var result = false;
            if (reader.Read()) result = true;
            conn.Close();
            return result;
        }
        /// <summary>
        /// 从数据表中获取指定数据
        /// </summary>
        /// <param name="key">查找键</param>
        /// <param name="value">查找值</param>
        /// <returns></returns>
        public List<Dictionary<string, object>> Select(string key, string value)
        {
            List<Dictionary<string, object>> result = new List<Dictionary<string, object>>();
            var conn = DataBase.GetConnection();
            conn.Open();
            var restric = new string[4];
            restric[2] = TableName;
            var table = conn.GetSchema("Columns", restric);
            List<string> colums = new List<string>();
            foreach (DataRow r in table.Rows)
            {
                colums.Add(r["column_name"].ToString());
            }
            var findstr = string.Format("select * from {0} where {1}='{2}'", TableName, key, value);
            MySqlCommand cmd = new MySqlCommand(findstr, conn);
            MySqlDataReader reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                Dictionary<string, object> dt = new Dictionary<string, object>();
                foreach (var cn in colums)
                {
                    dt.Add(cn, reader[cn]);
                }
                result.Add(dt);
            }
            reader.Dispose();
            cmd.Dispose();
            conn.Close();
            return result;
        }
        /// <summary>
        /// 从数据表中获取指定数据
        /// </summary>
        /// <param name="limit">查找条件</param>
        /// <returns></returns>
        public List<Dictionary<string, object>> Select(Dictionary<string, object> limit)
        {
            List<Dictionary<string, object>> result = new List<Dictionary<string, object>>();
            var conn = DataBase.GetConnection();
            conn.Open();
            var restric = new string[4];
            restric[2] = TableName;
            var table = conn.GetSchema("Columns", restric);
            List<string> colums = new List<string>();
            foreach (DataRow r in table.Rows)
            {
                colums.Add(r["column_name"].ToString());
            }
            var udstr = string.Format("select * from {0} where", TableName);
            StringBuilder sb = new StringBuilder(udstr);
            foreach (var kv in limit)
            {
                var key = kv.Key;
                if ((!"<>=".Contains(key.TrimEnd(' ').Last())) && (!key.ToUpper().Contains(" LIKE"))) key += '=';
                sb.AppendFormat(" {0}'{1}' and", key, kv.Value);
            }
            sb.Remove(sb.Length - 3, 3);
            udstr = sb.ToString();
            MySqlCommand cmd = new MySqlCommand(udstr, conn);
            MySqlDataReader reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                Dictionary<string, object> dt = new Dictionary<string, object>();
                foreach (var cn in colums)
                {
                    dt.Add(cn, reader[cn]);
                }
                result.Add(dt);
            }
            reader.Dispose();
            return result;
        }
        /// <summary>
        /// 将数据插入到数据表
        /// </summary>
        /// <param name="data">要插入的数据</param>
        /// <returns></returns>
        public bool Insert(Dictionary<string, object> data)
        {
            return InsertGetID(data) == -1 ? false : true;
        }
        /// <summary>
        /// 将数据插入到数据表，并得到当前自增数
        /// </summary>
        /// <param name="data">要插入的数据</param>
        /// <returns></returns>
        public long InsertGetID(Dictionary<string, object> data)
        {
            var connection = DataBase.GetConnection();
            connection.Open();
            StringBuilder sb = new StringBuilder();
            sb.Append("insert into ");
            sb.Append(TableName + " set ");
            foreach (var c in data)
            {
                sb.Append(c.Key + "=");
                sb.Append("'" + c.Value + "' ,");
            }
            sb.Remove(sb.Length - 1, 1);
            MySqlCommand cmd = new MySqlCommand(sb.ToString(), connection);
            cmd.ExecuteNonQuery();
            long result = cmd.LastInsertedId;
            cmd.Dispose();
            connection.Close();
            return result;
        }
        /// <summary>
        /// 从数据表中删除符合条件的数据
        /// </summary>
        /// <param name="limit">查找条件</param>
        /// <returns>受影响的数据的数量</returns>
        public long Delete(Dictionary<string, object> limit)
        {
            //返回删除的数据条数
            var conn = DataBase.GetConnection();
            conn.Open();
            var delstr = string.Format("delect from {0} where", TableName);
            StringBuilder sb = new StringBuilder(delstr);
            foreach (var kv in limit)
            {
                var key = kv.Key;
                if ((!"<>=".Contains(key.TrimEnd(' ').Last())) && (!key.ToUpper().Contains(" LIKE"))) key += '=';
                sb.AppendFormat(" {0}'{1}' and", key, kv.Value);
            }
            sb.Remove(sb.Length - 3, 3);
            delstr = sb.ToString();
            MySqlCommand cmd = new MySqlCommand(delstr, conn);
            long result = cmd.ExecuteNonQuery();
            cmd.Dispose();
            conn.Close();
            return result;
        }
        /// <summary>
        /// 从数据表中删除符合条件的数据
        /// </summary>
        /// <param name="key">查找键</param>
        /// <param name="value">查找值</param>
        /// <returns>受影响的数据的数量</returns>
        public int Delete(string key, string value)
        {
            //返回删除的数据条数
            var conn = DataBase.GetConnection();
            conn.Open();
            var findstr = string.Format("delete from {0} where {1}='{2}'", TableName, key, value);
            MySqlCommand cmd = new MySqlCommand(findstr, conn);
            var result = cmd.ExecuteNonQuery();
            cmd.Dispose();
            conn.Close();
            return result;
        }
        /// <summary>
        /// 更新数据表中的数据
        /// </summary>
        /// <param name="key">查找键</param>
        /// <param name="value">查找值</param>
        /// <param name="newdata">新数据</param>
        /// <returns>受影响的数据数量</returns>
        public long Update(string key, string value, Dictionary<string, object> newdata)
        {
            string udstr = string.Format("update {0} set ", TableName);
            StringBuilder sb = new StringBuilder(udstr);
            foreach (var u in newdata)
            {
                sb.AppendFormat("{0} = '{1}',", u.Key, u.Value);
            }
            sb.Remove(sb.Length - 1, 1);
            sb.AppendFormat(" where {0} = '{1}'", key, value);
            udstr = sb.ToString();
            var conn = DataBase.GetConnection();
            conn.Open();
            MySqlCommand cmd = new MySqlCommand(udstr, conn);
            long result = cmd.ExecuteNonQuery();
            cmd.Dispose();
            conn.Close();
            return result;
        }
        /// <summary>
        /// 更新数据表中的数据
        /// </summary>
        /// <param name="limit">查找条件</param>
        /// <param name="newdata">新数据</param>
        /// <returns>受影响的数据的数量</returns>
        public long Update(Dictionary<string, object> limit, Dictionary<string, object> newdata)
        {
            string udstr = string.Format("update {0} set ", TableName);
            StringBuilder sb = new StringBuilder(udstr);
            foreach (var u in newdata)
            {
                sb.AppendFormat("{0} = '{1}',", u.Key, u.Value);
            }
            sb.Remove(sb.Length - 1, 1);
            sb.Append(" where ");
            foreach (var kv in limit)
            {
                var key = kv.Key;
                if ((!"<>=".Contains(key.TrimEnd(' ').Last())) && (!key.ToUpper().Contains(" LIKE"))) key += '=';
                sb.AppendFormat(" {0}'{1}' and", key, kv.Value);
            }
            sb.Remove(sb.Length - 3, 3);
            udstr = sb.ToString();
            var conn = DataBase.GetConnection();
            conn.Open();
            MySqlCommand cmd = new MySqlCommand(udstr, conn);
            long result = cmd.ExecuteNonQuery();
            cmd.Dispose();
            conn.Close();
            return result;
        }
    }
```
在这个类当中包含了对数据库进行增删查改的通用操作。不同的应用所操作的数据表可能不同，因此数据表名定义为可覆写的类型。同时也考虑到，不同的应用操作的都是同一个数据库，因此应该有一个类负责管理数据库的配置和连接。
``` cs
class DataBase
    {
        /// <summary>
        /// 数据库名
        /// </summary>
        private static string Name { get; set; } = "test";
        /// <summary>
        /// 数据库地址
        /// </summary>
        private static string Host { get; set; } = "127.0.0.1";
        /// <summary>
        /// 数据库端口
        /// </summary>
        private static int Port { get; set; } = 3306;
        /// <summary>
        /// 用户名
        /// </summary>
        private static string UserName { get; set; } = "root";
        /// <summary>
        /// 密码
        /// </summary>
        private static string PassWord { get; set; } = "root";

        private static string ConnStr { get; set; }

        public static MySqlConnection GetConnection()
        {
            if (ConnStr == null) throw new Exception("数据库未初始化");
            else return new MySqlConnection(ConnStr);
        }

        public static void Initialise(string name, string host, int port, string user, string password)
        {
            ConnStr = string.Format(
                "Database = {0}; datasource = {1}; port = {2}; user={3};pwd={4};SslMode = none; charset=utf8;"
                , name, host, port, user, password);
        }

        public static void Initialise()
        {
            if (Name == null || Host == null || UserName == null || PassWord == null) return;
            Initialise(Name, Host, Port, UserName, PassWord);
        }
    }
```
数据库的连接及配置统一由这个类进行管理，从而避免各应用操作数据库时可能发生的错误和冲突。
## Http消息的封装
为了方便程序处理，以及架构的清晰，应对Http消息进行封装。
首先是Http请求参数：
``` cs
    class HttpArgs
    {
        protected Dictionary<string, string> getArgs;
        protected JObject postArgs_j;
        /// <summary>
        /// 创建HttpArgs消息对象
        /// </summary>
        /// <param name="get_args"></param>
        /// <param name="response"></param>
        public HttpArgs(Dictionary<string, string> get_args, JObject post_args)
        {
            getArgs = get_args;
            postArgs_j = post_args;
        }
        /// <summary>
        /// 获取指定Get参数的值
        /// </summary>
        /// <param name="key">参数的键</param>
        /// <returns></returns>
        public string GetArgValue(string key)
        {
            if (getArgs.ContainsKey(key))
            {
                return getArgs[key];
            }
            else
            {
                return string.Empty;
            }
        }
        /// <summary>
        /// 获取Post参数
        /// </summary>
        /// <returns></returns>
        public JObject GetPostValue()
        {
            if (postArgs_j == null) return new JObject();
            return postArgs_j;
        }
    }
```
对于Get方式传递的参数，使用字典进行存放，并提供GetArgValue方法进行查询；对于Post方式传递的参数，封装为JObject对象(这里借助了Newton.json包)，并提供GetPostValue方法，直接返回JObject对象。
接下来是Http相应参数的封装：
``` cs
class HttpResponseArgs
    {
        public ResponseType ResponseType { get; set; }
        private JObject obj { get; set; }
        private string rawStr { get; set; }
        public HttpResponseArgs(JObject obj, ResponseType type)
        {
            this.obj = obj;
            ResponseType = type;
        }
        public HttpResponseArgs(string msg)
        {
            rawStr = msg;
            ResponseType = ResponseType.Raw;
        }
        public override string ToString()
        {
            if (ResponseType == ResponseType.Raw)
            {
                return rawStr;
            }
            var jstr = obj.ToString(Newtonsoft.Json.Formatting.Indented);
            switch (ResponseType)
            {
                case ResponseType.Json:
                    return jstr;
                case ResponseType.Xml:
                    return Newtonsoft.Json.JsonConvert.DeserializeXmlNode(jstr).InnerXml;
            }
            return jstr;
        }
    }

enum ResponseType
    {
        Json,
        Xml,
        Raw
    }
```
首先是Http相应的类型。作为服务端接口，常用的返回格式是xml格式或者json格式。为了保持可拓展性，也应留出回应任意文本的接口。对于json格式，利用Newton.Json包将数据对象转换为JObject，从而得到json字符串；对于xml格式，先将数据对象转换为JObject，再转换为xml对象，从而得到xml文本。
## 按照我们的思路对Http服务器进行封装
最后，对原始的Http服务器进行封装，以实现我们思路中的架构：
``` cs
class HttpServer
    {
        private HttpListener httpListener;
        /// <summary>
        /// Http请求传递的参数
        /// </summary>
        /// <param name="args"></param>
        /// <returns>回应参数,无需回应时应返回null</returns>
        public delegate HttpResponseArgs HttpGetRequestDelegate(HttpArgs args);
        public event HttpGetRequestDelegate HttpGotRequest;
        /// <summary>
        /// 使用指定端口构造Http服务器
        /// </summary>
        /// <param name="port">监听端口</param>
        public HttpServer(int port)
        {
            httpListener = new HttpListener();
            httpListener.Prefixes.Add(string.Format("http://+:{0}/", port));
        }
        /// <summary>
        /// 启动Http服务器
        /// </summary>
        public void Start()
        {
            httpListener.Start();
            httpListener.BeginGetContext(new AsyncCallback(GetContext), httpListener);
        }
        private void GetContext(IAsyncResult ar)
        {

            HttpListener httpListener = ar.AsyncState as HttpListener;
            HttpListenerContext context = httpListener.EndGetContext(ar);
            httpListener.BeginGetContext(new AsyncCallback(GetContext), httpListener);
            HttpListenerRequest request = context.Request;
            HttpListenerResponse response = context.Response;
            Dictionary<string, string> getArgs = new Dictionary<string, string>();
            JObject postArgs = null;
            string[] keys = request.QueryString.AllKeys;
            if (keys.Count() > 0)
            {
                foreach (var key in keys)
                {
                    getArgs.Add(key, request.QueryString[key]);
                }
            }
            if (request.HttpMethod == "POST")
            {
                using (Stream stream = request.InputStream)
                {
                    StreamReader reader = new StreamReader(stream, Encoding.UTF8);
                    string body = reader.ReadToEnd();
                    string jsonText = "{}";
                    if (body.IndexOf("<xml>") == 0)
                    {
                        XmlDocument doc = new XmlDocument();
                        doc.LoadXml(body);
                        jsonText = JsonConvert.SerializeXmlNode(doc, Newtonsoft.Json.Formatting.None, true);
                    }
                    if (body.IndexOf("{") == 0)
                    {
                        jsonText = body;
                    }
                    postArgs = JObject.Parse(jsonText);
                }
            }
            var callbackValue = HttpGotRequest?.Invoke(new HttpArgs(getArgs, postArgs));
            #region 消息回应
            if (callbackValue == null) return;
            response.ContentType = "json";
            response.ContentEncoding = Encoding.UTF8;
            using (Stream output = response.OutputStream)
            {
                byte[] buffer = Encoding.UTF8.GetBytes(callbackValue.ToString());
                output.Write(buffer, 0, buffer.Length);
            }
            #endregion
        }
    }
```
利用HttpClient在指定的域和端口监听Http请求。GetContext是Http请求的处理方法，它是一个异步方法，因此不会造成程序的阻塞，从而可以处理较高并发的请求。在处理Http请求时，首先对请求链接中的QueryString进行处理，获取到Get方式传递的参数，构造字典。然后读取Post流，判断Post的数据是xml格式还是json格式，然后进行相应的处理，构造JObject对象。利用字典和JObject对象即可构造Http请求参数，以事件参数的形式传递给监听者，待监听者进行处理后，拿到监听者返回的Http响应对象，以网络流的方式进行Http响应。
而上文所说的监听者要做的事情很简单，只需要将请求参数传递给Application类，Application类就会根据参数进行下发、处理，并返回响应参数。拿到这个响应参数后再返回给HttpServer就好了。
``` cs
class Program
    {
        private static HttpServer Server;
        static void Main(string[] args)
        {
            Server = new HttpServer(80);
            Server.HttpGotRequest += Server_HttpGotRequest;
            Server.Start();
            Console.WriteLine("服务启动成功.");
            while(true) ;
        }

        private static HttpResponseArgs Server_HttpGotRequest(HttpArgs args)
        {
            return Application.Application.Handle(args);
        }
    }
```
这样，一个微型Http服务框架就写好了。
## 添加一个应用
接下来将演示如何利用我们刚写好的这个服务框架搭建一个最简单的Http服务
首先新建一个类，名字可以叫作Index。在我们的框架中，所有的应用都应实现IApplication接口，所有应用的数据都继承BaseModel类，它也不例外:
``` cs
class Index : IApplication
    {
        public string Name => "";

        public HttpResponseArgs Handle(HttpArgs args)
        {
            var msg = new IndexModel
            {
                Code = 200,
                Message = "Welcome to use ThinkNet frame!",
                Data = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")
            };
            return new HttpResponseArgs(JObject.FromObject(msg), ResponseType.Json);
        }
    }

class IndexModel : BaseModel
    {
        public int Code { get; set; }
        public string Message { get; set; }
        public string Data { get; set; }
    }
```
应用的名字设置成空字符串，说明这是默认的app，当找不到合适的app进行处理时就由这个app来进行处理。在Handle方法中，我们没有用到请求参数，只是简单地返回了一个状态码、一条消息以及当前时间。这个数据结构被定义在IndexModel这个类中。由于并不需要进行数据库操作，因此无需对数据表字段进行重写。
建立这两个类之后，就完成了一个最简单的app的定义。接下来只需要激活这个app即可。在Main方法中添加一行:
``` cs
Application.Application.LoadApp(typeof(Application.Index.Index));
```
完整的Main方法：
``` cs
static void Main(string[] args)
{
    Server = new HttpServer(80);
    Application.Application.LoadApp(typeof(Application.Index.Index));
    Server.HttpGotRequest += Server_HttpGotRequest;
    Server.Start();
    Console.WriteLine("服务启动成功.");
    while(true) ;
}
```
此时即可按下F5，运行服务器。运行开服务器之后打开浏览器，地址栏输入127.0.0.1，回车，应该能够看到类似下面的页面：
![img][img1]
如果将app的Name进行修改，比如将Name改成index:
``` cs
    public string Name => "index";
```
那么访问的链接中应带上app的名字，指定访问名为index的app：127.0.0.1?app=index
访问的效果是一样的:
![img2][img2]
那么，再添加多个app，进行不同的处理，道理都是一样的。有了这套框架，我们就可以很容易地搭建Http服务，而无需考虑底层Http服务器的运作，方便快捷，而且架构整洁。
## 在Linux系统中运行
前面已经介绍过，.Net Core是跨平台的，我们以 .Net Core为基础编写的程序当然也是跨平台的。生成面向Linux系统的发布版，我们的代码就可以在Linux系统中运行。
在解决方案根目录打开命令提示符，输入以下命令：
``` powershell
dotnet publish -c Release -r linux-x64
```
等待生成完成后，在/bin/Release目录下即可找到生成的结果。将publish文件夹复制到Linux系统中，直接输入./项目名称，程序即可在Linux系统中正常运行。访问上文中的地址，结果也是完全一样的。
本文可能存在各种错误与疏漏，请读者包涵指正。


[img1]: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAosAAAE2CAYAAAAJVtghAAAgAElEQVR4nO3dDXRU55ng+UcZpzf2kfwhQZCIsFtqMGyDIeAEWWLi9tLGjjFpxuLgacREO7Qn2Nn0Hp/sSXd7jjFxMJx1d7zJcZ/OxibtYVZpiV5zkEOCsQNuxsGLZOgYxTLMCKNGiVFKIkgyNrSV+Et73/tRdavq3tL9rA/p/ztHVHHr1q333irVffS87/vcsgmNAAAAAA4+UegGAAAAoHgRLAIAAMDVFYVuAKaODz/6SD788CP5SLtVoxv0n0I3CgAAhEKwiFBUQPjb99+XDz74sNBNAQAAMSBYRCAEiQAATA8Ei/DtYy1Q/Nf3xqWQE+nVK5cV7NUBAJg+CBbhy8cffyz/Ov7bggaKCoEiAAD5QbAIzz5SgeJ744VuBgAAyCNK58Cz8fHfFroJAAAgzwgW4cl7WqD4MRf7AQBg2iFYxKTe/+BDvYYiAACYfvI2ZnFkZFTee+89uf76Ofl6Sd/evXRJPnnFFXLllVcWuilp1LEbGR1L/n9GVaXMmFHluK46xldddVWkr/+7938XyXYGB38t/f9yVsbHx/VjPPcP6qW29jORbBsAAMQjL8HiW2+dk8e//V15TwsS/uvffz8fLxnIseM/lyu0YPHmZZ+VqysqCt0c6Tv9pvzox8/rt5mun1MrLX+6XhbMvzG57Af/5f+RL6xoTFsWlqqlGLb3+dVj/yw//ekhGR0by3qsqrJSvvjFO6Rh+efCvUimkR/LV1dulbp/+IX85eLsh3v/+rPy5Xb7kibZdvj/lrUzrBW+LUv+Q3v2E+02tsnrf+Ww8Yw2dKWeID/s/QvJ8QzX9jU99pJ8P9k4DybZfwAAvIo9WLQHiiuabon75UKpqamWoaFhee3ELwoeMKrA72jXq/r9qqpKWbZ0iVx15VXacXxPTvS8Lm+dG9SP679tapT/9GetyfVVsBilMEW3VZazY/ez0vvGSf3/lZXXyU03LZQrP3WVjP/2PXnjjVN6ANne8Y/a/ZPSsuHe8FnRjACtzmmVff+bfPmX2+Sfev9EkrGhCs5WflZetIKyxX8hr2uBneNLaM//40dEtt2XIwozg00V5L2uB3kjsu+rt8uXtafkDhiN9bYetQWW+rZul6+Kh4DRw/4DAOBH2USMBfMyA8Wv/Nn/GtdLRebkf/8fesBYyAyjFfhdeeWn5ME//6pjpvBEzy+k/R/3yOjomMyoqpKR0VF9+UN/8fXIMothS+X84O93yRsnT+n7cc+/Wyu3NHw+ax2VdXzuR/v0mdaLb1ok/+m+/xj49fSATwWB3/8TET2g65KNDpm1kZERmTEjM+hyCNKcX0X+ZnGrDOTM9JnbEqMtybXMQE5yPdcMMjPbbWQac7fN6/4DAOBHbBNcSjFQVBb94f+sZxg//PBDPcOoxjHm0ytHu/VAUWUT/6+/3uEa+C1b+ll57JsP64GYFShG7YMPg2cVVRBoBYr/+9e+mgwUz/T/i7zw4kH9VlHL1eNqPZWBVM8LavFf/UJetwdnLrIDRX2p1P3+5K/R+9etooVy8tVcGb6RLnnxqEjTF5vS2zKjSb64QqTrxS4tnHTZ/guq73mjfDEjwFt810bt33Z5sdf9Zb3uPwAAfsQSLJZqoGgpZMD4ox/v128f/NoDk3bJqsxinLUPP/7o48DPffGnB/XbjS3/Pm0Sy5kzZrB45l+Sy9TjKvOoqLGNRWvkx/L9djV+cFPOcYcjR1+ULlGBYWbYZgak2uNHHaPFXnlRjxXvzN7+7PnaFrX3/IUc0SIAADGIPFgs9UDRUoiAUXUtq27lpZ9dPOmscfuYxrgErauoZj2Pjb2tj1FcfNNNaY/Nm/cH8sU779Bv7VSGUa2vxjCq5+efGaitmC+zXdZwDwLTJfq6XB+bvaDJ/Ykjv5QBt8dm/D7jDwEABRHpBBdV4sUKFK+68kqZOWOGPpvXq2UegqQwVMB34YJbB2C2Kz/1KX3sohUw3rL8c7GW1fnVW4P6repizkUF5Gqc4tov3Z31mFoelaDDWa0uZjWZxUmZy4Wd1fo/+9n/pz8/vyV11BhD1b0ssvGrbt24vfJfHunSZ0B7m5RcJ3UB+4ObFriFqwAA5F+kwaIab6cCRUXdWl2qXqngJM5gUQWKZwd+Gei5KmBMDA3LH9THn9+Z6VJD0aKOUT7qVQYNFq2ucTXrOZPqflZd1Hp2cW56dtFaP7+XFTQmrOhVarRA0G0yyMi+p4xg8i5miwAAppdIg8U7V63UawKefvOM/v9Vt/8verkXL6666srIy75kUrUJVdFtrxM3VICoStQo1113rf786aSsrCxQwKgmqyiqPI4f1vrW82Nnq6WYu47hiBx9sUtkxTb5M8+x4oAMjIgsDpBd7OpLiMf0JQAAsYs0WFQTMh788wfk//yb78i5wV/L6dNn9FIuUV9RJKhPfvKTnjNyH3zwgd71rFRUlMuSmxbpz4+TCpiV13p+4av8jer+V9yu6hJU0GDRyhiefOO/y7p7/p3n56m6i/bnx2nELC2TVYzbcWVzdvNjTZ5mGucal2iMZ9zo3EVtjkt0HPFojmekixoAkG+RT3BRgeF//sv/Q+bUfiZZOFoVZy4lVqB46fJlPVC8eelnYw8UlZvNsYo9Pd5nvKpjq47x1m/tiLw9n3AbXDgJNd7QmqzS+8Ybnp6jSuZYk2JiH6/Y+20zUFR1CycJFMX7xBbLjDoj5DudyNqS6KMgXCfRzJb5K7SbX/4yu7RO4rQeRNYFHQgJAEBAsZTOKeWAsVCBoqIyg2oGuaqb2P6Pz3p6znM/3q+vv3Tpksjb84l/E/zjocYkKh0dz6bNbnaaDa0e/9GPfpz2vPiMyL7vG7UMvV16z+yC9jNhZfGdoldFzCxz41Z/MWmGrNAecyqt41Z/EQCAuMVWlLsUA8ZCBoqWe/5kjT5m79BL/23SmeQqoFTrqfU3/un6yNuixncGpUrh3LRooT7R6e++95R+3W1FdTGvvis1uUVlFNXjaj21vtNVXqKVkNNawOZYyzDg+qpLe8niz8pX91kR3mL5y3/QwsX2VtsyLUh9ZKt0rdgmj6UuQC1/oz1vyeJvixVWzli7Tbat6JKtK1PLVCZUXSd64z+kgtvs1wQAIB6xXhvaChjVGEYVMP7t957WxzAWq9ffOFnQQFFR2cX//BfaMfv2d/TZ5GrC0L9d0aiXFVLHU5XNeUtl4vYZGUUVKKr14xgX+m8+8YnA4xYVVZC7veP/1a/koq7//OKLB41rQ195pYxrwaF1bWhFBYpq/dhZtQy1QG5Ju/MqaZNdgo4VVNeW/gfRr+m85BFz2cY2ef2vJgtRZ8ja778kol9H2mqgGlf5C+a8AAAKItZrQ1tURtGa9PJf//77cb9cYIf+6b8VNFC0U0GhukKLNbPcyfwb5+kZxTjL6Pz2d7+T9z8Iftk/RWUPVbkcNSYxkxqjqLqe488oAgCAIPISLCoqYFSzdvNRHzAoVbRbFeIudKBopzKLqn6lNeNZuf76Wj2g9TNjOox3L/9rJNtRYxNVwW1VR1FlRFVXdH6LbwMAAL/yFiyidP3u/Q+0n/cL3QwAAFAAsU1wwdTxP/3eJ/XxiwAAYPohAoAnV135qcB1FwEAQOkiWIQnala0dYUZAAAwfTBmEb6oj8u/jo/Lxx/zsQEAYDogWEQg7/32t/Lhhx8VuhkAACBmBIsI7IMPP5Tf/u79wEW7QykrkzKZ0F5bjaP8OP+vDwDANEGwiNBUwPj+Bx8UuhkAACAGBIuIzIcffaR3TX+k3aqPlf5T6EYBAIBQCBYBAADgitI5AAAAcEWwCAAAAFcEiwAAAHBFsAgAAABXBIsAAABwdcULz79Q6DYAAACgSJWNj49P29I5iURC6uvrC90MAACAokU3NAAAAFwRLAIAAMAVwSIAAABcESwCAADAFcEiAAAAXBEsAgAAwBXBIgAAAFwRLAIAAMAVwSIAAABcESwCAADAFcFi1E50yte/9aQ8/sKFeNYHAADIoyui2lD/oTbpGtLulM+XNc0NUhnVhkvKKdn1k3Mi1y6RTXfNjGF9d+df2CWPH383fWH97fLdLy90foIKUtVrJ10tq7+ySVbNzvEiiZfl8R+8Lue1u4u/9KBsWuanPXNk0zebZbHbE3y354IcerJDDlxMLcnZJo/bdzyOWVzalojx+ORj+5nP1T6XDz14m8xyXMvn8QcAlKzwmcXRY9LZZgaK09z5F45Lrwok1rudYMOt78jKTDoFOGdfkq8/+bIeXKS/7q6MwEl5Vw78wD3D2fvDJ+XrZqCSmwoinNpzTnZp7dx1IvsZ/tujBdnfSg9U9Db+JKrt+xfn8cnH9pMSL8uuSYNlf8cfAFDaQmUWk9lEKZf5dy+Wd57vkuFo2lV6EsZJdtbyltzZuaDr5+SQNbIyaRcHpDdxW+o1EqlgwJ4JsrJJ54/vl0NLNqWtb2WzVKbyoRnHc2bezr+w3wwi7G1KZaF6f9IpvctsbfXbHrGCbEnLnKpgatdZ7fbUKZFltmyqz+3PumuTfPcut72z9uMamZXH4xPr9jP3b8/kAamv4w8AKHmhM4vVja3S2tosDVVRNKdUWSfZObLaU3ey3/Vz0E7833XqXtSWb6pXd96VQVsE3/tPzt2Ys+5abm7jXXnt9Yxsm+qO/OaD7l3aSafkgB7IZAavM2XVgy2y+lp1/5y8Zss+BWnP+RH1GlfL6j9OtWfxHy9xzM4G2l8XViC2+EsZxzvG45OX7Zt6f2gElIuXOx9Li5/jDwAofaGCxbmrWuWOeVE1pYSdeMU5iIhq/VCultpq6/4pee2sup0jN6eNLVOZp5eMbJHm/JunUtml2bflGLeW4cRpM+M032G/ZsoscyCrnn0K2h7NrBlXS1aQNzymrzNrxqdtawbbvvO+dRoZvfrb08flxXp88rD95HM79cygvn9Lcr+M9+MPAJgKmA0dWmqSympPg/v9rh9Q8uS/3NalfMHszkwPJlIZMzP7dHHMw9g4d5MGDGNmOwK2x8oKnj/eYY451IK/V9SYxIxMbWT7m3rPHpo0uzc5z8cnb9v3t3+ejz8AYEogWAypIJNaJmONV8w8+Q87BEVpGTMr+/SOnE8EeN3qSn2fnDN1VpZPUsFZ4PYslE1fmqPfUwHL1/XJFmqGckamNpL9tbKQEbxnfo9Pnrbf+0O/++fx+AMApgSCxTAS5uQJe/YuyvWDsAeKLt2XqcxTtBkzmb1QbtYzda/LrrRZxmr2rBGQzLo2rvakj82MavvJcXxfmqSskBcBj0+c21cTfVQGetbyNSH3z/34AwBKW2R1Fqef1CSVTZ4CD7/r+5esj5ervqJab+Q3+m1kGbOkmbLqC3PkgBaQ6Rmn4/bHVOZpuQz+4KXsUj4+25O1n2aArEq3PH6+RR7K6AoNvL/2cXyRDBkIdnxi237C/OPFZ51Pv8cfAFDayCwGVWSTWqwTuCrF4xoomt2UaszaIceM2QU5P6ZubaVh/NJnZ9+esY9qdq72OmKOlbvWbEeQ9iSs7KwtILa9pgqSkrX+Qu2vmYWMOrj3c3xi3v751weM/198XR7/1pN6vU79xyrVY1uerEeZ8HH8AQBTApnFQIpsUos5Dk8FijmzOrNnGkGCFgTotfgyM2aJU/KaWh4mWNEt1IKT7AArGZxUGu0I0h5jG+llW5Kv+ZULek3CZK2/EPtr1RKMJ7j3eHxi3n6QDKav4w8AmBIIFgMorkktZiCqCjZP2v23UG6uf0l6Vdeqw7g9qybhrBsXxtJOowagPdDw3x6jxp8Y4+MmzX4G3d8L0vumUa/w5jhnrKdxOj7xbt+1AHnCLATuMO7V3/EHAEwFdEP7lSiuSS3JDNhCbwFGsniyyrbZugutiQ4qmLh5ScRjzhJa8PEts65hxnHw2x6jxp+6EskuOWSfwZxIXenEXjom2P7+RgYjybB6lHA/PsW2fb/HHwBQ+srGx8cngj45dbk/d+UL1kjz8sqgLxGrRCIh9fX1Pp5hXTbN4fJ6kazvX3KyQS4ZE15yPSerKzthu9ycK9v+5VrfZYa2r/YkZ/Z6aEug7dteI8eM8qREzMcn7u07SbhnFoMcfwBAaaMb2gfXy71FtH6+6N2Ps8wSO0lqtmwE5WFcZF5uL3h7jPF41rWI07jMAi/E/vqV6/gU1/b9H38AQGkLlVksdf4ziwAAANMLYxYBAADgimARAAAArggWAQAA4IpgEQAAAK4IFgEAAOCKYBEAAACuCBYBAADgimARAAAArggWAQAA4IpgEQAAAK4IFgEAAOCKYBEAAACuCBYBAADgimAxaic65evfelIef+FCPOtnOP/CLv35u04EevrUk+fjjyLH+wsAoV0RfhP9crCtS4bTlpXL/LubpaEq/NZLyynZ9ZNzItcukU13zYxhfeQW8fFPvCyP/+B1Oa/dXfylB2XTMvctqaD98ePvpi+sv12+++WFk6+X5WpZ/ZVNsmq2h11wo4IktW9RbjOf21cS3o+/M36/ACAKIYLFMTnWuV9OX3Z67LKcfr5N3mlslTvmBX+FUnP+hePSq06a62+TWTGsHynrZO8Q0ESyfgFEefx7f/ik7DrrYSNZQZPN2Zfk609ekIcezO/76xyQvisHfvCkvLa8RR4KGTjFvX3F8/HPoaC/XwAwhYTOLJYvWCPNyyttS1JB5PAv+0XmzQ37EqUh8bLs0k6gs7STpafsit/1kVsiouOfSGWzVGD80IzjHjKBc2TTN5tlsX2RFUReHJDexG3J15h11yb57l1u27kgh57skAMXr5FZQT8TCWO/FHs2zgrwzh/fL4eWhMgA5mH7/o+/ezv5/QKA8EKMWayUhubWjEDRXP5H86Vc3b00qoWO04F2kt+jTnBzZLWnrIrf9ZFbxMf/2iXy0Dcf9JZBXdYs380MFM3lm+rVnXdlcDj7aU7Ov7BfCxRVEOawPY96/8m523bWXcvNbb4rr70efPxe3NvX+Tn+jvj9AoAoRTBmMYeKKskMJaekE6/4O8n7Xd9XW+zdojGMI5uE1X2YDCbS2uOQgbNLrjvJelnPi/D4z75NHnrQ6wt7cbXUVntpU6eRQau/PcDYPMspeU3vup0jN6dtQ2UsX5Je83/n3zwl5+8K0jUb9/YlmuMf5+8XAExDMQSLY3LsZ6flsprksmw6dEGnBtGv9nSS97u+d9ljydQ4sl0iyYBRe+1vpU7qOjWuTltmNys57szv+umyx52d07bX6RIIagHHK+eS6x144YIs9jlJpdDHP40W/On7Xr/cQ7CeatNDYcaDJi6Y3bfz045vKmPZIrWvqG7uMX0938Fc3NuPRJ7eXwCYRiIunZMar1jdOD1mQxfNpJZTqczUd1UXnvZjdYOG7hYM4Pwru/RgSQWSRntaZPW16hEjEMw2U1Z9YY5533v3YdEcfzsrQ+op+LOychG0adgI0jLbkspYzpRZeqr/HTmfKMLtR4BJLQAQvQgzi/ZAcZrMgk6Yg/21E6WfSRWe1/eh9+y5rAzf4oVa8KUtPz/yG1HBmMhC2fRNM3jxNLvZ7/op5y+KrP7Kg7b91ILB9UvkNTV5IdmeDGr8n59sUKJ4jn+SPVD0MAu694cdZlYuuuECs2Z82rwXUcYyz9sPLJGH9xcApqGIgkWr1uJ0qq+YGkS/ydOJ0u/6PqmZo0U0mN8x+Jk9Uw+ezo9diKCbssiOv9iGAfgoR2R0VYcZp+jQDj0Yl+gylnnefjDxv78AMF1FWJS7Wppa75DpMEpRV0yTWkRlEUvgBGmNeYtCkR1/K1B0G7+ZzczKRRncVFdqQds5PRg/9MOXHDKW2vHXyxMELM0T9/bDYFILAMQm5JhF1fU8DQPFYp1UUeysMW+VM0Nmoors+Jvj9rwHitbYuoiDGzNzKxdflwNOGcvEKXlNC6jk2sqAM5Vj3n5g/H4BQJxCBYtjx1+W05dV1/N0ChSLdFKFX9U+T+h+18+Smu0cNgtaXMffDFR8DQO4IL1vqlnrmSVowlooN9ebdx3GEVo1EmfduDDgcYh7+8EU5e8XAEwhIYLFfvl532WR8s/IvGkxRtGUsAbReymJEmD9fDv7kuw6kb6o94WX3buL/a6vUyV4jIkcOcfnqckh33pS++lML9djlyiu45/MEPoKgH8jg0EycB6Oz+I/XpLK/tneJ9VNbpQxulpuXuIS1BbB9n1LFPnvFwBMAeHHLF4+LfvbTrs8ONUmvBTfpIrAZt8mq+tf10/wvT/RTuA/sT2mskaZl6TzuX7WOrpcx8FLncU8HP+E7XJzNun7k1003Hl/TZFcT9tjHUrtfdq0fEDvFndq06zla1yCqiLZfsLP8S/i3y8AmEIirrM4tfm9HFsUl2+L0+IvPygPLb86fWGOki9+10+j13/MdRwmr7M41Y6/P97rUOrXn/7SnIylV+uljNy7yotn+15NrfcXAIpX2fj4+EShG1EoiURC6uvrJ18RnmVd7g8AAJQ0MosAAABwRbAIAAAAVwSLAAAAcMWYRcYsAgAAuCKzCAAAAFcEiwAAAHBFsAgAAABXBIsAAABwRbAIAAAAVwSLAAAAcEWwCAAAAFcEiwAAAHBFsAgAAABXBIsAAABwdUWoZ585KG3dw9nLa5qkddXcUJsGAABA4YXILI7JsTccAkVlqEvaOo9pawAAAKCUlY2Pj09EusXRY9L5/Gm5rN2tbmyVO+ZFuvVIJRIJqa+vL3QzAAAAilb0YxarGqS5sVq/e/ltcosAAAClLJYJLmNvX9Zvy6+rjGPzAAAAyJPIg8Wx452yv08LFsvny+eKuAsaAAAAkws3G1pswaEds6EBAACmhHjqLKrZ0If6Y9k0AAAA8if62dDSLwfbukQvqlPkGUZmQwMAAOQWQ2ZxrtzRukbml2t3h3rl2Gj0rwAAAID8iOlyf5VSVRHPlgEAAJA/MQWLYzJ6KZ4tAwAAIH9Cz4bOZh+zuFgaqqJ/BQAAAORHiGDRFhQ6qpamIp7cAgAAgMmFCBbnSn2NFiwOZT9S7NeEBgAAgDcxlM4pHZTOAQAAyC2mCS4AAACYCggWAQAA4IpgEQAAAK4IFgEAAOCKYBEAAACuCBYBAADgimARAAAArggWAQAA4IpgEQAAAK4IFgEAAOCKYBEAAACuCBYBAADgimARAAAArq4odANK34gc3d0ppy7ZFlUslOYNK2RGwdoEAAAQjYiDxX452NYlw9q96sZWuWNetFsvRn0HMgJFAACAKSTSYLH/kBEoTh990j+obitk4T0bZMXMQrcHAAAgWtGNWTxzULqGIttaiamQKgJFAAAwBUUTLI4ek87uYZHy+TK/JpItlogFMrdW3V6S0QuFbgsAAED0IggWx+TYz07LZSmX+X/UIFXhN1hSZlxbISpYfHu00C0BAACIXuhgcez4y3L6sprQ0iwN0y1S1MyorNBvL42NFLglAAAA0Qs3wWX0mLzcp0WKNU35n/k81CV7DvXLeNYDV8rcVeulaVp1hwMAAMQjRGaxXw4+r7qfq6Vp1dzoWuSVFqA23PB7WYt/74aGPAaKfbL/SEK7nS1Lm6iqCAAApp7AmUWrTE514x1SgFBRd/3yxTLzrZ/LhQlzQdlMWbz8+phfVQsQdx6RRPL/qmzOGlkQ86sCAAAUQsBgsV/OmmVyhrvbpK07e43U8mppao0poLzyD6XppjOyr/cd/b/X3NQkf3hlHC8EAAAwPZX85f6uWdIgc88c1MLXudKw5Jo8vOICWbPZyiMaWcZTz+2Xqs1kFwEAwNRTNj4+PjH5at71H2rTi3Pn9XJ/vzomx6RBGm7w97REIiH19fXhXrtvv+w8kpCKRc2ygXGLAABgiin5zKLuBi1QLHQbAAAApqDoLvc3TY2MXdJvKyrJKgIAgKmHYDGkkYsqWKyQ66ZhQXIAADD1Rd4NPXdVa8FK6eRfn/QPqtsKqZpZ6LYAAABEj8xiJC7J6IVCtwEAACB6U2OCS8EskLm1RyQxeElOPbdTTlmLKxZK84YVwihGAABQ6sgshrRg9Wa5tbbQrQAAAIhH5HUWS0kkdRYBAACmMDKLAAAAcEWwCAAAAFcEiwAAAHBFsAgAAABXBIsAAABwRbAIAAAAVwSLAAAAcEWwCAAAAFcEiwAAAHBFsIjgXt0hZWVlUvZYd6FbAgAAYnJF8KeOybHO/XL6svsa5QvWSPPyyuAvMV2pIKxxi3Znu3RNPCyN0i07yppEX9I9IQ/f4vbE1HpJ69tl6NkWqY67zWEE3t/4jOzeIp0tb6Yv3LZVNj+yyPkJrz4jOxt/altwoyw8t11WuF033O/6AAAUCJnFYlRbJ81pC+qkbn3up3Q/ViZlmYGismej1BR75i/A/sZGBXFl92YHisrWbbLz3p/KSMZiFVimB37Km3Jqzr2ye/dvsjbjd30AAAopRGbRUi7z726WhqrwW4JJC56WaTed67WgSV9QLXU3aTd7mqXOIfM0vHudNG017mdl4gY7ZN2uuBscks/9jd+dcuvEfbLAvsjKBO55RU4P3ikzrHYN/lQOmYHl7O5nZY157K3M5KWWv5WjX7BlDP2uDwBAgZFZLEpmZu2muozu42XZwZMWDH6tpVO70yzt5xy6bGtbZO8jjTG2NQo+9jdut9wnmzMDRXP5rdvUnTfl7cHU4r5dz8glSQ/8lBkb/lRmi7H+W6/8JvD6AAAUGsFiiaib2yySzLylDL+yV/RQseN70uInsLImp9h+drya+ylGV7ftpzGr09veMum4N339dbuHPTfPbX/T9O2XnTt3aj/7pc/zlsO6Ua5LHueT0q9ndO+UuWlB+m/k6L3bJGH+79JzPWbXtd/1AQAovAi6oS/L6efb5LT13/L5sqa5QZjWEka1tDw7IS32JRv2ysSGzPWG5fBzRlZx3Re8T2FRQZ/VbW23pbFMtmzrkonMTKTqyp6zUQ9KPXFZv7OlRsr6HbbveX/tRuRojxVeJaSna4cmbKgAACAASURBVEQWNM3w2kL/Xn1Gjqhjtu1PbV3Kv9azhLKtIS0TObL7b+XUHpU9/Du57jt/rt0f1IO/GX7Xj29vAADwLPrM4uXTsr+tTdoO9Ue+aWQakIE96tZHd+2rO8xA0ei2npgwf861G5NMtjZlZBi7ZYcV+KmZ1RO253Rvd3iB1PrNHUOpdSeGpH290/aDmiErls4278+WpTEHivp4xfX3SbN9NvTgoBH8ZayrT47ZtlXW3PJpqVJjL7X3aXQwwPoAABSBEJnFSmlobpWGjKX9h9qka0i7M9QlnccrKZ2TD5N119p0HzK6jrd3703vtlZjG7sH9K7lLYe65eFbjOzf8O4njBnWThlHJ68e1tdXgeLeDfZsZ7W0fKdd9u7ZmLb9UBaskc1ZgwsjZg8Un73TMdtXMffT5r2Tst8pqAy5PgAAhRR5ZnHuqlZpvXu+lGv3L/f9XMgv5sGeARnwtGK3HNazittlpVPtwltWip4r3HpYjGI7qW7u9k3egjsrGNW7nDPGRJZZGco3BsT76MXCSZa4UfUVXQJF5VK/mpBijTu8URZ+x33dIOsDAFBI8UxwqWqQxTXqzmUZHY3lFaCz6hGekAE/3ZaumcjM+oZmN/f6dbJympVysUrZVHT8nXsh7tpaqVC3b/xajj725+a4Q3vZm9/I6Bvqtk6qagOsDwBAEWA2dEmrlpX3qJGGnbJxl4/C266ZSGsMZFgZ4yEzf4r+ijLPJAPFDRs+7b5e7WeM4G/PM3JKn/yyNa0cjgz2yFt6sF1rZA79rg8AQBGIJ1gcPSa9atyilEsVxbpjVf2FdebElCekY9LsYqOs1GsFbpHDTpNMzPGGsm2lpHU6OwWXasazQ+mcxlWqI7tT9r5SCh3NTsxxhFoglzNQ1C2SudvMuw7jDq2aihX3LDWDP7/rAwBQeCGCxX452HYwe0yiFih2Pn9a1CWjyxd8TuaGaBw8qG2R73WY2cU5DrUSVVBnu9yfEcypMjnr0oNLW/C3fZUVKqaCyybbNtQVY9T4Q1mffpE+nTnuUR+z6HCZQf25UV1+MIY6iyO7/1Gvdzh7lbcJJws23ZfMFr5qO/aqG1svtSM3yvVf+HTg9dMk93e3HL3gY6cAAAihbHx8fCLYU1Ww2OU+UaGmSVpXFXeomEgkpL6+vtDNiIRb7URdxkxmFbDVtDhXTcyaxexWY1Ftc9VhozB35kzpyeoyep1ZndOIHN3dKafMWjQVi5plQwTlc6yxijmpCS+2rGCu5zh1Zftd33xW2v5K7a2yeXXcU8EBAAiVWZwrd5izntOpa0W3Fn2gONU0PqLGA3ZJVuVDVRsxIzDTC1471EhU15VOL3cjRkkdqwajbb2cwZ56jlVXMf0VpGtikud6lsc6i5O1ZMN22dx9Z8bSG2XhuWcdAz+/65vPsu2v5p1RrvICAMiLEJnF0jeVMouYLvpk/84jkqhYKM0bVjC2EQAQO2ZDAyWk78ARfUxlxQ3zCRQBAHkRwbWhAcTPNmaxYqGsKmC3OwBgeiFYBEoJE1sAAHnGmEXGLAIAALhizCIAAABcESwCAADAFcEiAAAAXBEsAgAAwBXBIgAAAFwRLAIAAMAVwSIAAABcESwCAADAFcEiAAAAXHG5PwAeDcuB3TWy71L60qW3TsgDXIEQRWCoa508erIz0GfSeq7UdsnTqxsjalG3PLWzSXoq2uXRDS1SE9FWS0nPgTJ5ajB9Gd8ZpSeizOKYHOtsk7a29J/O42PRbH66eXWHlJWVaT87tK8apVt26P8vkx2vplYb3r1OW7ZOOgZdtjPYIevStgNPPB7/vOnbIffvLNN+dkiP2Z6n9P9rX8J9qdXUl/L9O9fJgQvWEms9+zIV8GnLDvCJQHEzPs/efuy/B6XL6ffV6XHre6DYGe3NDBSnrAsd8qiXz6PH7/NiEz5YPHNQCwz3y+nLEbQGhto6aU5bUCd1691W7pS9rww7PjL8yl7tUfjm6/jnQVVdRkaiTq6vyF6t+lrV6k55a9RccGFAjE+GbZkMyFuXRGqurQvQkGpZvWFCnt5s/Dy6qHnyp0xX1gmBoLxk1DTtNT7bkWUVg+qUff8c8+cmD5/Poa4njGBIZVU3p7431M+0zip6/D4vNuG6oUePSWe3cToqX7BGmpdXRtEmaMHKMu2mc70WpOgLqqXuJu1mT7PU1Wav3vncYRne0KKtZdctz7QQKgbi8/jHbmad/t4OVdSZ73G11Fyj3VxqluurUqvVVOqtTi0YHZAh8+7w2LD+PCuArK5M/7QAxWbpai2wsC9QmZvnNmq/B9OgS3ewSZ7qK+Wgalh6fqW+i5pl7e1T/L3yy+P3ebEJlVnsP3FaVEKRQDFqZibrprqMAHCZc7CyZ68czkz1v3pYtsTVvCnP5/HPQ3v0vzyvyfyLdJnUzMxe2wgMtS+jsRPaX/XN+nOGLg4YD9oCSADFq6eno4R/V40eDKlYJ0sdvqOmN3/f58UieGZx9Jj06p/kallMoBi7urnNIslMV0rztu0iW7foXdEtG1KhTfchLVTc1i7tb2yUjXuctjgsHffWpD3W3DEkezc4Z5zU+MiazEzlti6ZeMS5y8bv+k7tsWzvnpCHb/G+vvPr+NvfTG7HP03fftl5JKHdmS23bl4jcSYF9C7ndzKCWbN7wwgMq2X4onb8r9ku1Zc6ZegdFSQ2ml9OTn/BZk9eqVk0JI82RZiBtDJD9mVuWSLVTXYk/c+dzEHx1oQEq53JgfS2baYG12+XBzY/LEuTz87D/vrhYX+DsPbfcVvmazrtd3Kyh13OiR9Fdjwtace1Wdbes1dW207ITvuZq91ZE2gm2X4u9tdOf3+2y9LaLdIzuFH29bV4/AwU6fH3IPOYZr0njp8758l2SuZ2SuH7wfH7vMgEDxbH3tGzilJTL3PVf493yv6+1MDF6sZWuWNe6PZNU9XS8uyEtNiXbNgrExscVp27UtatF9nY8ox0b3hYjF+pbjm8VQVZdTKw1eE5auLLnI1Z4xk7W2qkrD870Op+rEyanLaztUnWzc0OuPyu79YeVxGt77a/vo5/0ogc7UmY9xPS0zUiC5pmeG3hJIyxgqttS/TxVU0Zq1ndG/p/hrUAUVvvhpVyvXbbc8kcv6iyjZl/wToFcWo7J2vk/ovRzAx1mhGpu7RRHj1Ql/Yabuv2HCmT+886tyftOdo2n+5aKWsv1ti2s0X2dd0nS9WXfWz7a858tS8abJL7d6avlXnSCbK/cXJ9r7R9ebTL4YSZh89PENmBYKfse26diI+ALq7tp56bGaQYGj7fLsNasKhnFxdM0o3r+fgH+3z6kvlHj/r93rnRtkLugNr1s2fnsr+5FMf3g8Xj93mRCdwNPfa2ERiWX1Mp/Yfa0gJFZbi7TdoO9YdrHTyok5X3qIkGW+SwNVNX74J2G1/XLTvMwEll1iYmJsyfIWlXXa9aQJc+49cIPNUvefu5Cdv6EzLU4TTBwf/6VntkfbsMhVh/ont7BPsb1AxZsXS2eX+29qUTVaDoh9m9oWcRzW4gbZk+HkZOyNAFMbKNFfa/YLUTiPnFqE4SqUHoQ7JWbUsfOxWuVerEaHwpqxNF+kD3pzd3pZ8otZON47r3tBsnTKf2XHxG9g2qk65t0o3Tsjztry9B9jdW3XLM5b1yntBUZMfTcnZHqgyONalC/z7slONnUhMCk5Na/E7Y8rh9J8lAUZ/4kR0o6ma2yFq1PT2wybW9Ij3+AQz3rEtmwfV9uNX5+9za38yJM67vXyl/PxSR8LOhB1+WLv3INklra6vxc/d8KVePDXXJwTOhXwGTqP7COn327pZDZqEX1QW9fp2sdAoWzbGM2V2w1dLynfa07aRx2J7Ktrl243pcf3j3E8bYStV1/GzmJJ1sftcPvL9BLFgjmzdv1n7i7YKelMoi2iayLK1XX7r2GdE2fYf1TEN2NkH76/d2I2DpORvm+HTLvpPmQHfHjEKjPGDPKp41shJLb81YVzt5PmqePDLbMzS4RapvTT/pOi3Txbq/jfrJJ+1EZwsmkic1e1YxwP7mhcNYMxVYZWWcYv/8BNMzaHSv27NAxu+BbfxuAbafFihOMkln6WrjD6mhk8+4l8rxdfz9fz59W/Bw+h+BWTOh3bOKQ9oft+oPlFzd7skZ1qrdHic5Fc/3Q2kLXZT78uXLIuXzZc2quamFVQ3S3PiOtHUPy/Av+0XmzXXfAMKrNbqiO7celu5HRM/sNXes1D7e2V9aeiApZhdsS9bDhjdUoNFoBmKNcl9Hs2xp2Sg1ZRuNbF7OIM3P+sNy+DkjkGjf5CW173f9IPtbyqxZdSdk6Izqjm6W5frYRGMs4/BYt941bR9YbQUrehfLSZfNpo139Mn88pXab3jo+rOyWtulwemEsWClLD2iTtRqm7b3Szshrc1c32mZ5GF/ffG3v44ZqMg1ytpFzdJz0uw+nCSoKa7jaaMFE7GO1wuwfdUVui9jzFxu1nthdZE6bLNYj38AWX8wqcBT+0mxzbD+vI+u4JL9figugTOLldeVJ++X186TrCkulddIeeYyxKQ61RX9mNEFve4L0X1R6uP1JrpE/1t0jwoCjQLVZfd2iFMHiff1B2RATThxy4Jm8bv+9GPUWhR566JtbKI1lvHi4RA1FsPx9ZoVbgO9neuRqXGZmV/cTsuKls/9jZvRNWtmhvQxZ2bx692lMzt3aX284yR9b187jvsCFKeuafrG5NnFaSPYDOuS/34oEsEzi3owOGyUzrnOYTa0NQEGeWF0RXfKlq1b9Gxe7mBKjSfcKy2+Aq5GeXhiQoy/89QVTZpkix4IDkjXhDWxJsz62Qb6o6oTGWR/S1mnDL/TbAtCjKCj550TjsG931mcQVgztD0xJ+Nkf5lb4zDDin9/fYl9f51ZWRRnRpelwZwYoQeOAw4TMorseBYjPZu4Unr0GbbZk7rc2bOL7S6/QRx/RR+PHQmOp5PgYxar5slnzNSh3tWcof+Xxmmp+vfpgs4Lsytaab5npetpuXGVMX7N7aov3hiBoDGhZIs8sXuybU2y/p6BrA5zVXrHcUa1y/r6jOfG7JNfNPtbOozC3Gr8T6etu9nqnu7UM0P2gtzWeMbJBuUHprpS1e3gEy6XMLNrlAY9oN8ix5wGkSe7tFcG7pKNfX8tWVdpcBL//rpJTTrywhzrdqvx+7vPNuEib8dzSlCzYM2MrT6z3Nsxq2n6nj65YuhXe7P+2At8/D19PouUVdnBxt/n2V2oz7MaZzyFr1ATYoJLpcyrNaPFoa6060CrMjpdZg3Gesrn5IlR7kXN9M1ZO/CWlXr3sD6G77Hsgbr69abty9V1kh27m4el4zsOmQlf6zfKym3qdos02V5Tld6paRHZvi1zdpvz+nqb56jxkQ6z4fzubxiqzuLOndrPfinYhDnbScDe9Wt1T2fVWDSDOX2MjsOlv9SXcLhLghmZEausiFPA2NOX2r41SaDnSMa6qpyFWZIjVBdj7PubwWH2ZE9Xqjs37v213nf7oHw1dk6va1frMNtUlT5x7G4elgM9Dr/v+T6eJU8F3l3JY+YtYNSCzKXbk3/spQl7/Cf5fBaX1B9XTx3I/Dxrvye1EVx+tJg/z8nzy245Oukf3tELNcGlcvltMn/QuC70ZW1H2tI+dOUy/+47hLxisWmUh8+1ywlVTmZrk5Q5Ze+2fSP9/3vMySqOtss3MoNTH+s3bmqX5q3ZbWnu+J48XPeMbNnaKScGtC/UW6pzrq9mR+9ddVjK9mR2RQTY30DirLPog63Woj2DmLoUYOZVArST1z3tRl0xh5prutrcx0evB3jE/I/D4H01Bu6Bi6rOmQoYy2Rf1va75GnrL/EFD8ujYyf0YMZpXTVLMdxf7eH31xO99MlGPduRdnwUdYysyQox72/NvHVSo207c1/17VY+I/e71b7c6f77uzZtYkeejmce2Sc3xFOI2XbMtNd6qtJDJmrBfbJWC9azi1AHPP5eP59FZunn26VmMHtfaxZ9z/w8d6YubRpIiM+zrT5jFAX109nPL5fk1D/3yYrV+U1fhiydUykNza2yZkHmVJZqaWptloYivs7htFbbInutOoNptkuX6i62F6m+5eFUTcJMep3DjPGHftdXbTlnlLAxGPUZ7dnRzv6BHOsbV3ixt7l5bsZkCj/7G1gx1Fm0y8gg5up20rtPzDpiaYw6ZFEUVdav82vVDsx8jYzt6xMsHGqsqS/gSE7cedhfRe1zVu03l2A6tv1V+5px3HNuVy994nRsxL0uYJ6O55RiL42kBWuTZxjN7KLbtgIcf6+fz6KS9Xk26oHaP8+hSyMV5efZfn7RvDOqhY/5VTY+Pj4x+WpTUyKRkPr6+kI3AxGxrhzjfHlAAABKWZ/s33lEEhULpXnDCslnSiJ8UW4gb9T1ncscr7iSusTgdllJoAgAmGL6DmiBonZbccP8vAaKSuii3EC+bWksE7eiH9u7vZXlAQCgNIzI0d2dckqvM7lQVhVgmBOZRZQQc8Z3xphFnXmdaLqfAQBTUu2tsjnP3c8WxiwyZhEAAMAVmUUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgCuCRQAAALgiWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgKsrgj+1Xw62dcnwZKvVNEnrqrnBX2Y6enWHlDVu0e5sl66Jh6VRumVHWZPoS7rTr388vHud1LR0OmzEem4J8LG/edG3Q+4/YrTngc0Py1KtPU/tbJIebcnSWyfkgQXZTxnqWiePnsx4H2q75OnVLu9A8jUszbL2nr2yemaOdl3okEef2yhDOdrh3h5rXyLiu/3DcmB3jey7lFoy2T7E0R7H9ymLh/fCi2J6v4qxPZmvVdEuj25okRrHtXx8fgL8/gLIjcxiMaqt005XdnVSt97vRrZIU1mZ7Hg1ojapgE7bXtlj3RFt0CaS/Y1QVV3GCatOrq9wWVedmHaWOQcgg01y/+4O/eRsp06O6YGN0in7ntO20+X851fPgTK53zzR56ZOqk7t2aKdMMvkqb5JNzAp/+1XJ+v0E73Sc6RQ7YlfMb1fxdieJC2AfXrS4N3n58fP7y8AT0JkFufKHa05MoZnDkpb97CUX1MZ/CWmKy14WqbddK7XgiZ9QbXU3aTd7GmWulrnp2Rm4LofK5OmrdpXfOMOWVnsGcYA+xurmXVaC7QgpMK4Ve2puUa7udQs11c5PcEh62JlNy7tlZ4LLVJjZalsJ0d7lsPKrgyd/JocmGfLatmyQSpT+ei1T+TMjA11fc08qdrblMrK9BzZIT0LQmSI/LZff+wJPatjz7Sq4OWpQe32rPbHx4IQn06f7alp2itPN7ltzDpOy1LvV4D2FNv7VVTtSaNt96XJA1jfnx/fv78AJhNTZrFfDmqBopTPl9uWEyz6Z2bWbrK+7CzLPAdPjY9MSNc2dW+LPLG7MNkV78Lvb9Tt0TMR12RmKByCCO3E+bRT95y2/AG97Z3y1mhqcc8/O3cD1jR9w9xGpxw/k/F+qe65zRPuXdpJ3bJPDwQyg9dqWb1hSNbq2ZUtcixEdihI+4cvqjY1y9rPp9q/9PPtLt2N8bfHjRUoLb01ZDBURO9XUbbH1HPACECXLsr9WfD/+fHx+wvAkxiCxTE51qnGMpbL/D9qEELFaNTNbRZJZt68adzUrnfvdvYPODw6LB33lhldyxk/0XRdh9u+p/3t2y87d+7UfvZL1L1jmaqv1dpTkRnMemHPZnTLsUF1u10a0sZNqcyNMaZKGfrV4VS2ZWZLjnFcGfoOmxmYlQ7BjpldETMbE0iA9ot57DKDttEBfZ2aa/18oqNpj6O+HUbGrbYr3Ji2fL5f5hCI+3fuSO5r0bfHtq7KDOrHe17uVaP4/AT//QWgRB4sjh1/WU5f1n45G5ulgZR/QNXS8uyETDyS+ku6esNemXi2xd+Xndm9K28MpE9EGuyQdWU1snHPZBtQE03MIK/RHBO2tSkr+FuXmbn0vH1LkP0dkaM9CfN+Qnq6Rry+mKf2rN6QnonRuy+9nnSV5MnwG7YuZfN9yDgZpzJaZvbm0sDkE8dymPQE+s6Ah7FrDgK238ryDZ2sMccQasFcjzEBYW1TiNN3ZMezW55SQwZUBm7S7Fv0gr1f1jFUtsi+CMdmxt8ef8fb/+cngt9fAGmiDRbPHJT9fZf1GdB3TPLXIvLB7N7dMyCp3KIWAM7ZKPqopfXtMjShBWnmz1BHs9uGfIh7+5YZsmLpbPP+bFnaNCPCbYdkjVfMPBmOOpxk0zJaVvbmhAxdCPC65sB+50yalYWT4MFo4PY3ygO3btfvqRP+/fpkBTXjOGR3byTH08pCau25Pc/BRKj3SwuIlm4374cMuvPcnp4Dfo93TJ8fAJ5FFyyOHpNOc5ziGkrlFK3h3U/oJWlkW5eHTGWjPGwFe93miUA9zxYAqp+9G1Jb8bf9kBaskc2bN2s/a6RoqmHYA0WXTEYqcxNxRmvmSlmuZ9I2ytNpmR2rdEiz1EQwKzSa9qeP5SxUe5Lj5m6NoFSOX2HfL3287ITzmNkibY+aeKQy7jWLvhfyeEf3+QEwuRCzoe3G5NjPTstlxikWuWE5/JwxWLx9UxzdbXFvv7gl68Xlqq+o1rto5Hmjz2gZ2Z19WsCkZ2BO2h9TmZhvyFva+5OeNcquX5cue6a33/ZnHRczoFalTx4dG5JH07JQ8bcnyT5uriB/bQR5v0q4PdbMdS2Yv99HJtTf5wdAHCLJLFrjFMsX3MY4xaIyIANq3GByooj1/3WyMpZZxnFvv3hZJ7SaRUPugaJV/+2dATngmNEalqF31G2IWZt6dqcrI7OjAiztdcTsPgw60D9I+60AwR5A29qogpLAtftCHU8zC6mOTQHGKSbF+X4VWXuGzuw1As1LG+VRfSKM+WOV9rEtT9bHjPPzA8CzCDKL/fJzNU6RMjnFZ3BATqjbrJI0zgb6JyuOG07c2y8Yc5ycChRzZjms+m/aSVHPnGVmtC4cluNqeejgoFE7uU9kLU2erNNKihiTAVZ72WyA9huvmV72JNnGe9r1GoDptfLibY/Fqt0XukxOJPy8X9OrPf4/PwDiED5YPHNW/2uzvHYe3c9FpnuXMdFk+6qML1Jzwkta59/udXoRb1fmVVY8hXtBtl+yzAyVKng8aXdYozTUivSork+HcXVWzcCaG1bGcDK2aug5nXi98t9+o0aeGOPLIh8TGPR4DkvPr4x6gg1FM9g1UxTvV5TCt8e1ILpVONxhnG+8nx8AXoXuhh57+7J+W34doWIxsa7gomYk35e8skujrDQLdTfZLtun1q1p0U6d2zzMVt7alFUnsXt3hzk7MoLt+5HHOotukhmqem8n0GQxYZUNszXaGvivTsbL50Xc6ahOxua1cdNK+QTgt/1GjTx15Y91csA+I9l2ZZEwtRaDHc8BeSuSDG5MvL5ffuoalmh74v78APCmbHx8PLu/wYex4516uZzqxtaSK5eTSCSkvr6+0M0IRWXsalrc8n3N0n5ur7TYxw+qGohWaRv7mh1DsrfuGb2eon5/Q/ZpNBmAZlIlcqyZzyG278+IHN3dKafMiRAVi5plQwHK5yQH3+eSMeEl13OyurLtl2tzZZv0kWv9HDO0/fDV/uRMWjcOl0qMtT22NkV0PNLk7f1KnwjkOgSi2NrjJEdmMR+fHwCTi+lyfyg4vcZhRqCo1LbI3nPGlV0MKqBML3/jfMUX4xKCWbUS7YFiyO37U8R1Fiehd8fduj1jqZptOhHbzE51ObyoihL7a78x/u0BpwlPKoiO4ERfiOMZt8nfrxjqLBZle+L//ACYXOjMYimbCplFAACAOJFZBAAAgCuCRQAAALgiWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgCuCRQAAALgiWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgKsrCt2A0tIn+3cekYRtScWiZtnQNKNgLQIAAIhTNMHimYPS1j2c+n/5fFnT3CCVkWy8uF062Sm7hYARAABMTWXj4+MTYTYwdrxT9vddzn6gBALGRCIh9fX1wTfQt192HkmIVCyU5g0rhHARAABMNSEzi/3ycz1QLJf5dzdLQ5V2d/SYdD5/Wi5f/rWcGRVj2VRVdZ1USEIuFbodAAAAMQk3wWV0VPScYs3iVFBY1SCLa8I2q0SMvq0HihU3zCerCAAApqRwwWJVlZSr26Gz0p9cOCajeqqtXD08pfWdNaa6VFQSKgIAgKkpZOmcufK5BSpcHJautoNGwHjm53L6shYqLvic9uh0UCHXTfGgGAAATF+h6yxWLr9N5uvpRRUwthmzomuapHl5MU9tAQAAgBfxFOW+NCpjsWy4uCyon639e0lOvXRURgrdGAAAgBiELJ3TLwfbumQ4ORt6TI517te7oUWqpan1jqLuig5dOkexyudYKKMDAACmkFCZxf5D9kBRLamUhuZWWWONY+w8Ni0yjAAAAFNViDqL/XJ2SNLL5pgqlzdL0ztt0jU0xWstUpQbAABMccEzi1aNxWkyPtGJUTqnQhbeTqAIAACmpuDBolVj8fJp2X+oP+2h/kMqq6juTf1aiwAAAFNZiG5oVWOx17gu9FCXtLV1Za0xPWotXpK3R7WbmYVuBwAAQPRCTXBRYxNb755vZBjTqEkvrVO+1qJROkcLF8conAMAAKamEJlFU1WDNLc2RNCUElR1nVRIQi796rSMNM1g3CIAAJhy4inKPV2Mvi2XCt0GAACAGIXPLE4rfbJ/5xFJZCytuGE+WUUAADAlkVkMqWJRs2xoIlQEAABTU8jL/ZW2SC73BwAAMIWRWQQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgCuCRQAAALgiWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgCuCRQAAALgi5Weo4wAAAE1JREFUWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgCuCRQAAALgiWAQAAIArgkUAAAC4IlgEAACAK4JFAAAAuCJYBAAAgKv/H8Z4KCi3k6N1AAAAAElFTkSuQmCC
[img2]:data:image/png;base64,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
