Purpose
=======
Invented by Jimmy Nilsson and first described in an article for InformIT in 2002, a COMB is a GUID with an embedded date/time value, making the values sequential over time.

This library is designed to create "COMB" `Guid` values in C#, and to be able to extract the `DateTime` value from an existing COMB value. I've found some random code here and there that purports to do something similar, but it either didn't support both variants of COMB (more on this later) or it was not independent enough to be grafted into my code.

Status
======
While I've created code to do this before, this library is my attempt to clean it up, add PostgreSQL support, and make it available as open source. Suggestions welcome.

Revision History
================
 - 1.1		2016-01		First release
 - 1.2		2016-01		Clean up, add unit tests
 - 1.3		2016-01-18	Major revision of interface
 - 1.4      2016-08-10  Upgrade to .NETCore 1.0, switch tests to Xunit
 - 2.0      2016-11-19	Corrected byte-order placement, reorganized for better DI, upgraded to .NETCore 1.1, downgraded to .NET 4.5.1
 - 2.1      2016-11-20  Simplified API and corrected/tested byte order for PostgreSql, more README rewrites, git commit issue

Background
==========
When GUIDs (`uniqueidentifier` values in MSSQL parlance, `UUID` in PostgreSQL) are part of a database index, the randomness of new values can result in reduced performance, as insertions can fragment the index (or, for a clustered index, the table). In SQL Server 2005, Microsoft provided the `NEWSEQUENTIALID()` function to alleviate this issue, but despite the name, generated GUID values from that function are still not guaranteed to be sequential over time, nor do multiple instances of MSSQL produce values that would be sequential in relationship to one another.

The COMB method, however, takes advantage of the database's native sort ordering for GUID values and replaces the bytes that are sorted first with a date/time value, so values generated at the same time will always be generated in a *nearly* sequential order, regardless of server reboots or values generated on different machines (to the degree, of course, that the system clocks are synchronized).

As a side benefit, the COMB's sequential portion has the semantic value of being the date and time of insert, which can be useful from time to time.

While this does not alleviate all potential worries when deciding to use GUID fields in a table (they are still rather large, and can be unfriendly if exposed to end users), it does make them more palatable when they are called for.

Using this Library
====================
A NuGet package is available, dual-targeted for .NET Core 1.1 and Microsoft.NET 4.5.1:
https://www.nuget.org/packages/RT.Comb/

Everything is within the `RT.Comb` namespace.

The `ICombProvider` interface defines a `Create()` function (with several signature variants) to return new COMB `Guid` values, and a `GetTimestamp()` function to return the embedded `DateTime` within an existing COMB `Guid` value.

Two implementations of `ICombProvider` are provided:

- `SqlCombProvider`: This creates and decodes COMBs in GUIDs that are compatible with the way Microsoft SQL Server sorts `uniqueidentifier` values -- i.e., starting at the 11th byte.

- `PostgreSqlCombProvider`: This creates and decodes COMBs in GUIDs that are compatible with the way PostegreSQL sorts `uuid` values -- i.e., starting with the first byte shown in string representations of a `Guid`.

Both of these implementations have a single constructor that takes a single argument of type `IDateTimeStrategy`, which handles the actual conversion between `System.DateTime` values and the bytes that are written into the GUIDs. Two `IDateTimeStrategy` implementations are included:

 - `SqlDateTimeStrategy` encodes the DateTime in a way that is highly compatible with Microsoft SQL Server's `datetime` data type and the original COMB implementation.
 - `UnixDateTimeStrategy` encodes the DateTime using the millisecond version of the Unix epoch timestamp.

You can use either DateTimeStrategy with either Provider--or write your own if neither of these fits your needs.

Prior to version 2.0, I used static classes with hard-coded options. With 2.0, I switched to a traditional class to provide more flexibility and to allow me to inject the preferred options as a Singleton using ASP.NET Core. The class is thread-safe.

UUIDs and GUIDs
===============
*(For a more comprehensive treatment, see Wikipedia.)*

Standard UUIDs/GUIDs are 128-bit (16-byte) values, wherein most of those bits hold a random value. When viewed as a string, the bytes are represented in hex in five groups, separated by hyphens:

    xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    version       ^
    variant            ^

