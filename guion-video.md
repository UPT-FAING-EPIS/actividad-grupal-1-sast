# Guion del video · Actividad Grupal 1 SAST · máximo 5 minutos

**Duración objetivo: 4 min 30 s.** Margen bajo el límite.

Grabación de pantalla con voz en off. El texto es lo que se dice; la acción es
lo que se ve. Todo lo de este guion está verificado contra el repositorio: si
una pantalla no coincide, es que algo cambió, no que el guion esté mal.

> **No muestres `app/main.go` en el editor durante mucho tiempo**: es código
> deliberadamente inseguro. Se ve, pero sin leerlo línea por línea.

---

## 0:00 — Apertura

**Voz:**
> En los laboratorios analizamos el código de una aplicación con SonarQube,
> Semgrep y Snyk. En esta actividad cambiamos el punto de mira: usamos
> **gosec** y **OWASP Dependency-Check**, dos herramientas que no usamos en los
> laboratorios. Y lo que resultó más interesante no fue el informe: fue lo que
> pasó al intentar que el escaneo se repitiera solo en cada push.

**Pantalla:** el repositorio en GitHub.

---

## 0:25 — La aplicación de prueba

**Voz:**
> Para comprobar que gosec detecta de verdad, escribimos una aplicación a
> propósito con los fallos más comunes. No es una aplicación de producción: es
> un objetivo de prueba, y el README lo advierte.

**Pantalla:** `app/README.md`, luego el archivo `app/main.go` en el árbol.

```bash
# 148 líneas, un solo archivo, seis rutas con un fallo cada una
/saludo   plantilla sin escapar y redirección con datos del usuario
/descarga escritura en ruta construida sin restringir
/archivo  lectura de archivo con la ruta de la petición
/token    secreto concatenado sin validar
/tipo     comando del sistema con valor del usuario
/hash     MD5 y SHA1 para derivar contraseñas
```

---

## 0:55 — El escaneo, y por qué se ve en GitHub Actions

**Voz:**
> El escaneo no se grabs a mano: corre en cada push, en cada pull request y
> una vez por semana, porque la base de avisos cambia a diario.

**Pantalla:** pestaña **Actions**, el último run en verde.

**Voz:**
> Y este es el punto que más nos costó. Si abrierte la terminal y escribieras
> gosec, verías los 17 hallazgos en un segundo. Lo que no se ve es si el paso
> automático está escaneando **lo mismo**.

---

## 1:20 — Los 17 hallazgos

**Pantalla:** el paso "Resumen legible" del run, y luego el artefacto.

**Voz:**
> El resultado real: **17 hallazgos. Cinco de severidad alta, once medios, uno
> bajo.** Es el mismo número que da el escaneo manual y el que da el pipeline.
> Que coincidan no es casualidad: es la comprobación de que el automatismo
> está mirando lo mismo que la persona.

| Severidad | Cantidad |
| --- | ---: |
| HIGH | 5 |
| MEDIUM | 11 |
| LOW | 1 |

**Voz:**
> Lo interesante está en las reglas. Cuatro de los hallazgos no salen de buscar
> una palabra clave: gosec sigue el valor desde que entra en la función hasta
> donde se usa.

**Pantalla:** resaltar en la tabla G702, G703, G705, G710.

> G204 dice "subproceso con variable", que es cierto pero genérico. **G702, sobre
> la misma línea, dice inyección de comandos**, porque sabe que el valor viene
> de la petición sin validar. Es la diferencia entre un filtro por patrón y un
> análisis de flujo de datos.

---

## 2:05 — El fallo que más nos costó: un escaneo que no ocurrió

**Voz:**
> Y aquí está el hallazgo más importante del trabajo, que no tiene nada que ver
> con vulnerabilidades.

**Pantalla:** el log del paso, mostrando la línea de resumen.

**Voz:**
> Después de arreglar los workflows, el paso pasaba en verde. Pero el informe
> decía cero hallazgos. No era que el código estuviera limpio: era que **gosec
> no había escaneado nada**.

**Pantalla:** el `go.mod` dentro de `app/`.

> La causa: `go.mod` está en `app/`, no en la raíz del repositorio. Al ejecutar
> gosec desde la raíz no hay módulo que resolver, y la herramienta responde
> `Files: 0` **sin explicar por qué**. Cero hallazgos y escaneo inexistente se
> ven exactamente igual en el log.

**Pantalla:** el paso de resumen del workflow.

> La corrección de verdad no es el `working-directory`. Es no confiar en el cero:

