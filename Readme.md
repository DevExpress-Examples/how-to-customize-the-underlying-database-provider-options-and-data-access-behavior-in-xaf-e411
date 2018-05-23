# How to customize the underlying database provider options and data access behavior in XAF


<p><strong>IMPORTANT NOTE</strong></p>
<p>This article describes some advanced customization techniques and low-level entities of the framework with regard to data access, which may be required in complex scenarios only.<br>So, if you just want to change the connection string, e.g. to use the Oracle instead of the Microsoft SQL Server database, then you would better refer to the <a href="https://documentation.devexpress.com/#Xaf/CustomDocument3155">Connect an XAF Application to a Database Provider</a> article and documentation on your database provider instead. The XAF integration of supported ORM libraries is also described in the <a href="https://documentation.devexpress.com/#Xaf/CustomDocument3461">Business Model Design</a> section of the framework's documentation.</p>
<p><br><strong>Introducing IObjectSpaceProvider and IObjectSpace</strong><br>XAF accesses data from a data store through special abstractions called - <em>IObjectSpaceProvider</em> and <em>IObjectSpace</em>.<br>The <a href="https://documentation.devexpress.com/#Xaf/clsDevExpressExpressAppBaseObjectSpacetopic">IObjectSpace</a> is an abstraction above the ORM-specific database context (e.g., the DBContext used in Entity Framework or the Session in XPO) allowing you to query or modify data.<br>The IObjectSpaceProvider is a provider/creator of IObjectSpace entities, which also manages which business types these IObjectSpace are supposed to work with, how to set up the underlying connection to the database, create and update it and other low level data access options.<br><br><img src="https://raw.githubusercontent.com/DevExpress-Examples/how-to-customize-the-underlying-database-provider-options-and-data-access-behavior-in-xaf-e411/8.3.2+/media/deacd98f-10a9-11e4-80b8-00155d624807.png"><br><br>An XafApplication can use one or several IObjectSpaceProvider objects at the same time, and you can access this information through the <a href="https://documentation.devexpress.com/#Xaf/DevExpressExpressAppXafApplication_ObjectSpaceProvidertopic">XafApplication.ObjectSpaceProvder</a> or <a href="https://documentation.devexpress.com/#Xaf/DevExpressExpressAppXafApplication_ObjectSpaceProviderstopic">XafApplication.ObjectSpaceProviders</a> properties. Check out these help links to learn more on how to plug in custom IObjectSpaceProvider objects a well.<br><br>There are several built-in implementations of the IObjectSpaceProvider and IObjectSpace interfaces in our framework, which are usually specific to a target ORM (Entity Framework or XPO). I suggest you check out the source code of the default framework classes to better understand the role of the IObjectSpaceProvider:</p>
<p><em>...\DevExpress.ExpressApp.Xpo\XPObjectSpaceProvider.cs</em><br><em>...\DevExpress.ExpressApp.EF\EFObjectSpaceProvider.cs</em> <br><br><strong>Typical customization considerations</strong><br>You may want to provide a fully custom IObjectSpaceProvider implementation or inherit from the built-in implementors when you want to customize how data access is performed for a chosen ORM.<br>Below is a list of typical scenarios where a custom IObjectSpaceProvider may be required:</p>
<p>1. <a href="https://www.devexpress.com/Support/Center/p/K18356">How to use XPO caching in XAF</a> <br>2. <a href="https://www.devexpress.com/Support/Center/p/E1150">How to prevent altering the legacy database schema when creating an XAF application</a> <br>3. <a href="https://www.devexpress.com/Support/Center/p/E5137">How to connect to remote data store and configure WCF end point programmatically</a> <br>4. <a href="https://www.devexpress.com/Support/Center/p/Q439032">How do I map persistent classes to another schema, e.g. other than the default "dbo" in MS SQL Server?</a><br>5. <a href="https://www.devexpress.com/Support/Center/p/Q305734">How to use a custom ObjectSpace throughout the application by handling the CreateCustomObjectSpaceProvider event?</a> <br>6. <a href="https://www.devexpress.com/Support/Center/p/E4896">How to connect different ORM data models to several databases within a single application</a><br>7. <a href="https://www.devexpress.com/Support/Center/p/T590896">How to customize the Object Space behavior in XPO-based XAF applications</a><br>8. <a href="https://www.devexpress.com/Support/Center/p/T591324">How to customize the UnitOfWork behavior in XPO-based XAF applications</a><br><br>In most cases, it is not required to implement the IObjectSpaceProvider interface from scratch since you can inherit from existing implementors or customize/just their parts.<br><br><br><strong>XPO-specific customizations<br><br>1. Implementing IXpoDataStoreProvider</strong><br>For instance, in XPO one of such replaceable parts is the <em>IXpoDataStoreProvider</em> interface, which enables you to provide a custom or configured <a href="https://documentation.devexpress.com/#CoreLibraries/clsDevExpressXpoDBIDataStoretopic">IDataStore</a> object, which is used for underlying data access with this ORM:</p>


