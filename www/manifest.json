'use strict';

const CACHE = 'dolby-player-v1';

// Files to pre-cache for offline use
const PRECACHE = [
  '/',
  '/index.html',
  '/manifest.json',
  '/icon-192.png',
  '/icon-512.png',
];

// ── Install: pre-cache shell ──
self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(CACHE).then(c => c.addAll(PRECACHE).catch(() => {}))
  );
  self.skipWaiting();
});

// ── Activate: clean old caches ──
self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k)))
    )
  );
  self.clients.claim();
});

// ── Fetch: cache-first for shell, network-first for audio blobs ──
self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);

  // Let blob: URLs (local audio object URLs) pass straight through —
  // they live in the page's memory and cannot be cached by the SW
  if (url.protocol === 'blob:') return;

  // For same-origin requests, try cache first then fall back to network
  if (url.origin === self.location.origin) {
    e.respondWith(
      caches.match(e.request).then(cached => {
        if (cached) return cached;
        return fetch(e.request).then(resp => {
          // Only cache successful, non-opaque responses for the app shell
          if (resp && resp.status === 200 && resp.type === 'basic') {
            const clone = resp.clone();
            caches.open(CACHE).then(c => c.put(e.request, clone));
          }
          return resp;
        }).catch(() => cached); // offline fallback
      })
    );
    return;
  }

  // For cross-origin (e.g. CDN fonts) just fetch normally
  e.respondWith(fetch(e.request).catch(() => new Response('', { status: 408 })));
});

// ── Background audio keepalive ──
// The SW stays alive while audio plays by responding to periodic messages
// from the page. This prevents the browser from suspending the audio context.
self.addEventListener('message', e => {
  if (e.data === 'keepalive') {
    // Acknowledge — keeping the SW alive
    e.ports[0]?.postMessage('ack');
  }
  if (e.data === 'skipWaiting') {
    self.skipWaiting();
  }
});