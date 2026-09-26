# `promisify` captura la referencia a la función al importar — tu mock llega tarde

[English](README.md) · **Español**

`jest` · `node` · `correctness` · `2026`

## Contexto

Un módulo de Node que envuelve una herramienta de línea de comandos — un binario de OCR,
`ffmpeg`, `git`, cualquier cosa a la que invoques por shell. La forma idiomática de
escribirlo pone una llamada a `promisify` en el ámbito de módulo:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);   // se ejecuta una vez, al importar

export async function isToolAvailable(): Promise<boolean> {
  try {
    await execFileAsync("mytool", ["-v"]);
    return true;
  } catch {
    return false;
  }
}
```

Entonces un test necesita comprobar qué pasa cuando el binario no está en el runtime —
un escenario real, porque la imagen del contenedor y el portátil del desarrollador casi
nunca coinciden en qué binarios existen. El movimiento obvio es mockear `child_process`.

## Qué falló

El test mockeaba `child_process` de forma que `execFile` fallara, y comprobaba un rechazo.
La función resolvía en su lugar — con un valor real, de un subproceso real.

```
expect(received).rejects.toThrow()
Received promise resolved instead of rejected
```

La parte engañosa es que el mock no está roto y Jest no se está portando mal. Si haces un
log dentro del test, `require("node:child_process").execFile` es efectivamente el mock. La
aserción es correcta. Simplemente, el módulo bajo prueba no lo está usando.

Peor en una máquina donde el binario *sí* está instalado: el test no falla con ruido, sino
que invoca calladamente la herramienta real. En CI, donde el binario puede no estar, el
mismo test pasa por el motivo equivocado. Es un test que mide la máquina, no el código.

## Por qué

`promisify(execFile)` es una llamada, no una referencia. Se ejecuta en el instante en que
el módulo se importa por primera vez, y lo que devuelve es una función nueva que ha
capturado en su clausura *el valor de `execFile` en ese momento*.

El orden, entonces:

1. El fichero de test importa el módulo bajo prueba (directa o transitivamente).
2. Se ejecuta `const execFileAsync = promisify(execFile)`. El original queda capturado.
3. `jest.doMock("node:child_process", …)` sustituye el export del módulo.
4. Las llamadas pasan por `execFileAsync`, que sigue sosteniendo el original.

Jest puede sustituir lo que el módulo `child_process` *exporta*. No puede meter la mano en
una clausura que ya se copió fuera el valor viejo. La misma trampa aplica a cualquier
`const x = someModule.fn` en el ámbito de módulo — `promisify` es solo la forma más
frecuente de escribir uno sin darte cuenta de que lo has escrito.

El hoisting esconde el orden. `jest.mock` se eleva por encima de los imports y aquí sí
funcionaría; `jest.doMock` no se eleva, que es precisamente por lo que pierde esta
carrera. Cambiar de uno a otro para que el test pase es tratar el síntoma.

## El patrón

Mockea el módulo que *posee* el envoltorio, no la primitiva que envuelve. Esa es la
frontera de la que tu código depende de verdad:

```ts
test("rechaza cuando la herramienta no está instalada en el runtime", async () => {
  jest.doMock("../src/services/tool.service", () => ({
    isToolAvailable: jest.fn().mockResolvedValue(false),
    runTool: jest.fn().mockRejectedValue(
      new Error("mytool is not installed in this runtime"),
    ),
  }));

  // Importar DESPUÉS del doMock — es lo que hace que doMock sirva de algo.
  const mod = await import("../src/services/tool.service");

  await expect(mod.runTool(Buffer.from("input")))
    .rejects.toThrow("mytool is not installed");
});
```

Si de verdad necesitas ejercitar la lógica propia del envoltorio — reintentos,
construcción de argumentos, parseo de stderr — entonces no captures la referencia en el
ámbito de módulo, de entrada. Resuélvela en cada llamada: es mockeable y no cuesta nada
medible:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

export async function runTool(args: string[]) {
  // promisify en tiempo de llamada: lee lo que child_process exporte en ese momento
  return promisify(execFile)("mytool", args);
}
```

O inyéctala, lo que deja la dependencia visible en la firma y elimina del todo la
necesidad de mockear módulos:

```ts
export async function runTool(
  args: string[],
  exec = promisify(execFile),   // argumento por defecto, evaluado en cada llamada
) {
  return exec("mytool", args);
}
```

La regla, enunciada en general: **todo lo que se captura en un `const` a nivel de módulo
queda congelado antes de que tu test se ejecute.** Mockea al nivel del dueño de ese
`const`, o deja de capturar.

## Cómo verificarlo

**Encuentra el patrón en un código base.** Un `promisify` en ámbito de módulo es un solo
grep — la pista es que no lleva indentación:

```bash
grep -rn "^const .* = promisify(" src/
```

Cada coincidencia es un módulo cuyos mocks de `child_process` no harán nada, en silencio.

**Demuestra que un test sospechoso no prueba lo que dice.** Añade al mock un marcador que
solo puede aparecer si el mock se usó:

```ts
jest.doMock("node:child_process", () => ({
  execFile: (...args: unknown[]) => {
    throw new Error("MOCK_WAS_REACHED");
  },
}));
```

Ejecuta el test. Si falla con `MOCK_WAS_REACHED`, el mock está bien conectado y el test es
real. Si falla con cualquier otra cosa — o pasa — el mock nunca se consultó, y el test
llevaba tiempo aprobando gracias a los binarios instalados en la máquina.

**Caza la divergencia CI-frente-a-portátil.** Ejecuta la suite con el binario fuera del
`PATH`; un test que se comporta distinto está probando el entorno:

```bash
PATH=/usr/bin:/bin npx jest path/to/tool.service.test.ts
```

## Véase también

- [Cloud Run pierde el estado en memoria en cada reinicio](../cloudrun-sync-status-persistence/README.es.md) —
  otra consecuencia de la misma costumbre: valores capturados una sola vez, a nivel de
  módulo, que todo lo de abajo da por vivos.
