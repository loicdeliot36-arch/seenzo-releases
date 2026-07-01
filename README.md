# Seenzo — Scraper purstream (flux vidéo PRINCIPAL)

> **But de ce document** : permettre à une IA (ou un dev) de **ré-intégrer entièrement** le scraper
> purstream dans Seenzo **de zéro**, jusqu'au panel admin (changement de lien) et au rafraîchissement
> automatique toutes les 6 h. Si tu perds tout, donne ce fichier : il contient le contrat de l'API,
> le code des services, l'intégration lecture, l'admin et le job planifié.

---

## 0. Vue d'ensemble

purstream est la **source vidéo principale** de Seenzo (fallback : nakastream, puis CineFuse).
Deux domaines distincts, à ne pas confondre :

| Rôle | Domaine (exemple) | Qui le change | Stocké où |
|------|-------------------|---------------|-----------|
| **API de résolution** | `api.purstream.mx` | l'admin, **à la main** (rotation rare) | `config.purstream_base_url` (DB) + cache Redis 5 min |
| **CDN des flux `.m3u8`** | `zebi.senpai-stream.club` | **rotate/tombe souvent** → rattrapé par le **job 6 h** | renvoyé par l'API, jamais stocké en dur |

Chaîne de résolution d'un média Seenzo → flux jouable :

```
media (tmdbId, type, title, posterPath, releaseYear)
   └─ resolveInternalId()  → cherche sur api.purstream.mx  → id interne purstream  (cache Redis 7 j)
        └─ resolveStreamUrl() → api.purstream.mx/stream/{id} → stream_url sur zebi.senpai-stream.club
             └─ le player lit le .m3u8 EN DIRECT depuis zebi (CORS ouvert) → 0 bande passante VPS
```

**Pourquoi un scraper 6 h ?** À l'ajout d'un média, on résout son lien purstream. Mais les liens
(surtout le domaine CDN zebi) changent/tombent. Le job refait la résolution **pour tout le catalogue**
toutes les 6 h → rattrape les liens morts et rend disponibles les titres nouvellement présents.

---

## 1. Contrat de l'API purstream (reverse-engineering)

Base : `https://api.purstream.mx` (rotative, voir §3). En-têtes requis sur chaque requête :

```
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36
Referer:    https://purstream.mx/
Accept:     application/json
```

### 1.1 Recherche (tmdb → id interne)
```
GET /api/v1/search-bar/search/{query}
→ { data: { items: { movies: { items: [ {
      id, type: 'movie'|'tv', title, release_date,
      large_poster_path, small_poster_path   // chemins TMDB "/abc.jpg"
   } ] } } } }
```
- **Films ET séries** sont dans le bucket `movies`, distingués par le champ `type`
  (`movie` / `tv`).
- Le **poster** renvoyé est un chemin TMDB → match EXACT possible sur `media.posterPath`.

### 1.2 Résolution du flux (id interne → URL zebi)
```
GET /api/v1/stream/{internalId}                          (film)
GET /api/v1/stream/{internalId}/episode?season=&episode= (épisode de série)
→ { data: { items: { sources: [ { stream_url, source_name, format } ] } } }
```
- On retient la source **PREMIUM** (1080p+) : `source_name` contient `premium` ou `stream_url`
  contient `/premium-`. À défaut, la 1ʳᵉ source.
- `stream_url` a la forme :
  - film  : `https://zebi.senpai-stream.club/movie/{tmdb}/premium-{tok}/master.m3u8`
  - série : `https://zebi.senpai-stream.club/tv/{tmdb}/S{n}/E{m}/premium-{tok}/master.m3u8`
- Les slashs peuvent être échappés (`\/`) dans le JSON → `JSON.parse` nettoie.

---

## 2. Le service `src/services/purestream.js`

Résout un média Seenzo vers son URL HLS. **Cache** : domaine 5 min, mapping tmdb→id 7 j,
absence (négatif) 1 h.

Fonctions exportées :
- `getBaseUrl()` → lit `config.purstream_base_url` (fallback `https://api.purstream.mx`), cache Redis 5 min.
- `invalidateBaseUrlCache()` → supprime le cache du domaine (appelé quand l'admin change le lien).
- `resolveInternalId({ tmdbId, type, title, posterPath, year })` → id interne purstream (cache Redis).
  Matching : 1) poster TMDB EXACT, 2) titre normalisé + année, 3) titre normalisé.
