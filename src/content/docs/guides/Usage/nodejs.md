---
title: Node.js
description: Using Mapepire with Node.js
sidebar:
    order: 2
---

Full API docs can be found on the [client SDK project page](https://github.com/Mapepire-IBMi/mapepire-js), but the basics are summarized here.

`mapepire-js` is the pure TypeScript/JavaScript client for connecting to Db2 for IBM i. It supports multiple transport modes and provides both single-job and connection-pool APIs.

## Requirements

* Node.js 18 or later
* A running `mapepire-server` daemon on the target IBM i **or** SSH access to an IBM i (see [transport modes](#transport-modes) below)

## Installation

```sh
npm install @ibm/mapepire-js
```

## Sample Application

See our sample Node.js/TypeScript applications in the [mapepire-samples](https://github.com/Mapepire-IBMi/samples/tree/main/typescript) repository.

## Quick Start

The simplest way to connect and run a query:

```ts
import { SQLJob } from '@ibm/mapepire-js';

const job = new SQLJob();
await job.connect({
  host: process.env.DB2_HOST,
  user: process.env.DB2_USER,
  password: process.env.DB2_PASS,
  rejectUnauthorized: false, // set true and use getCertificate() for self-signed certs
});

const result = await job.execute<{ CURRENT_USER: string }>('values current user');
console.log(result.data);

await job.close();
```

## Transport Modes

`mapepire-js` supports three transport modes. The right choice depends on your deployment:

| Transport | How it connects | Requires |
|---|---|---|
| **WebSocket** (default) | Persistent WebSocket to Mapepire daemon | Daemon running on IBM i |
| **SSH Single** | Launches server in single mode over SSH | SSH access; no daemon needed |
| **Local Single** | Spawns server as child process (IBM i only) | Running on IBM i; no credentials |

### WebSocket Transport (default)

Connect to a running Mapepire daemon using a `DaemonServer` credentials object:

```ts
import { SQLJob, DaemonServer, DEFAULT_PORT } from '@ibm/mapepire-js';

const creds: DaemonServer = {
  host: process.env.DB2_HOST,
  port: DEFAULT_PORT, // 8076; can be omitted — DEFAULT_PORT is exported from the package
  user: process.env.DB2_USER,
  password: process.env.DB2_PASS,
  rejectUnauthorized: true, // set false only for development with self-signed certs
};

const job = new SQLJob();
await job.connect(creds);
```

### SSH Single Transport

No daemon required. `mapepire-js` connects via SSH and launches the server JAR in single-session mode. A private-install step uploads the bundled JAR automatically when needed.

:::note[Install an SSH library first]
SSH Single requires either `ssh2` or `node-ssh` — install whichever you prefer before using this transport:

```sh
npm install ssh2
# or
npm install node-ssh
```
:::

**Recommended — credential-first (SSH lifecycle managed internally):**

```ts
import { SQLJob } from '@ibm/mapepire-js';

// Using ssh2
const job = await SQLJob.ssh2({ host: 'ibmi.example.com', username: 'USER', password: 'PASS' });
await job.connect();
const result = await job.execute('SELECT * FROM QIWS.QCUSTCDT');
await job.close(); // SSH client torn down automatically

// Using node-ssh
const job2 = await SQLJob.nodeSSH({ host: 'ibmi.example.com', username: 'USER', password: 'PASS' });
await job2.connect();
const result2 = await job2.execute('SELECT * FROM QIWS.QCUSTCDT');
await job2.close(); // SSH instance disposed automatically
```

**Bring your own SSH client** (reuse an existing connection):

```ts
// ssh2
import { SQLJob, connectSSH2, createSSH2Connection } from '@ibm/mapepire-js';

const client = await connectSSH2({ host: 'ibmi.example.com', username: 'USER', password: 'PASS' });
const job = SQLJob.withConfig({ transport: 'ssh-single', sshSingle: createSSH2Connection(client) });
await job.connect();
const result = await job.execute('SELECT * FROM QIWS.QCUSTCDT');
await job.close();
client.end(); // caller manages SSH client lifetime

// node-ssh
import { SQLJob, connectNodeSSH, createNodeSSHConnection } from '@ibm/mapepire-js';

const ssh = await connectNodeSSH({ host: 'ibmi.example.com', username: 'USER', password: 'PASS' });
const job2 = SQLJob.withConfig({ transport: 'ssh-single', sshSingle: createNodeSSHConnection(ssh) });
await job2.connect();
const result2 = await job2.execute('SELECT * FROM QIWS.QCUSTCDT');
await job2.close();
ssh.dispose(); // caller manages SSH instance lifetime
```

:::tip[Private Install — how it works]
When you use `createSSH2Connection` (or `createNodeSSHConnection`), `mapepire-js` automatically manages the server JAR:

1. Searches `$HOME/.mapepire` **and** `$HOME/.vscode` for an existing JAR
2. If a version ≥ bundled is found → uses it as-is, no upload
3. If nothing usable is found → uploads the bundled JAR to `$HOME/.mapepire/`, verifies its SHA-256, then launches

You can control this behaviour with the `privateInstall` option in `SSHSingleConfig`:
- **`true`** — force private install (requires `upload`)
- **`false`** — disable entirely, use `serverPath` or the default RPM path
- **`undefined`** (default) — automatic: runs when `upload` is provided and `serverPath` is omitted
:::

### Local Single Transport (IBM i only)

For Node.js code running directly **on** IBM i. No daemon, no SSH, and no credentials are needed — the current IBM i job's user profile is used automatically.

**Auto-detection**: When running on IBM i (`process.platform === 'os400'`) with no credentials supplied, `mapepire-js` automatically selects `local-single` transport.

```ts
import { SQLJob } from '@ibm/mapepire-js';

// Explicit config — or omit transport entirely on IBM i when no user is provided
const job = SQLJob.withConfig({
  transport: 'local-single',
  localSingle: {
    serverPath: '/QOpenSys/pkgs/lib/mapepire/mapepire-server.jar', // optional, defaults to RPM path
  },
});

await job.connect(); // spawns JVM as child process — no credentials needed
const result = await job.execute('SELECT * FROM QIWS.QCUSTCDT');
await job.close();   // sends exit request and kills child JVM
```

:::tip[Zero-Install: Using the bundled JAR]
If the global `mapepire-server` RPM is not installed on your IBM i system, you can use the JAR bundled inside `@ibm/mapepire-js`:

```js
const { SQLJob, SERVER_VERSION_FILE } = require('@ibm/mapepire-js');
const path = require('path');

const distPath = path.dirname(require.resolve('@ibm/mapepire-js'));
const bundledServerPath = path.join(distPath, SERVER_VERSION_FILE);

const job = SQLJob.withConfig({
  transport: 'local-single',
  localSingle: { serverPath: bundledServerPath },
});
```
:::

## Running Queries

### `job.execute()` — one-shot query

`execute()` opens, runs, and closes the cursor in one call. Use this for most queries.

```ts
const result = await job.execute<{ OBJNAME: string; OBJTYPE: string }>(
  `SELECT OBJNAME, OBJTYPE FROM TABLE(QSYS2.OBJECT_STATISTICS('QGPL', '*ALL', '*ALLSIMPLE'))`
);
result.data.forEach(row => console.log(`${row.OBJNAME} (${row.OBJTYPE})`));
```

### `job.query()` — cursor-based query

Use `job.query()` when you need to page through large result sets or control cursor lifetime manually.

```ts
const query = job.query<{ NAME: string }>('SELECT * FROM SAMPLE.SYSCOLUMNS');
const result = await query.execute(100); // rows to fetch per call; defaults to 100 if omitted
while (!result.is_done) {
  const more = await query.fetchMore(100);
  console.table(more.data);
}
await query.close();
```

### Parameters

Pass parameter values to avoid building SQL strings by hand:

```ts
const result = await job.execute<any>(
  'SELECT * FROM SAMPLE.EMPLOYEE WHERE WORKDEPT = ?',
  { parameters: ['A00'] }
);
```

Batch parameters (for `INSERT` / `UPDATE` with multiple rows):

```ts
const query = job.query<any>(
  'UPDATE SAMPLE.EMPLOYEE SET PHONE = ? WHERE NAME = ?',
  {
    parameters: [
      ['789-678-6543', 'SANJULA'],
      ['222-456-1234', 'TONGKUN'],
      ['123-456-7891', 'JAMES'],
    ],
  }
);
await query.execute();
await query.close();
```

### `query.addToBatch()` — incremental batch building

Use `addToBatch()` to push parameter rows onto a query incrementally, instead of supplying the full 2D array upfront. Call `execute()` once all rows have been added.

```ts
const query = job.query<any>(
  'INSERT INTO SAMPLE.EMPLOYEE (NAME, PHONE) VALUES (?, ?)'
);

query.addToBatch([['SANJULA', '789-678-6543']]);
query.addToBatch([['TONGKUN', '222-456-1234']]);
query.addToBatch([['JAMES',   '123-456-7891']]);

await query.execute();
await query.close();
```

### `sql` Tagged Template

`Pool` exposes a `sql` tagged template literal for safe, inline-parameterized queries without needing to manage placeholder arrays:

```ts
const dept = 'A00';
const result = await pool.sql`SELECT * FROM SAMPLE.EMPLOYEE WHERE WORKDEPT = ${dept}`;
```

### CL Commands

CL commands can be executed through an `SQLJob`:

```ts
const query = job.clcommand('CRTLIB LIB(MYLIB1) TEXT(\'My library\')');
const result = await query.execute();
console.log(result);
```

### CL Command Documentation

Retrieve the HTML and UIM documentation for any CL command:

```ts
const doc = await job.getClDoc('/QSYS.LIB/CRTLIB.CMD');
console.log(doc.html);
console.log(doc.uim);
```

### BLOB Columns

Read BLOB columns by fetching the `BlobRef` first, then calling `fetchBlob`:

```ts
import { BlobRef } from '@ibm/mapepire-js';

const result = await job.execute<{ DATA: BlobRef }>('SELECT DATA FROM MYLIB.BLOBFILE WHERE ID = 1');
const blobRef = result.data[0].DATA;
const buffer: Buffer = await job.fetchBlob(blobRef);
```

### `job.explain()` — Visual Explain

Run Visual Explain on a SQL statement. By default the statement is also executed (`ExplainType.RUN`); pass `ExplainType.DO_NOT_RUN` to analyse the query plan without running it.

`ExplainType` is exported under the `States` namespace:

```ts
import { SQLJob, States } from '@ibm/mapepire-js';

// Explain and run
const result = await job.explain('SELECT * FROM SAMPLE.EMPLOYEE WHERE WORKDEPT = ?');
console.log(result.vemetadata); // explain metadata
console.log(result.vedata);     // explain data

// Explain only — do not execute
const dryRun = await job.explain(
  'SELECT * FROM SAMPLE.EMPLOYEE',
  States.ExplainType.DO_NOT_RUN
);
```

## Pooling

`Pool` is **WebSocket-only** — its `PoolOptions.creds` field takes a `DaemonServer` and manages a set of persistent WebSocket connections to the Mapepire daemon. SSH Single and Local Single transports are single-session by design and do not support pooling.

For production workloads, initialize a pool during your app's startup process.

```ts
import { Pool } from '@ibm/mapepire-js';

const pool = new Pool({ creds, maxSize: 5, startingSize: 3 });
await pool.init();
```

### `pool.execute()` — execute on any free job

Automatically finds the least-busy job and runs the query:

```ts
const result = await pool.execute('VALUES CURRENT_USER');
console.log(result.data);
```

### `pool.query()` — cursor from pool

```ts
const query = pool.query('SELECT * FROM SAMPLE.SYSCOLUMNS');
const result = await query.execute(50);
```

### Closing the pool

```ts
pool.end();
```

## `UrlToDaemon()` — Connection URI Helper

`UrlToDaemon` converts a `db2i://` connection URI string into a `DaemonServer` object. Useful when connection details arrive as a single URI (e.g. from an environment variable or config file). The password in the URI must be **base64-encoded** — pass a plain-text password through `Buffer.from('yourpassword').toString('base64')` before embedding it in the URI.

```ts
import { UrlToDaemon, SQLJob } from '@ibm/mapepire-js';

// Password must be base64-encoded
const b64pass = Buffer.from('mypassword').toString('base64');
const creds = UrlToDaemon(`db2i://USER:${b64pass}@myibmi.example.com:8076`);

const job = new SQLJob();
await job.connect(creds);
```

## JDBC Options

When creating an `SQLJob` or `Pool`, you can pass [JDBC options](https://www.ibm.com/docs/en/i/7.4?topic=jdbc-toolbox-java-properties) to control naming convention, library list, and more:

```ts
import { SQLJob, JDBCOptions } from '@ibm/mapepire-js';

const jdbcOptions: JDBCOptions = {
  naming: 'system',
  libraries: ['MYLIB1', 'MYLIB2'],
  'transaction isolation': 'none',
};

const job = new SQLJob(jdbcOptions);
await job.connect(creds);

// With a pool:
const pool = new Pool({ creds, opts: jdbcOptions, maxSize: 5, startingSize: 3 });
```

## Application Name

Pass a custom application name when connecting for connection tracking on the server:

```ts
await job.connect(creds, 'my-node-app');
```

## Job Status and Query State

Check the current status of an `SQLJob`:

```ts
const status = job.getStatus();
// 'notStarted' | 'connecting' | 'ready' | 'busy' | 'ended'
```

Check the state of a `Query`:

```ts
const query = job.query('SELECT * FROM SAMPLE.EMPLOYEE');
const state = query.getState();
// 'NOT_YET_RUN' | 'RUN_MORE_DATA_AVAILABLE' | 'RUN_DONE' | 'ERROR'
```

## Exception Handling

Query failures throw a standard `Error` whose message includes the SQL error, `sql_state`, and `sql_rc` from the server. Wrap calls in `try/catch`:

```ts
try {
  const result = await job.execute('SELECT * FROM NONEXISTENT.TABLE');
} catch (err) {
  console.error('Query failed:', err.message);
}
```

## Server-Side Tracing

Enable server-side tracing to capture protocol-level diagnostics. Three methods work together on a connected `SQLJob`:

```ts
import { SQLJob } from '@ibm/mapepire-js';

// Enable tracing — dest: 'FILE' | 'IN_MEM', level: 'OFF' | 'ON' | 'ERRORS' | 'DATASTREAM'
await job.setTraceConfig('IN_MEM', 'DATASTREAM');

// ... run queries ...

// Read trace data back (only meaningful for IN_MEM dest)
const trace = await job.getTraceData();
console.log(trace.tracedata);

// If dest was 'FILE', get the remote IFS path
const filePath = job.getTraceFilePath(); // e.g. '/tmp/mapepire-trace.log', or undefined for IN_MEM

// Turn tracing off
await job.setTraceConfig('IN_MEM', 'OFF');
```

`setTraceConfig('FILE', level)` writes trace output to an IFS file on the IBM i system. `getTraceFilePath()` returns that path after `setTraceConfig` completes, or `undefined` when in-memory tracing is active.

## Secure Connections

By default, `mapepire-js` always connects over TLS and validates the server certificate.

### Allow All Certificates

Set `rejectUnauthorized: false` on the `DaemonServer` object to skip certificate validation. Only use this for local development.

```ts
const creds: DaemonServer = {
  host: 'myibmi.example.com',
  user: 'USER',
  password: 'PASS',
  rejectUnauthorized: false,
};
```

:::danger
Never use `rejectUnauthorized: false` in production. With it set, credentials are exposed to anyone who can intercept traffic on the network.
:::

### Validate a Self-Signed Certificate

Use `getRootCertificate()` to fetch and pin the server's root CA certificate before connecting. This is the recommended approach when your server uses a self-signed certificate.

```ts
import { getRootCertificate, DaemonServer, Pool } from '@ibm/mapepire-js';

async function getDbPool() {
  const creds: DaemonServer = {
    host: process.env.DB2_HOST,
    user: process.env.DB2_USER,
    password: process.env.DB2_PASS,
  };

  // Fetch and pin the self-signed root certificate
  const ca = await getRootCertificate(creds);
  if (ca) {
    creds.ca = ca; // only set when the cert is not from a public CA
  }

  return new Pool({ creds, maxSize: 5, startingSize: 3 });
}
```

`getRootCertificate()` returns `undefined` when the certificate chain resolves to a publicly trusted CA, so the `if (ca)` guard ensures you never override the default trust store unnecessarily.

If you need the full peer certificate object instead, use `getCertificate()`:

```ts
import { getCertificate } from '@ibm/mapepire-js';

const cert = await getCertificate(creds);
// validate cert.raw, cert.subject, etc.
creds.ca = cert.raw;
```

### Validate a Certificate Signed by a Recognized CA

No extra configuration is needed. Leave `rejectUnauthorized` unset (or `true`) and the TLS handshake will validate the server certificate against the system trust store automatically.