```python
if not iss:
    raise SystemExit("::error::0 hallazgos: el escaneo no ocurrio, no salio limpio")
```

**Voz:**
> Un escaneo que no encuentra nada y uno que no se ejecutó producen el mismo
> archivo. Si el pipeline no los distingue, no está midiendo nada.

---

## 2:55 — Los otros cinco fallos, todos de automatización

**Voz:**
> Ninguno de los otros fallos estaba en el código de la aplicación. Estaban en
> el pipeline.

**Pantalla:** `security.yml`, recorriendo los jobs. Mencionar rápido, sin
detenerse en cada uno:

| Qué pasaba | Error real |
| --- | --- |
| El workflow era de otro proyecto | `MSBUILD : error MSB1009: Project file does not exist` |
| Una acción que dejó de existir | `Unable to resolve action tfsec-action@v1` |
| Un secret sin registrar | `--token ""` → *"missing a value"* |
| El script de instalación desapareció | `404: Not Found` ejecutado por bash |
| El destino de salida no existía | `Invalid 'out' argument: path does not exist` |

**Voz:**
> Uno merece comentario: el del secret. El comando era `vercel deploy --token ""`
> y el error que salía hablaba de sintaxis de comandos, cuando el problema era
> de configuración. Un mensaje que no nombra la causa cuesta más que el fallo:
> el secret vacío nos costó una ejecución entera.

**Pantalla:** el paso "Verificar secrets de Vercel" del workflow.

> Ahora el workflow falla nombrando el secret que falta.

---

## 3:40 — Publicar el informe, no solo el código

**Voz:**
> El segundo workflow corre el mismo gosec y publica el resultado como
> aplicación web, para que el informe se pueda leer sin clonar nada.

**Pantalla:** <https://sast-taskflow-iota.vercel.app>, recorriendo la tabla por
severidad.

**Voz:**
> La URL se despliega sola en cada push. El detalle que costó encontrar: el CLI
> de Vercel no recibe el proyecto por variables de entorno, las lee de
> `web/.vercel/project.json`. Con solo el token y el scope, el comando responde
> que olvidaste el ID del proyecto, o peor, despliega en otro sitio.

---

## 4:10 — Lo que la automatización no puede hacer sola

**Voz:**
> Dos cosas quedan fuera del alcance de un pipeline, y conviene decirlas.

**Pantalla:** el paso de Dependency-Check, con su aviso en ámbar.

> La primera: Dependency-Check descarga la base de avisos de la NVD, que supera
> los 400 mil registros. Sin una API key la descarga tarda **horas**, más de lo
> que dura el job. Lo que hicimos fue no inventar el resultado: si el informe no
> existe, el workflow escribe la constancia y lo dice, en vez de publicar un
> cero que nadie midió. Con una API key gratuita el escaneo real se completa
> sin tocar nada.

**Voz:**
> Y la segunda, la más incómoda: automatizar el escaneo garantiza que se
> repita, **no que acierte**. Un análisis estático no ve lógica de negocio
> incorrecta si no hay un patrón peligroso detrás.

---

## 4:30 — Cierre

**Voz:**
> Resultado: 17 hallazgos correctamente clasificados, cinco de severidad alta,
> sobre una aplicación escrita para contenerlos. Y cinco fallos de automatización
> que ninguno estaba en el código.

> Lo que queda fuera de alcance, y no es menor: la base de avisos completa no
> cabe en un runner sin API key. Un análisis de dependencias que no puede
> descargar la base de datos no es un análisis de dependencias, y conviene que
> el pipeline lo diga en lugar de mostrar un cero.

**Pantalla:** el repositorio y los dos artículos del equipo.

---

## Notas de producción

- **Duración estimada:** 4 min 30 s. Margen bajo el límite de 5.
- **Terminal en 1080p**, escalada al 80 %, para que el texto se lea sin ampliar.
- **Sin audio de fondo** en los segmentos de terminal: distrae.
- **Subtítulos** en el segmento de la tabla de hallazgos, porque se lee rápido.
- **Si el run de Actions tarda**, abre el artefacto `gosec` ya descargado en vez
  de esperar en vivo.
- **No muestres los secretos** ni abras los logs de los runs en rojo: ahí se
  ve `--token` con el valor censurado, que es feo pero no grave. Mejor solo el
  run en verde.
- **Comprobación antes de grabar:** que la app de Vercel cargue y que el run más
  reciente esté en `success`. Si el run está en rojo, el guion miente.