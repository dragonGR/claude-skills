# Untrusted input reaching dangerous sinks

Read this when user-controlled data reaches a query, shell, template, the filesystem, an archive extractor, an outbound HTTP request, a redirect, a parser or a CORS decision. The rule throughout: parse into a strict type first, then use an API that cannot be confused, and check the value that is actually used, not an earlier copy of it.

## SQL: identifiers and raw APIs

Bound parameters cover values only. Identifiers, sort direction and raw fragments need an allowlist.

```ts
// Before
const rows = await sql.unsafe(`SELECT * FROM orders WHERE org_id = $1 ORDER BY ${req.query.sort} ${req.query.dir}`, [orgId]);

// After
const SORT_COLUMNS = { created: 'created_at', total: 'total_cents', status: 'status' } as const;
const ListOrdersQuery = z.object({
  sort: z.enum(['created', 'total', 'status']).default('created'),
  dir: z.enum(['asc', 'desc']).default('desc'),
});

const { sort, dir } = ListOrdersQuery.parse(req.query);
const rows = await sql`
  SELECT id, status, total_cents, created_at FROM orders
  WHERE org_id = ${orgId}
  ORDER BY ${sql(SORT_COLUMNS[sort])} ${dir === 'asc' ? sql`ASC` : sql`DESC`}, id`;
```

Raw-query escape hatches to grep for: `sql.unsafe`, `$queryRawUnsafe`, `$executeRawUnsafe`, `Sequelize.literal`, `knex.raw` with template strings, `cursor.execute(f"...")`, `text(f"...")` in SQLAlchemy, `.extra()` and `RawSQL` in Django. `LIKE` patterns also need `%` and `_` escaped when the user supplies a substring, or the search becomes a table scan the user controls.

## NoSQL operator injection

```ts
// Before: {"email": "a@b.c", "password": {"$ne": null}} matches any password.
const user = await db.collection('users').findOne({ email: req.body.email, password: req.body.password });

// After: types enforced, and the password is never a query predicate.
const { email, password } = z.object({ email: z.string().email(), password: z.string().max(PASSWORD_MAX) }).parse(req.body);
const user = await db.collection('users').findOne({ email });
const ok = await argon2.verify(user?.passwordHash ?? config.auth.dummyHash, password);
```

Also reject keys starting with `$` or containing `.` in any user object stored as a document, and never pass user input to `$where`, `$function` or `mapReduce`.

## Command execution

```ts
// Before: shell injection via the URL, and option injection such as `--upload-pack=...`.
exec(`git clone --depth 1 ${repoUrl} ${workDir}`);

// After
const RepoUrl = z
  .string()
  .url()
  .refine((value) => new URL(value).protocol === 'https:', 'https only');

execFile('git', ['clone', '--depth', '1', '--', RepoUrl.parse(repoUrl), workDir], {
  timeout: config.imports.cloneTimeoutMs,
});
```

`workDir` is a server-generated path. The scheme check also blocks git's `ext::` and `file://` transports. `--` stops a value starting with `-` being read as an option for tools that honour it; for tools that do not (check the tool's own docs), validate that the value cannot start with `-`. In Python: `subprocess.run([...], shell=False, timeout=...)`, never `shell=True` with user data, and `shlex.quote` is not a substitute for argv.

## Paths

```ts
import path from 'node:path';

export function resolveInside(baseDir: string, untrusted: string): string {
  const base = path.resolve(baseDir);
  const target = path.resolve(base, untrusted);
  if (target !== base && !target.startsWith(base + path.sep)) throw new ForbiddenPathError(untrusted);
  return target;
}
```

```python
from pathlib import Path

def resolve_inside(base_dir: Path, untrusted: str) -> Path:
    base = base_dir.resolve()
    target = (base / untrusted).resolve()
    if not target.is_relative_to(base):
        raise ForbiddenPathError(untrusted)
    return target
```

Prefix checks without the separator accept `/data/uploads-evil` for base `/data/uploads`. If users can create symlinks inside the base (archive extraction, git checkouts, shared volumes), resolve with `fs.realpath` / `Path.resolve()` after the file exists and open with `O_NOFOLLOW` where the platform supports it. On Windows also reject drive letters, UNC prefixes, alternate data streams (`:`) and reserved device names. The simplest safe design stores content under a generated id and never uses the user's filename as a path at all.

## Archives

For each entry before writing anything:

1. Reject absolute names, names containing `..` segments after normalisation, and names whose resolved target fails `resolveInside(dest, name)`.
2. Reject symlink and hardlink entries unless the feature needs them; if it does, resolve the link target with the same check.
3. Keep running totals of entry count and uncompressed bytes while streaming, and abort when either passes a configured cap. The sizes in the archive header are attacker-supplied; count bytes actually written.
4. Extract into a fresh temporary directory owned by the job, then move the validated result.

