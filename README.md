# Actividad Grupal 1 · SAST

Análisis de vulnerabilidades de una aplicación con herramientas que **no** se
usaron en los laboratorios: **OWASP Dependency-Check** y **gosec**.

Los laboratorios de la unidad employaron SonarCloud, Semgrep, Snyk y tfsec.
Estas dos cubren otro terreno: las dependencias de terceros y el análisis
estático con reglas mapeadas a CWE.

## Contenido

| Ruta | Qué es |
|---|---|
| `app/` | Aplicación Go deliberadamente insegura, caso de estudio |
| `articulos/` | Los tres artículos del equipo, escritos para publicar |
| `informe-gosec.json` | Informe real del escaneo: 17 hallazgos |
| `web/` | Informe de SAST publicado como aplicación web |

## Enlaces

| | |
|---|---|
| **Aplicación publicada** | https://sast-taskflow-iota.vercel.app |
| **Repositorio** | https://github.com/UPT-FAING-EPIS/actividad-grupal-1-sast |
| **Artículo 1** (Nicole Rios Cohaila) | https://dev.to/korins707/los-bugs-mas-caros-del-escaneo-de-vulnerabilidades-no-estan-en-el-codigo-estan-en-el-pipeline-lf0 |
| **Artículo 2** (Patrick Rodriguez Cardenas) | https://dev.to/dejameingresar/auditar-dependencias-con-owasp-dependency-check-ei8 |
| **Artículo 3** (Patrick Rodriguez Cardenas) | https://dev.to/dejameingresar/detectar-vulnerabilidades-en-go-con-gosec-2h46 |

## Resultados

### gosec sobre `app/`

17 hallazgos: **5 HIGH, 11 MEDIUM, 1 LOW**.

| Regla | Qué detecta |
|---|---|
| G101 | Credencial y clave privada RSA escritas en el código |
| G702 | Inyección de comandos (análisis de taint) |
| G703 | Recorrido de rutas (análisis de taint) |
| G705 | XSS (análisis de taint) |
| G710 | Redirección abierta (análisis de taint) |
| G112 | Slowloris por falta de timeouts |
| G202 | SQL por concatenación de cadenas |
| G204 | Subproceso con variable |
| G304 | Inclusión de archivo vía variable |
| G401 / G501 / G505 | MD5 y SHA1 |
| G104 | Errores sin manejar |

```bash
# requiere el compilador de Go
curl -sSfL https://raw.githubusercontent.com/securego/gosec/master/install.sh | sh
cd app && gosec -no-fail -fmt=json -out=informe.json ./...
```

**Advertencia:** sin el compilador de Go, gosec termina sin errores y reporta
`Files: 0`. Eso no es un escaneo limpio: es un escaneo que no ocurrió.

### OWASP Dependency-Check

Revisa el grafo de dependencias contra la base de avisos de la OWASP. La
primera ejecución descarga la base de la NVD, que supera los 400.000 registros;
sin API key tarda horas. Con una key gratuita baja a minutos:
https://nvd.nist.gov/developers/request-an-api-key

## Automatización

`.github/workflows/security.yml` corre los escaneos en cada `push`, en cada pull
request y una vez por semana, porque la base de avisos cambia a diario.

## Advertencia

`app/` contiene código inseguro **a propósito**, escrito para la demostración.
No desplegar, no exponer, no usar con datos reales.

## Artículos

- [Auditar dependencias con OWASP Dependency-Check](articulos/01-dependency-check.md)
- [Detectar vulnerabilidades en Go con gosec](articulos/02-gosec.md)
- [Los bugs más caros del escaneo de vulnerabilidades no están en el código, están en el pipeline](articulos/03-automatizacion-del-escaneo.md)