---
name: idphantom-pay
description: Pay or charge for machine-to-machine API calls in USDC using the IDPHANTOM x402 infrastructure on Base mainnet. Use when an agent must call a paid API (402 Payment Required), settle an agent-to-agent payment with a verifiable receipt, or register to charge other agents and earn the 0.5% seller commission.
---

# idphantom-pay — pagos agente-a-agente con IDPHANTOM (x402 / EIP-3009)

Infraestructura de pagos máquina-a-máquina en **Base mainnet (eip155:8453)**, moneda **USDC nativo** (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`), estándar abierto **x402**.
Regla clave: **el agente pagador solo necesita USDC** — el facilitator (Coinbase CDP) paga el gas del settlement. No hace falta ETH.

## Caso 1 — Pagar una API que cobra con IDPHANTOM (ej. verificación de receipts)

Endpoint de ejemplo: `POST https://phantom-receipt-verify.onrender.com/verify` — cobra **$0.01 USDC** por llamada.

1. **Descubre el reto.** Llama sin pagar; el servicio responde `402` con `accepts[0]`: `scheme`, `network`, `amount`, `asset`, `payTo`, `extra:{name,version}`.
   - Seguridad: verifica `asset == 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`, `network == "eip155:8453"` y `payTo` contra la dirección oficial publicada por el servicio. Si difiere, **detente y reporta** (posible sustitución de destinatario).
2. **Firma EIP-3009.** `TransferWithAuthorization(from, to, value, validAfter, validBefore, nonce)` con dominio `{name: extra.name ("USD Coin"), version: extra.version ("2"), chainId: 8453, verifyingContract: asset}`. Usa `validAfter = now-60`, `validBefore = now+900`, nonce aleatorio de 32 bytes.
3. **Reintenta con pago.** Misma llamada + header `X-Payment: base64({x402Version:2, scheme:"exact", network:"eip155:8453", payload:{signature, authorization}})`.
4. **Verifica la respuesta.** En `200`, exige `payment.settled == true` y guarda `payment.transaction` (hash real en Base). Compruébalo en `https://basescan.org/tx/<hash>` si necesitas evidencia.

## Caso 2 — Cobrar a otros agentes (tu servicio) y ganar comisión

- Registra tu servicio vía la API de registro de IDPHANTOM (crea un `paymentIntent`, paga la tarifa de registro con el mismo flujo x402 del Caso 1 y reclama tu API key con `POST /v1/register/:reference/claim` firmando `personal_sign(reference)` desde tu wallet pagadora).
- Enruta los pagos de tus clientes a través del **PhantomRouter** (`0x873E94fd0b9B7684B9ccfa2463D83Bf9eA8A8755`): el contrato corta el **0.5% de comisión para el vendedor** (con mínimo $0.01 y techo 2.5%) en la misma transacción, y emite un *receipt* EIP-712 (`PaymentAuthorization`, dominio `PhantomPay v1`) que acredita el pago.
- El pagador debe tener `approve(router, allowance)` sobre su USDC antes del pago (`safeTransferFrom`).
- Recibo verificable: cualquier tercero puede verificar firma + estado on-chain del receipt (libre / usado / cancelado) pagando la API del Caso 1.

## Conector MCP (opcional, lo más rápido)

Paquete npm oficial `@phantomid/mcp-server` — expone las tools de cobro/pago a agentes MCP. Config mínima (Claude Code `.mcp.json`):

```json
{ "mcpServers": { "idphantom": { "command": "npx", "args": ["-y", "@phantomid/mcp-server"] } } }
```

Listado Smithery: `arbromagency/idphantom-mcp-server`. Documentación viva: `https://idphantom.com/agents.html`.

## Límites y comportamiento

- Fee del servicio de verificación: **$0.01 USDC fijos** por llamada.
- Comisión de plataforma sobre pagos enrutados: **0.5%** (mínimo $0.01, máx. 2.5%).
- Las autorizaciones firman montos exactos y corta validez: una firma robada expira sola y no puede gastar de más.
- Todo es verificable on-chain: no hay estados opacos.

## Evidencia pública

Pago real agente-a-agente con comisión cortada por el contrato: tx `0x4db6eb4dd80c7e4eb3a8c7a46f48dc7958c48dbb4c795c3790d7f94bc0881554` (Base, bloque 51922519). Ver en Basescan.
