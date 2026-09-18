proxy-games/
├── package.json
├── server.js
├── tunnel.sh                 (optional – Cloudflare tunnel helper)
└── public/
    ├── index.html
    ├── home.css
    ├── home.js
    ├── app.html
    ├── app.css
    ├── app.js
    ├── style.css
    ├── browse.html           (older simpler browser UI – optional)
    └── games/
        ├── snake.html
        ├── pong.html
        ├── tetris.html
        ├── 2048.html
        ├── breakout.html
        └── memory.html
{
  "name": "proxy-games",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node server.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "commonjs",
  "dependencies": {
    "compression": "^1.8.2",
    "express": "^5.2.1",
    "lru-cache": "^11.5.2",
    "undici": "^7.29.1",
    "ws": "^8.21.3"
  }
}
const express = require('express');
const path = require('path');
const zlib = require('zlib');
const compression = require('compression');
const { URL } = require('url');
const { Agent, request } = require('undici');
const { LRUCache } = require('lru-cache');

const app = express();
const PORT = process.env.PORT || 3000;

// ===========================================================================
// PlayHub v3 — advanced web proxy + games portal + AI assistant
// ===========================================================================

app.use(compression());
app.set('trust proxy', true);
app.use(express.static(path.join(__dirname, 'public'), {
  maxAge: '1h',
  setHeaders: (res, filePath) => {
    if (/\.html?$/i.test(filePath)) res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
  },
}));
app.use(express.urlencoded({ extended: true, limit: '25mb' }));
app.use(express.json({ limit: '25mb' }));

const PROXY_PREFIX = '/proxy?url=';

// --- Structured logging -----------------------------------------------------
const LOG_LEVEL = process.env.LOG_LEVEL || 'info';
function log(level, msg, meta = {}) {
  const levels = { debug: 0, info: 1, warn: 2, error: 3 };
  if (levels[level] < levels[LOG_LEVEL]) return;
  console.log(JSON.stringify({ t: new Date().toISOString(), level, msg, ...meta }));
}

// --- Metrics ----------------------------------------------------------------
const metrics = {
  startedAt: Date.now(),
  requests: 0,
  cacheHits: 0,
  cacheMisses: 0,
  coalesced: 0,
  errors: 0,
  bytesOut: 0,
  byHost: new Map(),
};
function bumpHost(host, ms) {
  const e = metrics.byHost.get(host) || { count: 0, totalMs: 0 };
  e.count++; e.totalMs += ms;
  metrics.byHost.set(host, e);
}

// --- Connection pool --------------------------------------------------------
const agent = new Agent({
  connections: 256,
  pipelining: 10,
  keepAliveTimeout: 30000,
  keepAliveMaxTimeout: 60000,
  connect: { timeout: 15000 },
  headersTimeout: 30000,
  bodyTimeout: 60000,
});

// --- Cache with stale-while-revalidate --------------------------------------
const CACHE_TTL = 1000 * 60 * 5;
const CACHE_STALE = 1000 * 60 * 30;
const cache = new LRUCache({
  maxSize: 512 * 1024 * 1024,
  sizeCalculation: (v) => (v.body ? v.body.length : 0) + 512,
  ttl: CACHE_STALE,
});
const inflight = new Map();

// --- Rate limiting (token bucket per session) -------------------------------
const RATE_CAPACITY = 120;
const RATE_REFILL = 20;
const buckets = new Map();
function rateLimit(sid) {
  const now = Date.now();
  let b = buckets.get(sid);
  if (!b) { b = { tokens: RATE_CAPACITY, last: now }; buckets.set(sid, b); }
  b.tokens = Math.min(RATE_CAPACITY, b.tokens + ((now - b.last) / 1000) * RATE_REFILL);
  b.last = now;
  if (b.tokens < 1) return false;
  b.tokens -= 1;
  return true;
}

// --- Circuit breaker per host -----------------------------------------------
const breakers = new Map();
const BREAKER_THRESHOLD = 5;
const BREAKER_COOLDOWN = 30000;
function breakerOpen(host) {
  const b = breakers.get(host);
  if (!b) return false;
  if (b.failures >= BREAKER_THRESHOLD && Date.now() - b.lastFail < BREAKER_COOLDOWN) return true;
  if (Date.now() - b.lastFail >= BREAKER_COOLDOWN) { b.failures = 0; }
  return false;
}
function recordFail(host) {
  const b = breakers.get(host) || { failures: 0, lastFail: 0 };
  b.failures++; b.lastFail = Date.now();
  breakers.set(host, b);
}
function recordSuccess(host) { breakers.delete(host); }

