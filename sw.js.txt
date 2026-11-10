/* LN_Education — service worker
   - Le site (index.html) : réseau d'abord, copie en mémoire si pas de connexion
     (les mises à jour du site arrivent donc toutes seules, sans rien changer ici).
   - Les PDF : gardés en mémoire après la première ouverture (lisibles hors ligne).
   Si un jour il faut vider tout ce que les élèves ont en mémoire, changer VERSION. */
const VERSION = 'v1';
const SHELL = 'lne-shell-' + VERSION;
const RUNTIME = 'lne-runtime-' + VERSION;
const MAX_DOCS = 80;
const SHELL_FILES = ['./', './index.html', './manifest.json', './icon-192.png', './icon-512.png'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(SHELL).then((c) => c.addAll(SHELL_FILES)).then(() => self.skipWaiting())
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(keys.filter((k) => k !== SHELL && k !== RUNTIME).map((k) => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});

function timeout(ms) {
  return new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms));
}

async function networkFirst(req, cacheName, ms) {
  const cache = await caches.open(cacheName);
  try {
    const res = await Promise.race([fetch(req), timeout(ms)]);
    if (res && res.ok) cache.put(req, res.clone());
    return res;
  } catch (e) {
    const hit = (await cache.match(req)) || (await cache.match('./index.html')) || (await cache.match('./'));
    if (hit) return hit;
    return new Response('Hors ligne', { status: 503, headers: { 'Content-Type': 'text/plain; charset=utf-8' } });
  }
}

async function trim(cache, max) {
  const keys = await cache.keys();
  const docs = keys.filter((r) => /\.pdf(\?|$)/i.test(r.url));
  for (let i = 0; i < docs.length - max; i++) await cache.delete(docs[i]);
}

async function cacheFirstDoc(req) {
  const cache = await caches.open(RUNTIME);
  const hit = await cache.match(req);
  if (hit) return hit;
  try {
    const res = await fetch(req);
    if (res && res.status === 200) {
      await cache.put(req, res.clone());
      trim(cache, MAX_DOCS);
    }
    return res;
  } catch (e) {
    return new Response(
      'Ce document n\'est pas encore disponible hors ligne. Ouvre-le une première fois avec une connexion Internet.',
      { status: 503, headers: { 'Content-Type': 'text/plain; charset=utf-8' } }
    );
  }
}

async function staleWhileRevalidate(req, cacheName) {
  const cache = await caches.open(cacheName);
  const hit = await cache.match(req);
  const net = fetch(req).then((res) => {
    if (res && (res.ok || res.type === 'opaque')) cache.put(req, res.clone());
    return res;
  }).catch(() => hit);
  return hit || net;
}

self.addEventListener('fetch', (event) => {
  const req = event.request;
  if (req.method !== 'GET' || req.headers.has('range')) return;
  const url = new URL(req.url);

  if (req.mode === 'navigate') {
    event.respondWith(networkFirst(req, SHELL, 4000));
    return;
  }
  if (url.origin === self.location.origin) {
    if (/\.pdf$/i.test(url.pathname)) {
      event.respondWith(cacheFirstDoc(req));
    } else {
      event.respondWith(staleWhileRevalidate(req, SHELL));
    }
    return;
  }
  if (url.hostname === 'fonts.googleapis.com' || url.hostname === 'fonts.gstatic.com') {
    event.respondWith(staleWhileRevalidate(req, RUNTIME));
  }
});