```cs
    public interface IXpoDataStoreProvider {
        IDataStore CreateWorkingStore(out IDisposable[] disposableObjects);
        IDataStore CreateUpdatingStore(out IDisposable[] disposableObjects);
        IDataStore CreateSchemaCheckingStore(out IDisposable[] disposableObjects);
        string ConnectionString { get; }
    }

```


<p> </p>
<p>In its turn, XAF provides several ready to use implementations of this interface for most popular scenarios: </p>
<p>1. ConnectionDataStoreProvider - can provide IDataStore by the IDbConnection object;</p>
<p>2. ConnectionStringDataStoreProvider - can provide IDataStore by just connecting string information;</p>
<p>3. MemoryDataStoreProvider - DataSet based in-memory IDataStore providers.</p>
<p><br>Technically, such an IXpoDataStoreProvider part is passed into the XPObjectSpaceProvider constructor as a parameter, which means that you can just customize it instead of re-implementing the whole XPObjectSpaceProvider logic:</p>


```cs
//In your XafApplication descendant class (e.g., in the YourSolutionName.Win/WinApplication.cs file).
protected override void CreateDefaultObjectSpaceProvider(CreateCustomObjectSpaceProviderEventArgs args) {
    args.ObjectSpaceProvider = new XPObjectSpaceProvider(new MyIXpoDataStoreProvider(args.ConnectionString, args.Connection, false), true);
    args.ObjectSpaceProviders.Add(new NonPersistentObjectSpaceProvider(TypesInfo, null));
}
```


<p>You can find IXpoDataStoreProvider implementation examples in the following articles:<br><a href="https://www.devexpress.com/Support/Center/p/K18356">How to use XPO caching in XAF</a><br><a href="https://www.devexpress.com/Support/Center/p/E1150">How to prevent altering the legacy database schema when creating an XAF application</a><br><br>Refer to the <em>...\DevExpress.ExpressApp.Xpo\XPObjectSpaceProvider.cs</em> file within the XAF source code to better understand the role of this part.<br><br><strong>2. Handling the DataStoreCreated event of ConnectionDataStoreProvider and ConnectionStringDataStoreProvider <br></strong>Instead of creating your own IXpoDataStoreProvider  from scratch, it is often simpler to explicitly create a ConnectionDataStoreProvider or ConnectionStringDataStoreProvider instance (an <em>IXpoDataStoreProvider</em> implementer) and subscribe to its <em>DataStoreCreated </em>event (available starting with version 17.2).  The event arguments (<em>DataStoreCreatedEventArgs</em>) expose the <em>DataStore </em>and <em>Destination </em>parameters that provide access to the current data store and its type (SchemaChecking,  Updating or Working). Refer to the <a href="https://www.devexpress.com/Support/Center/p/Q439032">How do I map persistent classes to another schema, e.g. other than the default "dbo" in MS SQL Server?</a>  example for more details.</p>
<p> </p>
<p><strong>3. Overriding XPO connection or database providers</strong></p>
<p>To learn more on this approach, check out the <a href="https://documentation.devexpress.com/#CoreLibraries/CustomDocument2114"><u>Database Systems Supported by XPO</u></a> help topic and <a href="https://www.devexpress.com/Support/Center/Question/Details/K18098">How to create a custom XPO connection provider and then use it in an XAF application</a> article in particular.  To learn more on customizing data access settings for XPO, please refer to the corresponding product documentation: <a href="https://documentation.devexpress.com/#XPO/CustomDocument2121">Data Access Layer</a>.</p>
<p><br><br><strong>See Also:</strong><br><a href="https://www.devexpress.com/Support/Center/p/T400009">Can I connect an XAF application to a custom data source (Web service, OData service, NoSQL database, etc.)?</a></p>

<br/>