// --- User-agent rotation ----------------------------------------------------
const USER_AGENTS = [
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36',
  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15',
  'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0 Safari/537.36',
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:121.0) Gecko/20100101 Firefox/121.0',
];
function pickUA() { return USER_AGENTS[Math.floor(Math.random() * USER_AGENTS.length)]; }

// --- Cookie jar -------------------------------------------------------------
const jars = new Map();
function parseCookies(str) {
  const out = {};
  str.split(';').forEach((pair) => {
    const idx = pair.indexOf('=');
    if (idx > -1) out[pair.slice(0, idx).trim()] = pair.slice(idx + 1).trim();
  });
  return out;
}
function getSessionId(req, res) {
  let sid = req.headers['x-proxy-session'] || parseCookies(req.headers.cookie || '')['ph_sid'];
  if (!sid) {
    sid = Math.random().toString(36).slice(2) + Date.now().toString(36);
    res.setHeader('Set-Cookie', `ph_sid=${sid}; Path=/; SameSite=Lax`);
  }
  return sid;
}
function storeCookies(sid, host, setCookieHeaders) {
  if (!setCookieHeaders) return;
  if (!jars.has(sid)) jars.set(sid, new Map());
  const hostJar = jars.get(sid);
  if (!hostJar.has(host)) hostJar.set(host, new Map());
  const jar = hostJar.get(host);
  const list = Array.isArray(setCookieHeaders) ? setCookieHeaders : [setCookieHeaders];
  list.forEach((c) => {
    const [pair] = c.split(';');
    const idx = pair.indexOf('=');
    if (idx > -1) jar.set(pair.slice(0, idx).trim(), pair.slice(idx + 1).trim());
  });
}
function cookieHeaderFor(sid, host) {
  const hostJar = jars.get(sid);
  if (!hostJar || !hostJar.has(host)) return '';
  return [...hostJar.get(host).entries()].map(([k, v]) => `${k}=${v}`).join('; ');
}

// --- URL rewriting ----------------------------------------------------------
function proxify(targetUrl) { return PROXY_PREFIX + encodeURIComponent(targetUrl); }

