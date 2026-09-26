# Cinco deploys fallaron con un mensaje de error cierto, preciso y que apuntaba al sitio equivocado

[English](README.md) · **Español**

`node` · `outage` · `2026`

## Contexto

Un servicio Node.js/TypeScript sobre una plataforma de contenedores gestionada. Los secretos
los inyecta la plataforma como variables de entorno, y el servicio los valida al arrancar con
el guard de siempre: un `validateRequiredEnvVars()` que lee `process.env` y aborta con
estruendo si falta algo.

Esta entrada aplica a cualquier runtime donde un error de configuración ausente sea lo
*primero* que ves, y donde la configuración está demostrablemente presente.

## Qué falló

Cinco deploys consecutivos murieron con la misma línea:

```
GEMINI_API_KEY not configured
```

Lo engañoso es que el mensaje era **cierto**. Cuando el validador se ejecutaba, la variable
realmente no estaba en `process.env`. Y era **preciso**: el nombre correcto de la variable,
desde el guard correcto, en el punto correcto del arranque. Nada en él era una pista falsa en
el sentido habitual.

Y sin embargo la plataforma la estaba inyectando. La configuración de la revisión listaba el
secreto. El secreto existía y tenía versión vigente. Volcar el entorno desde una shell sobre
la misma imagen mostraba el valor.

Así que la investigación fue adonde apuntaba el mensaje: almacén de secretos e IAM. Se
concedió un rol de acceso redundante a la identidad de ejecución — no cambió nada. Se
revisó un permiso, se reaplicó un binding, se reversionó el secreto, se redesplegó la
revisión. Cinco revisiones quemadas. Unos noventa minutos.

## Por qué

En algún punto del código, un getter hacía esto:

```ts
delete process.env.GEMINI_API_KEY;   // "defensivo": leer una vez y borrar
```

La intención era evitar filtrar la clave en un volcado posterior del entorno. Inofensivo por
sí solo. El problema es *cuándo* se ejecutaba.

Al getter se llegaba desde el inicializador de una propiedad de clase. Esa clase se
instanciaba en el **nivel superior del módulo** en un fichero de rutas:

```ts
// routes/something.ts
const service = new SomeService();   // corre durante el import, no durante el arranque
```

El código de nivel superior de un módulo se ejecuta mientras se resuelve el grafo de módulos
— durante el `import`, antes de que corra una sola línea de tu `main()`. La secuencia era:

1. El `import` recorre los módulos de rutas.
2. `new SomeService()` ejecuta sus inicializadores de propiedad.
3. Uno de ellos lee la clave a través del getter, que la borra de `process.env`.
4. `main()` por fin arranca y llama a `validateRequiredEnvVars()`.
5. La variable ya no está. El validador acierta. El mensaje acierta. El diagnóstico no.

La plataforma sí inyectaba la variable. La aplicación la eliminaba unos milisegundos antes de
comprobar que estuviera. Ninguna cantidad de trabajo sobre IAM habría encontrado esto, porque
el error no estaba donde se reportaba.

## El patrón

La disciplina que habría encontrado esto en diez minutos en vez de noventa:

- **Causa raíz antes que cualquier arreglo.** Un arreglo aplicado a una causa no confirmada es
  una conjetura, y una conjetura que por casualidad cambia el comportamiento es peor que una
  que no lo cambia: te enseña algo falso. El rol de IAM era una conjetura. «Funcionó» en el
  sentido de que se aplicó limpiamente, y alejó la investigación de la respuesta.
- **Una hipótesis y un cambio mínimo por intento.** Si cambian dos cosas y el síntoma se
  mueve, no has aprendido nada de ninguna de las dos.
- **Tres arreglos fallidos significan que el modelo está mal, no el arreglo.** Deja de
  parchear. La pregunta ya no es «por qué falta esta variable» sino «quién más toca
  `process.env` para esta clave, y cuándo». Esa pregunta está a un grep de distancia:

  ```bash
  grep -rn "process\.env\.MY_VAR\|delete process\.env" src/
  ```

- **Rastrea el valor hacia arriba, no el error hacia abajo.** El error te dice dónde el valor
  está *ausente*. Localiza cada sitio donde se escribe, se lee o se elimina, y ordénalos por
  momento de ejecución. En Node, «momento de ejecución» incluye todo el grafo de imports, que
  corre antes de tu entrypoint.
- **Instrumenta fronteras, no interioridades.** Una línea de log al principio del entrypoint y
  otra al principio de cada módulo que toca el valor habrían mostrado el borrado ocurriendo
  antes de la validación, sin necesidad de razonar nada más.
- **Separa una regresión real de un fallo preexistente antes de depurarlo.** Cuando una suite
  se pone en rojo mientras estás a medio cambio, haz stash exactamente del fichero que tocaste
  y vuelve a lanzarla:

  ```bash
  git stash push -- src/path/to/changed-file.ts
  npm test -- path/to/failing.spec.ts
  git stash pop
  ```

  ¿Sigue en rojo? Ya estaba roto y estás persiguiendo el bug de otro. ¿Verde? Es tuyo.

## Cómo verificarlo

Dos comprobaciones, menos de cinco minutos.

**¿Algo muta el entorno antes que tu validador?** Imprime el orden. Pon esto como primerísima
sentencia de tu entrypoint, por encima de todos los demás imports:

```ts
// entrypoint: first line, above all other imports
console.log('[boot] pre-import env keys:', Object.keys(process.env).length);
process.on('exit', () => console.log('[boot] exiting'));
```

Añade después un segundo conteo justo tras los imports. Si el número baja, algo del grafo de
imports está borrando variables.

**¿Algo se instancia en el ámbito del módulo?** En un código que no escribiste tú, esto
encuentra a los candidatos:

```bash
grep -rn "^const .* = new " src/ --include="*.ts"
grep -rn "delete process\.env" src/
```

Cualquier resultado del primer comando es código que corre durante el `import`. Cualquier
resultado del segundo, en un fichero alcanzable desde el primero, es este bug.

## Véase también

- [Un guard de deduplicación que solo mira el histórico no puede ver el duplicado que está creando ahora mismo](../bounded-agent-loops/README.es.md)
  — otro caso donde el primer instinto («la fuente de datos está mal») apuntaba lejos del
  mecanismo real.