Structurally, this breaks down into one 32-bit unsigned integer (`Data1`), two 16-bit unsigned integers (`Data2`, `Data3`), and 8 bytes (`Data4`). The most significant nybble of Data3 (the 7th byte in string order, 8th in the internal bytes of `System.Guid`) is the GUID "version" number--generally 4 (0100b). Also, the most significant 2-3 bits of the first byte of Data4 is the "variant" value. For common GUIDs, the first two of these bits will be `10`, and the third is random. The remaining bits for a version 4 GUID are random.

The UUID standard (RFC 4122) specifies that the `Data1`, `Data2`, and `Data3` integers are stored and shown in *network byte order* (a.k.a. "big endian" or most significant byte first). However, while Microsoft's GUID implementation *shows and sorts* the values as if they were in big endian form, internally it stores the bytes for these values in the native form for the processor (which, on x86, means little endian).

This means that the bytes you get from `Guid.ToString()` are actually stored in the GUID's internal byte array in the following order:

    33221100-5544-7766-8899-AABBCCDDEEFF

Npgsql reads incoming bytes for these three fields using `GetInt32()` and `GetInt16()`, which assume the bytes are in little endian form. This ensures that .NET will return the same *string* as non-.NET tools that show PostgreSQL UUID values, but the bytes inside .NET and inside PostgreSQL are stored in a different order. Reference:

https://github.com/npgsql/npgsql/blob/4ef74fa78cffbb4b1fdac00601d0ee5bff5e242b/src/Npgsql/TypeHandlers/UuidHandler.cs

IDateTimeStrategy
=================
`IDateTimeStrategy` implementations are responsible for returning a byte array, most significant byte first, representing the timestamp. This byte order is independent of whatever swapping we might have to do to embed the value in a GUID at a certain index. They can return either 4 or 6 bytes, both built-in implementations use 6 bytes.

SqlDateTimeStrategy
-------------------
This strategy returns bytes for a timestamp with *part* of an MSSQL `datetime` value. The `datetime` type in MSSQL is an 8-byte structure. The first four bytes are a *signed* integer representing the number of days since January 1, 1900. Negative values are permitted to go back to `1752-01-01` (for Reasons), and positive values are permitted through `9999-12-12`. The remaining four bytes represent the time as the number of unsigned 300ths of a second since midnight. Since a day is always 24 hours, this integer never uses more than 25 bits. 

For the COMB, we use all four bytes of the time and two bytes of the date. This means our date has no sign bit and has a maximum value of 65,535, limiting our date range to January 1, 1900 through June 5, 2079 -- a very reasonable spread.

If you use this in conjunction with `SqlCombProvider`, your COMB values will be compatible with the original COMB article, and you can easily use plain T-SQL to encode and decode the timestamps without requiring .NET (the overhead for doing this in T-SQL is minimal):

Creating a COMB `uniqueidentifier` in T-SQL with the current date and time:

    CAST(CAST(NEWID() AS binary(10)) + CAST(GETUTCDATE() AS binary(6)) AS uniqueidentifier)

