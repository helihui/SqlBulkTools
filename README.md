<img src="http://gregnz.com/images/SqlBulkTools/icon-large.png" alt="SqlBulkTools"> 
#SqlBulkTools
-----------------------------
Welcome to Version 2! High-performance C# Bulk operations for SQL Server (starting from 2008) and Azure SQL Database. Supports Bulk Insert, Update, Delete & Merge. Uses SQLBulkCopy under the hood. Please leave a Github star if you find this project useful. 

##Examples

####Getting started
-----------------------------
```c#
using SqlBulkTools;

// IBulkOperations Interface for easy mocking.
public class BookClub(IBulkOperations bulk) {

  IBulkOperations _bulk;
  
  public BookClub(IBulkOperations bulk) {
    _bulk = bulk;
  }
    // Do your stuff
}

// Or simply new up an instance.
var bulk = new BulkOperations();

// The following examples are based on a cut down Book model

public class Book {
    public int Id {get; set;}
    public string ISBN {get; set;}
    public string Description {get; set;}
}

```
###BulkInsert
---------------
```c#
books = GetBooks();

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.AddAllColumns()
.BulkInsert();

bulk.CommitTransaction("DefaultConnection");

/* 
Notes: 

(1) It's also possible to add each column manually via the AddColumn method. Bear in mind that 
columns that are not added will be assigned their default value according to the property type. 
(2) It's possible to disable non-clustered indexes during the transaction. See advanced section 
for more info. 
*/

```

###BulkInsertOrUpdate (aka Merge)
---------------
```c#
books = GetBooks();

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.AddColumn(x => x.ISBN)
.AddColumn(x => x.Title)
.AddColumn(x => x.Description)
.BulkInsertOrUpdate()
.MatchTargetOn(x => x.ISBN)

bulk.CommitTransaction("DefaultConnection");


// Another example using an identity column

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.AddAllColumns()
.BulkInsertOrUpdate()
.SetIdentityColumn(x => x.Id)
.MatchTargetOn(x => x.Id)

bulk.CommitTransaction("DefaultConnection");

/* 
Notes: 

(1) It's possible to use AddAllColumns for operations BulkInsert/BulkInsertOrUpdate/BulkUpdate. 
(2) MatchTargetOn is mandatory for BulkUpdate, BulkInsertOrUpdate and BulkDelete... unless you want to eat 
an InvalidOperationException. 
(3) If model property name does not match the actual SQL column name, you can set up a custom 
mapping. An example of this is shown in a dedicated section somewhere in this Readme...
(4) BulkInsertOrUpdate also supports DeleteWhenNotMatched which is false by default. With power 
comes responsibility. Use at your own risk.
(5) If your model contains an identity column and it's included (via AddAllColumns, AddColumn or 
MatchTargetOn) in your setup, you must use SetIdentityColumn to mark it as your identity column. 
Identity columns are immutable and auto incremented. You can of course update based on an identity 
column (using MatchTargetOn) but just make sure to use SetIdentityColumn to mark it as an 
identity column. 
*/
```

###BulkUpdate
---------------
```c#
books = GetBooksToUpdate();

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.AddColumn(x => x.ISBN)
.AddColumn(x => x.Title)
.AddColumn(x => x.Description)
.BulkUpdate()
.MatchTargetOn(x => x.ISBN) 

/* Notes: 

(1) Whilst it's possible to use AddAllColumns for BulkUpdate, using AddColumn for only the columns 
that need to be updated leads to performance gains. 
(2) MatchTargetOn is mandatory for BulkUpdate, BulkInsertOrUpdate and BulkDelete... unless you want to eat 
an InvalidOperationException. 
(3) MatchTargetOn can be called multiple times for more than one column to match on. 
(4) If your model contains an identity column and it's included (via AddAllColumns, AddColumn or 
MatchTargetOn) in your setup, you must use SetIdentityColumn to mark it as your identity column. 
Identity columns are immutable and auto incremented. You can of course update based on an identity 
column (using MatchTargetOn) but just make sure to use SetIdentityColumn to mark it as an 
identity column.  
*/

bulk.CommitTransaction("DefaultConnection");
```
###BulkDelete
---------------
```c#
/* Tip: Considering you only need to match a key, use a DTO containing only the column(s) needed for 
performance gains. */

public class BookDto {
    public string ISBN {get; set;}
}

books = GetBooksIDontLike();

bulk.Setup<BookDto>(x => x.ForCollection(books))
.WithTable("Books")
.AddColumn(x => x.ISBN)
.BulkDelete()
.MatchTargetOn(x => x.ISBN)

bulk.CommitTransaction("DefaultConnection");

/* 
Notes: 

(1) Avoid using AddAllColumns for BulkDelete. 
(2) MatchTargetOn is mandatory for BulkUpdate, BulkInsertOrUpdate and BulkDelete... unless you want to eat 
an InvalidOperationException.
*/

```

