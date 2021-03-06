# ManagedEsent Overview

ManagedEsent provides managed interop access to the ESENT database engine. In this overview, you will get a brief introduction to ESENT and ManagedEsent, including sample code and links to API documentation.

## ESENT

ESENT is an embeddable, transactional database engine. It first shipped with Microsoft Windows 2000 and has been available for developers to use since then. You can use ESENT for applications that need reliable, high-performance, low-overhead storage of structured or semi-structured data. The ESENT engine can help with data needs ranging from something as simple as a hash table that is too large to store in memory to something more complex such as an application with tables, columns, and indexes. 

Many teams at Microsoft, including The Active Directory, Windows Desktop Search, Windows Mail, Live Mesh, and Windows Update, currently rely on ESENT for data storage. And Microsoft Exchange stores all of its mailbox data (a large server typically has dozens of terrabytes of data) using a slightly modified version of the ESENT code.

#### Features
Significant technical features of ESENT include:
* ACID transactions with savepoints, lazy commits, and robust crash recovery.
* Snapshot isolation.
* Record-level locking (multi-versioning provides non-blocking reads).
* Highly concurrent database access.
* Flexible meta-data (tens of thousands of columns, tables, and indexes are possible).
* Indexing support for integer, floating point, ASCII, Unicode, and binary columns.
* Sophisticated index types, including conditional, tuple, and multi-valued.
* Columns that can be up to 2GB with a maximum database size of 16TB.

#### Benefits
*_No additional download needed._ ManagedEsent uses the native esent.dll that already comes as part of every version of Microsoft Windows.

*_No administration required._ ESENT automatically manages logfiles, database recovery, and even the database cache size.

Note: The ESENT database file cannot be shared between multiple processes simultaneously. ESENT works best for applications with simple, predefined queries; if you have an application with complex, ad-hoc queries, a storage solution that provides a query layer will work better for you. 

## ManagedEsent
ManagedEsent provides interop access to the ESENT API so that managed code can use ESENT for data storage.  I developed the ManagedEsent code with these principles:
* By default, a program written with ManagedEsent will work with the esent.dll from Windows XP onwards. ESENT features that are not present in Windows XP (the baseline OS) but were added in later versions should be available in a separate namespace.
* ManagedEsent should only require version 2.0 of the .NET Framework.
* The API should be fully documented and Intellisense should be able to provide useful help.
* There should be minimal editorialization of the ESENT API. Whenever possible, ManagedEsent should exactly match the ESENT API - in particular, the names of structs, types, and functions should not be changed. API changes that will be made:
	* Group the constants from esent.h into useful enumerations to eliminate common API usage errors.
	* Remove cbStruct from structures.
	* Remove unused/reserved entries from structures.
	* Throw exceptions instead of returning errors.
* Helper methods should be provided for common operations.
* The overhead of ManagedEsent, compared to using ESENT natively, should be as small as possible.

## [API Reference](ManagedEsentApis.md)
This _API Reference_ section offers links to the MSDN documentation for individual ESENT APIs.