Extracting a `datetime` value from a COMB `uniqueidentifier` created using the above T-SQL:

	CAST(SUBSTRING(CAST(0 AS binary(2)) + CAST(value AS binary(16), 10, 6) AS datetime)

UnixDateTimeStrategy
--------------------
This implementation was inspired by work done on the Marten project, which uses a modified version of RT.Comb. This also returns 6 bytes, but as a 48-bit unsigned integer representing the number of *milliseconds* since the UNIX epoch date (1970-01-01). Since this method is far more space-efficient than the MSSQL `datetime` structure, it will cover you well into the 85th Century. This is the recommended implementation for use with PostgreSQL, or with SQL Server if you aren't concerned with precise T-SQL based conversions to SQL `datetime` values.

ICombProvider
=============
Regardless which strategy is used for encoding the timestamp, we need to overwrite the portion of the GUID that is sorted *first*, so our GUIDs are sortable in date/time order and minimize index page splits. This differs by database platform.

MSSQL and `System.Data.SqlTypes.SqlGuid` sort *first* by the *last 6* `Data4` bytes, *left to right*, then the first two bytes of `Data4` (again, left to right), then `Data3`, `Data2`, and `Data1` *right to left*. This means for COMB purposes, we want to overwrite the the last 6 bytes of `Data4` (byte index 10), left to right.

However, PostgreSQL and `System.Guid` sort GUIDs in the order shown as a string, which means for PostgreSQL COMB values, we want to overwrite the bytes that are *shown first* from the left. Since `System.Guid` *shows* bytes for `Data1`, `Data2`, and `Data3` in a different order than it *stores* them internally, we have to account for this when overwriting those bytes. For example, our most significant byte inside a `Guid` will be at index 3, not index 0.

Here is a fiddler illustrating some of this:

https://dotnetfiddle.net/rW9vt7

SqlCombProvider
---------------
As mentioned above, MSSQL sorts the *last* 6 bytes first, left to right, so we plug our timestamp into the GUID structure starting at the 11th byte, most significant byte first:

    00112233-4455-6677-8899-AABBCCDDEEFF
    xxxxxxxx xxxx 4xxx Vxxx MMMMMMMMMMMM  UnixDateTimeStrategy, milliseconds
    xxxxxxxx xxxx 4xxx Vxxx DDDDTTTTTTTT  SqlDateTimeStrategy, days and 1/300s
	4 = version
    V = variant
	x = random

PostgreSqlCombProvider
----------------------
For PostgreSQL, the *first* bytes are sorted first, so those are the ones we want to overwrite.

    MMMMMMMM MMMM 4xxx Vxxx xxxxxxxxxxxx  UnixDateTimeStrategy, milliseconds
    DDDDTTTT TTTT 4xxx Vxxx xxxxxxxxxxxx  SqlDateTimeStrategy, days and 1/300s
	4 = version
	V = variant
	x = random

Recall that `System.Guid` stores its bytes for `Data1` and `Data2` in a different order than they are shown, so we have to reverse the bytes we're putting into those areas so they are stored and sorted in PostgreSQL in network byte order.

**This is a breaking change from RT.Comb version 1.4.** In prior versions, I wasn't actually able to test on PostgreSQL and got this all wrong, along with misplacing where the version nybble was and doing unnecessary bit-gymbastics to avoid overwriting it. My error was kindly pointed out by Barry Hagan and has been fixed.

Note about entropy
------------------
The default implementations overwrite 48 bits of random data with the timestamp, and another 6 bits are also pre-determined (the verion and variant). This still leaves 74 random bits per unit of time (1/300th of a second for SqlDateTimeStrategy, 1/1000th of a second for UnixDateTimeStrategy). This provides well beyond a reasonable amount of protection against collision.

How To Contribute
=================
Some missing pieces:
- More unit tests.
- Please keep all contributions compatible with .NET Core.
- Please use the same style (tabs, same-line opening braces, compact line spacing, etc.)
- Please keep project/solution files compatible with Visual Studio 2015 Community Edition and Visual Studio Code.

Security and Performance
========================
(1) It's a good idea to always use UTC date/time values for COMBs, so they remain highly sequential regardless of server locale or daylight savings, etc. This is even more important if the values will be generated on multiple machines that may have different time zone settings.

(2) It should go without saying, but using a COMB "leaks" the date/time to anyone knows how to decode it, so don't use one if this information (or even the relative order of their creation) should remain private.

(3) Don't use this with MySQL if you plan to store GUIDs as varchar(36) -- performance will be terrible.

(4) Test. Don't assume that a GUID key will be significantly slower for you than a 32-bit `int` field. Don't assume it will be roughly the same. The relative size of your tables (and especially of your indexed columns) compared to your primary key column will impact the overall database size and relative performance. I use COMB values frequently in moderate-sized databases without any performance issues, but YMMV.

(5) The best use for a COMB is where (a) you want the range and randomness of the GUID structure without index splits under load, and (b) any actual use of the embedded timestamp is rare (for example, for debugging purposes).

(6) When generating values, note that on Windows platforms, the Windows system timer only has a resolution of around 15ms. But even with a higher-resolution timer, multiple COMB values can have the same timestamp if you generate them quickly enough, so don't rely on the COMB alone if you need to sort your values in exactly the same order as they were created.

More Information
=================================
The original COMB article:
http://www.informit.com/articles/article.aspx?p=25862

Another implementation (not compatible):
http://www.siepman.nl/blog/post/2013/10/28/ID-Sequential-Guid-COMB-Vs-Int-Identity-using-Entity-Framework.aspx

License (MIT "Expat")
=====================
Copyright 2015-2016 Richard S. Tallent, II

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.