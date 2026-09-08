# Extractor y Estandarizador de Estados Financieros (SEC EDGAR)

Herramienta en Python para acceder a la fuente oficial (SEC EDGAR) de estados contables de cualquier ticker que cotice en EE.UU. (incluyendo emisores extranjeros que presentan 20-F/40-F, US-GAAP e IFRS), estandarizarlos y calcular métricas financieras validadas contra fuentes de mercado.

> 📌 **Este repositorio es una vidriera del proyecto.** Documenta su funcionamiento y resultados con capturas reales. El código fuente es privado; este README explica arquitectura, alcance y stack técnico.

**Autor:** Alan Yapaze — [LinkedIn](https://www.linkedin.com/in/alan-yapaze/)
**Vigencia:** 2026

---

## Qué es

Un pipeline propio de ingesta y estandarización de reportes 10-K/20-F/40-F directamente desde SEC EDGAR, con un visor en Streamlit que permite:
- explorar Income Statement, Balance Sheet, Cash Flow, Stockholders' Equity y las notas a los estados contables de cualquier ticker (US-GAAP e IFRS),
- ver cada estado contable en modo Absoluto, Horizontal (% variación interanual) o Vertical (% de una base común, detectada por tag XBRL real),
- calcular un set de **ratios financieros clave** (rentabilidad, liquidez, solvencia, eficiencia) directamente sobre los tags XBRL, con **validación cruzada sistemática contra yfinance**,
- seguir las **transacciones de insiders** (Form 4, compras/ventas y grants de opciones) y los **eventos materiales** (8-K) de cualquier ticker — sin depender de proveedores de datos pagos.

La lógica de negocio y las decisiones de arquitectura son de autoría propia; la implementación de código fue desarrollada con asistencia de agentes de IA (Claude, Gemini) bajo un enfoque de *AI-augmented development* — dirigido por criterio financiero, no por experiencia previa como programador.

**Alcance actual: el universo CEDEAR.** El proyecto hoy estandariza y valida específicamente el universo de tickers que un inversor argentino puede adquirir como CEDEAR (Certificado de Depósito Argentino) — poco más de 300 empresas, un subconjunto acotado del total de emisores en SEC EDGAR (que son varios miles). Esta decisión de alcance es deliberada: permite auditar cada ticker a fondo, sector por sector, contra una fuente independiente (yfinance) y documentar cada divergencia real encontrada, algo mucho menos viable si el objetivo fuera cubrir el universo completo de la SEC de entrada.

## Capturas (ejemplo: Microsoft, 10-K 2026)

16 años de reportes anuales disponibles, exportación individual o masiva (ZIP) a Excel, y exploración interactiva por año y tipo de estado contable.
<img width="3440" height="1440" alt="image" src="https://github.com/user-attachments/assets/3f98d173-e21e-496f-9eea-de69d83e11af" />

### Income Statement
<img width="3440" height="1440" alt="image" src="https://github.com/user-attachments/assets/181ba165-f806-4b5a-a3df-6a07cefb1024" />

### Balance Sheet
<img width="3440" height="1440" alt="image" src="https://github.com/user-attachments/assets/564c3c57-c055-486b-a61a-68bc652b9ab0" />

### Cash Flow
<img width="3440" height="1440" alt="image" src="https://github.com/user-attachments/assets/ed1ec386-dab8-484f-8e54-af1207d02ffd" />

### Análisis Horizontal y Vertical (por filing)
Cada estado contable admite un segundo modo de lectura sin salir del filing original: Horizontal (% de variación interanual, con detección de cruce de signo — pérdida a ganancia o viceversa se marca "N/M" en vez de un % engañoso) y Vertical (% de Revenue en el Income Statement, % de Total Assets en el Balance Sheet — la base se detecta por el tag XBRL real de la fila, no por el texto de su label, así que funciona igual para un filer US-GAAP que para uno IFRS).
<!-- TODO: agregar captura -- selector "Modo de análisis" en Horizontal y en Vertical, ej. sobre el Income Statement de una empresa con varios años -->

### Ratios Clave — Anual y Trimestral (TTM)
Un set de ratios estilo Damodaran (rentabilidad, liquidez, solvencia, apalancamiento, eficiencia) calculado directamente sobre el Company Facts XBRL, con dos modos: Anual (estrictamente sobre 10-K/20-F/40-F) y Trimestral con TTM (Trailing Twelve Months para las métricas de flujo, valor puntual de cierre para las de balance). Cada ratio muestra qué tag XBRL exacto se usó para calcularlo, auditable con un clic contra el filing original.
<!-- TODO: agregar captura -- tabla de Ratios Clave, modo Anual, con la columna "tag XBRL usado" visible -->

### Insiders & Eventos
Transacciones de insiders (Form 4, incluyendo enmiendas 4/A) parseadas del **XML estructurado** del filing original, no del HTML pre-renderizado — captura compras/ventas directas de acciones y también grants de opciones/RSUs (transacciones derivadas), con fecha, cargo del insider (oficial, director, accionista >10%), tipo de transacción, cantidad, precio y tenencia resultante. Debajo, los últimos 8-K (eventos materiales) con link directo al documento original — sin intentar estandarizarlos, ya que no tienen un formato de estado contable fijo.
<!-- TODO: agregar captura -- tabla de transacciones Form 4 de un insider conocido, ej. un director vendiendo/ejerciendo opciones -->

## Qué resuelve — y por qué el alcance evolucionó

La primera versión de este proyecto apuntaba a un motor de valuación completo (DCF, múltiplos comparables, ROIC ajustado a lo Damodaran) montado directamente sobre los datos de SEC EDGAR. Al construirlo apareció un problema de fondo: la taxonomía XBRL varía entre emisores y a lo largo del tiempo para la misma empresa, lo que hacía frágil cualquier análisis automatizado sobre esos datos sin resolver antes ese problema de raíz.

En lugar de forzar un análisis poco confiable, el proyecto se reconstruyó en dos etapas: primero **integridad de la fuente** — ingesta directa, parseo estructurado y exportación limpia, sin agregación de terceros — y recién sobre esa base sólida, una **capa de métricas** con un motor de alias por tag XBRL (para tolerar que distintos emisores, o el mismo emisor en distintos años, usen tags distintos para el mismo concepto) y una metodología de auditoría sector por sector contra yfinance, documentando cada divergencia real encontrada en vez de ignorarla. La misma decisión que aplico al auditar datos financieros: sin integridad de datos, no hay análisis posterior confiable — pero una vez asegurada esa integridad, sí vale la pena construir el análisis encima, con evidencia de que es correcto.

## Lo que hace hoy

- Ingesta desde SEC EDGAR con un rate-limiter propio auto-limitado a 5 req/s (por debajo del máximo de 10 req/s que permite la SEC), con reintentos y backoff ante 429/errores de conexión.
- Parser propio (sin dependencias externas) para interpretar archivos XBRL R-files y extraer estados contables y notas de forma estructurada — soporta emisores US-GAAP e IFRS, y múltiples monedas de reporte (USD, EUR, y cualquier otra que la SEC muestre en el R-file).
- Visor interactivo en Streamlit por ticker, año y tipo de estado contable, con modo Absoluto / Horizontal / Vertical por filing.
- Motor de ratios financieros clave (Anual y Trimestral TTM), calculado sobre Company Facts XBRL con manejo explícito de los casos donde la taxonomía no es uniforme entre emisores.
- Validación cruzada sistemática contra yfinance, sector por sector, con métricas de cobertura publicadas y cada divergencia real documentada (no descartada).
- Transacciones de insiders (Form 4/4-A) parseadas del XML estructurado, y últimos eventos materiales (8-K) con link directo al filing original.
- Exportación a Excel individual (por año) o masiva (ZIP con todos los años disponibles).

## Precisión de métricas: metodología de validación

Los datos se extraen **directamente de los filings oficiales XBRL** presentados a la SEC: máxima fidelidad a lo reportado, con divergencias respecto a plataformas comerciales documentadas caso por caso en vez de asumidas como error propio.

**Benchmark (234 de los +300 tickers CEDEAR, todos los sectores, contra yfinance):**

| Métrica | % OK vs yfinance |
|---|---|
| Cash & Equivalents | **97.3%** |
| Cash Flow from Operations | **96.9%** |
| Accounts Payable | **92.9%** |
| Inventory | **93.4%** |
| Cost of Revenue | **90.0%** |
| CapEx / Free Cash Flow | ~89% |
| Accounts Receivable | ~84% |
| Operating Income | ~82% |

Cada gap por debajo del 100% tiene una causa raíz identificada y documentada (ej. empresas que reportan en moneda local sin conversión, corrimiento de año fiscal en cierres de enero/febrero, segmento financiero mezclado con el industrial, partidas que la propia empresa nunca tagea como concepto XBRL separado) — no son errores silenciosos, son límites conocidos y explicados de la fuente de datos.

## Stack técnico

| Capa | Tecnologías |
|---|---|
| Ingesta | aiohttp, con rate-limiter propio y reintentos con backoff |
| Parsing XBRL | Parser HTML propio (stdlib, sin BeautifulSoup) — US-GAAP e IFRS, multi-moneda |
| Motor de ratios | Cálculo propio sobre Company Facts XBRL, con resolución de alias por tag |
| Validación | Cross-check sistemático contra yfinance, sector por sector |
| Almacenamiento | SQLite (índice de filings y estados) |
| Exportación | openpyxl |
| Dashboard | Streamlit |
| Tests | pytest — parser de R-files, motor de ratios, validación cruzada, formato numérico |

## Contacto

¿Preguntas sobre el proyecto o interés en el desarrollo? [Alan Yapaze en LinkedIn](https://www.linkedin.com/in/alan-yapaze/)