Python: `zipfile` strips `..` and absolute prefixes but has no size limit; `tarfile.extractall(path, filter='data')` refuses links outside the destination, absolute paths and device files. Pass the filter explicitly, since it only became the default in 3.14.

## SSRF

Validation has to happen on the address the socket connects to. Checking a hostname, then letting the HTTP client resolve it again, loses to DNS rebinding; following redirects loses to a 302 to an internal address.

```ts
import dns from 'node:dns';
import http from 'node:http';
import https from 'node:https';
import net from 'node:net';

const BLOCKED_V4: Array<[string, number]> = [
  ['0.0.0.0', 8], ['10.0.0.0', 8], ['100.64.0.0', 10], ['127.0.0.0', 8], ['169.254.0.0', 16],
  ['172.16.0.0', 12], ['192.0.0.0', 24], ['192.168.0.0', 16], ['198.18.0.0', 15],
  ['224.0.0.0', 4], ['240.0.0.0', 4],
];
// IPv4-mapped, NAT64 and 6to4 ranges embed IPv4 addresses, so they are blocked outright.
const BLOCKED_V6: Array<[string, number]> = [
  ['::', 128], ['::1', 128], ['::ffff:0:0', 96], ['64:ff9b::', 96], ['64:ff9b:1::', 48], ['2002::', 16],
  ['fc00::', 7], ['fe80::', 10], ['ff00::', 8],
];

const blocked = new net.BlockList();
for (const [addr, prefix] of BLOCKED_V4) blocked.addSubnet(addr, prefix, 'ipv4');
for (const [addr, prefix] of BLOCKED_V6) blocked.addSubnet(addr, prefix, 'ipv6');

export function isPublicAddress(address: string): boolean {
  const family = net.isIP(address);
  if (family === 0) return false;
  return !blocked.check(address, family === 6 ? 'ipv6' : 'ipv4');
}

const guardedLookup: net.LookupFunction = (hostname, options, callback) => {
  dns.lookup(hostname, options, (err, address, family) => {
    if (err) return callback(err, address, family);
    const results = Array.isArray(address) ? address : [{ address, family }];
    if (results.length === 0 || results.some((r) => !isPublicAddress(r.address))) {
      return callback(new BlockedDestinationError(hostname), address, family);
    }
    callback(null, address, family);
  });
};

export function fetchPublicUrl(rawUrl: string, limits: { timeoutMs: number; maxBytes: number }): Promise<Buffer> {
  const url = new URL(rawUrl);
  if (url.protocol !== 'https:' && url.protocol !== 'http:') throw new BlockedDestinationError(url.protocol);
  if (url.username || url.password) throw new BlockedDestinationError('credentials in URL');
  // Node skips the lookup function for IP literals, so they are checked here.
  const literal = url.hostname.replace(/^\[(.*)\]$/, '$1');
  if (net.isIP(literal) && !isPublicAddress(literal)) throw new BlockedDestinationError(literal);

  const client = url.protocol === 'https:' ? https : http;
  return new Promise((resolve, reject) => {
    const req = client.get(url, { lookup: guardedLookup, signal: AbortSignal.timeout(limits.timeoutMs) }, (res) => {
      if (res.statusCode !== 200) {
        res.resume();
        return reject(new UpstreamStatusError(res.statusCode));
      }
      const chunks: Buffer[] = [];
      let size = 0;
      res.on('data', (chunk: Buffer) => {
        size += chunk.length;
        if (size > limits.maxBytes) return req.destroy(new ResponseTooLargeError(limits.maxBytes));
        chunks.push(chunk);
      });
      res.on('end', () => resolve(Buffer.concat(chunks)));
      res.on('error', reject);
    });
    req.on('error', reject);
  });
}
```

Notes on the template:

- `http.get` does not follow redirects. If the feature needs them, follow manually up to a configured count, running each `Location` back through `fetchPublicUrl`.
- The WHATWG `URL` parser normalises `http://2130706433/` and `http://0x7f.1/` to `127.0.0.1`, so the literal check sees the real address.
- With `autoSelectFamily` on (the Node default in current releases), the lookup is called with `all: true` and returns an array; the wrapper handles both shapes.
- Using `fetch` (undici) with an `Agent` whose connect options carry the same lookup is possible, but verify against the installed undici version before relying on it.
- Adjust the blocklist to your network: if services run on public IPs of your own, block those too. Keep network-level egress rules as well; the in-process check is one layer.

Python: resolve with `socket.getaddrinfo`, classify each address, and make the HTTP client connect only to the checked address. The classifier:

