# Seenzo — Lecture vidéo côté React (front web)

> **But de ce document** : permettre à une IA (ou un dev) de **ré-intégrer entièrement** la chaîne
> de **lecture vidéo dans le front React** (`web/`) **de zéro** — de la demande de session jusqu'au
> lecteur (hls.js + Plyr), la bascule de flux, la reprise, l'auto-play épisode suivant et le Cast.
> C'est le **pendant front** de `PURSTREAM_SCRAPER.md` (qui, lui, couvre le backend Node/purstream).
> Ici on ne parle **jamais** de purstream/zebi en direct : le front ne connaît **que Seenzo**.

---

## 0. Principe fondateur (à ne JAMAIS casser)

**Le navigateur ne voit que `seenzo.site`.** Il ne connaît ni `purstream`, ni `nakastream`, ni
`zebi.senpai-stream.club`, ni CineFuse. Tout passe par le **proxy Seenzo**. La résolution du flux
réel (purstream → zebi) est faite **côté serveur** (cf. `PURSTREAM_SCRAPER.md`). Le front reçoit une
**session opaque** avec un **token** et lit un `.m3u8` **relatif à Seenzo**.

```
React                       Backend Seenzo                    Amont (caché du navigateur)
─────                       ──────────────                    ───────────────────────────
POST /api/play/:slug   ──►  résout la source (purstream…)  ──► api.purstream.mx / nakastream
   { pow, flux? }              mint playToken (TTL 30 min)
        ◄── session { kind, token, master, proxyBase, resumeSec, … }
hls.js charge:
  /api/proxy/{base}/{token}/{master}  ──► proxy 302 → zebi .m3u8   (0 bande passante VPS)
```

Stack front : **React 18 + Vite + Tailwind**, lecteur **hls.js** (MSE) habillé par **Plyr**
(skin custom, i18n FR), `socket.io-client` (viewers temps réel), Google Cast (Chromecast).
Fichier principal : `web/src/pages/Player.jsx`. Client API : `web/src/lib/api.js`.

---

## 1. Le client API (`web/src/lib/api.js`)

Tout part d'un `fetch` same-origin préfixé par `BASE` (`/` en dev, `/cine/` en prod sous-chemin).

```js
export const BASE = import.meta.env.BASE_URL || '/';

async function request(method, url, body) {
  const opts = { method, headers: {}, credentials: 'same-origin' }; // cookie JWT HTTP-Only
  if (body !== undefined) {
    opts.headers['Content-Type'] = 'application/json';
    opts.body = JSON.stringify(body);
  }
  const res = await fetch(`${BASE}api${url}`, opts);
  const isJson = (res.headers.get('content-type') || '').includes('application/json');
  const data = isJson ? await res.json() : null;
  if (!res.ok) { const e = new Error(data?.error || `Erreur ${res.status}`); e.status = res.status; e.code = data?.code; throw e; }
  return data;
}
export const api = { get:(u)=>request('GET',u), post:(u,b)=>request('POST',u,b), /* put, patch, del */ };
```

Points **non négociables** :
- **`credentials: 'same-origin'`** → envoie le cookie JWT HTTP-Only (auth). Sans ça, `/play` renvoie 401.
- L'erreur porte `.status` (le code HTTP) et `.code` → le front distingue 401 (→ redirige) des autres.
- **`img(path, size)`** sert les images via **notre** proxy `/img/:size/:file` (WebP + edge Cloudflare),
  jamais `image.tmdb.org` en direct (CSP `'self'` + LCP).

```js
export function img(path, size = 'w500') {
  if (!path) return null;
  return `${BASE}img/${size}/${String(path).replace(/^\/+/, '')}`;
}
```

---

## 2. Construire l'URL de flux (proxy Seenzo)

Le front ne fabrique **jamais** une URL amont. Il compose l'URL **du proxy** à partir de la session :

```js
// base = session.proxyBase : 'hls' (nakastream) | 'manual' (source admin). CineFuse = iframe (pas ici).
const hlsUrl = (token, path, base = 'hls') => `${BASE}api/proxy/${base}/${token}/${path}`;
```