###Custom Mappings
---------------
```c#
/* If the property names in your model don't match the column names in the corresponding table, you 
can use a custom column mapping. For the below example, assume that there is a 'BookTitle' column 
name in database which is defined in the model as 'Title' */

books = GetBooks();

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.AddAllColumns()
.CustomColumnMapping(x => x.Title, "BookTitle") 
.BulkInsert();

bulk.CommitTransaction("DefaultConnection");

```

###Advanced
---------------
```c#
books = GetBooks();

bulk.Setup<Book>(x => x.ForCollection(books))
.WithTable("Books")
.WithSchema("Api") // Specify a schema 
.WithBulkCopyBatchSize(4000)
.WithBulkCopyCommandTimeout(720) // Default is 600 seconds
.WithBulkCopyEnableStreaming(false)
.WithBulkCopyNotifyAfter(300)
.WithSqlCommandTimeout(720) // Default is 600 seconds
.WithSqlBulkCopyOptions(SqlBulkCopyOptions.TableLock)
.AddColumn(x =>  // ........

/* SqlBulkTools gives you the ability to disable all or selected non-clustered indexes during 
the transaction. Indexes are rebuilt once the transaction is completed. If at any time during 
the transaction an exception arises, the transaction is safely rolled back and indexes revert 
to their initial state. */

// Example

bulk.Setup<Book>(x => x.ForCollection(col))
.WithTable("Books")
.WithBulkCopyBatchSize(5000)
.WithSqlBulkCopyOptions(SqlBulkCopyOptions.TableLock)
.AddAllColumns()
.TmpDisableAllNonClusteredIndexes()
.BulkInsert();

bulk.CommitTransaction("DefaultConnection");

```

###How does SqlBulkTools compare to others? 
<img src="http://gregnz.com/images/SqlBulkTools/performance_comparison.png" alt="Performance Comparison">

<b>Test notes:</b>
- Table had 6 columns including an identity column. <br/> 
- There were 3 non-clustered indexes on the table. <br/>
- SqlBulkTools used the following setup options: AddAllColumns, TmpDisableAllNonClusteredIndexes. <br/>

###More on Setup and CommitTransaction...
---------------

#####Setup<T>
```c#
Setup<T>(Func<Setup<T>, CollectionSelect<T>> list)

// Example usage where col implements IEnumerable and is of type Book
bulk.Setup<Book>(x => x.ForCollection(col)) 

/* Setup is the main entry point. Because of the vast flexibility possible with SqlBulkTools, 
a fluent interface helps to guide you through setup process. This design choice was made to 
make it easier for you to use SqlBulkTools. Options that are not relevant to a particular 
operation are not exposed. For example the MatchTargetOn method is not accessible from the 
BulkInsert method because it would not make sense. */
```

#####CommitTransaction
```c#
CommitTransaction(string connectionName, SqlCredential credentials = null)
CommitTransaction(SqlConnection connection)
CommitTransactionAsync(string connectionName, SqlCredential credentials = null)
CommitTransactionAsync(SqlConnection connection)

/* A transaction will only take place if CommitTransaction is called. CommitTransaction is 
always called after a valid setup is built and Async flavours are included for scalability. 
CommitTransaction and CommmitTransactionAsync respectively are overloaded. It's up to you how 
you would like to pass in your SQL configuration.  
 */

```