function rewriteHtml(html, baseUrl, origin) {
  const base = new URL(baseUrl);
  const proxifyAbs = (abs) => origin + PROXY_PREFIX + encodeURIComponent(abs);
  const toAbs = (val) => {
    try {
      if (!val || /^(data:|javascript:|mailto:|tel:|blob:|about:|#)/i.test(val)) return null;
      return new URL(val, base).href;
    } catch { return null; }
  };
  html = html.replace(/<base\b[^>]*>/gi, '');
  html = html.replace(/\b(href|src|action|poster|data-src|data-href)=("|')([^"']*)(\2)/gi,
    (m, attr, q, val) => { const abs = toAbs(val); return abs ? `${attr}=${q}${proxifyAbs(abs)}${q}` : m; });
  html = html.replace(/srcset=("|')([^"']*)(\1)/gi, (m, q, val) => {
    const parts = val.split(',').map((part) => {
      const t = part.trim(); if (!t) return t;
      const [url, descriptor] = t.split(/\s+/, 2);
      const abs = toAbs(url);
      return (abs ? proxifyAbs(abs) : url) + (descriptor ? ' ' + descriptor : '');
    });
    return `srcset=${q}${parts.join(', ')}${q}`;
  });
  html = html.replace(/url\(\s*(['"]?)([^'")]+)(\1)\s*\)/gi, (m, q, val) => {
    const abs = toAbs(val); return abs ? `url(${q}${proxifyAbs(abs)}${q})` : m;
  });
  html = html.replace(/\s+integrity=("|')[^"']*(\1)/gi, '');
  html = html.replace(/\s+crossorigin(=("|')[^"']*(\2))?/gi, '');
  const injected = `<base href="${origin}/">`;
  const realPath = base.pathname + base.search + base.hash;
  const pathFix = `<script>(function(){try{
    var p=${JSON.stringify(realPath)};
    if(location.pathname!==p){history.replaceState(null,'',p);}
  }catch(e){}})();<\/script>`;
  if (/<head[^>]*>/i.test(html)) html = html.replace(/<head([^>]*)>/i, `<head$1>${injected}${pathFix}`);
  else html = injected + pathFix + html;

  const navScript = `<script>(function(){
    function report(url){ try { parent.postMessage({type:'ph-navigate', url:url}, '*'); } catch(e){} }
    function decodeProxy(href){ try { var u=new URL(href,location.href);
      if(u.pathname==='/proxy'&&u.searchParams.get('url')) return u.searchParams.get('url'); }catch(e){} return null; }
    document.addEventListener('click',function(e){ var a=e.target.closest&&e.target.closest('a[href]'); if(!a)return;
      var t=decodeProxy(a.getAttribute('href')); if(t)report(t); },true);
    document.addEventListener('submit',function(e){ var f=e.target; if(!f||!f.action)return;
      var t=decodeProxy(f.getAttribute('action')||f.action); if(t)report(t); },true);
  })();<\/script>`;
  if (/<\/body>/i.test(html)) html = html.replace(/<\/body>/i, navScript + '</body>');
  else html += navScript;
  return html;
}

function rewriteCss(css, baseUrl, origin) {
  const base = new URL(baseUrl);
  const proxifyAbs = (abs) => origin + PROXY_PREFIX + encodeURIComponent(abs);
  return css.replace(/url\(\s*(['"]?)([^'")]+)(\1)\s*\)/gi, (m, q, val) => {
    try { if (/^(data:|#)/i.test(val)) return m; return `url(${q}${proxifyAbs(new URL(val, base).href)}${q})`; }
    catch { return m; }
  });
}

// --- MIME sniffing ----------------------------------------------------------
const EXT_MIME = {
  '.html': 'text/html; charset=utf-8', '.htm': 'text/html; charset=utf-8',
  '.css': 'text/css; charset=utf-8', '.js': 'application/javascript; charset=utf-8',
  '.mjs': 'application/javascript; charset=utf-8', '.json': 'application/json; charset=utf-8',
  '.xml': 'application/xml; charset=utf-8', '.svg': 'image/svg+xml',
  '.png': 'image/png', '.jpg': 'image/jpeg', '.jpeg': 'image/jpeg', '.gif': 'image/gif',
  '.webp': 'image/webp', '.avif': 'image/avif', '.ico': 'image/x-icon', '.bmp': 'image/bmp',
  '.woff': 'font/woff', '.woff2': 'font/woff2', '.ttf': 'font/ttf', '.otf': 'font/otf',
  '.eot': 'application/vnd.ms-fontobject', '.mp4': 'video/mp4', '.webm': 'video/webm',
  '.mp3': 'audio/mpeg', '.wav': 'audio/wav', '.ogg': 'audio/ogg', '.pdf': 'application/pdf',
  '.wasm': 'application/wasm', '.txt': 'text/plain; charset=utf-8',
  '.webmanifest': 'application/manifest+json', '.map': 'application/json',
};
function sniffMime(url, upstreamType) {
  const t = (upstreamType || '').split(';')[0].trim().toLowerCase();
  if (t && t !== 'application/octet-stream' && t !== 'binary/octet-stream' && t !== 'text/plain') {
    return upstreamType;
  }
  try {
    const path = new URL(url).pathname.toLowerCase();
    const dot = path.lastIndexOf('.');
    if (dot !== -1) {
      const ext = path.slice(dot);
      if (EXT_MIME[ext]) return EXT_MIME[ext];
    }
  } catch {}
  return upstreamType || 'application/octet-stream';
}

function decompress(buf, enc) {
  enc = (enc || '').toLowerCase();
  try {
    if (enc.includes('br')) return zlib.brotliDecompressSync(buf);
    if (enc.includes('gzip')) return zlib.gunzipSync(buf);
    if (enc.includes('deflate')) return zlib.inflateSync(buf);
  } catch { /* raw */ }
  return buf;
}

function errorPage(title, detail) {
  return `<!DOCTYPE html><html><head><meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1"><title>${title}</title>
  <style>body{font-family:system-ui,sans-serif;background:#0f172a;color:#e2e8f0;display:flex;
    align-items:center;justify-content:center;min-height:100vh;margin:0}
    .box{max-width:520px;padding:32px;background:#1e293b;border:1px solid #334155;border-radius:16px;text-align:center}
    h1{font-size:1.4rem;margin:0 0 12px}p{color:#94a3b8;line-height:1.5;margin:0 0 20px}
    a{display:inline-block;padding:10px 20px;border-radius:10px;text-decoration:none;
    background:linear-gradient(90deg,#38bdf8,#818cf8);color:#0f172a;font-weight:700}</style></head>
    <body><div class="box"><h1>${title}</h1><p>${detail}</p><a href="/">← Back to PlayHub</a></div></body></html>`;
}

// --- Core fetch with retry + backoff ----------------------------------------
async function fetchUpstream(targetUrl, method, headers, body, attempt = 0) {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 45000);
  try {
    const res = await request(targetUrl.href, {
      method, headers, body, signal: controller.signal, dispatcher: agent, maxRedirections: 0,
    });
    clearTimeout(timeout);
    return res;
  } catch (err) {
    clearTimeout(timeout);
    if (attempt < 2) {
      const delay = 200 * Math.pow(2, attempt);
      await new Promise((r) => setTimeout(r, delay));
      return fetchUpstream(targetUrl, method, headers, body, attempt + 1);
    }
    throw err;
  }
}

// --- Proxy handler ----------------------------------------------------------
async function handleProxy(req, res) {
  const t0 = Date.now();
  metrics.requests++;

  const target = req.query.url;
  if (!target) return res.status(200).send(errorPage('No URL provided', 'Enter a website address to browse.'));

  let targetUrl;
  try {
    targetUrl = new URL(target);
    if (!['http:', 'https:'].includes(targetUrl.protocol)) throw new Error('bad protocol');
  } catch {
    return res.status(200).send(errorPage('Invalid address', "That doesn't look like a valid website address."));
  }

  const sid = getSessionId(req, res);
  const host = targetUrl.host;
  const method = req.method === 'POST' ? 'POST' : 'GET';
  const origin = `${req.protocol}://${req.get('host')}`;
  const fwdHost = req.headers['x-forwarded-host'];
  const fwdProto = (req.headers['x-forwarded-proto'] || '').split(',')[0].trim();
  const publicOrigin = fwdHost ? `${fwdProto || 'https'}://${fwdHost.split(',')[0].trim()}` : origin;

  if (!rateLimit(sid)) {
    res.setHeader('Retry-After', '2');
    return res.status(200).send(errorPage('Slow down a moment', 'You\'re browsing very fast. Please wait a second and try again.'));
  }

  if (breakerOpen(host)) {
    return res.status(200).send(errorPage('Temporarily unavailable', `<b>${host}</b> has been failing repeatedly. We'll try again shortly.`));
  }

  const ckey = method + ' ' + targetUrl.href;

  const render = (result) => {
    if (!result.body) return result;
    if (result.kind === 'html') {
      return { body: Buffer.from(rewriteHtml(result.body.toString('utf8'), targetUrl.href, publicOrigin)), contentType: 'text/html; charset=utf-8' };
    }
    if (result.kind === 'css') {
      return { body: Buffer.from(rewriteCss(result.body.toString('utf8'), targetUrl.href, publicOrigin)), contentType: 'text/css; charset=utf-8' };
    }
    return result;
  };

  if (method === 'GET') {
    const hit = cache.get(ckey);
    if (hit && Date.now() - hit.time < CACHE_TTL) {
      metrics.cacheHits++;
      const out = render(hit);
      res.setHeader('X-Proxy-Cache', 'HIT');
      res.setHeader('Content-Type', out.contentType);
      metrics.bytesOut += out.body.length;
      return res.status(200).send(out.body);
    }
    if (hit) {
      metrics.cacheHits++;
      const out = render(hit);
      res.setHeader('X-Proxy-Cache', 'STALE');
      res.setHeader('Content-Type', out.contentType);
      metrics.bytesOut += out.body.length;
      res.status(200).send(out.body);
      refreshInBackground(ckey, targetUrl, sid, host);
      return;
    }
  }

  if (method === 'GET' && inflight.has(ckey)) {
    metrics.coalesced++;
    try {
      const result = await inflight.get(ckey);
      const out = render(result);
      res.setHeader('X-Proxy-Cache', 'COALESCED');
      res.setHeader('Content-Type', out.contentType);
      return res.status(200).send(out.body);
    } catch {
      return res.status(200).send(errorPage("Couldn't reach that site", `We couldn't connect to <b>${host}</b>.`));
    }
  }

  const headers = {
    'user-agent': pickUA(),
    accept: req.headers['accept'] || 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'accept-language': req.headers['accept-language'] || 'en-US,en;q=0.9',
    'accept-encoding': 'gzip, deflate, br',
  };
  const cookie = cookieHeaderFor(sid, host);
  if (cookie) headers.cookie = cookie;

  const doFetch = async () => {
    const upstream = await fetchUpstream(targetUrl, method, headers,
      method === 'POST' ? new URLSearchParams(req.body).toString() : undefined);
    const status = upstream.statusCode;
    const h = upstream.headers;

    if ([301, 302, 303, 307, 308].includes(status) && h.location) {
      let abs; try { abs = new URL(h.location, targetUrl).href; } catch { abs = h.location; }
      upstream.body.dump();
      return { redirect: abs };
    }

    storeCookies(sid, host, h['set-cookie']);
    const contentType = sniffMime(targetUrl.href, h['content-type']);
    const isText = /text\/html|text\/css|application\/javascript|text\/javascript|application\/json|text\/plain|application\/xml|text\/xml|image\/svg/i.test(contentType);

    if (isText) {
      const chunks = [];
      for await (const chunk of upstream.body) chunks.push(chunk);
      const body = decompress(Buffer.concat(chunks), h['content-encoding']);
      const kind = contentType.includes('text/html') ? 'html' : (contentType.includes('text/css') ? 'css' : 'text');
      return { body, contentType, kind };
    }
    return { stream: upstream.body, contentType };
  };

  try {
    const promise = doFetch();
    if (method === 'GET') inflight.set(ckey, promise);
    const result = await promise;
    if (method === 'GET') inflight.delete(ckey);

    recordSuccess(host);
    bumpHost(host, Date.now() - t0);

    if (result.redirect) return res.redirect(proxify(result.redirect));

    ['x-frame-options', 'content-security-policy', 'content-security-policy-report-only',
     'x-content-security-policy', 'content-encoding', 'content-length', 'transfer-encoding',
     'strict-transport-security', 'permissions-policy', 'cross-origin-opener-policy',
     'cross-origin-embedder-policy', 'cross-origin-resource-policy'].forEach((k) => res.removeHeader(k));
    res.setHeader('Access-Control-Allow-Origin', '*');
    res.setHeader('X-Content-Type-Options', 'nosniff');

    if (result.body) {
      if (method === 'GET' && result.body.length < 8 * 1024 * 1024) {
        cache.set(ckey, { body: result.body, contentType: result.contentType, kind: result.kind, time: Date.now() });
      }
      const out = render(result);
      metrics.cacheMisses++;
      metrics.bytesOut += out.body.length;
      res.setHeader('X-Proxy-Cache', 'MISS');
      res.setHeader('Content-Type', out.contentType);
      return res.status(200).send(out.body);
    }

    res.setHeader('Content-Type', result.contentType);
    res.status(200);
    result.stream.pipe(res);
    result.stream.on('error', () => res.end());
  } catch (err) {
    metrics.errors++;
    recordFail(host);
    log('warn', 'proxy fetch failed', { host, err: err.message });
    return res.status(200).send(errorPage("Couldn't reach that site",
      `We couldn't connect to <b>${host}</b>. It may be offline, blocking automated access, or the address may be wrong.`));
  }
}

async function refreshInBackground(ckey, targetUrl, sid, host) {
  if (inflight.has(ckey)) return;
  const headers = { 'user-agent': pickUA(), accept: 'text/html,*/*;q=0.8', 'accept-encoding': 'gzip, deflate, br' };
  const cookie = cookieHeaderFor(sid, host);
  if (cookie) headers.cookie = cookie;
  const p = (async () => {
    try {
      const upstream = await fetchUpstream(targetUrl, 'GET', headers);
      const contentType = sniffMime(targetUrl.href, upstream.headers['content-type']);
      if (!/text\/html|text\/css/i.test(contentType)) { upstream.body.dump(); return; }
      const chunks = [];
      for await (const c of upstream.body) chunks.push(c);
      const body = decompress(Buffer.concat(chunks), upstream.headers['content-encoding']);
      const kind = contentType.includes('text/html') ? 'html' : 'css';
      cache.set(ckey, { body, contentType, kind, time: Date.now() });
    } catch { /* ignore */ }
  })();
  inflight.set(ckey, p);
  p.finally(() => inflight.delete(ckey));
}

app.get('/proxy', handleProxy);
app.post('/proxy', handleProxy);

// --- AI assistant relay -----------------------------------------------------
app.post('/api/assistant', async (req, res) => {
  const { apiKey, baseUrl, model, messages } = req.body || {};
  if (!apiKey) return res.status(400).json({ error: 'Missing API key' });
  const base = (baseUrl || 'https://api.openai.com/v1').replace(/\/$/, '');
  const mdl = model || 'gpt-4o-mini';
  try {
    const upstream = await request(`${base}/chat/completions`, {
      method: 'POST',
      headers: { 'content-type': 'application/json', authorization: `Bearer ${apiKey}` },
      body: JSON.stringify({ model: mdl, messages, temperature: 0.4 }),
      dispatcher: agent, headersTimeout: 60000, bodyTimeout: 60000,
    });
    const chunks = [];
    for await (const c of upstream.body) chunks.push(c);
    res.status(upstream.statusCode).setHeader('Content-Type', 'application/json');
    return res.send(Buffer.concat(chunks).toString('utf8'));
  } catch (err) {
    return res.status(502).json({ error: 'Assistant request failed: ' + err.message });
  }
});

// --- Stats / health ---------------------------------------------------------
app.get('/health', (req, res) => res.json({ ok: true, uptime: process.uptime() }));
app.get('/api/stats', (req, res) => {
  const topHosts = [...metrics.byHost.entries()]
    .map(([host, e]) => ({ host, count: e.count, avgMs: Math.round(e.totalMs / e.count) }))
    .sort((a, b) => b.count - a.count).slice(0, 10);
  res.json({
    uptimeSec: Math.round((Date.now() - metrics.startedAt) / 1000),
    requests: metrics.requests,
    cacheHits: metrics.cacheHits,
    cacheMisses: metrics.cacheMisses,
    coalesced: metrics.coalesced,
    errors: metrics.errors,
    cacheEntries: cache.size,
    cacheBytes: cache.calculatedSize || 0,
    bytesOut: metrics.bytesOut,
    topHosts,
  });
});

app.use((req, res) => res.status(200).send(errorPage('Page not found', "That page doesn't exist on PlayHub.")));

const server = app.listen(PORT, '0.0.0.0', () => log('info', 'PlayHub v3 started', { port: PORT }));

// --- WebSocket passthrough --------------------------------------------------
const WebSocket = require('ws');
const wss = new WebSocket.Server({ noServer: true });
server.on('upgrade', (req, socket, head) => {
  try {
    const u = new URL(req.url, `http://${req.headers.host}`);
    if (u.pathname !== '/ws') { socket.destroy(); return; }
    const target = u.searchParams.get('url');
    if (!target) { socket.destroy(); return; }
    const t = new URL(target);
    const upstream = new WebSocket(target, {
      headers: { 'User-Agent': pickUA(), Origin: `${t.protocol}//${t.host}` },
      handshakeTimeout: 15000,
    });
    wss.handleUpgrade(req, socket, head, (client) => {
      const queue = [];
      upstream.on('open', () => {
        while (queue.length) upstream.send(queue.shift());
        client.on('message', (d) => { if (upstream.readyState === 1) upstream.send(d); });
        upstream.on('message', (d) => { if (client.readyState === 1) client.send(d); });
      });
      client.on('message', (d) => { if (upstream.readyState === 1) upstream.send(d); else queue.push(d); });
      const closeBoth = () => { try { client.close(); } catch {} try { upstream.close(); } catch {} };
      client.on('close', closeBoth);
      upstream.on('close', closeBoth);
      client.on('error', closeBoth);
      upstream.on('error', closeBoth);
      metrics.requests++;
    });
  } catch (err) {
    log('warn', 'ws upgrade failed', { err: err.message });
    socket.destroy();
  }
});

// --- Graceful shutdown ------------------------------------------------------
function shutdown(sig) {
  log('info', 'shutting down', { sig });
  server.close(() => { agent.close().finally(() => process.exit(0)); });
  setTimeout(() => process.exit(0), 5000);
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
#!/bin/bash
# Persistent Cloudflare quick-tunnel wrapper.
set -u

URLFILE="/workspace/PUBLIC_URL.txt"
LOGFILE="/workspace/tunnel.log"

echo "Starting PlayHub tunnel..." > "$LOGFILE"

while true; do
  cloudflared tunnel --url http://localhost:3000 --no-autoupdate 2>&1 | while IFS= read -r line; do
    echo "$line" >> "$LOGFILE"
    url=$(echo "$line" | grep -oE 'https://[a-z0-9-]+\.trycloudflare\.com' | head -1)
    if [ -n "$url" ]; then
      echo "$url" > "$URLFILE"
      echo "[$(date -u +%FT%TZ)] Public URL: $url" >> "$LOGFILE"
    fi
  done

  echo "[$(date -u +%FT%TZ)] cloudflared exited; restarting in 3s..." >> "$LOGFILE"
  sleep 3
done