- `resolveStreamUrl({ internalId, type, season, episode })` → `{ streamUrl, quality }`. Lève
  `{ code: 'NO_SOURCE' }` si pas de source.
- `resolveStream({ media, season, episode })` → enchaîne les deux ci-dessus.

Constantes clés :
```js
const DEFAULT_BASE = 'https://api.purstream.mx';
const URL_TTL = 300;          // cache domaine 5 min
const MAP_TTL = 86400 * 7;    // mapping tmdb → id interne : 7 j
const NEG_TTL = 3600;         // absence mémorisée 1 h
```

> Le fichier complet vit déjà dans le repo : `src/services/purestream.js`. S'il manque, le
> reconstruire depuis le contrat §1 + les signatures ci-dessus.

---

## 3. Clés Redis / config (`src/config/redis.js`)

```js
export const KEYS = {
  // ...
  purstreamBaseUrl: () => 'config:purstream_base_url',      // cache du domaine API
  purstreamMap: (tmdbId, type) => `pure:map:${type}:${tmdbId}`, // tmdbId+type → id interne
};
```
Le domaine vit aussi en base : table `config`, clé `purstream_base_url` (via
`getConfig`/`setConfig` de `src/services/config.js`).

---

## 4. Intégration LECTURE (`src/routes/play.js` + `src/routes/proxy.js`)

- **`play.js`** : flux principal = purstream. Appelle `resolveStream({ media, season, episode })`.
  En cas d'échec (`NO_SOURCE`), **fallback** nakastream puis CineFuse.
- **`proxy.js`** : le flux zebi est **CORS-ouvert** → le proxy fait une simple **redirection 302**
  vers l'URL zebi (le player lit ensuite tout en direct depuis zebi). ⇒ **0 bande passante VPS**
  sur la vidéo (contrairement à CineFuse qui est proxifié octet par octet).
- Le token de lecture (playToken) encapsule l'URL résolue ; voir `src/services/playToken.js`.

---

## 5. SÉCURITÉ / CSP (`src/middleware/security.js`)

Le domaine CDN doit être autorisé côté navigateur, sinon le player est bloqué par la CSP :
```js
connectSrc: ["'self'", 'https://www.gstatic.com', 'https://challenges.cloudflare.com', 'https://zebi.senpai-stream.club'],
mediaSrc:   ["'self'", 'blob:', 'https://cinefuse.cc', 'https://cdn.plyr.io', 'https://zebi.senpai-stream.club'],
```
> ⚠️ Si le domaine CDN change (ex : `zebi.senpai-stream.club` → autre), **mettre à jour la CSP**
> ici (sinon écran noir / erreurs `connect-src`/`media-src` en console).

---

## 6. ADMIN — changer le lien de l'API (`src/routes/admin.js`)

Routes derrière `requireAdmin` + `requireAdminSession` (session OTP admin) :
```js
// Lire le domaine courant
adminRouter.get('/purstream', asyncHandler(async (_req, res) => {
  res.json({ url: await getConfig('purstream_base_url', 'https://api.purstream.mx') });
}));

// Changer le domaine (rotation API)
adminRouter.post('/purstream', /* validate({ url }) */ asyncHandler(async (req, res) => {
  await setConfig('purstream_base_url', req.body.url);
  await invalidatePurstreamCache();   // import { invalidateBaseUrlCache as invalidatePurstreamCache }
  res.json({ ok: true });
}));
```
Côté front admin : un champ « URL API purstream » + bouton Enregistrer qui POST `/api/admin/purstream`.

---

## 7. LE JOB 6 H — rafraîchir tout le catalogue (`src/jobs/purstreamRefresh.js`)