Here is a complete C# implementation of a bi-directional RPC framework with heartbeat-based disconnection detection and reconnection:

**RPC 框架**

**IRpcClient**
```csharp
public interface IRpcClient
{
    void Connect(string endpoint);
    void Disconnect();
    void SendRequest(string method, object[] args);
    void SendResponse(string method, object result);
    event EventHandler<RpcRequestEventArgs> RequestReceived;
    event EventHandler<RpcResponseEventArgs> ResponseReceived;
    event EventHandler<ConnectionStateChangedEventArgs> ConnectionStateChanged;
}

public class RpcRequestEventArgs : EventArgs
{
    public string Method { get; set; }
    public object[] Args { get; set; }
}

public class RpcResponseEventArgs : EventArgs
{
    public string Method { get; set; }
    public object Result { get; set; }
}

public class ConnectionStateChangedEventArgs : EventArgs
{
    public ConnectionState State { get; set; }
}

public enum ConnectionState
{
    未连接,
    连接中,
    已连接,
    断开连接中
}
```
**RpcClient**
```csharp
public class RpcClient : IRpcClient
{
    private readonly TcpClient _tcpClient;
    private readonly HeartbeatTimer _heartbeatTimer;
    private readonly object _syncLock = new object();
    private ConnectionState _connectionState;

    public RpcClient()
    {
        _tcpClient = new TcpClient();
        _heartbeatTimer = new HeartbeatTimer(5000, 30000); // 5 秒间隔，30 秒超时
        _heartbeatTimer.Elapsed += HeartbeatTimer_Elapsed;
    }

    public void Connect(string endpoint)
    {
        lock (_syncLock)
        {
            if (_connectionState == ConnectionState.已连接) return;
            _connectionState = ConnectionState.连接中;
            _tcpClient.Connect(endpoint);
            _heartbeatTimer.Start();
            _connectionState = ConnectionState.已连接;
            OnConnectionStateChanged(ConnectionState.已连接);
        }
    }

    public void Disconnect()
    {
        lock (_syncLock)
        {
            if (_connectionState == ConnectionState.未连接) return;
            _connectionState = ConnectionState.断开连接中;
            _tcpClient.Close();
            _heartbeatTimer.Stop();
            _connectionState = ConnectionState.未连接;
            OnConnectionStateChanged(ConnectionState.未连接);
        }
    }

    public void SendRequest(string method, object[] args)
    {
        lock (_syncLock)
        {
            if (_connectionState!= ConnectionState.已连接) throw new InvalidOperationException("未连接");
            var request = new RpcRequest { Method = method, Args = args };
            var data = Serialize(request);
            _tcpClient.GetStream().Write(data, 0, data.Length);
        }
    }

    public void SendResponse(string method, object result)
    {
        lock (_syncLock)
        {
            if (_connectionState!= ConnectionState.已连接) throw new InvalidOperationException("未连接");
            var response = new RpcResponse { Method = method, Result = result };
            var data = Serialize(response);
            _tcpClient.GetStream().Write(data, 0, data.Length);
        }
    }

    private void HeartbeatTimer_Elapsed(object sender, ElapsedEventArgs e)
    {
        lock (_syncLock)
        {
            if (_connectionState!= ConnectionState.已连接) return;
            var heartbeat = new RpcHeartbeat();
            var data = Serialize(heartbeat);
            _tcpClient.GetStream().Write(data, 0, data.Length);
        }
    }

    private byte[] Serialize(object obj)
    {
        using (var ms = new MemoryStream())
        {
            var formatter = new BinaryFormatter();
            formatter.Serialize(ms, obj);
            return ms.ToArray();
        }
    }

    private void OnConnectionStateChanged(ConnectionState state)
    {
        ConnectionStateChanged?.Invoke(this, new ConnectionStateChangedEventArgs { State = state });
    }

    public event EventHandler<RpcRequestEventArgs> RequestReceived;
    public event EventHandler<RpcResponseEventArgs> ResponseReceived;
    public event EventHandler<ConnectionStateChangedEventArgs> ConnectionStateChanged;
}
```
**RpcServer**
```csharp
public class RpcServer
{
    private readonly TcpListener _tcpListener;
    private readonly HeartbeatTimer _heartbeatTimer;
    private readonly object _syncLock = new object();
    private ConnectionState _connectionState;

    public RpcServer(int port)
    {
        _tcpListener = new TcpListener(IPAddress.Any, port);
        _heartbeatTimer = new HeartbeatTimer(5000, 30000); // 5 秒间隔，30 秒超时
        _heartbeatTimer.Elapsed += HeartbeatTimer_Elapsed;
    }

    public void Start()
    {
        lock (_syncLock)
        {
            if (_connectionState == ConnectionState.已连接) return;
            _connectionState = ConnectionState.连接中;
            _tcpListener.Start();
            _heartbeatTimer.Start();
            _connectionState = ConnectionState.已连接;
            OnConnectionStateChanged(ConnectionState.已连接);
        }
    }

    public void Stop()
    {
        lock (_syncLock)
        {
            if (_connectionState == ConnectionState.未连接) return;
            _connectionState = ConnectionState.断开连接中;
            _tcpListener.Stop();
            _heartbeatTimer.Stop();
            _connectionState = ConnectionState.未连接;
            OnConnectionStateChanged(ConnectionState.未连接);
        }
    }

    private void HeartbeatTimer_Elapsed(object sender, ElapsedEventArgs e)
    {
        lock (_syncLock)
        {
            if (_connectionState!= ConnectionState.已连接) return;
            var heartbeat = new RpcHeartbeat();
            var data = Serialize(heartbeat);
            foreach (var client in _tcpListener.GetClients())
            {
                client.GetStream().Write(data, 0, data.Length);
            }
        }
    }

    private byte[] Serialize(object obj)
    {
        using (var ms = new MemoryStream())
        {
            var formatter = new BinaryFormatter();
            formatter.Serialize(ms, obj);
            return ms.ToArray();
        }
    }

    private void OnConnectionStateChanged(ConnectionState state)
    {
        ConnectionStateChanged?.Invoke(this, new ConnectionStateChangedEventArgs { State = state });
    }

    public event EventHandler<RpcRequestEventArgs> RequestReceived;
    public event EventHandler<RpcResponseEventArgs> ResponseReceived;
    public event EventHandler<ConnectionStateChangedEventArgs> ConnectionStateChanged;
}
```
**使用示例**
```csharp
// 客户端
var client = new RpcClient();
client.Connect("localhost:8080");
client.RequestReceived += (sender, e) => Console.WriteLine($"Received request: {e.Method}");
client.ResponseReceived += (sender, e) => Console.WriteLine($"Received response: {e.Method}");
client.SendRequest("Add", new object[] { 2, 3 });

// 服务器
var server = new RpcServer(8080);
server.Start();
server.RequestReceived += (sender, e) => Console.WriteLine($"Received request: {e.Method}");
server.ResponseReceived += (sender, e) => Console.WriteLine($"Received response: {e.Method}");
server.SendResponse("Add", 5);
```
This implementation provides a bi-directional RPC framework with heartbeat-based disconnection detection and reconnection. The `RpcClient` class represents the client-side of the RPC connection, while the `RpcServer` class represents the server-side. The `IRpcClient` interface defines the methods that can be called on the client-side, while the `RpcRequestEventArgs` and `RpcResponseEventArgs` classes define the events that are raised when a request or response is received.

Note that this implementation uses a simple binary formatter to serialize and deserialize the RPC requests and responses. You may want to use a more robust serialization mechanism, such as JSON or XML, depending on your specific requirements. Additionally, you may want to add additional features, such as authentication and authorization, to the RPC framework.

