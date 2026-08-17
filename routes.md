# Curvi — Rotas API necessárias

Este documento lista as rotas HTTP e contratos (payloads/respostas) necessários para o frontend existente em `frontend/src`.

Observação sobre autenticação: recomenda-se usar cookies httpOnly (sessão) ou JWT no header `Authorization`. Rotas marcadas com `Auth` exigem autenticação e verificação de ownership quando aplicável.

---

## Autenticação

- POST /api/auth/signup
  - Body: `{ "email": "...", "password": "...", "username": "..." }`
  - Response 201: `{ "id": "...", "email": "...", "username": "...", "displayName": "..." }`
  - Nota: gravar `password_hash`; iniciar sessão (cookie) na resposta.

- POST /api/auth/signin
  - Body: `{ "email": "...", "password": "..." }`
  - Response 200: `{ "user": { ... }, "token": "..." }` ou set-cookie httpOnly
  - Erros: 401 para credenciais inválidas.

- POST /api/auth/signout
  - Auth required. Invalida sessão / remove cookie.
  - Response 204.

- POST /api/auth/reset-password
  - Body: `{ "email": "...", "newPassword": "..." }`
  - Response 200: `{ "ok": true }` (em produção: enviar e-mail com token/fluxo seguro)

---

## Usuários / Perfil

- GET /api/users/me
  - Auth required.
  - Response: usuário completo incluindo `links` e `categories`.

- GET /api/users/:id
  - Auth required (ou admin) — retorna perfil privado.

- PATCH /api/users/:id
  - Auth required (usuário dono)
  - Body: Partial das propriedades permitidas (displayName, bio, avatar_url, theme, button_color, background_color, background_image_url, categories)
  - Response 200: usuário atualizado

- GET /api/public/:username
  - Public route (sem auth)
  - Response: `{ username, displayName, bio, avatar_url, button_color, background_color, background_image_url, plan, links: [...] }`
  - Usado para renderizar `/$username` (página pública). Deve ser rápido e cacheável.

---

## Links (CRUD)

- GET /api/users/:userId/links
  - Auth required / owner
  - Response: `[{ id, title, url, active, clicks, category, position }]`

- POST /api/users/:userId/links
  - Auth required
  - Body: `{ title, url, category, active? }`
  - Response 201: novo link

- PATCH /api/users/:userId/links/:linkId
  - Auth required
  - Body: Partial do link (title, url, active, category, position)
  - Response 200: link atualizado

- DELETE /api/users/:userId/links/:linkId
  - Auth required
  - Response 204

- POST /api/users/:userId/links/reorder
  - Auth required
  - Body: `{ order: ["linkId1","linkId2", ...] }` — atualiza `position` em lote
  - Response 200

---

## Cliques / Métricas

- POST /api/:username/links/:linkId/click
  - Public route (chamada ao clicar em link na página pública). Pode ser chamada pelo frontend ou redirecionada via servidor.
  - Behavior: incrementa `links.clicks` atomically e opcionalmente grava em `link_clicks` para histórico.
  - Response 204 (ou redirect para `url`).

- GET /api/users/:userId/metrics
  - Auth required
  - Query params (opcional): `from`, `to`, `group=day|week|month`
  - Response: métricas agregadas para painel (totais, por link, por categoria)

---

## Avaliações (Reviews)

- GET /api/reviews
  - Public: lista avaliações recentes

- POST /api/reviews
  - Body: `{ name, rating, comment }`
  - Response 201: avaliação criada

---

## Checkout / Pagamentos

- POST /api/checkout/init
  - Auth required
  - Body: `{ userId, plan }`
  - Response: `{ checkoutId, redirectUrl }` (redirectUrl do provedor)

- POST /api/checkout/webhook
  - Provider webhook (no-auth endpoint protegido por secret)
  - Body: provider payload
  - Behavior: validar, criar `payments` e atualizar `users.plan` + `pending_checkouts.status` quando confirmado

- GET /api/checkout/:id
  - Auth required
  - Retorna status do checkout

---

## Uploads (avatares / fundos)

- POST /api/uploads
  - Auth required; multipart/form-data com `file` ou aceitar `data:` URLs
  - Behavior: enviar para storage (S3/GCS) e retornar `{ url }` ou armazenar data-URL em `uploads`/`users` conforme arquitetura
  - Response 201: `{ id, url }`

---

## Observações de implementação

- Proteção: use CSRF para forms/cookies; valide ownership nas rotas de usuário/links; sanitize URLs e campos de usuário.
- Consistência: incrementos de `clicks` devem ser atômicos.
- Cache: `GET /api/public/:username` é candidato a cache (CDN). Evite expor dados sensíveis.
- Idempotência: webhooks de pagamento devem ser idempotentes.

---

Se desejar, eu posso gerar exemplos concretos de payloads/respostas (JSON), um `OpenAPI`/Swagger básico, ou handlers iniciais (Express/Nitro/Fastify) para as rotas principais.