- `token` : le playToken renvoyé par `/play` (une **session** de lecture, TTL 30 min, multi-Range).
- `path` : `session.master` (le master.m3u8) puis, en interne, les playlists/segments **relatifs**.
- Tout est **relatif au master** → immunisé au sous-chemin `/cine` et au changement de domaine amont.

---

## 3. Le contrat de la session `/play` (ce que le front reçoit)

`POST /api/play/:slug` ou `/api/play/:slug/:episode`, body `{ pow, flux? }`. Réponse = une **session** :

| Champ | Type | Rôle |
|-------|------|------|
| `kind` | `'hls' \| 'file' \| 'iframe'` | comment lire : HLS (hls.js), fichier mp4 direct, ou iframe (CineFuse) |
| `token` | string | playToken pour le proxy |
| `master` | string | nom du master (`master.m3u8`) — pour `kind: 'hls'` |
| `file` | string | nom du fichier — pour `kind: 'file'` |
| `embedUrl` | string | URL d'iframe — pour `kind: 'iframe'` (CineFuse) |
| `proxyBase` | `'hls' \| 'manual'` | segment de base de l'URL proxy |
| `resumeSec` | number | position sauvegardée → overlay « Reprendre / Recommencer » |
| `flux` | string | flux réellement servi (peut différer du flux demandé) |
| `availableFlux` | string[] | flux proposés dans le menu Source (déf. `['hls','api']`) |
| `subtitles` | `[{ path, lang, label, default }]` | pistes de sous-titres (via proxy) |

Les **trois `kind`** décident du rendu de la surface vidéo (cf. §7).

---

## 4. Récupérer la session (effet React)

`useEffect` déclenché sur `[slug, episode, flux]`. **PoW invisible** anti-bot avant le mint,
`socket.io` pour les viewers, garde d'annulation (`cancelled`) contre les courses.

```js
useEffect(() => {
  const socket = getSocket();
  let cancelled = false;
  const onViewers = ({ slug: s, count }) => { if (s === slug) setViewers(count); };
  (async () => {
    try {
      setError(''); setSession(null);
      if (flux !== 'api') fallbackTriedRef.current = false;     // nouvelle tentative HLS → fallback réarmé
      const url = episode ? `/play/${slug}/${episode}` : `/play/${slug}`;
      const pow = await solvePow();                             // captcha invisible
      if (cancelled) return;
      const data = await api.post(url, flux ? { flux, pow } : { pow });
      if (cancelled) return;
      setSession(data); setNotice('');
      socket.emit('watch:join', slug); socket.on('viewers', onViewers);
    } catch (err) {
      if (cancelled) return;
      if (err.status === 401) { navigate(`/media/${slug}`); return; } // pas connecté
      if (flux !== 'api') { fallbackToCinefuse(); return; }            // HLS KO → bascule CineFuse
      setError(err.message || 'Lecture indisponible');
    }
  })();
  return () => { cancelled = true; socket.emit('watch:leave'); socket.off('viewers', onViewers); };
}, [slug, episode, flux, navigate]);
```

---

## 5. Le lecteur hls.js + Plyr (le cœur)

`useEffect` sur `[session]`. On monte hls.js **uniquement** pour `kind:'hls'`/`'file'`.
Réglages hls.js **spécifiques à un host amont lent** (gros buffer, timeouts longs, pas d'abandon
prématuré de segment) — critiques pour éviter les coupures :

