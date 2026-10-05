# Los bugs más caros del escaneo de vulnerabilidades no están en el código, están en el pipeline

En los laboratorios de la unidad revisamos código con SonarQube, Semgrep, Snyk y
tfsec. Para esta actividad cambiamos el punto de mira: **OWASP Dependency-Check**
(dependencias de terceros, base de avisos de la OWASP Foundation) y **gosec**
(análisis estático de Go con reglas mapeadas a CWE). Ninguna de las dos se usó en
los laboratorios.

La parte interesante no fue el informe final. Fue lo que pasó al intentar que el
escaneo se repitiera solo en cada `push`.

<!-- more -->

## Los enlaces del proyecto

- **Repositorio público con el código, los workflows y el informe:**
  <https://github.com/UPT-FAING-EPIS/actividad-grupal-1-sast>
- **Informe de SAST publicado como aplicación web (Vercel, despliegue por
  GitHub Actions):** <https://sast-taskflow.vercel.app>
- **Artículo del equipo sobre gosec:** [*Detectar vulnerabilidades en Go con
  gosec*](https://dev.to/dejameingresar/detectar-vulnerabilidades-en-go-con-gosec-2h46)

## El resultado que importa

La aplicación de caso de estudio (`app/main.go`) está escrita **a propósito** con
las clases de fallo más comunes. Es un objetivo de prueba, no una aplicación de
producción.

```bash
gosec -no-fail -fmt=json -out=informe.json ./...
```

| Severidad | Cantidad |
| --- | ---: |
| HIGH | 5 |
| MEDIUM | 11 |
| LOW | 1 |
| **Total** | **17** |

Cinco de los altos son una credencial escrita en el código, una clave privada RSA
embebida, y tres encontrados por **análisis de taint** (inyección de comandos,
recorrido de rutas, XSS), que no aparecen buscando palabras clave: la herramienta
sigue el valor desde que entra en la función hasta donde se usa.

Ese 17/5/11/1 es el mismo número que produce el escaneo manual y el que produce
el pipeline. Que coincidan no es casualidad: es la comprobación de que el
automatismo está escaneando lo mismo que la persona.

## El fallo más caro: un escaneo que no ocurrió pareciendo limpio

gosec **no necesita el compilador de Go para no fallar**. Sin él, el comando
termina con código de salida `0` y escribe:

```
Archivos: 0 | Lineas: 0 | Hallazgos: 0
```

Eso es indistinguible de "el código no tiene nada". Un panel que muestra una
casilla verde y un `0` no está diciendo que el código sea seguro: está diciendo que
no miró nada.

Esto lo sufrimos de verdad. Corregimos el workflow para que escaneara
`./app/...`, lo subimos, y el paso pasó en verde. El informe, en cambio, decía
cero hallazgos. La causa: **`go.mod` vive en `app/`, no en la raíz del
repositorio**, así que al ejecutar gosec desde la raíz no había módulo que
resolver y el analizador no tenía nada que cargar:

```yaml
# mal: sin modulo en la raiz, gosec responde "Files: 0" sin explicar por que
- run: gosec -no-fail -fmt=json -out=informe.json ./app/...

# bien: hay que escanear desde el directorio del modulo
- name: Escanear
  working-directory: app
  run: gosec -no-fail -fmt=json -out="$GITHUB_WORKSPACE/informe.json" ./...
```

La corrección que de verdad importa no es el `working-directory`. Es no trusting
el cero:

```python
if not iss:
    raise SystemExit("::error::0 hallazgos: el escaneo no ocurrio, no salio limpio")
```

Un escaneo con cero hallazgos y un escaneo que no se ejecutó producen el mismo
archivo. Si el pipeline no los distingue, no está midiendo nada: está
decorándose. Y una automatización que miente es peor que no tener automatización,
porque ocupa el lugar de una revisión.

## Los otros cuatro fallos, todos de automatización

Ninguno estaba en el código de la aplicación. Los cuatro estaban en el pipeline.

**1. El workflow estaba escrito para otro proyecto.** El `security.yml`apuntaba
`dotnet restore TaskFlow.Api/TaskFlow.Api.csproj` sobre un repositorio que no
tenía ese proyecto, y escaneaba `seguridad/banco-vulnerable/`, un directorio que
tampoco existía. El error que salía era `MSBUILD : error MSB1009: Project file
does not exist`, que señala un problema de rutas pero no dice que el archivo de
workflow está describiendo otro repositorio. Cuatro ejecuciones seguidas en rojo
por lo mismo.

**2. Una acción que dejó de existir.** El job de tfsec usaba
`aquasecurity/tfsec-action@v1`, y el runner respondió `Unable to resolve action
aquasecurity/tfsec-action@v1, unable to find version v1`. Las etiquetas de las
acciones son referencias: pueden desaparecer, y cuando lo hacen el job no falla
por código, falla al no resolverse.

**3. Un secret ausente que se veía como otro problema.** El despliegue ejecutaba

```
npx vercel deploy --prod --yes --token ""
```

con `VERCEL_TOKEN` sin registrar, y la CLI respondía `Error: You defined
"--token", but it's missing a value`. Un mensaje que habla de sintaxis de
comandos cuando el problema es de configuración. La versión que obliga a nombrar
el secreto que falta:

```yaml
- name: Verificar secrets de Vercel
  env:
    VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
    VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  run: |
    for v in VERCEL_TOKEN VERCEL_ORG_ID; do
      if [ -z "${!v}" ]; then
        echo "::error::Falta el secret $v (Settings > Secrets > Actions)"
        exit 1
      fi
    done
```

**4. Un `.gitignore` que se comió un resultado.** El `.gitignore` traía
`informe.json`, el mismo nombre que produce el escaneo. El artefacto se generaba,
se subía, y desaparecía antes de llegar a ninguna parte.

## Lo que la automatización no puede hacer sola

Dos cosas de este ejercicio quedan fuera del alcance de un pipeline, y conviene
decirlo en vez de dejarlas para cuando falle.

**La base de avisos de la NVD no cabe en un runner.** Dependency-Check descarga
la base de la NVD, que supera los 400.000 registros. Sin una API key la descarga
inicial tarda **horas**; con una key gratuita baja a minutos
(<https://nvd.nist.gov/developers/request-an-api-key>). En un job con límite de
tiempo, la primera ejecución nunca va a completar. Lo que decidimos fue no
inventar el resultado: si el informe no existe, el workflow escribe la constancia
y lo dice con un `::warning::` en vez de publicar un cero. Registrando el secret
`NVD_API_KEY`, el escaneo real se completa sin tocar el workflow.

**Las métricas solo existen si alguien las midió.** El artefacto `informe-gosec.json`
está versionado en el repositorio, con fecha y entorno, precisamente para que el
número que se cita en el artículo se pueda contrastar contra el archivo. Un
escaneo que corre y nadie guarda no deja nada que revisar.

## Publicar el informe, no solo el código

El informe de escaneo sirve poco si leerlo exige clonar el repositorio y abrir un
JSON. El segundo workflow corre el mismo gosec en cada `push` a `web/**` y publica
el resultado como aplicación web en Vercel, con el detalle por severidad y regla.

El detalle que costó encontrar: **la CLI de Vercel no recibe el proyecto por
variables de entorno.** Las lee de `web/.vercel/project.json`. Con solo `--token`
y `--scope` el comando responde `You specified VERCEL_PROJECT_ID but you forgot
VERCEL_ORG_ID`, o peor, despliega en un proyecto que no es el que creías. El
workflow ahora escribe ese archivo con los secrets:

```bash
mkdir -p .vercel
printf '{"projectId": "%s", "orgId": "%s"}\n' "$VERCEL_PROJECT_ID" "$VERCEL_ORG_ID" > .vercel/project.json
npx --yes vercel@25 deploy --prod --yes --token "$VERCEL_TOKEN"
```

## Limitación honesta

Automatizar el escaneo garantiza que se repita, no que acierte. Estas
automatizaciones se ejecutan en cada `push`, en cada *pull request* y una vez por
semana, porque la base de avisos cambia a diario y una vulnerabilidad puede
publicarse después del último `push`. Aun así:

- **Un análisis estático no ve lógica de negocio incorrecta** si no hay un patrón
  peligroso detrás. La base de este ejercicio lo demuestra: una aplicación
  deliberadamente insegura en clase de datos, pero sin ninguna regla de las que
  busca gosec, daría un informe limpio sin tener un solo problema resuelto.
- **El caso de estudio no se despliega y no debe usarse con datos reales.**
  Publicar el informe es seguro; publicar la aplicación vulnerable, no.
- **`gosec` solo analiza Go.** Sobre un proyecto en C# devuelve `Files: 0`, que es
  el mismo cero sin información del que hablaba antes.

## Conclusiones

1. **El escaneo que no se ejecutó y el escaneo limpio producen el mismo archivo.**
   La primera defensa no es la herramienta, es un paso que compare `0` contra
   "no se pudo ejecutar" y aborte si no puede distinguirlos.
2. **En un pipeline, la mayoría de los fallos son de rutas, de secretos y de
   versiones, no de vulnerabilidades.** Ninguno de los cuatro fallos de este
   artículo estaba en el código de la aplicación.
3. **Un mensaje de error que no nombra la causa cuesta más que el fallo.** El
   `missing a value` de Vercel ocultaba un secret sin registrar durante una
   ejecución completa.
4. **Un resultado sin su archivo no es evidencia.** Versionar el informe con fecha
   y entorno es lo que permite que alguien lo contraste en lugar de creerlo.
5. **La automatización multiplica el valor de un control, y también multiplica
   el alcance de un error de configuración.** Un escaneo manual equivocado se
   detecta en la revisión. Un escaneo automático que no escanea pasa desapercibido
   mientras todo luzca verde.

Lo que queda fuera de alcance, y no es menor: la base de avisos completa no cabe
en un runner sin API key. Un análisis de dependencias que no puede descargar la
base de datos no es un análisis de dependencias, y conviene que el pipeline lo
diga en lugar de mostrar un cero.