Refait la résolution purstream (comme à l'ajout) pour **chaque média du catalogue**, en forçant
la re-résolution (purge du cache `purstreamMap` puis recherche), throttlé à 300 ms/titre.

```js
import { prisma } from '../config/prisma.js';
import { redis, KEYS } from '../config/redis.js';
import { resolveInternalId } from '../services/purestream.js';

const THROTTLE_MS = 300;
const ptype = (t) => (t === 'SERIES' ? 'tv' : 'movie');
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
let running = false;

export async function refreshCatalogPurstream() {
  if (running) return { skipped: true };
  running = true;
  let ok = 0, miss = 0;
  try {
    const medias = await prisma.media.findMany({
      select: { tmdbId: true, type: true, title: true, posterPath: true, releaseYear: true },
    });
    for (const m of medias) {
      try {
        await redis.del(KEYS.purstreamMap(m.tmdbId, ptype(m.type)));   // force la re-résolution
        const id = await resolveInternalId({
          tmdbId: m.tmdbId, type: m.type, title: m.title,
          posterPath: m.posterPath, year: m.releaseYear,
        });
        id ? ok++ : miss++;
      } catch { miss++; }
      await sleep(THROTTLE_MS);
    }
    console.log(`[purstream] refresh catalogue: ${ok} OK / ${miss} KO sur ${medias.length}`);
    await redis.set('telemetry:purstream:lastrun',
      JSON.stringify({ total: medias.length, ok, miss, at: Date.now() }), 'EX', 86400).catch(() => {});
    return { total: medias.length, ok, miss };
  } finally { running = false; }
}
```

### Câblage dans le scheduler (`src/jobs/scheduler.js`)
```js
import { refreshCatalogPurstream } from './purstreamRefresh.js';

const PURSTREAM_INTERVAL = 6 * 3600_000; // 6 h

export function startScheduler() {
  // ... (tick, warmHomeCache, imgWarm)
  const purstreamBoot  = setTimeout(() => { refreshCatalogPurstream().catch((e) => console.error('[purstream]', e.message)); }, 30_000);
  const purstreamTimer = setInterval(() => { refreshCatalogPurstream().catch((e) => console.error('[purstream]', e.message)); }, PURSTREAM_INTERVAL);
  return () => {
    // ... clearTimeout/clearInterval des autres
    clearTimeout(purstreamBoot);
    clearInterval(purstreamTimer);
  };
}
```
`startScheduler()` est appelé au démarrage du serveur (`src/server.js`).

---

## 8. RECETTE — intégrer de zéro (checklist IA)

1. **Service** : créer `src/services/purestream.js` depuis le contrat §1 + signatures §2.
2. **Redis** : ajouter `purstreamBaseUrl` et `purstreamMap` dans `KEYS` (`src/config/redis.js`, §3).
3. **Config** : s'assurer que `getConfig`/`setConfig` existent (`src/services/config.js`).
4. **Lecture** : dans `play.js`, brancher `resolveStream()` comme flux principal (fallback nakastream/CineFuse) ; dans `proxy.js`, **rediriger 302** vers l'URL zebi (§4).
5. **CSP** : autoriser le domaine CDN dans `connectSrc` + `mediaSrc` (`src/middleware/security.js`, §5).
6. **Admin** : ajouter `GET`/`POST /api/admin/purstream` (lire/écrire `config.purstream_base_url` + invalider le cache) et le champ correspondant au front admin (§6).
7. **Job 6 h** : créer `src/jobs/purstreamRefresh.js` (§7) et le câbler dans `scheduler.js` (boot différé + intervalle 6 h).
8. **Redémarrer** le serveur (`pm2 restart seenzo`, `PM2_HOME=/home/seenzo/.pm2`).

### Vérifs rapides
```bash
# Résolution d'un titre (doit renvoyer un id interne)
node -e "import('./src/services/purestream.js').then(m=>m.resolveInternalId({tmdbId:920,type:'MOVIE',title:'Cars',posterPath:null,year:2006}).then(x=>console.log(x)))"

# Dernière passe du job 6 h (clé Redis)
redis-cli GET telemetry:purstream:lastrun
```

---

## 9. Dépannage

| Symptôme | Cause probable | Action |
|----------|----------------|--------|
| Tous les titres en `MISS` | API purstream down ou domaine changé | changer `config.purstream_base_url` en admin (§6) |
| Écran noir / erreurs `media-src` en console | domaine CDN zebi changé | mettre à jour la CSP (§5) |
| Lecture qui coupe | source non premium / lien zebi expiré | relancer le job (§7) ou re-résoudre le titre |
| Job qui ne tourne pas | scheduler non câblé | vérifier `scheduler.js` (§7) + `startScheduler()` dans `server.js` |

---

*Généré pour Seenzo — flux principal purstream. Domaines à surveiller : `api.purstream.mx` (API,
changé à la main) et `zebi.senpai-stream.club` (CDN, rattrapé par le job 6 h + CSP).*