```js
useEffect(() => {
  if ((session?.kind !== 'hls' && session?.kind !== 'file') || !videoRef.current) return;
  const video = videoRef.current;
  const base = session.proxyBase || 'hls';
  let hls;

  // Reprise : si resumeSec > 5 on N'AUTOPLAY PAS (l'overlay décide). Sinon autoplay depuis le début.
  const resumeSec = Number(session.resumeSec || 0);
  const autoplay = !(resumeSec > 5);

  const baseOpts = {
    controls: ['play-large','play','progress','current-time','duration','mute','volume','captions','settings','pip','fullscreen'],
    settings: ['captions','quality','speed'],
    speed: { selected: 1, options: [0.5,0.75,1,1.25,1.5,2] },
    keyboard: { focused: true, global: true },
    iconUrl: `${BASE}plyr.svg`,   // sprite LOCAL : la CSP 'self' bloque le CDN Plyr (icônes invisibles sinon)
    i18n: PLYR_I18N,              // interface 100% FR
    storage: { enabled: true, key: 'seenzo-plyr' },
  };

  if (session.kind === 'file') {                       // mp4 direct : lecture native
    video.src = hlsUrl(session.token, session.file || 'file', base);
    plyrRef.current = new Plyr(video, baseOpts);
    if (autoplay) video.play?.().catch(() => {});
  } else if (Hls.isSupported()) {
    hls = new Hls({
      enableWorker: true, lowLatencyMode: false,
      maxBufferLength: 60, maxMaxBufferLength: 600, backBufferLength: 60, // buffer large → lisse les segments lents
      fragLoadingTimeOut: 90_000, fragLoadingMaxRetry: 8, fragLoadingMaxRetryTimeout: 64_000, // NE PAS lâcher un segment lent
      manifestLoadingTimeOut: 20_000, levelLoadingTimeOut: 20_000,
    });
    hlsRef.current = hls;
    hls.loadSource(hlsUrl(session.token, session.master, base));
    hls.attachMedia(video);
    hls.on(Hls.Events.MANIFEST_PARSED, () => {
      const heights = [...new Set(hls.levels.map(l => l.height).filter(Boolean))].sort((a,b)=>b-a);
      plyrRef.current = new Plyr(video, { ...baseOpts, quality: {
        default: 0, options: [0, ...heights], forced: true,
        onChange: (q) => { if (q === 0) { hls.currentLevel = -1; return; }
          hls.levels.forEach((l,i)=>{ if (l.height === q) hls.currentLevel = i; }); },
      }});
      if (autoplay) video.play?.().catch(() => {});
    });
    hls.on(Hls.Events.ERROR, (_e, data) => {
      if (!data.fatal) return;
      const d = data.details || '';
      if (d.includes('manifestLoad') || d.includes('manifestParsing') || d.includes('levelLoad')) {
        fallbackToCinefuse(d); return;                 // lien HLS cassé → bascule CineFuse
      }
      if (data.type === Hls.ErrorTypes.NETWORK_ERROR) hls.startLoad();       // récup réseau : re-mint amont côté proxy
      else if (data.type === Hls.ErrorTypes.MEDIA_ERROR) hls.recoverMediaError();
      else fallbackToCinefuse(d || data.type || 'hls fatal');
    });
  } else if (video.canPlayType('application/vnd.apple.mpegurl')) {           // Safari : HLS natif
    video.src = hlsUrl(session.token, session.master, base);
    plyrRef.current = new Plyr(video, baseOpts);
  }

  return () => { try { plyrRef.current?.destroy(); } catch {} try { hls?.destroy(); } catch {}
    plyrRef.current = null; hlsRef.current = null; };   // NETTOYAGE OBLIGATOIRE (fuite mémoire / double lecteur sinon)
}, [session]);
```

> ⚠️ **Le cleanup `return () =>` est obligatoire** : sans `plyr.destroy()` + `hls.destroy()` on
> empile des instances (double son, fuite mémoire) à chaque changement d'épisode/flux.

---

## 6. Bascule de flux + fallback CineFuse

Le flux vit dans un state `flux` (`null` = défaut média · `'hls'` = nakastream · `'api'` = CineFuse
· `'manual'` = source admin). **Changer `flux` re-déclenche l'effet §4** → nouvelle session.

```js
const fallbackToCinefuse = (reason) => {
  if (fallbackTriedRef.current) return;                 // une seule bascule par tentative HLS
  fallbackTriedRef.current = true;
  if (reason) api.post('/play/hls-error', { slug, episode, reason }).catch(()=>{}); // prévient le staff (Discord)
  setNotice('Source HLS indisponible — bascule sur CineFuse…');
  setFlux('api');
};
```