```python
import ipaddress

def is_public(address: str) -> bool:
    ip = ipaddress.ip_address(address)
    if ip.version == 6 and ip.ipv4_mapped is not None:
        ip = ip.ipv4_mapped
    return ip.is_global and not ip.is_multicast
```

`requests` and `httpx` resolve the hostname again when connecting, so a pre-check with `socket.gethostbyname` followed by `httpx.get(url)` is rebinding-prone, and `gethostbyname` ignores IPv6 entirely. For arbitrary user URLs in Python, route through an egress proxy that enforces the policy on the connected address (Smokescreen is one), and set `follow_redirects=False`.

Cloud metadata: on AWS require IMDSv2 (`HttpTokens=required`) with hop limit 1 on nodes that run containers that do not need instance credentials; GCP and Azure require a metadata header (`Metadata-Flavor: Google`, `Metadata: true`), which stops naive SSRF but not one where the attacker controls headers.

## Redirects

```ts
export function safeReturnPath(raw: unknown, fallback: string): string {
  if (typeof raw !== 'string' || !raw.startsWith('/')) return fallback;
  const origin = new URL(config.web.publicOrigin).origin;
  // Inputs such as `//` fail to parse; they must fall back, not throw.
  if (!URL.canParse(raw, origin)) return fallback;
  const resolved = new URL(raw, origin);
  if (resolved.origin !== origin) return fallback;
  return resolved.pathname + resolved.search + resolved.hash;
}
```

Resolving against your origin and comparing origins handles `//evil.example`, `/\evil.example` and tab or newline tricks, because the URL parser applies the same normalisation the browser will. For cross-domain returns (a partner site), store allowed destinations server-side and pass an id.

## Deserialization and parsers

| Format | Unsafe | Use instead |
| --- | --- | --- |
| Python pickle, `shelve`, `joblib`, `torch.load(..., weights_only=False)` | any load of attacker data | JSON with a schema; for model files, formats without code execution (safetensors) |
| YAML (PyYAML) | `yaml.load(data, Loader=yaml.Loader)` or `UnsafeLoader` | `yaml.safe_load` |
| Java | `ObjectInputStream.readObject` on network or file input | JSON binding with explicit types; if unavoidable, a strict `ObjectInputFilter` allowlist |
| .NET | `BinaryFormatter`, `NetDataContractSerializer`, JSON.NET `TypeNameHandling` other than `None` | `System.Text.Json` with concrete types |
| PHP | `unserialize` on user data | `json_decode` |
| XML | DTDs and external entities enabled | disable DTD processing; `defusedxml` in Python |

Signed or encrypted blobs (cookies, cache entries) reduce exposure only while the key stays secret; many deserialization breaches start with a leaked signing key.

## Prototype pollution

```ts
const FORBIDDEN_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function assertSafeKeys(value: unknown): void {
  if (value === null || typeof value !== 'object') return;
  for (const key of Object.keys(value)) {
    if (FORBIDDEN_KEYS.has(key)) throw new ValidationError(`forbidden key ${key}`);
    assertSafeKeys((value as Record<string, unknown>)[key]);
  }
}
```

Better still, parse with a schema that lists known keys, so there is nothing to merge dynamically. `JSON.parse` creates an own property named `__proto__`, which becomes dangerous only when a deep merge or path setter copies it onto a normal object.

## CORS

```ts
const allowedOrigins = new Set(config.web.corsOrigins);

app.use((req, res, next) => {
  res.vary('Origin');
  const origin = req.get('Origin');
  if (origin && allowedOrigins.has(origin)) {
    res.set('Access-Control-Allow-Origin', origin);
    res.set('Access-Control-Allow-Credentials', 'true');
  }
  if (req.method === 'OPTIONS') {
    res.set('Access-Control-Allow-Methods', CORS_METHODS);
    res.set('Access-Control-Allow-Headers', CORS_HEADERS);
    return res.status(204).end();
  }
  next();
});
```

`Access-Control-Allow-Origin: *` on a public, unauthenticated endpoint is fine; browsers refuse to combine it with credentials. The dangerous version is reflecting the origin with credentials, or matching origins with a suffix test or regex.

## Uploads

- Decide the type from content and your allowlist, not the client's `Content-Type` or extension.
- Serve from a separate registrable domain (not a subdomain of the app, which is same-site) or with `Content-Disposition: attachment`.
- Always send `X-Content-Type-Options: nosniff`.
- Re-encode images server-side when practical; this strips embedded scripts and metadata such as GPS location.
- Cap size at the proxy and in the app; scan or sandbox parsers for formats with a history of parser bugs (images, PDFs, office documents).
