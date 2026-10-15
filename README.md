# foodc-caja

Pantalla de caja (cashier screen) de FoodC: lista de pedidos `Completado` pendientes de pago, acción "Pagada" y cierre de día ("cierre de día"), desde una tablet, mediante sondeo (polling) al mismo backend que usan el comensal y la cocina. Sitio estático puro (sin framework, sin bundler), publicado por GitHub Pages.

El backend (Google Apps Script) vive en el repositorio privado de FoodC (la [FoodC backend repository](https://github.com/MelvinRamirez/FoodC)). Este repositorio no duplica ningún archivo de ese backend; solo consume sus endpoints HTTP documentados en sus specs (`caja-pin-api`, `order-payment-status`, `day-close-archival`, `api-transport`).

## Desarrollo local

Requiere Node.js >= 22.

```bash
npm test
```

## Tailwind CSS

El CSS se genera una vez con el CLI standalone de Tailwind v4 y se commitea (no hay build en el deploy). Binario fijado para este proyecto:

- Versión: **v4.3.3**
- Binario: `tailwindcss-macos-x64` (ajustar a la arquitectura de quien regenere el CSS)
- Descarga:

```bash
mkdir -p bin
curl -sSLo bin/tailwindcss https://github.com/tailwindlabs/tailwindcss/releases/download/v4.3.3/tailwindcss-macos-x64
chmod +x bin/tailwindcss
```

Regenerar el CSS tras cambiar clases en `index.html` o `js/`:

```bash
npm run build:css
```

## Estado actual

Este repositorio está en construcción, en orden (mismo patrón que `foodc-kds`):

- **K1 (completo)**: scaffold y módulos puros de dominio (`store`, `api`, `pin`, `time`, `config`, `messages`, `queue`, `payment`, `close`, `actions`, `persistence`), sin DOM, sin service worker, sin nada desplegado.
- **K2 (pendiente)**: `js/effects.js` — polling, runner de marcado Pagada, orquestación del cierre de día.
- **K3 (pendiente)**: vistas, `index.html`, Tailwind, manifest/service worker instalable, deploy a GitHub Pages.

El backend necesario (`verifyCajaPin`, `getCajaOrders`, `markOrderPaid`, `closeDay`) ya está implementado y probado en `FoodC` (trabajo B1-B3 de `openspec/changes/foodc-caja`), pero **todavía no desplegado a producción** (Phase 8, acción del propietario) al momento de escribir este archivo. El `API_URL` en `js/config.js` queda como placeholder hasta entonces.

## Deploy

Pendiente (K3): URL publicada será `https://melvinramirez.github.io/foodc-caja/`, habilitada solo después de confirmación explícita del propietario para crear el repositorio público remoto y habilitar GitHub Pages (ver `openspec/changes/foodc-caja/tasks.md` tarea 16.1 en `FoodC`).
