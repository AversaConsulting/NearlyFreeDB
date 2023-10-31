# NearlyFreeDB - Azure Table Storage Pumped Up!
Nearly Free Document Database - easy storage of stronly typed and indexed documents on Azure for next to nothing!
# The Idea
One of the lowest cost and most scalable cloud databases is Azure Table Storage. It is a noSQL key-value store, with multiple columns in addition to it's PartitionKey and RowKey. What if we can leverage this to store and query more complex .NET objects?
## Azure Table Downsides
- Only PartitionKey and RowKey are indexed. For performance, and avoid full row scans, we should use these where possible.
- Certain .NET types are NOT supported in Azure Tables.
- Columns are limited to 32kb - a barrier for storing larger text or binary data (although ideally larger data should be in a blob - more later!)
## What we have done
We already have an in-use (available on NuGet) system called _Alte_. This used .NET API libraries, which have now been depreciated. And again... and again... We'll describe how Alte works later - which is now becoming NearlyFreeDB.
## What we are doing
We are re-writing Alte as NearlyFreeDB. It will not be dependant on any MS APIs for Azure Storage. At present it is only dependant on Newtonsoft JSON - we will aim to change this to MS JSON. We will keep the current Alte principles as we know they work in production, but also enhance, improve and remove en-route.
## How it works
By following lots of Microsoft guidance on Table structure, insights from similar projects, and our own experience developing and using _Alte_ - the following describes the basics of how **NearlyFreeDB** works. Bear in mind that there ARE compromises - but we think the gains are worth it.
* Table Name
  * There are a few options for this:
    * The type-name of the object (e.g. Order) is used as the table name. This is best for larger databases for individual users, rather than multi-tenant applications
    * The type-name prefixed by a pseudo database name (e.g. Woolworth-Order) - this allows for multiple tenants in one storage account
    * The pseudo database name (e.g. Woolworth). This would then rely on the PartitionKey (see below) to determine the object type
* PartitionKey
  * Normally, the partition key should be used to best distribute the data. However - we will be relying on batch updates (more on this later) so we need the partition keys to be the same for all data within a type, so....
  * In the event the type-name is set as the table name - or part of it, the partition key is simply "00"
  * In the event the type-name is NOT part of the table name, the partition key is the type-name e.g. Order
* The Document Object
  * Yes - ideally we would like to save simple POCO... plain old common objects... However, we need to be a little clever later on. So we will force you to inherit from a NearlyFree object. This object will in addition to your normal properties include:-
   * An enforced ID field. We will include utilties to generate a whole host of IDs from GUID to reverse integer timestamps... Most of the time a simple 6 digit random aplhanumeric will suffice for smaller applications. Self inremement will also be included - but best avoided!
   * Timestamp - pulled in from Azure.
   * The Etag - this will be pulled from the table storage - so that we can have optomistic concurrency as provided by Table Storage. However, as you'll see later, allongside Etag, it also has an "EtagKey".
   * A list of indexed column values. See later for how this works. It is not persisted, but is used to modifiy index objects on saving.
   * Decorations - allowing you to specify indexed properties/columns.
* RowKey
   * You'll have noticed that PartitionKey and RowKey are actually missing from the Document Object. This is where the clever stuff comes in. We only use these when saving and retrieving data. You don't need to know above how this works, unless you're using Storage Exporer for instance. For a bog-standard object with no secondary indexes - the PartitionKey is as described above, the RowKey is PK@ID e.g. PK@1234. PK here indicating primary key. So when we want to get an object by ID we simply query RowKey=PK@1234.
   * For any indexed properties, we save a SECOND (third etc.) copy of the whole object. Remember Azure Tables are cheap - so this is not an issue, and is a recommended route by Microsoft. For each of these index copies, the RowKey is PropertyName@Value@ID e.g. Surname@Smith@1234. The value element of the key is encoded to be Azure Key safe, and truncated. When we load any object, we "save" into the list of indexed columns (see The Document Object above) the values of those properties. When saving we can compare the new values to see if we need to delete any index copies from the Table Storage, or add any new ones.
   * For querying - if we have an index on a property being queried, we can use a single partition and row range scan, which is quite efficient, such as Surname = Smith, would translate to Surname >=Surname@Smith and Surname <=Surname@Smith% (where percent is acsii character 255). Any subsequent parts of the query would just be conventional OData queries. And of course LINQ could be used once we have our list of objects for anything more clever!
* Query Builder
   * To save you effort, we have a basic fluent query builder which is strongly typed. Just make sure your indexed property is at the start (we only use one).
* Unsupported Types
   * We have added other types which are not normally support in Azure tables. They are translated back and forth during the serialisation into and back from JSON. Dates are also modified to allow MinValue.
   * Complicated properties, such as lists or other objects are simply stored as a JSON encoded string column in Azure Tables (they can't be indexed)
   * Long text is split across multiple columns e.g. Notes, Notes_01, Notes_02 automatically
   * Long binary data is dealt with the same as text above - but really we shouldn't be storing binary data here should we! 
