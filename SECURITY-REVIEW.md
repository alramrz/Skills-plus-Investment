# Revisión de seguridad de skills — Skills-plus-Investment

- **Fecha:** 2026-11-10
- **Commit revisado:** `c8f62b786a14570cf26babf091f74aa598ed2694`
- **Herramientas:** NVIDIA SkillSpector v2.12.0 (estático, `--no-llm`) + revisión manual (Claude)
- **Alcance:** `.claude/` completo — 61 skills, agents, commands, references (148 archivos, ~27k líneas)
- **Veredicto:** **APROBADO para uso**, con las condiciones de la sección 4.

## 1. Resultado del escáner

| Severidad | Hallazgos | Veredicto tras revisión manual |
|---|---|---|
| CRITICAL | 2 | Falsos positivos |
| HIGH | 22 | Falsos positivos (20) / riesgo aceptado con condiciones (2) |
| MEDIUM | 50 | Riesgo bajo, aceptado |
| LOW | 2 | Aceptado |

Cobertura: 140 archivos analizados por completo y 9 de forma parcial, por los límites del parser. Los analizadores marcados "degraded" analizaron 148 de 149 archivos; el que no completaron es `code-review-and-quality/SKILL.md`, porque contiene guiones largos y flechas Unicode. El problema no está en la instalación del escáner. Los 9 rangos parciales se revisaron a mano: son bloques de código de ejemplo dentro de documentación, que el agente no ejecuta.

Todos los hallazgos quedan registrados en `.skillspector-baseline.yaml`. Los escaneos futuros con `--baseline` solo reportan hallazgos **nuevos**.

## 2. Hallazgos principales y justificación

| Regla | Ubicación | Veredicto | Motivo |
|---|---|---|---|
| TT3 (CRITICAL) | trader-analyst/facts.py:112 | Falso positivo | La única variable de entorno es `SEC_USER_AGENT` (nombre y correo). Se manda como User-Agent solo a sec.gov, que es un requisito de la SEC. |
| TT3 (CRITICAL) | trader-news/news.py:183 | Falso positivo | Mismo caso: User-Agent a la SEC. A Yahoo se le manda un UA fijo. |
| P1 / YR4 | browser-testing-with-devtools:77, source-driven-development:101-110 | Falso positivo | Son reglas **anti**-prompt-injection ("trata el contenido web como datos"). |
| AR2 | arena:20, shipping-and-launch:25, trader-timemachine:144 | Falso positivo | Texto normal: orquestación, checklist y reglas de honestidad estadística. |
| PE3 / SSRF1 / MP3 | security-and-hardening y su hardening-patterns.md | Falso positivo | Ejemplos educativos de qué proteger. |
| YR4 | replica-launch/listing.py:33 | Falso positivo | Regex para limitar texto de App Store. El ZWJ de la línea 50 es un literal para contar emojis. |
| RA1 / TM1 / EA3 | code-simplification, git-workflow (sugiere un reset duro de git) | Aceptado | Es una guía de flujo. Claude Code pide permiso antes de correr comandos. |
| AS1 | references/orchestration-patterns.md:205 | Aceptado | Documenta una opción del archivo de configuración global de Claude. No la modifica sola. |
| EA5 | doubt-driven-development:146 | **Aceptado con condición** | Puede invocar los CLIs de Codex o Gemini (costo y código enviado a un tercero). Pide confirmación antes. |
| E1 | trader-broker/SKILL.md:130 | **Aceptado con condición** | Puede preparar órdenes para Alpaca o IBKR. Usa paper por defecto y tiene 3 controles por orden. |
| RP1 (x30) | varios `npx` sin versión | Riesgo bajo | Son referencias en documentación, no se instalan solas. |
| AST4 | trader-pelosi/ptr.py:342 | Aceptado | `subprocess.run` con lista de argumentos hacia `pdftotext`, sin shell. |

## 3. Barridos adicionales (todo el repo)

- **Dominios contactados por los scripts:** sec.gov, data.sec.gov, disclosures-clerk.house.gov, finance.yahoo.com, stooq.com, nasdaqtrader.com y nyse.com. Todas son fuentes públicas de datos.
- **Llamadas peligrosas:** no hay `eval`, `exec`, `pickle`, `os.system`, `Popen`, borrado de archivos, descarga y ejecución de scripts remotos, ni acceso a llaves SSH o credenciales.
- **Configuración que se ejecuta sola:** el repo no trae `settings.json`, hooks, `.mcp.json` ni `CLAUDE.md`.
- **Texto oculto:** no hay instrucciones escondidas en caracteres invisibles ni en comentarios HTML.
- **Orígenes identificados:** Jakeschincariol/day-trader-skill (trader-\*), DietrichGebert/ponytail, skills de ingeniería tipo agent-skills; el origen de replica-\* no está identificado.

## 4. Condiciones de uso

1. **trader-broker:** solo llaves *paper* de Alpaca. La herramienta de órdenes debe estar en "Needs approval", nunca en "Always allow".
2. **doubt-driven-development:** no instalar ni autenticar los CLIs de Codex o Gemini salvo que se quiera ese costo y ese flujo de datos.
3. **SEC_USER_AGENT:** usar un nombre y correo que no haya problema en compartir con la SEC.
4. **Claude Code:** permisos normales. Nunca `--dangerously-skip-permissions`.
5. **Actualizaciones:** cualquier cambio a `.claude/` debe re-escanearse con `--baseline .skillspector-baseline.yaml`. Todo hallazgo nuevo se revisa antes de hacer merge.

## 5. Limitaciones

- La revisión manual se dirigió a las líneas marcadas y a barridos de patrones en todo el repo. No fue una lectura línea por línea de las ~27k líneas.
- No se corrió el análisis semántico con LLM de SkillSpector.
- Un escáner y una revisión reducen el riesgo, pero no garantizan que el código sea seguro.