## Sample Code
Here is a very [simple sample application](ManagedEsentSample.md) that creates a database with one table, inserts a record, and then retrieves the data from the record.
For other versions, see these links: ([C++/CLI version](CppManagedEsentSample.md)) ([VB.NET version](VbNetManagedEsentSample.md)) ([IronPython version](IpyManagedEsentSample.md)) ([F# version](FSharpManagedEsentSample.md))

Here is a more [advanced sample application](StockSample.md) that creates a database; inserts, updates, and deletes records; and performs several different types of queries.

## [How Do I...?](HowDoI.md)
This _How Do I_ section offers sample code for common ESENT programming tasks.

## Tutorials
[Using multi-valued columns](MultiValueTutorial.md)

## Performance
Performance is highly dependent on database schema, how much of the database is cached, and API patterns. To get an accurate sense of your application's performance, you will need to measure your application with realistic workloads and specific scenarios. 

Below are some performance measurements for a simple application that inserts 1,000,000 records. Each record has an 8 byte autoincrement key and 32 bytes of data. These test results are from a fast development machine with an SSD disk. The speed of the disk affects the insert/update results, but the database is fully cached so the read/scan/seek results are not affected. Lazy transactions are used everywhere, and modifications are done in batches of 100 modifications/transaction.

| Operation      | Performance                               |
| -------------- |:-----------------------------------------:|
| Insert records | 132,000 records/second                    |
| Update one record | 157,000 updates/second                 |
| Read one record | 1,149,000 retrieves/second               |
| Scan all records (sequential) | 794,000 records/second     |
| Seek to all records (random order) | 266,000 seeks/second  |


## API Versioning
Different versions of ESENT support different APIs, grbits, and structures. One of my goals in developing the ManagedEsent code was to make these differences as transparent as possible. 
1. The static EsentVersion class provides properties (for example, EsentVersion.SupportsVistaFeatures) that describe the capabilities of the current version of ESENT.
2. Options and APIs that are not available on Windows XP are provided in separate namespaces (for example, Microsoft.Isam.Esent.Interop.Windows7). This makes usage of the more advanced features explicit.  APIs in these namespaces throw an InvalidOperationException if they are not supported.
3. If Unicode support is available, APIs that take file paths use the Unicode APIs (for example, JetAttachDatabaseW). If Unicode support is not available, APIs use the ASCII APIs.
4. Structures that were expanded in later versions are silently downgraded when using older versions of ESENT (for example, the cbKeyMost member of JET_INDEXCREATE is ignored on Windows XP and Windows Server 2003).
5. Helper methods and objects do not throw exceptions if possible (for example, the SystemParameter class returns default values for unsupported parameters).

```c#
using Microsoft.Isam.Esent.Interop.Windows7;
Api.JetInit2(ref instance, Windows7Grbits.ReplayIgnoreLostLogs); // Uses new Windows 7 option

using Microsoft.Isam.Esent.Interop.Vista;
VistaApi.JetOpenTemporaryTable(session, opentemporarytable); // Uses Windows Vista API
```

## Helper Classes
Helper methods and classes do not start with â€˜Jetâ€™ to set them apart from the interop methods they wrap.

#### IDisposable Wrapper Classes
These classes provide an IDisposable wrapper around ESENT resources that should always be cleaned up. The objects are:
* **Instance** wraps a JET_INSTANCE along with JetInit and JetTerm. This class inherits from SafeHandle to make sure that ESENT instances are always terminated.
* **Session** wraps a JET_SESID.
* **Table** wraps a JET_TABLEID.
* **Transaction** wraps JetBeginTransaction and JetCommitTransaction/JetRollback. The transaction is automatically rolled back if disposed without being committed.
* **Update** wraps JetPrepareUpdate and JetUpdate. The update is automatically canceled if disposed without being finished.

The Instance, Session, and Table classes all provide implicit conversions to the underlying ESENT type so they can be used anywhere the wrapped type is required.

```c#
using (var instance = new Instance("stocksample"))
{
    instance.Parameters.CircularLog = true;
    instance.Init();
    using (var session = new Session(instance))
    {
        // Attach the database ...
        using (var table = new Table(session, dbid, TableName, OpenTableGrbit.None))
        {
            using (var transaction = new Transaction(session))
            {
                // Do something...
                transaction.Commit(CommitTransactionGrbit.None);
            }
        }
    }
}
```
			
#### ColumnStream
This class wraps a tableid/columnid pair and exposes a stream interface for reading and writing data.

```c#
using (var stream = new ColumnStream(sesid, tableid, columnid))
{
    stream.Seek(-100, SeekOrigin.End);
    stream.Read(data, 0, data.Length));
}
```

#### EsentStopwatch
This class combines JetGetThreadStats with System.Diagnostics.Stopwatch so you can measure ESENT activities.

```c#
EsentStopwatch stopwatch = EsentStopwatch.StartNew();
// do something
stopwatch.Stop();
Console.WriteLine("Elapsed time: {0}", stopwatch.Elapsed);
Console.WriteLine("Pages read: {0}", stopwatch.ThreadStats.cPageRead);
```

#### InstanceParameters/SystemParameters
These classes provide properties to allow setting/getting system parameters.

```c#
SystemParameters.DatabasePageSize = 8192;

var parameters = new InstanceParameters(instance);
parameters.LogFileDirectory = @"c:\logs";
```

#### SetColumn/MakeKey/EscrowUpdate/RetrieveColumn/RetrieveColumnAs
These convenience methods are strongly typed instead of taking a byte[](). The SetColumn and MakeKey overloads can be more efficient as they will pin data in-place instead of copying it. RetrieveColumnAsString optimizes memory allocation and conversion overhead. RetrieveColumn, RetrieveKey, and RetrieveBookmark all allocate memory so that the caller does not have to. 

```c#
Api.MakeKey(sesid, tableid, 56.87, MakeKeyGrbit.NewKey);
// ...
Api.SetColumn(sesid, tableid, textcolumn, "hello", Encoding.Unicode);
Api.SetColumn(sesid, tableid, int32column, 42);
// ...
int? value = Api.RetrieveColumnAsInt32(sesid, tableid, columnid);
// ...
Api.EscrowUpdate(sesid, tableid, columnid, -1);
```

#### TrySeek/TryMove/TrySetIndexRange
These navigation methods return false when a record is not found instead of throwing an exception.

```c#
Api.MoveBeforeFirst(sesid, tableid);
while (Api.TryMoveNext(sesid, tableid))
{
    // Do something...
}
```

#### IntersectIndexes
This convenience method takes JET_TABLEIDs and intersects them, returning an enumeration of bookmarks.

#### Meta-data Enumeration Helpers
These methods wrap the Jet functions that expose the meta-data of a database/table, returning an enumeration of meta-data objects. Methods found here include GetColumnDictionary, GetTableIndexes, GetTableNames.

```c#
foreach (string table in Api.GetTableNames(sesid, dbid))
{
	Console.WriteLine(table);
}

IDictionary<string, JET_COLUMNID> columnids = Api.GetColumnDictionary(sesid, tableid);
Api.JetSetColumn(sesid, tableid, columnids["age"](_age_), 39);
```

#### ColumnValue
These are strongly typed classes that combine the members of the {"JET_RETRIEVECOLUMN and JET_SETCOLUMN"} structures along with a data member. You can use these classes along with SetColumns and RetrieveColumns to set or retrieve columns efficiently.

```c#
var keyColumn = new Int64ColumnValue { Columnid = columnids["key"](_key_) };
var dataColumn = new StringColumnValue { Columnid = columnids["data"](_data_) };

keyColumn.Value = 1;
dataColumn.Value = "something";

Api.SetColumns(sesid, tableid, keyColumn, dataColumn);
```

#### TryOpenTable
This method tries to open a table. This does not throw an exception if the table does not exist.

```c#
JET_TABLEID tableid;
if (Api.TryOpenTable(this.sesid, this.dbid, this.table, OpenTableGrbit.None, out tableid))
{
    // do something with the table...
}
else
{
    // create the table...
}
```
