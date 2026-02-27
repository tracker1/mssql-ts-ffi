# FILESTREAM

SQL Server FILESTREAM allows storing large binary data (BLOBs) directly on the file system while maintaining transactional consistency.

::: warning Windows Only
FILESTREAM is only available on Windows with the [Microsoft OLE DB Driver 19 for SQL Server](https://learn.microsoft.com/en-us/sql/connect/oledb/download-oledb-driver-for-sql-server) installed.
:::

## Check Availability

Checks all three requirements: local OLE DB driver installed, server FILESTREAM
access level >= 2, and a FILESTREAM filegroup exists in the target database.

```ts
await using cn = await mssql.connect("Server=localhost;...");

// Check current database
if (await cn.filestreamAvailable()) {
  console.log("FILESTREAM is ready");
}

// Check a specific database
if (await cn.filestreamAvailable("MyFilestreamDB")) {
  console.log("FILESTREAM is ready on MyFilestreamDB");
}
```

Also works on pools:

```ts
const pool = await mssql.createPool("Server=localhost;...");
if (await pool.filestreamAvailable()) {
  // ...
}
```

## Two FILESTREAM APIs

`@tracker1/mssql` provides two ways to work with FILESTREAM blobs:

| Method | Returns | Best for |
|---|---|---|
| `cn.openFilestream()` | `node:stream` Readable / Writable / Duplex | `pipe()`, Node.js patterns, `node:fs` interop |
| `cn.openWebstream()` | Web Standard ReadableStream / WritableStream | `pipeTo()`, Deno patterns, Web API interop |

Both require an active transaction and the FILESTREAM path + transaction context from a query.

## Getting the Path and Context

```ts
await using cn = await mssql.connect("Server=localhost;...");
await using tx = await cn.beginTransaction();

const row = await cn.querySingle<{ path: string; ctx: Uint8Array }>(
  `SELECT file_data.PathName() AS path,
          GET_FILESTREAM_TRANSACTION_CONTEXT() AS ctx
     FROM Documents WHERE id = @id`,
  { id: docId },
  { transaction: tx }
);
```

## Node.js Streams (`openFilestream`)

Returns a `node:stream` Readable, Writable, or Duplex depending on the mode.
Works across Deno, Node.js, and Bun — all runtimes support `node:stream`.

### Read a blob

```ts
const readable = cn.openFilestream(row.path, row.ctx, "read");

const chunks: Uint8Array[] = [];
for await (const chunk of readable) {
  chunks.push(chunk);
}
```

### Write a blob

```ts
const writable = cn.openFilestream(row.path, row.ctx, "write");

writable.write(new Uint8Array([0x48, 0x65, 0x6c, 0x6c, 0x6f]));
writable.end();
```

### Pipe from a local file into FILESTREAM

```ts
import { createReadStream } from "node:fs";
import { pipeline } from "node:stream/promises";

const source = createReadStream("./upload.bin");
const writable = cn.openFilestream(row.path, row.ctx, "write");

await pipeline(source, writable);
await tx.commit();
```

### Pipe from FILESTREAM to a local file

```ts
import { createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";

const readable = cn.openFilestream(row.path, row.ctx, "read");
const dest = createWriteStream("./download.bin");

await pipeline(readable, dest);
await tx.commit();
```

## Web Streams (`openWebstream`)

Returns a Web Standard `ReadableStream<Uint8Array>` or `WritableStream<Uint8Array>`.
Native in all runtimes — ideal for Deno's file APIs and the Fetch/Streams spec.

### Read a blob

```ts
const readable = cn.openWebstream(row.path, row.ctx, "read");

const reader = readable.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  // value is Uint8Array
}
```

### Write a blob

```ts
const writable = cn.openWebstream(row.path, row.ctx, "write");

const writer = writable.getWriter();
await writer.write(new Uint8Array([0x48, 0x65, 0x6c, 0x6c, 0x6f]));
await writer.close();
```

### Pipe from a local file into FILESTREAM (Deno)

```ts
const writable = cn.openWebstream(row.path, row.ctx, "write");

using file = await Deno.open("./upload.bin", { read: true });
await file.readable.pipeTo(writable);

await tx.commit();
```

### Pipe from FILESTREAM to a local file (Deno)

```ts
const readable = cn.openWebstream(row.path, row.ctx, "read");

using file = await Deno.open("./download.bin", { write: true, create: true });
await readable.pipeTo(file.writable);

await tx.commit();
```

### Read/write mode

For `"readwrite"` mode, `openWebstream` returns an object with both streams:

```ts
const { readable, writable } = cn.openWebstream(row.path, row.ctx, "readwrite");
```

## Close and Commit

Always commit the transaction after FILESTREAM operations:

```ts
await tx.commit();
```

Streams are automatically closed when destroyed or when the underlying handle
reaches end-of-file. You can also close explicitly by calling `.destroy()` on
Node.js streams or cancelling the Web stream.
