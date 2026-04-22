# WebODM Technical Documentation

> **Scope:** This document covers two topics:
> 1. How authentication works in WebODM and how to replace it with AWS Cognito
> 2. How the dynamic Plant Health / multispectral map viewer works end-to-end

---

## Table of Contents

- [Part 1: Authentication](#part-1-authentication)
  - [1.1 Authentication Architecture Overview](#11-authentication-architecture-overview)
  - [1.2 Session-Based Login Flow](#12-session-based-login-flow)
  - [1.3 JWT / API Token Flow](#13-jwt--api-token-flow)
  - [1.4 OIDC / OpenID Connect (SSO) Flow](#14-oidc--openid-connect-sso-flow)
  - [1.5 External Auth Backend](#15-external-auth-backend)
  - [1.6 User Model & Permissions](#16-user-model--permissions)
  - [1.7 Replacing Authentication with AWS Cognito](#17-replacing-authentication-with-aws-cognito)
- [Part 2: Dynamic Map & Plant Health Viewer](#part-2-dynamic-map--plant-health-viewer)
  - [2.1 Architecture Overview](#21-architecture-overview)
  - [2.2 Frontend Map Components](#22-frontend-map-components)
  - [2.3 Plant Health UI Controls](#23-plant-health-ui-controls)
  - [2.4 Tile Request & Re-render Cycle](#24-tile-request--re-render-cycle)
  - [2.5 Backend Tile Server (tiler.py)](#25-backend-tile-server-tilerpy)
  - [2.6 Vegetation Index Formulas](#26-vegetation-index-formulas)
  - [2.7 Cloud Optimized GeoTIFF (COG)](#27-cloud-optimized-geotiff-cog)
  - [2.8 Color Maps](#28-color-maps)
  - [2.9 GeoTIFF Export](#29-geotiff-export)
  - [2.10 Library Stack Summary](#210-library-stack-summary)
  - [2.11 Building Your Own Dynamic Map App](#211-building-your-own-dynamic-map-app)

---

# Part 1: Authentication

## 1.1 Authentication Architecture Overview

WebODM uses a **layered, multi-mechanism authentication system** built on top of Django. It supports four distinct authentication paths that can coexist simultaneously:

| Mechanism | Use Case | Key Library |
|---|---|---|
| Django Session Auth | Browser login (username/password form) | `django.contrib.auth` |
| JWT Token Auth | API clients, mobile apps | `djangorestframework-jwt` |
| OIDC / OpenID Connect | Enterprise SSO (any OIDC-compliant provider) | Custom (`app/views/oidc.py`) |
| External Auth Backend | Federated or proxy auth via external HTTP endpoint | Custom (`app/auth/backends.py`) |

All four share the same Django `User` model and `Profile` extension. The `django-guardian` library adds per-object permissions (e.g., per-project, per-task access control).

### Key Source Files

| File | Purpose |
|---|---|
| `webodm/settings.py` | All auth configuration (backends, JWT settings, OIDC providers) |
| `app/auth/backends.py` | External auth backend implementation |
| `app/views/oidc.py` | OIDC login initiation and callback handling |
| `app/api/authentication.py` | Custom JWT-from-query-string authentication class |
| `app/api/externalauth.py` | External token exchange API view |
| `app/api/urls.py` | API URL routing incl. `/api/token-auth/` |
| `webodm/urls.py` | Root URL routing incl. Django auth URLs |
| `app/models/profile.py` | Extended user profile (quota, OIDC subject) |

---

## 1.2 Session-Based Login Flow

This is the standard browser login path.

```
Browser                     Django
  |                            |
  |  GET /login/               |
  |--------------------------->|
  |  <HTML login form          |
  |<---------------------------|
  |                            |
  |  POST /login/              |
  |  {username, password}      |
  |--------------------------->|
  |          AuthenticationMiddleware
  |          calls AUTHENTICATION_BACKENDS in order:
  |            1. ModelBackend (checks DB)
  |            2. ExternalBackend (calls EXTERNAL_AUTH_ENDPOINT if set)
  |                            |
  |  Set-Cookie: sessionid=... |
  |  302 -> /dashboard/        |
  |<---------------------------|
```

### Settings (`webodm/settings.py`)
```python
AUTHENTICATION_BACKENDS = (
    'django.contrib.auth.backends.ModelBackend',
    'guardian.backends.ObjectPermissionBackend',
    'app.auth.backends.ExternalBackend',
)

LOGIN_REDIRECT_URL = '/dashboard/'
LOGIN_URL = '/login/'

SESSION_COOKIE_SAMESITE = None   # Required for cross-origin cookie use
CORS_ORIGIN_ALLOW_ALL = True
CORS_ALLOW_CREDENTIALS = True
```

### Middleware Stack
```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',         # CORS headers
    'django.contrib.sessions.middleware.SessionMiddleware',   # Session cookie
    'django.middleware.csrf.CsrfViewMiddleware',     # CSRF tokens
    'django.contrib.auth.middleware.AuthenticationMiddleware', # request.user
    ...
]
```

### Special Modes
- **`SINGLE_USER_MODE = True`**: Auto-creates an admin user and logs them in without any form
- **`AUTO_LOGIN_USER = 'username'`**: Auto-logs in a named user on every request (demo/kiosk use)

---

## 1.3 JWT / API Token Flow

For programmatic API access (scripts, mobile apps, integrations).

```
Client                      Django API
  |                            |
  |  POST /api/token-auth/     |
  |  {username, password}      |
  |--------------------------->|
  |  {token: "eyJ..."}         |
  |<---------------------------|
  |                            |
  |  GET /api/projects/        |
  |  Authorization: JWT eyJ... |
  |--------------------------->|
  |  [projects JSON]           |
  |<---------------------------|
```

Tokens can also be passed as a query string parameter:
```
GET /api/projects/?jwt=eyJ...
```

This is handled by the custom `JSONWebTokenAuthenticationQS` class:

```python
# app/api/authentication.py
class JSONWebTokenAuthenticationQS(BaseJSONWebTokenAuthentication):
    def get_jwt_value(self, request):
        return request.query_params.get('jwt')
```

### JWT Settings
```python
# webodm/settings.py
JWT_AUTH = {
    'JWT_EXPIRATION_DELTA': datetime.timedelta(hours=6),
}

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'app.api.authentication.JSONWebTokenAuthenticationQS',
    ),
    ...
}
```

---

## 1.4 OIDC / OpenID Connect (SSO) Flow

WebODM has a built-in OIDC client implementation that works with any standard OIDC provider (Okta, Auth0, Keycloak, Azure AD, **and AWS Cognito**).

```
Browser                   WebODM                 OIDC Provider
  |                          |                         |
  |  GET /oidc/login/0/      |                         |
  |------------------------->|                         |
  |                          | generate state token    |
  |  302 -> provider?        |                         |
  |  client_id=...&state=... |                         |
  |<-------------------------|                         |
  |  GET /oauth/authorize    |                         |
  |----------------------------------------->|
  |  [user logs in at provider]              |
  |  302 -> /oidc/callback?code=...&state=...|
  |<-----------------------------------------|
  |  GET /oidc/callback?code=...             |
  |------------------------->|               |
  |                          | POST /token   |
  |                          |-------------->|
  |                          | {access_token}|
  |                          |<--------------|
  |                          | GET /userinfo |
  |                          |-------------->|
  |                          | {email, sub}  |
  |                          |<--------------|
  |                          | get_or_create User by oidc_sub
  |                          | login(request, user)
  |  302 -> /dashboard/      |
  |<-------------------------|
```

### OIDC Provider Configuration
```python
# webodm/settings.py
OIDC_AUTH_PROVIDERS = [
    {
        'name': 'My SSO Provider',
        'client_id': 'your-client-id',
        'client_secret': 'your-client-secret',
        'auth_endpoint': 'https://provider/oauth2/authorize',
        'token_endpoint': 'https://provider/oauth2/token',
        'userinfo_endpoint': 'https://provider/oauth2/userInfo',
        'icon': 'fa fa-lock'
    }
]

# Optional: Only allow specific emails or domains
OIDC_AUTH_EMAILS = ['user@example.com', '@mycompany.com']
```

### How Users Are Matched
The OIDC `sub` (subject) claim is stored in `Profile.oidc_sub`. On each login, the system looks up the user by this field. If not found, a new user is created with `username = email`.

---

## 1.5 External Auth Backend

A simple HTTP-based delegation mechanism. When a user logs in, WebODM calls an external HTTP endpoint with the credentials. If that endpoint returns a valid user response, login succeeds.

```python
# webodm/settings.py
EXTERNAL_AUTH_ENDPOINT = 'https://your-auth-service/verify'
```

Expected response format from your external endpoint:
```json
{
    "user_id": 42,
    "username": "jsmith",
    "maxQuota": 5000.0,
    "cluster_id": 1,
    "node": {
        "hostname": "node.example.com",
        "port": 3000,
        "token": "abc123"
    }
}
```

---

## 1.6 User Model & Permissions

### Core User: `django.contrib.auth.models.User`
Standard Django user with username, password (hashed), email, is_staff, is_superuser.

### Extended Profile: `app/models/profile.py`
```python
class Profile(models.Model):
    user      = OneToOneField(User)
    quota     = FloatField(default=-1)        # Disk quota in MB (-1 = unlimited)
    cluster_id = IntegerField(null=True)       # Multi-node cluster assignment
    oidc_sub  = CharField(max_length=255)     # OIDC Subject (SSO identifier)
```

### Object-Level Permissions (`django-guardian`)
Every `Project` and `Task` is protected by per-object permissions, not just model-level permissions. This means user A can see project X but not project Y — managed through the `assign_perm` / `get_perms` API.

---

## 1.7 Replacing Authentication with AWS Cognito

AWS Cognito is a fully OIDC-compliant identity provider. WebODM already has a working OIDC client, so **the integration requires only configuration** — no code changes needed for the core SSO flow.

### Option A: OIDC Integration (Recommended — No Code Changes)

This replaces the login button with a Cognito-hosted UI. Users are redirected to Cognito to authenticate, then redirected back.

**Step 1: Set up a Cognito User Pool App Client**
- Create a User Pool in AWS Console
- Add an App Client with:
  - Allowed OAuth Flows: `Authorization code grant`
  - Allowed OAuth Scopes: `openid`, `email`, `profile`
  - Callback URL: `https://your-webodm-domain/oidc/callback/`
  - Sign-out URL: `https://your-webodm-domain/logout/`

**Step 2: Configure WebODM settings**
```python
# webodm/settings.py (or override in webodm/settings_local.py)

OIDC_AUTH_PROVIDERS = [
    {
        'name': 'AWS Cognito',
        'client_id': 'YOUR_COGNITO_APP_CLIENT_ID',
        'client_secret': 'YOUR_COGNITO_APP_CLIENT_SECRET',
        # Replace with your actual User Pool domain and region
        'auth_endpoint':     'https://YOUR_DOMAIN.auth.REGION.amazoncognito.com/oauth2/authorize',
        'token_endpoint':    'https://YOUR_DOMAIN.auth.REGION.amazoncognito.com/oauth2/token',
        'userinfo_endpoint': 'https://YOUR_DOMAIN.auth.REGION.amazoncognito.com/oauth2/userInfo',
        'icon': 'fa fa-amazon'
    }
]

# Optional: restrict to specific email domains from your Cognito pool
OIDC_AUTH_EMAILS = ['@yourcompany.com']
```

**Step 3: Disable or hide local login (optional)**
To force all users through Cognito, you can disable the username/password form by removing `ModelBackend` from `AUTHENTICATION_BACKENDS` and setting a login template that only shows the OIDC button.

---

### Option B: External Auth Backend (More Control)

Use this if you need to validate Cognito tokens server-side before granting access, or if you want to synchronize user attributes (groups, quotas) from Cognito on every login.

**Architecture:**
```
WebODM Login Form
      |
      | POST {username, password}
      v
ExternalBackend.authenticate()
      |
      | POST {username, password}
      v
Your custom service (e.g., Lambda or Express)
      |
      | Calls Cognito InitiateAuth API
      v
AWS Cognito
      |
      | Returns tokens / error
      v
Your service returns {user_id, username, maxQuota}
      |
      v
WebODM creates/updates local User and logs in
```

```python
EXTERNAL_AUTH_ENDPOINT = 'https://your-lambda-or-service/cognito-verify'
```

Your endpoint receives a POST with `username` and `password`, calls the Cognito `InitiateAuth` API using boto3, and returns the standard external auth response JSON.

---

### Option C: Replace JWT with Cognito Tokens (Advanced)

For fully Cognito-native API authentication (useful if you have other services also consuming Cognito tokens):

1. **Install** `django-cognito-jwt` or `python-jose`
2. **Write a custom DRF authentication class** that:
   - Reads the `Authorization: Bearer <CognitoIdToken>` header
   - Verifies the JWT signature against Cognito's JWKS endpoint:
     `https://cognito-idp.REGION.amazonaws.com/POOL_ID/.well-known/jwks.json`
   - Extracts the `sub` claim and looks up (or creates) the local user
3. **Register it** in `REST_FRAMEWORK['DEFAULT_AUTHENTICATION_CLASSES']`

```python
# Example skeleton
import requests
from jose import jwt
from rest_framework.authentication import BaseAuthentication

COGNITO_JWKS_URL = 'https://cognito-idp.REGION.amazonaws.com/POOL_ID/.well-known/jwks.json'

class CognitoJWTAuthentication(BaseAuthentication):
    def authenticate(self, request):
        header = request.headers.get('Authorization', '')
        if not header.startswith('Bearer '):
            return None
        token = header[7:]
        jwks = requests.get(COGNITO_JWKS_URL).json()
        try:
            claims = jwt.decode(token, jwks, algorithms=['RS256'],
                                audience='YOUR_APP_CLIENT_ID')
        except Exception:
            return None
        sub = claims['sub']
        email = claims.get('email', sub)
        user, _ = User.objects.get_or_create(
            profile__oidc_sub=sub,
            defaults={'username': email}
        )
        return (user, token)
```

---

### Summary: Cognito Integration Paths

| Option | Effort | User Experience | Best For |
|---|---|---|---|
| A: OIDC (built-in) | Low — config only | Redirect to Cognito hosted UI | Quickest path; standard SSO |
| B: External Backend | Medium — build a proxy service | Keeps WebODM login form | Need custom attribute sync |
| C: Custom JWT Auth | High — write DRF auth class | API-only, no browser UI change | Microservice / headless API |

---

# Part 2: Dynamic Map & Plant Health Viewer

## 2.1 Architecture Overview

The Plant Health viewer is a **dynamic tile-based map** where every map tile (256x256 px PNG) is generated on-the-fly by the server using the current user-selected parameters (formula, bands, color map, min/max rescale). Changing any setting in the UI causes Leaflet to discard cached tiles and re-request all visible tiles with the new parameters.

```
 ┌─────────────────────────────────────────┐
 │  Browser (Leaflet + React)              │
 │                                         │
 │  LayersControlLayer.jsx                 │
 │   [Formula ▼] [Bands ▼] [Color ▼]      │
 │   [Min ___] [Max ___]                   │
 │                                         │
 │  On change → updateLayer()              │
 │    layer.setUrl(newUrl)                 │
 │    layer._removeAllTiles()              │
 │    layer.redraw()                       │
 └────────────┬────────────────────────────┘
              │ GET /api/projects/1/tasks/2/orthophoto/tiles/14/3241/6891.png
              │   ?formula=NDVI&bands=RGN&color_map=rdylgn&rescale=-1,1
              ▼
 ┌─────────────────────────────────────────┐
 │  Django REST API (tiler.py)             │
 │                                         │
 │  1. Open Cloud Optimized GeoTIFF        │
 │  2. Extract tile x/y/z window           │
 │  3. Apply formula (NDVI expression)     │
 │  4. Rescale to [-1, 1]                  │
 │  5. Apply color map (RdYlGn palette)    │
 │  6. Render → PNG bytes                  │
 └────────────┬────────────────────────────┘
              │
              ▼
 ┌─────────────────────────────────────────┐
 │  Storage                                │
 │  Cloud Optimized GeoTIFF                │
 │  (GDAL 3.1 COG, 256px blocks,           │
 │   deflate compression, overviews)       │
 └─────────────────────────────────────────┘
```

---

## 2.2 Frontend Map Components

### Key Files

| File | Role |
|---|---|
| `app/static/app/js/MapView.jsx` | Root component; selects best tile type (orthophoto/plant/dsm/dtm) |
| `app/static/app/js/components/Map.jsx` | Main Leaflet map; loads tile layers, fetches tile.json metadata |
| `app/static/app/js/components/LayersControlLayer.jsx` | Plant Health / layer controls UI panel |
| `app/static/app/js/components/ColorMaps.js` | Color map decoding helpers |
| `app/static/app/js/Utils.js` | URL builder and query string helpers |

### MapView.jsx — Tile Type Selection
```javascript
// Automatically prefers 'plant' layer if the dataset has thermal/multispectral data
let preferredTypes = ['orthophoto', 'dsm', 'dtm'];
if (this.isThermalMap())
    preferredTypes = ['plant'].concat(preferredTypes);
```

### Map.jsx — Layer Loading Flow
1. Calls `/api/projects/{pid}/tasks/{tid}/{tileType}/metadata` to get bounds, zoom levels, band statistics, available algorithms, and encoded color maps
2. Computes default NDVI formula:
   ```javascript
   // Auto-detects NIR band availability
   let formula = this.hasBands(["red", "green", "nir"], meta.task.orthophoto_bands)
       ? "NDVI" : "VARI";
   ```
3. Creates `L.tileLayer(tileUrl, options)` with the tile endpoint URL
4. Passes layer reference and metadata down to `LayersControlLayer`

---

## 2.3 Plant Health UI Controls

**Component:** `LayersControlLayer.jsx`

The UI panel renders these controls:

| Control | What it does |
|---|---|
| **Algorithm** dropdown | Selects the vegetation index formula (NDVI, GNDVI, VARI, etc.) |
| **Bands** dropdown | Maps camera bands to formula variables (RGN, NRG, Auto, etc.) |
| **Color** dropdown | Sets the rendering color palette (Viridis, RdYlGn, etc.) |
| **Min / Max** sliders | Sets the rescale range for the index output |
| **Histogram** display | Shows band value distribution to help choose min/max |
| **Export GeoTiff (RGB)** button | Downloads the processed raster as a GeoTIFF |

### Update Flow on User Interaction
```javascript
// LayersControlLayer.jsx — simplified
handleSelectFormula(formula) {
    // Validate that camera has the required bands for this formula
    if (!this.formulaBandsAreValid(formula, this.state.bands)) {
        // Switch bands to 'auto' if current selection is invalid
        this.setState({ bands: 'auto' });
    }
    this.setState({ formula }, this.updateLayer);
}

updateLayer() {
    clearTimeout(this.updateTimer);
    this.updateTimer = setTimeout(() => {
        const newUrl = Utils.buildUrlWithQuery(this.props.url, {
            formula:   this.state.formula,    // e.g. "NDVI"
            bands:     this.state.bands,      // e.g. "RGN"
            color_map: this.state.colorMap,   // e.g. "rdylgn"
            rescale:   `${this.state.min},${this.state.max}`  // e.g. "-1,1"
        });
        this.props.layer.setUrl(newUrl, true);
        this.props.layer._removeAllTiles();
        setTimeout(() => this.props.layer.redraw(), 1);
    }, 200);  // Debounced 200ms to avoid rapid-fire requests
}
```

---

## 2.4 Tile Request & Re-render Cycle

Every time a setting changes, this exact sequence occurs:

```
1. User picks new formula/color/bands in UI
         ↓
2. handleSelectX() sets React state
         ↓
3. updateLayer() is called (debounced 200ms)
         ↓
4. New tile URL is built:
   /api/.../orthophoto/tiles/{z}/{x}/{y}.png
     ?formula=NDVI
     &bands=RGN
     &color_map=rdylgn
     &rescale=-1,1
         ↓
5. layer.setUrl(newUrl)     — Leaflet updates URL template
6. layer._removeAllTiles()  — Leaflet empties tile cache
7. layer.redraw()           — Leaflet re-requests all visible tiles
         ↓
8. Browser fires N tile requests (one per visible 256x256 grid cell)
         ↓
9. Server renders each tile (see Section 2.5)
         ↓
10. Tiles appear on map
```

---

## 2.5 Backend Tile Server (tiler.py)

**File:** `app/api/tiler.py`

### Endpoints

| URL Pattern | Class | Purpose |
|---|---|---|
| `.../{type}/tiles.json` | `TileJson` | TileJSON 2.1 metadata (bounds, zoom range) |
| `.../{type}/metadata` | `Metadata` | Band stats, color maps, available algorithms |
| `.../{type}/tiles/{z}/{x}/{y}.{ext}` | `Tiles` | Actual tile image generation |
| `.../{type}/export` | `Export` | Full-resolution GeoTIFF download |

### Tile Rendering Pipeline (`Tiles` view)

```python
# Simplified from app/api/tiler.py

def get(self, request, pk, project_pk, tile_type, z, x, y):
    # 1. Resolve GeoTIFF file path
    raster_path = get_raster_path(project_pk, pk, tile_type)

    # 2. Parse query parameters
    formula   = request.query_params.get('formula')       # "NDVI"
    bands     = request.query_params.get('bands')         # "RGN"
    color_map = request.query_params.get('color_map')     # "rdylgn"
    rescale   = request.query_params.get('rescale')       # "-1,1"
    hillshade = request.query_params.get('hillshade')     # "6" (DSM only)
    tilesize  = int(request.query_params.get('size', 256))

    # 3. Resolve formula expression and band expression
    #    e.g. formula="NDVI", bands="RGN" →
    #         expr = "(R2 - R1) / (R2 + R1)"  (R1=Red, R2=NIR in RGN order)
    expr = lookup_formula(formula, band_order=bands)

    # 4. Open Cloud Optimized GeoTIFF and extract tile
    with COGReader(raster_path) as src:
        if expr:
            tile = src.tile(x, y, z,
                            expression=expr,          # numexpr expression
                            tilesize=tilesize,
                            resampling_method='nearest')
        else:
            tile = src.tile(x, y, z, tilesize=tilesize)

    # 5. Rescale to output range
    min_v, max_v = [float(v) for v in rescale.split(',')]
    img = tile.post_process(in_range=((min_v, max_v),))

    # 6. Apply color map
    if color_map:
        cmap = colormap.get(color_map)   # rio-tiler built-in or custom
        img_data, alpha = apply_cmap(img.data, cmap)
    else:
        img_data = img.data

    # 7. Apply hillshading (elevation datasets only)
    if hillshade:
        intensity = ls.hillshade(elevation_data, vert_exag=float(hillshade))
        img_data = hsv_blend(img_data, intensity)

    # 8. Render to PNG and return
    return HttpResponse(
        render(img_data, alpha, img_format='PNG'),
        content_type='image/png'
    )
```

### Metadata Endpoint

The `/metadata` endpoint is called once when a layer loads. It returns:
- Per-band statistics (min, max, mean, 2nd/98th percentile)
- Available algorithms (filtered to those compatible with the camera's bands)
- All color maps with their RGBA values pre-encoded as 32-bit integers
- Histogram data for the UI sliders

---

## 2.6 Vegetation Index Formulas

**File:** `app/api/formulas.py`

Formulas are stored as symbolic expressions mapped to band variable names. The `lookup_formula()` function translates them into `numexpr`-compatible strings using the actual band order from the GeoTIFF.

### Selected Formulas (40+ total)

| Index | Full Name | Formula | Requires |
|---|---|---|---|
| NDVI | Normalized Difference Vegetation Index | `(N - R) / (N + R)` | NIR, Red |
| GNDVI | Green NDVI | `(N - G) / (N + G)` | NIR, Green |
| NDRE | Red Edge | `(N - Re) / (N + Re)` | NIR, RedEdge |
| VARI | Visual Atmospheric Resistance Index | `(G - R) / (G + R - B)` | RGB only |
| EVI | Enhanced Vegetation Index | `2.5 * (N - R) / (N + 6*R - 7.5*B + 1)` | NIR, Red, Blue |
| SAVI | Soil-Adjusted Vegetation Index | `1.5 * (N - R) / (N + R + 0.5)` | NIR, Red |
| OSAVI | Optimized SAVI | `(N - R) / (N + R + 0.16)` | NIR, Red |
| TGI | Triangular Greenness Index | `G - 0.39*R - 0.61*B` | RGB |
| RVI | Ratio Vegetation Index | `N / R` | NIR, Red |

### Band Order Filters

The camera's band configuration (stored in task metadata `orthophoto_bands`) determines which formulas are available. Supported configurations:

```
RGB       — Standard 3-band camera
RGN       — Red, Green, Near-Infrared (MicaSense RedEdge, Parrot Sequoia)
NRG       — NIR, Red, Green
RGBN      — 4-band with NIR
RGNRe     — 5-band with RedEdge
BGRNRe    — 5-band alternative ordering
RGBNRePL  — 6-band with Panchro + Luminance
L         — Single-band thermal (FLIR)
```

### Auto Band Detection
```python
def get_auto_bands(orthophoto_bands, formula):
    # Reads camera band metadata from the GeoTIFF
    # and picks the best band order for the requested formula
    # e.g., camera has [Blue, Green, Red, NIR] -> formula "NDVI"
    #       returns band order "BGRN" with R=band3, N=band4
```

---

## 2.7 Cloud Optimized GeoTIFF (COG)

**File:** `app/cogeo.py`

### What is a COG?

A Cloud Optimized GeoTIFF is a regular GeoTIFF that has been restructured so that data is organized internally in tiles (typically 256x256 pixels) with overview pyramids. This allows tile servers to read only the specific block needed for a given tile request — without loading the entire file.

### COG Creation in WebODM

After ODM processing completes, `assure_cogeo()` is called on the output GeoTIFF:

```bash
# Equivalent GDAL command (GDAL 3.1+)
gdal_translate \
    -of COG \
    -co BLOCKSIZE=256 \
    -co COMPRESS=deflate \
    -co OVERVIEW_RESAMPLING=average \
    input_odm_orthophoto.tif \
    output_cogeo.tif
```

### Why COG Matters for Tile Performance

| Without COG | With COG |
|---|---|
| Read entire file per tile | Read only the 256x256 block needed |
| Slow for large files (>1 GB) | Consistent speed regardless of file size |
| No zoom-level optimization | Overview pyramids provide fast low-zoom tiles |
| Cannot be served from S3 | Can be served directly from S3 via HTTP range requests |

### Tile Extraction (rio-tiler)
```python
with COGReader(path) as cog:
    tile = cog.tile(x, y, z, expression="(b4 - b1) / (b4 + b1)")
    # rio-tiler reads ONLY the COG blocks that overlap this tile
    # at the appropriate overview level for zoom z
```

---

## 2.8 Color Maps

### Available Color Maps

| Name | Description | Best For |
|---|---|---|
| `rdylgn` | Red → Yellow → Green | NDVI (dead → healthy vegetation) |
| `spectral` | Spectral rainbow | General vegetation indices |
| `viridis` | Blue → Green → Yellow | DSM elevation, generic data |
| `plasma` | Blue → Magenta → Yellow | High-contrast visualization |
| `inferno` | Black → Red → Yellow | Thermal, high-contrast |
| `magma` | Black → Purple → Yellow | Alternative to inferno |
| `jet` | Blue → Cyan → Yellow → Red | Legacy, not recommended |
| `terrain` | Green → Brown → White | DEM elevation |
| `discrete_ndvi` | 5-color discrete NDVI | Quick NDVI classification |
| `better_discrete_ndvi` | 20-color gradient NDVI | Detailed NDVI gradient |

### Color Map Encoding (for custom implementations)

The server encodes color maps as arrays of 32-bit integers for compact JSON transfer:

```python
# Server encodes: RGBA as a single integer
encoded = (R << 24) | (G << 16) | (B << 8) | A
```

```javascript
// Client decodes:
const r = (v >> 24) & 0xFF;
const g = (v >> 16) & 0xFF;
const b = (v >> 8)  & 0xFF;
const a =  v        & 0xFF;
```

---

## 2.9 GeoTIFF Export

**Endpoint:** `POST /api/projects/{pid}/tasks/{tid}/orthophoto/export`

### Supported Export Formats
| Format | Parameter | Notes |
|---|---|---|
| Raw GeoTIFF | `gtiff` | Original float data, preserves index values |
| RGB GeoTIFF | `gtiff-rgb` | Color-mapped 8-bit, visually matches map view |
| JPEG | `jpg` | Compressed, no geospatial metadata |
| PNG | `png` | Lossless, no geospatial metadata |
| KMZ | `kmz` | Google Earth overlay |

### Export Parameters (POST body)
```json
{
    "formula":    "NDVI",
    "bands":      "RGN",
    "color_map":  "rdylgn",
    "rescale":    "-1,1",
    "format":     "gtiff-rgb",
    "epsg":       4326
}
```

The export is processed asynchronously by a Celery worker task that applies the formula to the full-resolution raster (not just tiles) and returns a downloadable file.

---

## 2.10 Library Stack Summary

### Backend
| Library | Version | Role |
|---|---|---|
| `rio-tiler` | 2.1.2 (custom WebODM fork) | COG tile extraction, expression evaluation |
| `rasterio` | 1.3.10 | GeoTIFF I/O (GDAL Python wrapper) |
| `rio-cogeo` | — | COG validation and creation |
| `rio-color` | 1.0.4 | Color correction filters |
| `GDAL` | 3.1+ | Underlying raster engine, COG creation |
| `numexpr` | — | Fast evaluation of NDVI-style expressions on numpy arrays |
| `numpy` | — | Array math |
| `morecantile` | 3.2.0 | Tile coordinate math (TMS/XYZ) |

### Frontend
| Library | Version | Role |
|---|---|---|
| `leaflet` | 1.3.1 | Interactive map rendering, tile layer management |
| `react` | 16.4 | UI component framework |
| `rbush` | 3.0.1 | R-tree spatial indexing for map annotations |

---

## 2.11 Building Your Own Dynamic Map App

If you want to build a standalone dynamic multispectral map viewer (outside WebODM), here is the architecture you would replicate:

### Minimum Required Components

**1. Cloud Optimized GeoTIFF tile server**

The simplest modern option is [TiTiler](https://developmentseed.org/titiler/) — a FastAPI-based tile server built on top of the same `rio-tiler` library WebODM uses internally.

```bash
pip install titiler[full]
uvicorn titiler.application:app --port 8000
```

Key TiTiler endpoints (compatible with WebODM's approach):
```
GET /cog/tiles/{z}/{x}/{y}.png
    ?url=s3://bucket/orthophoto.tif
    &expression=(b4-b1)/(b4+b1)
    &rescale=-1,1
    &colormap_name=rdylgn

GET /cog/statistics?url=...    # Band statistics for min/max sliders
GET /cog/info?url=...          # Band count, CRS, bounds
```

**2. Frontend map (Leaflet or MapLibre)**

```javascript
// Leaflet example — same pattern WebODM uses
const formula = 'NDVI';
const bands = 'RGN';
const colorMap = 'rdylgn';
const rescale = '-1,1';

const tileUrl = `http://localhost:8000/cog/tiles/{z}/{x}/{y}.png`
    + `?url=${encodeURIComponent(geotiffUrl)}`
    + `&expression=(b4-b1)%2F(b4%2Bb1)`   // (b4-b1)/(b4+b1) = NDVI
    + `&rescale=${rescale}`
    + `&colormap_name=${colorMap}`;

const layer = L.tileLayer(tileUrl, { tileSize: 256 });
layer.addTo(map);

// When user changes formula/color:
function updateLayer(newFormula, newColorMap) {
    const newUrl = buildUrl(newFormula, newColorMap);
    layer.setUrl(newUrl, true);
    layer._removeAllTiles();
    layer.redraw();
}
```

**3. Vegetation index expressions**

Map human-readable formula names to `rio-tiler` band expressions:

```python
FORMULAS = {
    'NDVI':  '(b4-b1)/(b4+b1)',     # NIR=b4, Red=b1 in RGNB ordering
    'GNDVI': '(b4-b2)/(b4+b2)',     # NIR=b4, Green=b2
    'VARI':  '(b2-b1)/(b2+b1-b3)', # Green=b2, Red=b1, Blue=b3
    'EVI':   '2.5*(b4-b1)/(b4+6*b1-7.5*b3+1)',
}
```

Note: band numbering depends on the order bands appear in your GeoTIFF.

### Recommended Technology Choices

| Component | WebODM Choice | Modern Alternative |
|---|---|---|
| Tile server | Custom `tiler.py` + rio-tiler | TiTiler (ready-made, same rio-tiler core) |
| Map library | Leaflet 1.3 | MapLibre GL JS (WebGL, higher performance) |
| Color maps | Custom + rio-tiler built-ins | Same (rio-tiler colormaps) |
| COG creation | GDAL + rio-cogeo | Same — no good alternative |
| Band math | numexpr via rio-tiler | Same |
| Frontend framework | React 16 | React 18 or Vue 3 |

### Data Flow for a Custom App

```
Your GeoTIFF (from ODM or other source)
    ↓
gdal_translate -of COG (create COG)
    ↓
Upload to S3 or serve locally
    ↓
TiTiler reads COG, serves tiles
    ↓
Leaflet/MapLibre displays tiles
    ↓
User changes NDVI formula in your UI
    ↓
Frontend rebuilds tile URL with new expression parameter
    ↓
Leaflet refreshes tiles → new formula applied
```

### Key Insight: Everything is a URL Parameter

The entire power of this system comes from the tile URL being fully parameterized. There is no server-side state between tile requests. Each tile URL completely encodes what the output should look like:

```
/tiles/14/6241/7891.png
  ?expression=(b4-b1)/(b4+b1)   ← formula
  &rescale=-1,1                  ← normalization range
  &colormap_name=rdylgn          ← color palette
  &bidx=1,2,3,4                  ← which bands to load
```

Changing any of these parameters immediately changes what all new tiles look like — no server restart, no file regeneration.