Menu **Source** dans la barre haute : liste `session.availableFlux`, coche le `session.flux` courant,
`onClick` → `setFlux(key)`. `FLUX_LABELS` mappe la clé vers un libellé FR (`hls`→« Flux HLS », `api`→
« Flux API · CineFuse », `manual`→« Flux Manuel »).

---

## 7. Rendu de la surface vidéo (les 3 `kind`)

Une **`key` qui force le remontage** au changement de flux (sinon le wrapper Plyr noir reste collé
par-dessus l'iframe) :

```jsx
<div key={isVideo ? `vid-${session.token}` : session?.kind || 'idle'} className="absolute inset-0">
  {error ? <div>…{error}</div>
   : isVideo /* kind hls|file */ ? (
      <video key={session.token} ref={videoRef} playsInline crossOrigin="anonymous" className="w-full h-full">
        {session.subtitles?.map((st) => (
          <track key={st.lang} kind="captions" srcLang={st.lang} label={st.label || st.lang}
                 src={hlsUrl(session.token, st.path, session.proxyBase)} default={st.default} />
        ))}
      </video>
   ) : session?.kind === 'iframe' ? (
      <iframe src={session.embedUrl} allow="autoplay; fullscreen; encrypted-media; picture-in-picture" allowFullScreen className="w-full h-full border-0" />
   ) : <div>Préparation de la lecture…</div>}
</div>
```

- `crossOrigin="anonymous"` sur `<video>` : requis pour les pistes `<track>` servies via proxy.
- CineFuse = **iframe** (`kind:'iframe'`) → pas de `<video>` exploitable ⇒ **ni progression ni Cast**.

---

## 8. Reprise, progression, épisode suivant

- **Overlay reprise** : si `session.resumeSec > 5`, on affiche « Reprendre à hh:mm » / « Recommencer ».
  `decideResume(bool)` fait le `video.currentTime = sec` (après `loadedmetadata` si pas encore seekable)
  puis `play()`. Tant que l'overlay est ouvert, **pas d'autoplay**.
- **Progression** : `useEffect` sur `[session, slug, episode]` — sauvegarde toutes les 15 s (si non en
  pause), sur `pause`, sur `ended` (completed), et sur `visibilitychange` (onglet caché) + une sauvegarde
  finale au cleanup. **Ignore les < 5 s**. Endpoint `POST /watch/progress`
  `{ slug, episode, positionSec, durationSec, completed }`. `completed` = position/durée > 0.92.
  La réponse peut renvoyer `cardAwarded` (carte Cinédex débloquée → toast).
- **Épisode suivant** : carte « Next up » à ≤ 150 s de la fin ; à `ended`, compte à rebours 10 s puis
  `navigate(/lecture/:slug/:nextSlug)`. `nextRef` (ref) évite de recréer l'intervalle à chaque render.

---

## 9. Google Cast (Chromecast)

Uniquement pour `kind:'hls'` (l'iframe CineFuse n'est pas castable). On tient à jour un `castMediaRef`
avec l'**URL ABSOLUE** du master (`window.location.origin + hlsUrl(...)`) + sous-titres, on charge le
SDK `cast_sender.js` (gstatic), et on `loadMedia` un `MediaInfo(url,'application/x-mpegURL')` au démarrage
de session Cast. Le Chromecast récupère le flux **depuis le réseau** (URL absolue proxy Seenzo), pas
depuis l'onglet.

---

## 10. CSP côté navigateur (sinon écran noir / icônes manquantes)

La CSP (backend `src/middleware/security.js`) doit autoriser ce que **le navigateur** charge :
```js
connectSrc: ["'self'", 'https://www.gstatic.com', 'https://challenges.cloudflare.com', 'https://zebi.senpai-stream.club'],
mediaSrc:   ["'self'", 'blob:', 'https://cinefuse.cc', 'https://cdn.plyr.io', 'https://zebi.senpai-stream.club'],
```
- `zebi.senpai-stream.club` : le proxy **302-redirige** vers zebi → le `.m3u8`/segments sont chargés
  **directement par le navigateur** ⇒ le domaine CDN doit être en `connect-src` **et** `media-src`.
  **Si le domaine CDN change, mettre à jour la CSP** (cf. `PURSTREAM_SCRAPER.md` §5).
- Sprite Plyr servi **en local** (`web/public/plyr.svg` → `${BASE}plyr.svg`) car la CSP `'self'`
  bloque le CDN Plyr (sinon **toutes les icônes du lecteur disparaissent**).
- `www.gstatic.com` : SDK Google Cast.

---

## 11. RECETTE — intégrer de zéro (checklist IA, côté React)

1. **Client API** (`web/src/lib/api.js`) : `request()` avec `credentials:'same-origin'`, helpers
   `api.get/post`, `img()`, `BASE`.
2. **Deps** : `hls.js`, `plyr` (+ `plyr/dist/plyr.css`), `socket.io-client`. Copier le sprite
   `plyr.svg` dans `web/public/`.
3. **Page Player** (`web/src/pages/Player.jsx`) :
   - effet **session** : `POST /api/play/:slug[/:episode]` avec `{ pow, flux? }` (§4) ;
   - effet **lecteur** hls.js + Plyr sur `[session]`, avec les réglages buffer/timeout (§5) **et le
     cleanup destroy** ;
   - `hlsUrl(token, path, base)` = `${BASE}api/proxy/${base}/${token}/${path}` (§2) ;
   - rendu des **3 kind** (video / iframe / file) avec `key` de remontage (§7) ;
   - bascule flux + `fallbackToCinefuse` (§6) ;
   - reprise + progression + épisode suivant (§8).
4. **CSP** : autoriser `zebi.senpai-stream.club` (connect+media), gstatic, sprite Plyr local (§10).
5. **Build** : `npm run build` dans `web/` → `public/` (servi par Express). Dev : `npm run web:dev`
   (proxy `/api` → `:3020`).

### Vérifs rapides
- Un titre jouable (ex. **Dragon Ball Super Super Hero**, série **« Que ça vous serve de leçon »**)
  démarre → hls.js `MANIFEST_PARSED` → Plyr monté, qualités listées.
- Couper le HLS (mauvais token) → bascule auto CineFuse (iframe) sans écran noir.
- Console **sans** erreur `connect-src`/`media-src` (sinon CSP §10).
- Icônes Plyr présentes (sinon sprite `plyr.svg` manquant / CSP).

---

## 12. Dépannage (front)

| Symptôme | Cause probable | Action |
|----------|----------------|--------|
| 401 → redirige vers la fiche | pas connecté / cookie non envoyé | vérifier `credentials:'same-origin'` (§1) |
| Écran noir, erreurs `media-src`/`connect-src` | domaine CDN zebi absent de la CSP | mettre à jour la CSP (§10) |
| Icônes du lecteur invisibles | sprite Plyr chargé depuis le CDN (bloqué CSP) | servir `plyr.svg` en local (§5/§10) |
| Double son / fuite après changement d'épisode | cleanup manquant | `plyr.destroy()` + `hls.destroy()` dans le `return` (§5) |
| Coupures fréquentes sur host lent | buffer/timeouts trop courts | garder les réglages hls.js (§5) — ne PAS baisser `fragLoadingTimeOut` |
| Iframe CineFuse : pas de reprise ni Cast | `kind:'iframe'` = pas de `<video>` | normal (limite CineFuse) |
| Video/audio qui ne reprend pas à la bonne position | seek avant `loadedmetadata` | seeker après l'event `loadedmetadata` (§8) |

---

*Pendant front de `PURSTREAM_SCRAPER.md`. Règle d'or : le navigateur ne voit QUE Seenzo — toute URL
de flux passe par `/api/proxy/...`. La résolution purstream/zebi est 100% serveur.*
</content>
</invoke>
