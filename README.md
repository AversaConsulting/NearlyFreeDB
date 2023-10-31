# NearlyFreeDB
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
test
* PartitionKey
test
