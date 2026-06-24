# PRD: Haaland — Sports Trading Agent for Polymarket
**Version:** 1.0 · **Status:** MVP · **Fecha:** Junio 2026  
**Owner:** Builder (Andrés) · **Clasificación:** Internal

---

## Tabla de Contenidos
1. [Project Overview](#1-project-overview)
2. [User Stories & Acceptance Criteria](#2-user-stories--acceptance-criteria)
3. [UI/UX Requirements](#3-uiux-requirements)
4. [Technical Requirements](#4-technical-requirements)
5. [Success Metrics](#5-success-metrics)
6. [Implementation Roadmap](#6-implementation-roadmap)

---

## 1. Project Overview

### 1.1 Problema

Los traders de Polymarket que operan mercados de fútbol/soccer no tienen acceso rápido a:
- Contexto real (noticias, lesiones, alineaciones) integrado con datos de mercado en tiempo real.
- Una señal accionable que combine múltiples fuentes en un solo número.
- Análisis honesto de parlays con desglose de probabilidad por pierna.

El resultado: operan a ciegas o de forma manual, perdiendo edge frente a actores más informados.

### 1.2 Solución

**Haaland** es un agente de trading autónomo especializado en mercados de predicción de fútbol en Polymarket. Combina noticias en tiempo real, datos de mercado (precio, volumen, órdenes de ballenas) y análisis LLM para generar señales BUY/SELL/SKIP con un composite score 0–100. Puede ejecutar trades automáticamente y responde a consultas en lenguaje natural desde Telegram (privado o grupal).

### 1.3 Objetivos del MVP

| # | Objetivo | Métrica de éxito |
|---|----------|------------------|
| O1 | Generar señales con edge real en mercados de fútbol | ≥1 señal con edge positivo por match day |
| O2 | Responder análisis de parlays bajo demanda desde Telegram | `/analyze` funciona en grupo con respuesta < 15s |
| O3 | Ejecutar trades sin intervención manual | Al menos 1 trade auto-ejecutado en paper trading sin errores |
| O4 | Operar dentro del budget mensual de $4–11 | Costo mensual real ≤ $15 |

### 1.4 Usuario Target

#### Persona A — El Trader Semi-Pro (usuario primario)
- **Perfil:** 25–40 años, opera en Polymarket regularmente, familiarizado con predicción de mercados. Sigue fútbol europeo (Premier League, Champions League, Eurocopa).
- **Pain:** Tiene intuición sobre mercados pero no tiempo ni herramientas para validarla con datos.
- **Uso:** Ejecuta el bot en modo semi-auto, revisa señales por Telegram antes de aprobarlas, consulta `/score` y `/whales` para confirmar entrada.
- **Expectativa:** Que el bot le ahorre 2–3h de análisis manual por match day.

#### Persona B — El Builder-Trader (el propio Andrés)
- **Perfil:** Desarrollador que también opera. Quiere validar el sistema con dinero real progresivamente.
- **Uso:** Inicia en modo auto con límites pequeños (<$50/trade), monitorea via Telegram, itera sobre los parámetros del composite score.
- **Expectativa:** Que el sistema sea transparente (reasoning visible), debuggeable y extensible.

#### Persona C — El Miembro del Grupo de Telegram (usuario secundario)
- **Perfil:** Amigo o comunidad que accede al bot en un grupo compartido. No tiene wallet conectada.
- **Uso:** Usa `/markets`, `/signal`, `/analyze` para aprender y discutir sin ejecutar trades.
- **Expectativa:** Respuestas claras, honestas sobre la incertidumbre, sin ruido.

### 1.5 Fuera de Alcance (MVP)

- Web dashboard
- Mercados no-fútbol (NBA, UFC, etc.)
- Monte Carlo / modelos GBM
- Trading in-match de alta frecuencia
- Backtesting formal
- Kelly Criterion completo
- Historial de parlays por usuario/wallet
- Sistema Hashdive-style (requiere blockchain indexer)

---

## 2. User Stories & Acceptance Criteria

### Épica 1: Descubrimiento de Mercados

---

**US-01 — Listar mercados activos de fútbol**

> Como trader, quiero ver los mercados de fútbol activos en Polymarket para identificar oportunidades rápidamente.

**Acceptance Criteria:**
- [ ] `/markets` devuelve lista de mercados activos en ≤10s.
- [ ] Cada mercado muestra: nombre del evento, precio actual YES/NO, volumen, fecha de resolución.
- [ ] Solo aparecen mercados de categoría soccer/football (filtrado por tag en Gamma API).
- [ ] Si no hay mercados activos, responde "No hay mercados de fútbol activos en este momento."
- [ ] Máximo 10 mercados por respuesta (paginación en v1.1).

---

**US-02 — Monitoreo automático de mercados**

> Como builder, quiero que el agente escanee mercados activos periódicamente sin intervención manual.

**Acceptance Criteria:**
- [ ] El loop en `main.py` corre cada X minutos (configurable via ENV, default: 30min).
- [ ] Detecta mercados nuevos desde el último ciclo y los encola para análisis.
- [ ] Guarda snapshot de precios en SQLite cada 30min para detección de momentum.
- [ ] Logs muestran timestamp, mercados detectados y acción tomada por ciclo.

---

### Épica 2: Análisis y Señales

---

**US-03 — Señal completa para un partido**

> Como trader, quiero una señal BUY/SELL/SKIP con reasoning para un equipo o partido específico.

**Acceptance Criteria:**
- [ ] `/signal [team]` responde en ≤15s con: señal, composite score, confianza, y reasoning resumido.
- [ ] La búsqueda del mercado es tolerante a errores tipográficos (ej: "Manch United" encuentra Manchester United).
- [ ] Si hay múltiples mercados para el mismo equipo, lista las opciones y pide al usuario que elija.
- [ ] El reasoning incluye al menos: resumen de noticias relevantes, precio actual vs estimación del agente, momentum del precio.
- [ ] Si el spread es >10%, la señal es automáticamente SKIP con explicación.

---

**US-04 — Composite Score desglosado**

> Como trader avanzado, quiero ver el desglose del score 0–100 para entender de dónde viene la señal.

**Acceptance Criteria:**
- [ ] `/score [team]` devuelve tabla con los 4 componentes: LLM edge (0–40), momentum (0–20), liquidez (0–20), smart money (0–20).
- [ ] Cada componente muestra el valor obtenido y el máximo posible.
- [ ] Score total ≥75 se muestra en verde (emoji ✅), 50–74 en amarillo (⚠️), <50 en rojo (❌).
- [ ] Si el mercado no existe, responde con mensaje claro.

---

**US-05 — Detección de ballenas (smart money)**

> Como trader, quiero saber si hay órdenes grandes en el libro de órdenes que confirmen o contradigan la señal.

**Acceptance Criteria:**
- [ ] `/whales` lista todas las órdenes individuales >$500 detectadas en las últimas 24h.
- [ ] Cada entrada muestra: mercado, dirección (YES/NO), monto, timestamp.
- [ ] Si no hay órdenes grandes, responde "No se detectaron órdenes >$500 en las últimas 24h."
- [ ] La detección es pasiva (lectura del order book via CLOB API, sin ejecutar nada).

---

### Épica 3: Análisis de Parlays

---

**US-06 — Análisis de parlay en lenguaje natural**

> Como usuario del grupo, quiero analizar un parlay completo con desglose por pierna para saber si vale la pena.

**Acceptance Criteria:**
- [ ] `/analyze [query en lenguaje libre]` funciona en chat privado y en grupos.
- [ ] El bot parsea la query, identifica N legs (≥1), busca cada mercado en Gamma API, y corre `analyst.py` por leg.
- [ ] La respuesta incluye por cada leg: precio de mercado, estimación del agente, edge, veredicto (BUY/SKIP).
- [ ] La respuesta incluye: probabilidad combinada de mercado, probabilidad combinada del agente, veredicto final del parlay.
- [ ] Si una leg no encuentra mercado en Polymarket, lo indica explícitamente y excluye del cálculo.
- [ ] Tiempo de respuesta ≤20s para parlays de hasta 3 legs.
- [ ] El veredicto final compara probabilidad combinada de mercado vs estimación del agente y sugiere acción.

---

**US-07 — Consulta de evento único via /analyze**

> Como usuario, quiero poder usar `/analyze` para un solo evento, no solo parlays.

**Acceptance Criteria:**
- [ ] `/analyze Will Spain win?` funciona igual que un parlay de 1 leg.
- [ ] La respuesta es equivalente a `/signal` pero en formato de análisis de parlay.
- [ ] No requiere sintaxis especial (AND, etc.) para consultas de 1 leg.

---

### Épica 4: Ejecución de Trades

---

**US-08 — Ejecución automática (modo auto)**

> Como builder, quiero que el agente ejecute trades automáticamente cuando el score supera el umbral, sin que yo tenga que intervenir.

**Acceptance Criteria:**
- [ ] Si `MODE=auto` en `.env` y composite score ≥75, el trade se ejecuta via `py-clob-client`.
- [ ] Monto por trade viene de `TRADE_SIZE_USDC` en `.env` (default: $5 para paper trading).
- [ ] El trade ejecutado se registra en SQLite con: market_id, señal, score, precio de entrada, monto, timestamp.
- [ ] Una notificación Telegram se envía al `OWNER_CHAT_ID` confirmando el trade ejecutado.
- [ ] Si la ejecución falla (API error, saldo insuficiente), se loggea el error y se envía alerta por Telegram. No se reintenta automáticamente.

---

**US-09 — Aprobación manual (modo semi-auto)**

> Como trader, quiero recibir una alerta cuando el agente detecta una oportunidad con score 50–74, para aprobarla o rechazarla yo mismo.

**Acceptance Criteria:**
- [ ] Si `MODE=semi-auto` y score es 50–74, el bot envía al `OWNER_CHAT_ID`: señal, score, reasoning, botones inline "✅ Ejecutar" / "❌ Skip".
- [ ] Si el usuario toca "Ejecutar", se ejecuta el trade y se notifica confirmación.
- [ ] Si el usuario toca "Skip" o no responde en 10 minutos, la oportunidad se descarta y se loggea como SKIPPED_BY_USER.
- [ ] Los botones inline solo funcionan para el `OWNER_CHAT_ID`, no para miembros del grupo.

---

### Épica 5: Resumen y Seguimiento

---

**US-10 — Resumen diario de P&L**

> Como trader, quiero ver un resumen de los trades ejecutados y el P&L del día.

**Acceptance Criteria:**
- [ ] `/summary` muestra: número de trades ejecutados hoy, monto total apostado, ganancias/pérdidas realizadas (si el mercado ya resolvió), trades pendientes de resolución.
- [ ] Si no hay trades, responde "No hay trades ejecutados hoy."
- [ ] El P&L solo calcula trades en mercados ya resueltos; los pendientes se listan por separado.

---

## 3. UI/UX Requirements

### 3.1 Principios de Diseño de la Interfaz (Telegram)

- **Claridad sobre completitud:** Cada respuesta debe ser legible en móvil. Máximo 20 líneas por mensaje antes de truncar.
- **Emojis como señales visuales:** ✅ = positivo/BUY, ❌ = negativo/SKIP, ⚠️ = neutro/semi, 🐋 = whale, 📊 = datos.
- **Números siempre con contexto:** No "42%", sino "42% (mercado) vs 51% (agente) → edge +9%".
- **Acciones restringidas visualmente:** En grupos, los comandos de trading no aparecen / están deshabilitados.

### 3.2 Flujos de Usuario

#### Flujo A — Trader busca señal para un partido

```
Usuario: /signal Manchester City
    ↓
Bot: Buscando mercados para "Manchester City"... (mensaje temporal)
    ↓
[markets.py: Gamma API search]
[news.py: GNews últimas 48h]
[market_data.py: CLOB price + volume + spread + whales]
[analyst.py: Claude Haiku → JSON signal]
[scorer.py: composite score]
    ↓
Bot: 📊 Análisis — Man City vs Arsenal (Champions League)

Signal:     BUY YES ✅
Score:      81/100
Confianza:  Alta

Desglose:
• LLM edge:      34/40 (+14% sobre precio)
• Momentum:      18/20 (precio subiendo 4% en 6h)
• Liquidez:      16/20 (spread 3.2%)
• Smart money:   13/20 (1 orden $800 detectada)

Precio actual: 48% YES
Estimación:    62% YES

Reasoning: De Bruyne confirmado para jugar según nota oficial del club.
Arsenal sin centrocampista titular por sanción. Precio refleja incertidumbre
pre-partido, pero el contexto favorece a City.

[En modo semi-auto o si score <75]
¿Ejecutar trade? ✅ BUY $5  ❌ Skip
```

#### Flujo B — Análisis de parlay en grupo

```
Usuario (en grupo): /analyze Will Spain win the World Cup AND Mbappé scores in the final?
    ↓
Bot: 🔍 Analizando parlay de 2 piernas...
    ↓
[parlay.py: parser → extrae 2 legs]
[Para cada leg: markets.py → analyst.py → scorer.py]
    ↓
Bot: 🎯 Análisis de Parlay

① España gana el Mundial
   Precio mercado:  42%
   Estimación:      51%
   Edge:            +9% → BUY ✅

② Mbappé anota en la final
   Precio mercado:  38%
   Estimación:      29%
   Edge:            -9% → SKIP ❌

──────────────────────────
📊 Parlay combinado
   Mercado implica: 42% × 38% = 15.9%
   Agente estima:   51% × 29% = 14.8%

⚠️ Veredicto: SKIP
   El parlay está aproximadamente a precio justo.
   La pierna 2 está ligeramente sobrevaluada.
   Mejor jugar solo la pierna 1 si acaso.
```

#### Flujo C — Trade auto-ejecutado (modo auto)

```
[main.py loop detecta score ≥ 75]
    ↓
[executor.py: py-clob-client → order placed]
    ↓
Bot → OWNER_CHAT_ID:
⚡ Trade Ejecutado

Mercado:   Real Madrid gana la Liga
Señal:     BUY YES
Precio:    63% YES
Monto:     $5 USDC
Score:     82/100
TX:        [link a Polymarket]
Timestamp: 2026-06-15 14:32 UTC
```

### 3.3 Descripción de Mensajes de Error

| Situación | Mensaje al usuario |
|-----------|-------------------|
| Mercado no encontrado | "❌ No encontré un mercado activo para '[query]'. Prueba con `/markets` para ver los disponibles." |
| GNews sin resultados | Análisis continúa, el reasoning indica "Sin noticias recientes encontradas." |
| CLOB API timeout | "⚠️ No pude obtener datos de mercado en tiempo real. Reintentando en el próximo ciclo." |
| Fallo en ejecución de trade | "🚨 Error al ejecutar trade en [mercado]. Causa: [error]. Trade NO ejecutado. Revisa saldo y conectividad." |
| Score <50 | No se envía alerta. Se loggea como SKIP automático. |

### 3.4 Restricciones de Contexto (Grupos vs Privado)

| Feature | Chat Privado (OWNER) | Grupo Telegram |
|---------|---------------------|----------------|
| `/markets` | ✅ | ✅ |
| `/signal` | ✅ | ✅ |
| `/score` | ✅ | ✅ |
| `/whales` | ✅ | ✅ |
| `/analyze` | ✅ | ✅ |
| `/summary` | ✅ | ❌ (solo owner) |
| Botones de ejecución | ✅ | ❌ |
| Trade auto | ✅ | N/A |

---

## 4. Technical Requirements

### 4.1 Stack Tecnológico

| Componente | Tecnología | Justificación |
|------------|-----------|---------------|
| Lenguaje | Python 3.11+ | Ecosistema ML/API, base repo en Python |
| LLM | Claude Haiku (Anthropic API) | Costo ~$0.25/1M tokens input; suficiente para análisis estructurado |
| Mercados | Polymarket Gamma API | Fuente oficial de mercados y metadata |
| Trading | Polymarket CLOB API + py-clob-client | Ejecución de órdenes y datos de orderbook |
| Noticias | GNews API (free tier) | 100 req/día gratuitas; cubre necesidad del MVP |
| Alertas | python-telegram-bot | Soporte inline buttons, group chats, commands |
| Persistencia | SQLite | Zero-config, suficiente para MVP; migrable a Postgres en v2 |
| Scheduler | APScheduler o loop simple asyncio | Liviano, sin necesidad de Celery en MVP |
| Deploy | Railway free tier o local | $0–5/mes |

### 4.2 Estructura de Archivos y Responsabilidades

```
haaland/
├── main.py          ← Loop principal (APScheduler o asyncio). Orquesta el ciclo completo.
│                      Configura intervalo via SCAN_INTERVAL_MIN en .env.
├── markets.py       ← Gamma API client. Métodos: get_active_soccer_markets(), search_market(query).
│                      Filtra por tag "soccer" o "football". Retorna lista de Market objects.
├── news.py          ← GNews API client. Método: get_team_news(team_name, hours=48).
│                      Rate limiting: máx 100 req/día. Cache en memoria por 1h para evitar duplicados.
├── market_data.py   ← CLOB API client. Métodos: get_price(market_id), get_orderbook(market_id),
│                      detect_whales(market_id, threshold=500), get_price_history(market_id).
│                      Calcula spread, momentum (vs último snapshot en SQLite).
├── analyst.py       ← Prompt engineering + Anthropic API call. Input: Market + News + MarketData.
│                      Output JSON: {signal, confidence, llm_edge_score, reasoning}.
│                      Prompt incluye: instrucción de responder SOLO JSON, schema esperado.
├── scorer.py        ← Calcula composite score 0–100. Input: analyst output + market_data output.
│                      Retorna: {total_score, llm_pts, momentum_pts, liquidity_pts, whale_pts}.
├── parlay.py        ← Parser de lenguaje natural + orquestador multi-leg.
│                      Usa Claude Haiku para parsear query → lista de legs.
│                      Llama analyst.py por cada leg. Calcula probabilidad combinada.
├── executor.py      ← Wrapper de py-clob-client. Método: place_order(market_id, side, amount).
│                      Valida MODE antes de ejecutar. Loggea en SQLite. Notifica via alerts.py.
├── alerts.py        ← python-telegram-bot. Registra todos los handlers de comandos.
│                      Restringe ejecución de trades a OWNER_CHAT_ID.
│                      Maneja inline keyboard callbacks (aprobar/rechazar trade).
├── db.py            ← SQLite schema y métodos CRUD.
│                      Tablas: trades, price_snapshots, signals_log.
└── .env             ← Variables de entorno (ver sección 4.3)
```

### 4.3 Variables de Entorno (.env)

```env
# APIs
POLYMARKET_KEY=           # Clave privada wallet Polygon
CLAUDE_API_KEY=           # Anthropic API key
TELEGRAM_TOKEN=           # Token del bot de Telegram
GNEWS_KEY=                # GNews API key

# Configuración del agente
MODE=semi-auto            # "auto" o "semi-auto"
TRADE_SIZE_USDC=5         # Monto por trade en USDC
SCORE_AUTO_THRESHOLD=75   # Score mínimo para auto-ejecución
SCORE_ALERT_THRESHOLD=50  # Score mínimo para alertar en semi-auto
SCAN_INTERVAL_MIN=30      # Intervalo de escaneo en minutos
WHALE_THRESHOLD=500       # Monto mínimo en USD para considerar whale

# Telegram
OWNER_CHAT_ID=            # Chat ID del owner (único autorizado a ejecutar)
GROUP_CHAT_ID=            # (Opcional) Chat ID del grupo de Telegram

# Spread
MAX_SPREAD_PCT=10         # Auto-SKIP si spread > este porcentaje
```

### 4.4 Schema de Base de Datos (SQLite)

```sql
-- Trades ejecutados
CREATE TABLE trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL,
    market_name TEXT,
    signal TEXT NOT NULL,         -- BUY_YES, BUY_NO
    composite_score INTEGER,
    entry_price REAL,
    amount_usdc REAL,
    status TEXT DEFAULT 'OPEN',   -- OPEN, WON, LOST, VOID
    pnl_usdc REAL,                -- NULL hasta resolución
    executed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    resolved_at DATETIME,
    tx_hash TEXT,
    reasoning TEXT
);

-- Snapshots de precio para detección de momentum
CREATE TABLE price_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL,
    price_yes REAL,
    price_no REAL,
    volume_24h REAL,
    spread_pct REAL,
    captured_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Log de señales (incluyendo SKIPs)
CREATE TABLE signals_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL,
    market_name TEXT,
    signal TEXT,                  -- BUY_YES, BUY_NO, SKIP
    composite_score INTEGER,
    llm_edge_pts INTEGER,
    momentum_pts INTEGER,
    liquidity_pts INTEGER,
    whale_pts INTEGER,
    skip_reason TEXT,             -- NULL si no es SKIP
    action_taken TEXT,            -- AUTO_EXECUTED, SENT_TO_USER, SKIPPED_BY_SCORE, SKIPPED_BY_USER, SKIPPED_HIGH_SPREAD
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 4.5 Contrato del Prompt de analyst.py

**Input al LLM:**
```
Eres un analista experto en mercados de predicción deportiva. Analiza el siguiente contexto
y devuelve ÚNICAMENTE un objeto JSON con exactamente este schema, sin markdown ni texto adicional:

{
  "signal": "BUY_YES" | "BUY_NO" | "SKIP",
  "confidence": "HIGH" | "MEDIUM" | "LOW",
  "estimated_probability": <float entre 0 y 1>,
  "llm_edge_score": <integer entre 0 y 40>,
  "reasoning": "<string de 2-3 oraciones max>"
}

CONTEXTO DEL MERCADO:
- Nombre: {market_name}
- Precio actual YES: {price_yes}%
- Volumen 24h: ${volume_24h}
- Spread: {spread_pct}%

NOTICIAS RECIENTES (últimas 48h):
{news_summary}

MOVIMIENTO DE PRECIO:
{price_momentum_description}
```

### 4.6 Integración con py-clob-client

```python
# executor.py — interface esperada
from clob_client import ClobClient

client = ClobClient(
    host="https://clob.polymarket.com",
    key=os.getenv("POLYMARKET_KEY"),
    chain_id=137  # Polygon
)

def place_order(market_id: str, side: str, amount_usdc: float) -> dict:
    """
    side: "YES" o "NO"
    Retorna: {"tx_hash": str, "filled_price": float, "status": str}
    """
```

### 4.7 Constraints Técnicos

- GNews: máximo 100 requests/día. El cache de 1h en `news.py` es obligatorio para no agotar el cupo.
- Claude Haiku: máximo ~600 llamadas/mes dentro del budget. El caller debe validar que no se llame más de 20 veces por hora.
- Telegram: los comandos en grupos deben ignorar `/command@other_bot` patterns (filtrar por bot username).
- SQLite: no usar WAL mode. Un solo proceso escritor (main.py + alerts.py deben usar conexión compartida o cola).
- No usar threads para el bot + el loop: usar asyncio con `python-telegram-bot` en modo async.

---

## 5. Success Metrics

### 5.1 KPIs de Producto (semana 1–3, paper trading)

| KPI | Definición | Target MVP |
|-----|-----------|-----------|
| **Signal Rate** | Señales BUY generadas por match day con score ≥50 | ≥1 por match day activo |
| **Edge Accuracy** | % de señales BUY donde la estimación del agente fue más cercana al resultado que el precio de mercado | ≥55% (baseline: 50% random) |
| **False Positive Rate** | % de señales BUY con score ≥75 que resultaron en pérdida | <40% en paper trading |
| **Parlay Response Rate** | % de comandos `/analyze` que responden correctamente en <20s | ≥90% |
| **Uptime** | % del tiempo que el loop corre sin crashes | ≥95% en ventana de 7 días |
| **Cost/Analysis** | Costo promedio de Claude Haiku por análisis completo | <$0.01 |

### 5.2 KPIs Técnicos

| KPI | Target |
|-----|--------|
| Tiempo de respuesta `/signal` | <15s P90 |
| Tiempo de respuesta `/analyze` (3 legs) | <25s P90 |
| Requests GNews por día | <80 (buffer de 20%) |
| Errores de ejecución de trade | 0 errores silenciosos (todos notificados) |
| Cobertura de mercados de fútbol en Polymarket | ≥80% de mercados activos detectados por ciclo |

### 5.3 Criterios de Go/No-Go para Dinero Real

Antes de pasar de paper trading a dinero real, se deben cumplir **todos**:

- [ ] ≥50 señales generadas y loggeadas en SQLite.
- [ ] Edge Accuracy ≥55% sobre las señales generadas.
- [ ] 0 trades ejecutados involuntariamente (falsos positivos de ejecución).
- [ ] `/analyze` funciona correctamente para ≥10 consultas distintas de prueba.
- [ ] Costo mensual real ≤$15 verificado en billing de Anthropic.
- [ ] Al menos 1 semana de uptime continuo sin crash manual.

---

## 6. Implementation Roadmap

### Phase 0 — Setup (Día 1–2)

**Objetivo:** Repositorio funcional con acceso a las 4 APIs verificado.

**Tickets:**
- [ ] **P0-01:** Crear estructura de directorios del proyecto y `.env` template.
- [ ] **P0-02:** Instalar dependencias: `anthropic`, `python-telegram-bot`, `requests`, `py-clob-client`, `apscheduler`, `python-dotenv`.
- [ ] **P0-03:** Verificar conexión a Gamma API → imprimir lista de mercados activos en consola.
- [ ] **P0-04:** Verificar conexión a CLOB API → imprimir precio de un mercado de prueba.
- [ ] **P0-05:** Verificar GNews API → retornar noticias de "Real Madrid" en últimas 48h.
- [ ] **P0-06:** Verificar Anthropic API con prompt simple → recibir respuesta JSON válida.
- [ ] **P0-07:** Verificar Telegram bot → bot responde `/start` en chat privado.

**Entregable:** Script de smoke test que corre todos los checks anteriores y reporta PASS/FAIL.

---

### Phase 1 — Week 1: Data Pipeline

**Objetivo:** Datos fluyendo de las 3 fuentes, visibles en consola, guardados en SQLite.

**Tickets:**
- [ ] **W1-01:** Implementar `markets.py` — `get_active_soccer_markets()` y `search_market(query)`.
- [ ] **W1-02:** Implementar `news.py` — `get_team_news(team, hours=48)` con cache en memoria (1h TTL).
- [ ] **W1-03:** Implementar `market_data.py` — `get_price()`, `get_orderbook()`, `detect_whales()`, `get_spread()`.
- [ ] **W1-04:** Implementar `db.py` — schema SQLite, métodos insert/query para las 3 tablas.
- [ ] **W1-05:** Implementar `main.py` loop básico — scannea mercados, loggea datos en consola y SQLite cada 30min.
- [ ] **W1-06:** Test de integración — correr el loop por 2 horas, verificar que SQLite tiene snapshots de precio.

**Entregable:** Loop corriendo, datos guardándose en SQLite, visibles con un `sqlite3` query.

---

### Phase 2 — Week 2: Análisis y Telegram

**Objetivo:** Señales reales desde Claude Haiku, composite score, comandos básicos en Telegram.

**Tickets:**
- [ ] **W2-01:** Implementar `analyst.py` — prompt, llamada a Haiku, parsing de JSON response, manejo de errores.
- [ ] **W2-02:** Implementar `scorer.py` — composite score 0–100 con los 4 componentes.
- [ ] **W2-03:** Integrar analyst + scorer en `main.py` — generar señal por mercado detectado.
- [ ] **W2-04:** Implementar `alerts.py` — bot con handlers para `/start`, `/markets`, `/signal`, `/score`, `/whales`.
- [ ] **W2-05:** Implementar lógica de semi-auto en `alerts.py` — inline keyboard para aprobar/rechazar trade.
- [ ] **W2-06:** Implementar `executor.py` — `place_order()` en modo DRY RUN (log sin ejecutar).
- [ ] **W2-07:** Restricción de ejecución en grupos — verificar `OWNER_CHAT_ID` antes de mostrar botones.
- [ ] **W2-08:** Test manual: `/signal Real Madrid` en Telegram privado → señal completa recibida.

**Entregable:** Bot respondiendo en Telegram con señales reales para cualquier equipo activo.

---

### Phase 3 — Week 3: Parlays, Ejecución y Paper Trading

**Objetivo:** `/analyze` funcional, trades ejecutándose en paper, 1 semana de paper trading loggeada.

**Tickets:**
- [ ] **W3-01:** Implementar `parlay.py` — parser de lenguaje natural (via Haiku), orquestador multi-leg.
- [ ] **W3-02:** Implementar handler de `/analyze` en `alerts.py`.
- [ ] **W3-03:** Activar ejecución real en `executor.py` — `place_order()` con py-clob-client (montos mínimos $1–2).
- [ ] **W3-04:** Implementar `/summary` en `alerts.py` — P&L diario desde SQLite.
- [ ] **W3-05:** Ajustar umbrales de score si los primeros 20 análisis muestran señales poco calibradas.
- [ ] **W3-06:** Paper trading — correr 7 días con `TRADE_SIZE_USDC=2`, revisar SQLite diariamente.
- [ ] **W3-07:** Revisar costo real de Anthropic API tras 7 días → ajustar si supera budget.
- [ ] **W3-08:** Deploy en Railway free tier (o dejar corriendo en local/VPS).

**Entregable:** Sistema completo corriendo 7 días con señales, parlays y trades paper en SQLite.

---

### Post-MVP Backlog (v1.1 — v2)

| Feature | Versión | Justificación para diferir |
|---------|---------|--------------------------|
| Historial de parlays por usuario | v1.1 | Requiere auth por wallet, complejidad > beneficio en MVP |
| Kelly Criterion completo | v1.1 | Necesita historial de accuracy para calibrar |
| Backtesting formal | v1.1 | Requiere datos históricos de resolución de mercados |
| Monte Carlo / GBM | v2 | Validar primero que señales LLM tienen edge antes de añadir complejidad cuantitativa |
| Multi-sport (NBA, UFC) | v2 | Extensión natural; misma arquitectura, distintos prompts |
| Web dashboard | v2 | Telegram cubre el need del MVP; dashboard agrega superficie de mantenimiento |

---

## Apéndice A — Decisiones de Diseño

| Decisión | Alternativa considerada | Razón elegida |
|----------|------------------------|---------------|
| Claude Haiku como LLM | GPT-4o mini, Gemini Flash | Menor costo ($0.25/1M), integración nativa con Anthropic SDK, sin latencia adicional |
| SQLite como BD | PostgreSQL, Redis | Zero-config para MVP; migración sencilla si se necesita concurrencia |
| GNews (free tier) | Serper, NewsAPI, RapidAPI Sports | Único tier gratuito con 100 req/día suficientes para MVP |
| python-telegram-bot | Aiogram, Telethon | Más documentado, soporte async, inline keyboards out of the box |
| Señales BUY/SELL/SKIP | BUY/WATCH/SKIP | WATCH crea ambigüedad de acción; SELL es equivalente a BUY NO |
| Semi-auto + auto (configurable) | Solo manual | Permite testear gradualmente sin bloquear la automatización objetivo |

---

## Apéndice B — Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| GNews agota las 100 req/día | Media | Alto | Cache de 1h obligatorio en `news.py`; alert si se usan >80 req |
| Claude Haiku genera JSON inválido | Baja | Medio | Try/catch en `analyst.py`; fallback a SKIP si parsing falla |
| py-clob-client falla en ejecución | Media | Alto | Modo DRY RUN en Phase 2; solo activar ejecución real en Week 3 |
| Polymarket no tiene mercados de fútbol activos | Baja | Alto | Verificar en Gamma API antes del MVP; expandir a "sports" si escasean |
| Costo de Haiku supera budget | Baja | Medio | Rate limiter en `analyst.py`; no más de 20 llamadas/hora |
| Bot crashea silenciosamente | Media | Alto | Watchdog en `main.py` + alerta Telegram si el loop no corre en >2 ciclos |

---

*Haaland PRD v1.0 — Generado Junio 2026*  
*Próxima revisión recomendada: tras completar Phase 1 (fin de Week 1)*
