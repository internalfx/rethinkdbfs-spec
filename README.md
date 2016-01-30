# RethinkDBFS Spec

This is the official spec for the [RethinkDBFS](https://github.com/internalfx/rethinkdbfs) Nodejs library. Copied and adapted from the original [GridFS spec](https://github.com/mongodb/specifications) for MongoDB.

The purpose of this document is to adapt GridFS specifically for RethinkDB and allow compatibile drivers to be written in other languages.

- Authors [**Bryan Morris**](https://github.com/internalfx), [**Brian Chavez**](https://github.com/bchavez)
- Advisors [**Daniel Mewes**](https://github.com/danielmewes)

# Abstract

RethinkDBFS is a convention drivers use to store and retrieve JSON binary data that exceeds RethinkDB's JSON-document size limit of 64MB. When this data, called a **user file**, is written to the system, RethinkDBFS divides the file into **chunks** that are stored as distinct documents in a **chunks table**. To retrieve a stored file, RethinkDBFS locates and returns all of its component chunks. Internally, RethinkDBFS creates a **files table document** for each stored file. Files table documents hold information about stored files, and they are stored in a **files table**.

# Definitions

## META

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Terms

#### Bucket name
A prefix under which a RethinkDBFS system’s tables are stored. Table names for the files and chunks tables are prefixed with the bucket name. The bucket name MUST be configurable by the user. Multiple buckets may exist within a single database. The default bucket name is `fs`.

#### Chunk
A section of a user file, stored as a single document in the `chunks` table of a RethinkDBFS bucket. The default size for the data field in chunks is 255KB. Chunk documents have the following form:

```json
{
  "id": "<String>",
  "files_id": "<String>",
  "n": "<Number>",
  "data": "<Binary>"
}
```

| Key | Description |
|---|---|
| id | a unique ID for this document. |
| files_id | the id for this file (the id from the files table document). |
| n | the index number of this chunk, zero-based |
| data | a chunk of data from the user file |

#### Chunks table
A table in which chunks of a user file are stored. The name for this table is the word 'chunks' prefixed by the bucket name. The default is `fs_chunks`.

#### Empty chunk
A chunk with a zero length `data` field.

#### Files table
A table in which information about stored files is stored. There will be one files table document per stored file. The name for this table is the word `files` prefixed by the bucket name. The default is `fs_files`.

#### Files table document
A document stored in the files table that contains information about a single stored file. Files table documents have the following form:

```json
{
  "id" : "<String>",
  "length" : "<Number>",
  "chunkSize" : "<Number>",
  "completeDate" : "<Time>",
  "incompleteDate" : "<Time>",
  "deletedDate" : "<Time>",
  "sha256" : "<String>",
  "filename" : "<String>",
  "status" : "<String>",
  "metadata" : "<Object>"
}
```

| Key | Description |
|---|---|
| id | a unique ID for this document. |
| length | the length of this stored file, in bytes. |
| chunkSizeBytes | the size, in bytes, of each data chunk of this file. This value is configurable by file. The default is 255KB (1024 * 255). |
| completeDate | the date and time this files status was set to `Complete`. The value of this field MUST be the datetime when the upload completed, not the datetime when it was begun. |
| incompleteDate | the date and time this files status was set to `Incomplete`. The value of this field MUST be the datetime when the upload started, not the datetime when it was finished. |
| deletedDate | the date and time this files status was set to `Deleted`. The value of this field MUST be the datetime when file was marked `Deleted`. |
| sha256 | SHA256 checksum for this user file, computed from the file’s data, stored as a hex string (lowercase). |
| filename | the name of this stored file; this does not need to be unique. |
| status | Status may be "Complete" or "Incomplete" or "Deleted". |
| metadata | any additional application data the user wishes to store. |

#### Orphaned chunk
A document in the chunks tables for which the `files_id` does not match any `id` in the files table. Orphaned chunks may be created if write or delete operations on RethinkDBFS fail part-way through.

#### Stored File
A user file that has been stored in RethinkDBFS, consisting of a files table document in the files table and zero or more documents in the chunks table.

#### Stream
An abstraction that represents streamed I/O. In some languages a different word is used to represent this abstraction.

#### User File
A data added by a user to RethinkDBFS. This data may map to an actual file on disk, a stream of input, a large data object, or any other large amount of consecutive data.

# Specification

## Documentation

The documentation provided in code below is merely for driver authors and SHOULD NOT be taken as required documentation for the driver.

## Operations

All drivers MUST offer the Basic API operations defined in the following sections and MAY offer the Advanced API operations. This does not preclude a driver from offering more.

## Operation Parameters

All drivers MUST offer the same options for each operation as defined in the following sections. This does not preclude a driver from offering more. The options parameter is optional. A driver SHOULD NOT require a user to specify optional parameters.

## Deviations

A non-exhaustive list of acceptable deviations are as follows:

- Using named parameters instead of an options hash. For instance,

```javascript
var stream = bucket.createWriteStream(filename, meta, {chunkSizeBytes: 16 * 1024})
```

#### Naming

All drivers MUST name operations, objects, and parameters as defined in the following sections.

Deviations are permitted as outlined below.

#### Deviations

When deviating from a defined name, an author should consider if the altered name is recognizable and discoverable to the user of another driver.

A non-exhaustive list of acceptable naming deviations are as follows:

- Using `bucketName` as an example, Java would use `bucketName` while Python would use `bucket_name`. However, calling it `bucketPrefix` would not be acceptable.

- Using `maxTimeMS` as an example, .NET would use `MaxTime` where its type is a `TimeSpan` structure that includes units. However, calling it `MaximumTime` would not be acceptable.

- Using `RethinkDBFSUploadOptions` as an example, Javascript wouldn't need to name it while other drivers might prefer to call it `RethinkDBFSUploadArgs` or `RethinkDBFSUploadParams`. However, calling it `UploadOptions` would not be acceptable.

- Languages that use a different word than `Stream` to represent a streamed I/O abstraction may replace the word `Stream` with their language's equivalent word. For example, `open_upload_stream` might be called `open_upload_file` or `open_upload_writer` if appropriate.

- Languages that support overloading MAY shorten the name of some methods as appropriate. For example, `download_to_stream` and `download_to_stream_by_name` MAY be overloaded `download_to_stream` methods with different parameter types. Implementers are encouraged not to shorten method names unnecessarily, because even if the shorter names are not ambiguous today they might become ambiguous in the future as new features are added.

# API

This section presents two groups of features, a basic API that a driver MUST implement, and a more advanced API that drivers MAY choose to implement additionally.

## Basic API

Configurable RethinkDBFSBucket class

```javascript
var RethinkDBFSBucketOptions = {
  bucketName: String optional; // The bucket name. Defaults to 'fs'.
  chunkSizeBytes: Number optional; // The chunk size in bytes. Defaults to 255KB (1024 * 255).
}

var bucket = new RethinkDBFSBucket(dbConnectionOptions, RethinkDBFSBucketOptions)
```

Creates a new RethinkDBFSBucket object, managing a RethinkDBFS bucket within the given database.

RethinkDBFSBucket objects MUST allow the following options to be configurable:

- **bucketName:** the name of this RethinkDBFS bucket. The files and chunks table for this RethinkDBFS bucket are prefixed by this name followed by an underscore. Defaults to `fs`. This allows multiple RethinkDBFS buckets, each with a unique name, to exist within the same database.

- **chunkSizeBytes:** the number of bytes stored in chunks for new user files added through this RethinkDBFSBucket object. This will not reformat existing files in the system that use a different chunk size. Defaults to 255KB (1024 * 255).

RethinkDBFSBucket instances are immutable. Their properties MUST NOT be changed after the instance has been created. If your driver provides a fluent way to provide new values for properties, these fluent methods MUST return new instances of RethinkDBFSBucket.

## Indexes

For efficient execution of various RethinkDBFS operations the following indexes MUST exist:

*Index spec is tentative and will likely change*

An index on the `files` table:

```javascript
r.table('<bucketName>_files').createIndex('status_filename_createdat', [r.row('status'), r.row('filename'), r.row('createdAt')])
```

An index on the `chunks` table:

```javascript
r.table('<bucketName>_chunks').createIndex('filesid_n', [r.row('files_id'), r.row('n')])
```

RethinkDB does not automatically create tables, nor does it automatically use indexes. RethinkDBFS requires tables and indexes to be explicitly created before first use.

#### Initialization

Drivers MUST provide an `initBucket` method, which creates all necessary tables and indexes.

Drivers MUST check whether the tables and indexes already exist before attempting to create them.

If a driver determines that it should create the tables and indexes, it MUST raise an error if the attempt to create the them fails.

Drivers MUST wait for the indexes to finish building before writing.

#### Before read/write operations

Drivers MUST assume that the proper tables and indexes exist.

# File Upload

```javascript

var uploadOptions = {

  /**
   * The number of bytes per chunk of this file. Defaults to the
   * chunkSizeBytes in the RethinkDBFSBucketOptions.
   */
  chunkSizeBytes: Number optional;

  /**
   * User data for the 'metadata' field of the files table document.
   * If not provided the driver MUST omit the metadata field from the
   * files table document.
   */
  metadata : Object optional;

}

class RethinkDBFSBucket {

  /**
   * Opens a Stream that the application can write the contents of the file to.
   *
   * Returns a Stream to which the application will write the contents.
   */
  Stream createWriteStream(string filename, uploadOptions=null);

}
```

Uploads a user file to a RethinkDBFS bucket. For languages that have a Stream abstraction, drivers SHOULD use that Stream abstraction. For languages that do not have a Stream abstraction, drivers MUST create an abstraction that supports streaming.

In the case of `open_upload_stream`, the driver returns a Stream to which the application will write the contents of the file. As the application writes the contents to the returned Stream, the contents are uploaded as chunks in the chunks table. When the application signals it is done writing the contents of the file by calling close (or its equivalent) on the returned Stream, a files table document is created in the files table. Once the Stream has been closed (and the files table document has been created) a driver MUST NOT allow further writes to the upload Stream.

The driver MUST make the Id of the new file available to the caller. Typically a driver SHOULD make the Id available as a property named Id on the Stream that is returned. In languages where that is not idiomatic, a driver MUST make the Id available in a way that is appropriate for that language.

In the case of `upload_from_stream`, the driver reads the contents of the user file by consuming the the source Stream until end of file is reached. The driver does NOT close the source Stream.

Drivers MUST take an `options` document with configurable parameters. Drivers for dynamic languages MUST ignore any unrecognized fields in the options for this method (this does not apply to drivers for static languages which define an Options class that by definition only contains valid fields).

Note that in RethinkDBFS, `filename` is not a unique identifier. There may be many stored files with the same filename stored in a RethinkDBFS bucket under different ids. Multiple stored files with the same filename are called `revisions`, and the `createdAt` is used to distinguish newer revisions from older ones.

#### Implementation details:

If `chunkSizeBytes` is set through the options, that value MUST be used as the chunk size for this stored file. If this parameter is not specified, the default chunkSizeBytes setting for this RethinkDBFSBucket object MUST be used instead.

To store a user file, drivers first generate an ObjectId to act as its id. Then, drivers store the contents of the user file in the chunks table by breaking up the contents into chunks of size `chunkSizeBytes`. For a non-empty user file, for each Nth section of the file, drivers create a chunk document and set its fields as follows:

|Key|Description|
|---|---|
|files_id|the id generated for this stored file.|
|n|this is the Nth section of the stored file, zero based.|
|data|a section of file data, stored as JSON binary data with subtype. All chunks except the last one must be exactly `chunkSizeBytes` long. The last chunk can be smaller, and should only be as large as necessary.|

While streaming the user file, drivers compute an SHA256 digest. This SHA256 digest will later be stored in the files table document.

After storing all chunk documents generated for the user file in the `chunks` table, drivers create a files table document for the file and store it in the files table. The fields in the files table document are set as follows:

| Key | Description |
|---|---|
| id | a unique ID for this document. |
| length | the length of this stored file, in bytes. |
| chunkSizeBytes | the size, in bytes, of each data chunk of this file. This value is configurable by file. The default is 255KB (1024 * 255). |
| completeDate | the date and time this files status was set to `Complete`. The value of this field MUST be the datetime when the upload completed, not the datetime when it was begun. |
| incompleteDate | the date and time this files status was set to `Incomplete`. The value of this field MUST be the datetime when the upload started, not the datetime when it was finished. |
| deletedDate | the date and time this files status was set to `Deleted`. The value of this field MUST be the datetime when file was marked `Deleted`. |
| sha256 | SHA256 checksum for this user file, computed from the file’s data, stored as a hex string (lowercase). |
| filename | the name of this stored file; this does not need to be unique. |
| status | Status may be "Complete" or "Incomplete" or "Deleted". |
| metadata | any additional application data the user wishes to store. |

If a user file contains no data, drivers MUST still create a files table document for it with length set to zero. Drivers MUST NOT create any empty chunks for this file.

Note that drivers are no longer required to run the 'filemd5' to confirm that all chunks were successfully uploaded. We assume that if none of the inserts failed then the chunks must have been successfully inserted, and running the 'filemd5' command would just be unnecessary overhead.

#### Operation Failure

If any of the above operations fail against the server, drivers MUST raise an error. If some inserts succeeded before the failed operation, these become orphaned chunks. Drivers MUST NOT attempt to clean up these orphaned chunks. The rationale is that whatever failure caused the orphan chunks will most likely also prevent cleaning up the orphaned chunks, and any attempts to clean up the orphaned chunks will simply cause long delays before reporting the original failure to the application.

#### Aborting an upload

Drivers SHOULD provide a mechanism to abort an upload. When using `open_upload_stream`, the returned Stream SHOULD have an Abort method. When using `upload_from_stream`, the upload will be aborted if the source stream raises an error.

When an upload is aborted any chunks already uploaded MUST be deleted. Note that this differs from the case where an attempt to insert a chunk fails, in which case drivers immediately report the failure without attempting to delete any chunks already uploaded.

Abort MUST raise an error if it is unable to succesfully abort the upload (for example, if an error occurs while deleting any chunks already uploaded). However, if the upload is being aborted because the source stream provided to upload_from_stream raised an error then the original error should be re-raised.

Abort MUST also close the Stream, or at least place it in an aborted state, so any further attempts to write additional content to the Stream after Abort has been called fail immediately.

# File Download

```javascript
class RethinkDBFSBucket {

  /** Opens a Stream from which the application can read the contents of the stored file
   * specified by @id.
   *
   * Returns a Stream.
   */
  Stream open_download_stream(ObjectId id);

  /**
   * Downloads the contents of the stored file specified by @id and writes
   * the contents to the @destination Stream.
   */
  void download_to_stream(ObjectId id, Stream destination);

}
```

Downloads a stored file from a RethinkDBFS bucket. For languages that have a Stream abstraction, drivers SHOULD use that Stream abstraction. For languages that do not have a Stream abstraction, drivers MUST create an abstraction that supports streaming.

In the case of `open_download_stream`, the application reads the contents of the stored file by reading from the returned Stream until end of file is reached. The application MUST call close (or its equivalent) on the returned Stream when it is done reading the contents.

In the case of `download_to_stream` the driver writes the contents of the stored file to the provided Stream. The driver does NOT call close (or its equivalent) on the Stream.

Note: if a file in a RethinkDBFS bucket was added by a legacy implementation, its id may be of a type other than ObjectId. Drivers that previously used id’s of a different type MAY implement a download() method that accepts that type, but MUST mark that method as deprecated.

#### Implementation details:

Drivers must first retrieve the files table document for this file. If there is no files table document, the file either never existed, is in the process of being deleted, or has been corrupted, and the driver MUST raise an error.

The recommended query for retrieving the files table document is as follows:

```javascript
r.table('fs_files').between(['Completed', '<filename>', r.minval], ['Completed', '<filename>', r.maxval], {index: 'status_filename_createdat'}).orderBy({index: r.desc('status_filename_createdat')})
```

Then, implementers retrieve all chunks with files_id equal to id, sorted in ascending order on “n”.

However, when downloading a zero length stored file the driver MUST NOT issue a query against the chunks table, since that query is not necessary. For a zero length file, drivers return either an empty stream or send nothing to the provided stream (depending on the download method).

If a networking error or server error occurs, drivers MUST raise an error.

As drivers stream the stored file they MUST check that each chunk received is the next expected chunk (i.e. it has the expected "n" value) and that the data field is of the expected length. In the case of open_download_stream, if the application stops reading from the stream before reaching the end of the stored file, any errors that might exist beyond the point at which the application stopped reading won't be detected by the driver.

# File Deletion

```javascript
class RethinkDBFSBucket {

  /**
   * Given a @id, delete this stored file’s files table document and
   * associated chunks from a RethinkDBFS bucket.
   */
  void delete(id);

}
```

Deletes the stored file’s files table document and associated chunks from the underlying database.

As noted for download(), drivers that previously used id’s of a different type MAY implement a delete() method that accepts that type, but MUST mark that method as deprecated.

#### Implementation details:

There is an inherent race condition between the chunks and files tables. Without some transaction-like behavior between these two tables, it is always possible for one client to delete a stored file while another client is attempting a read of the stored file. For example, imagine client A retrieves a stored file’s files table document, client B deletes the stored file, then client A attempts to read the stored file’s chunks. Client A wouldn’t find any chunks for the given stored file. To minimize the window of vulnerability of reading a stored file that is the process of being deleted, drivers MUST first delete the files table document for a stored file, then delete its associated chunks.

If there is no such file listed in the files table, drivers MUST raise an error. Drivers MAY attempt to delete any orphaned chunks with files_id equal to id before raising the error.

If a networking or server error occurs, drivers MUST raise an error.

# Generic Find on Files Table

```javascript
class RethinkDBFSFindOptions {

  /**
   * The number of documents to return per batch.
   */
  batchSize : Number optional;

  /**
   * The maximum number of documents to return.
   */
  limit : Number optional;

  /**
   * The number of documents to skip before returning.
   */
  skip : Number optional;

  /**
   * The order by which to sort results. Defaults to not sorting.
   */
  sort : Document optional;

}

class RethinkDBFSBucket {

  /**
   * Find and return the files table documents that match @filter.
   */
  Iterable find(Document filter, RethinkDBFSFindOptions=null);

}
```

This call will trigger a find() operation on the files table using the given filter. Drivers returns a sequence of documents that can be iterated over. Drivers return an empty or null set when there are no matching files table documents. As the number of files could be large, drivers SHOULD return a cursor-like iterable type and SHOULD NOT return a fixed-size array type.

### Implementation details:

Drivers SHOULD NOT perform any validation on the filter. If the filter contains fields that do not exist within files table documents, then an empty result set will be returned.

Drivers MUST document how users query files table documents, including how to query metadata, e.g. using a filter like { metadata.fieldname : “some_criteria” }.

# Advanced API

## File Download by Filename

```javascript
class RethinkDBFSDownloadByNameOptions {

  /**
   * Which revision (documents with the same filename and different createdAt)
   * of the file to retrieve. Defaults to -1 (the most recent revision).
   *
   * Revision numbers are defined as follows:
   * 0 = the original stored file
   * 1 = the first revision
   * 2 = the second revision
   * etc…
   * -2 = the second most recent revision
   * -1 = the most recent revision
   */
  revision : Number optional;

}

class RethinkDBFSBucket {

  /** Opens a Stream from which the application can read the contents of the stored file
   * specified by @filename and the revision in @options.
   *
   * Returns a Stream.
   */
  Stream createReadStream(string filename, RethinkDBFSDownloadByNameOptions=null);

}
```

Retrieves a stored file from a RethinkDBFS bucket. For languages that have a Stream abstraction, drivers SHOULD use that Stream abstraction. For languages that do not have a Stream abstraction, drivers MUST create an abstraction that supports streaming.

#### Implementation details:

If there is no file with the given filename, or if the requested revision does not exist, drivers MUST raise an error with a distinct message for each case.

Drivers MUST select the files table document of the file to-be-returned by running a query on the files table for the given filename, sorted by createdAt (either ascending or descending, depending on the revision requested) and skipping the appropriate number of documents. For negative revision numbers, the sort is descending and the number of documents to skip equals (-revision - 1). For non-negative revision numbers, the sort is ascending and the number of documents to skip equals the revision number.

If a networking error or server error occurs, drivers MUST raise an error.

## Partial File Retrieval

In the case of open_download_stream, drivers SHOULD support partial file retrieval by allowing the application to read only part of the stream. If a driver does support reading only part of the stream, it MUST do so using the standard stream methods of its language for seeking to a position in a stream and reading the desired amount of data from that position. This is the preferred method of supporting partial file retrieval.

In the case of download_to_stream, drivers are not required to support partial file retrieval. If they choose to do so, drivers can support this operation by adding ‘start’ and ‘end’ to their supported options for download_to_stream. These values represent non-negative byte offsets from the beginning of the file. When ‘start’ and ‘end’ are specified, drivers return the bytes of the file in [start, end). If ‘start’ and ‘end’ are equal no data is returned.

If either ‘start’ or ‘end’ is invalid, drivers MUST raise an error. These values are considered invalid if they are negative, greater than the file length, or if ‘start’ is greater than ‘end’.

When performing partial reads, drivers SHOULD use the file’s ‘chunkSize’ to calculate which chunks contain the desired section and avoid reading unneeded documents from the ‘chunks’ table.

## Renaming stored files

```javascript
class RethinkDBFSBucket {

  /**
   * Renames the stored file with the specified @id.
   */
  void rename(id, string new_filename);

}
```

Sets the filename field in the stored file’s files table document to the new filename.

#### Implementation details:

Drivers construct and execute an update_one command on the files table using `{ _id: @id }` as the filter and `{ $set : { filename : "new_filename" } }` as the update parameter.

To rename multiple revisions of the same filename, users must retrieve the full list of files table documents for a given filename and execute `rename` on each corresponding `_id`.

If there is no file with the given id, drivers MUST raise an error.

## Dropping an entire RethinkDBFS bucket

```javascript
class RethinkDBFSBucket {

  /**
   * Drops the files and chunks tables associated with
   * this bucket.
   */
  void drop();

}
```

This method drops the files and chunks tables associated with this RethinkDBFS bucket.

Drivers should drop the files table first, and then the chunks table.

# Motivation for Change

RethinkDBFS documentation is only concerned with the underlying data model for this feature, and does not specify what basic set of features an implementation of RethinkDBFS should or should not provide. As a result, RethinkDBFS is currently implemented across drivers, but with varying APIs, features, and behavior guarantees. Current implementations also may not conform to the existing documentation.

This spec documents minimal operations required by all drivers offering RethinkDBFS support, along with optional features that drivers may choose to support. This spec is also explicit about what features/behaviors of RethinkDBFS are not specified and should not be supported. Additionally, this spec validates and clarifies the existing data model, deprecating fields that are undesirable or incorrect.

# Design Rationale

#### Why is the default chunk size 255KB?
On MMAPv1, the server provides documents with extra padding to allow for in-place updates. When the ‘data’ field of a chunk is limited to 255KB, it ensures that the whole chunk document (the chunk data along with an id and other information) will fit into a 256KB section of memory, making the best use of the provided padding. Users setting custom chunk sizes are advised not to use round power-of-two values, as the whole chunk document is likely to exceed that space and demand extra padding from the system. WiredTiger handles its memory differently, and this optimization does not apply. However, because application code generally won’t know what storage engine will be used in the database, always avoiding round power-of-two chunk sizes is recommended.

#### Why can’t I alter documents once they are in the system?
RethinkDBFS works with documents stored in multiple tables within RethinkDB. Because there is currently no way to atomically perform operations across tables in RethinkDB, there is no way to alter stored files in a way that prevents race conditions between RethinkDBFS clients. Updating RethinkDBFS stored files without that server functionality would involve a data model that could support this type of concurrency, and changing the RethinkDBFS data model is outside of the scope of this spec.

#### Why provide a ‘rename’ method?
By providing users with a reasonable alternative for renaming a file, we can discourage users from writing directly to the files tables under RethinkDBFS. With this approach we can prevent critical files table documents fields from being mistakenly altered.

#### Why is there no way to perform arbitrary updates on the files table?
The rename helper defined in this spec allows users to easily rename a stored file. While updating files table documents in other, more granular ways might be helpful for some users, validating such updates to ensure that other files table document fields remain protected is a complicated task. We leave the decision of how best to provide this functionality to a future spec.

#### What is the ‘md5’ field of a files table document and how is it used?
‘md5’ holds an MD5 checksum that is computed from the original contents of a user file. Historically, RethinkDBFS did not use acknowledged writes, so this checksum was necessary to ensure that writes went through properly. With acknowledged writes, the MD5 checksum is still useful to ensure that files in RethinkDBFS have not been corrupted. A third party directly accessing the 'files' and ‘chunks’ tables under RethinkDBFS could, inadvertently or maliciously, make changes to documents that would make them unusable by RethinkDBFS. Comparing the MD5 in the files table document to a re-computed MD5 allows detecting such errors and corruption. However, drivers now assume that the stored file is not corrupted, and applications that want to use the MD5 value to check for corruption must do so themselves.

#### Why store the MD5 checksum instead of creating the hash as-needed?
The MD5 checksum must be computed when a file is initially uploaded to RethinkDBFS, as this is the only time we are guaranteed to have the entire uncorrupted file. Computing it on-the-fly as a file is read from RethinkDBFS would ensure that our reads were successful, but guarantees nothing about the state of the file in the system. A successful check against the stored MD5 checksum guarantees that the stored file matches the original and no corruption has occurred.

#### Why is contentType deprecated?
Most fields in the files table document are directly used by the driver, with the exception of: metadata, contentType and aliases. All information that is purely for use of the application should be embedded in the 'metadata' document. Users of RethinkDBFS who would like to store a contentType for use in their applications are encouraged to add a 'contentType' field to the ‘metadata’ document instead of using the deprecated top-level ‘contentType’ field.

#### Why are aliases deprecated?
The ‘aliases’ field of the files table documents was misleading. It implies that a file in RethinkDBFS could be accessed by alternate names when, in fact, none of the existing implementations offer this functionality. For RethinkDBFS implementations that retrieve stored files by filename or support specifying specific revisions of a stored file, it is unclear how ‘aliases’ should be interpreted. Users of RethinkDBFS who would like to store alternate filenames for use in their applications are encouraged to add an ‘aliases’ field to the ‘metadata’ document instead of using the deprecated top-level ‘aliases’ field.

#### What happened to the put and get methods from earlier drafts?
Upload and download are more idiomatic names that more clearly indicate their purpose. Get and put are often associated with getting and setting properties of a class, and using them instead of download and upload was confusing.

#### Why aren't there methods to upload and download byte arrays?
We assume that RethinkDBFS files are usually quite large and therefore that the RethinkDBFS API must support streaming. Most languages have easy ways to wrap a stream around a byte array. Drivers are free to add helper methods that directly support uploading and downloading RethinkDBFS files as byte arrays.

#### Should drivers report an error if a stored file has extra chunks?
The length and the chunkSize fields of the files table document together imply exactly how many chunks a stored file should have. If the chunks table has any extra chunks the stored file is in an inconsistent state. Ideally we would like to report that as an error, but this is an extremely unlikely state and we don't want to pay a performance penalty checking for an error that is almost never there. Therefore, drivers MAY ignore extra chunks.

# Future work

The ability to alter or append to existing RethinkDBFS files has been cited as something that would greatly improve the system. While this functionality is not in-scope for this spec (see ‘Why can’t I alter documents once they are in the system?’) it is a potential area of growth for the future.
