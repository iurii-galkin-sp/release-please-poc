[English](./TESTING_STRATEGY.md) | [Español](./TESTING_STRATEGY.es.md) | [Русский](./TESTING_STRATEGY.ru.md)
***

# Estrategia de Pruebas Oficial y Plan de Verificación

## 1. Objetivos y Alcance

**Objetivo:** Verificar que el sistema de versionado automatizado implementado y las reglas de protección de ramas funcionan de acuerdo con los procesos documentados, manejan correctamente los escenarios estándar y bloquean acciones erróneas.

**Alcance de las Pruebas:**
*   Configuración de la protección de ramas a través de GitHub Rulesets.
*   Funcionalidad del linting local y del lado del servidor para commits y Pull Requests.
*   El flujo de trabajo de dos fases de extremo a extremo de `release-please.yml`.
*   Corrección de la generación de versiones, actualizaciones de `CHANGELOG.md` y creación de lanzamientos para una estructura de monorepo.
*   Escenarios avanzados, incluyendo reversiones (rollbacks) y validación de la ruta de archivo del commit.

## 2. Roles

*   **"Desarrollador":** Realiza acciones en ramas de característica (feature branches), crea PRs a `dev`.
*   **"Gerente de Lanzamiento":** Realiza fusiones de PR, incluidas las promociones entre `dev`, `stg` y `main`.

## 3. Protocolo de Ejecución de Pruebas

### Parte 0: Prerrequisitos del Entorno de Pruebas (Obligatorio)

Esta configuración inicial debe completarse antes de ejecutar cualquier caso de prueba para garantizar que el entorno local esté configurado correctamente.

| ID | Acción | Comandos a Ejecutar | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **0.1** | Instalar Dependencias y Git Hooks | En la raíz de un repositorio recién clonado, ejecute:<br>`npm install` | 1. El comando se completa con éxito.<br>2. Se crean los directorios `node_modules` y `.husky` en la raíz del proyecto. |

---
### Parte A: Verificación de Protección Local y del Servidor

**Objetivo:** Asegurar que los hooks locales y los Rulesets del servidor funcionen en conjunto para bloquear/permitir acciones correctamente.

| ID | Escenario | Acciones | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **A-1** | Validación **Local** del Formato de Commit | **1. Prueba Negativa (sin ámbito):**<br> `git commit --allow-empty -m "test: this must fail"` <br><br> **2. Prueba Positiva (con ámbito válido):**<br> `git commit --allow-empty -m "test(project): this must pass"` | 1. El comando `git commit` **FALLA**. La consola muestra un error de `commitlint` (`scope may not be empty`). El commit no se crea.<br><br> 2. El comando `git commit` **PASA** con éxito. |
| **A-2** | Push Directo a `dev` está Bloqueado | `git checkout dev && git pull && git commit --allow-empty -m "test(dev): push rejected" && git push` | El comando `git push` **FALLA** сon un error del servidor (`(protected branch hook declined)`). |
| **A-3** | Validación del Ámbito en PR a `dev` **en el Servidor** | Abra un PR a `dev` con el título `test(invalidscope): check server validation`. | La verificación de estado `Check PR Title` **FALLA**. El botón "Squash and merge" está DESHABILITADO. |
| **A-4** | Forzar Estrategia "Squash" en `dev` | Abra un PR válido a `dev`. Abra el menú de fusión. | El menú de fusión **SOLO** muestra la opción "Squash and merge". |
| **A-5** | Forzar Estrategia "Merge Commit" en `stg`/`main` | Abra un PR de `dev` a `stg`. Abra el menú de fusión. | El menú de fusión **SOLO** muestra la opción "Create a merge commit". |
| **A-6** | Bloqueo de Rama Desactualizada en `stg`/`main` | 1. Abra un PR a `stg`.<br>2. Antes de fusionar, fusione otro cambio en `stg`. | El primer PR ahora está **BLOQUEADO**, y aparece el botón "Update branch". |

---
### Parte B: Verificación de la Lógica Central de `release-please`

**Objetivo:** Verificar el ciclo completo de automatización de lanzamiento de dos fases.

| ID | Escenario | Acciones | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **B-1** | Creación de Release PR | Promueva un commit `fix(payment): ...` a `main`. Asegúrese de que el commit modifique un archivo **dentro** del directorio `services/payment/`. | Se crea automáticamente un **nuevo Pull Request** titulado `chore(release): prepare new release`. Contiene actualizaciones para `CHANGELOG.md`, `release-please-manifest.json` y el `version.txt` del componente. |
| **B-2** | Auto-Actualización de Release PR | 1. No fusione el PR de B-1.<br>2. Promueva otro commit `feat(activity): ...` a `main`. | **No se crea un nuevo PR.** El Release PR existente se actualiza automáticamente con un nuevo commit del bot, incluyendo ahora cambios para el componente `activity`. |
| **B-3** | Finalización del Lanzamiento | Fusione el Release PR actualizado de B-2 en `main`. | 1. El flujo de trabajo `release-please` se ejecuta. El job `prepare-release-pr` se **OMITE** (SKIPPED). <br>2. El job `finalize-release` se **EJECUTA y tiene éxito**. <br>3. Se crean nuevos tags de Git (ej., `payment-vX.Y.Z`) y Releases de GitHub correspondientes. |
| **B-4** | "Cancelación de Ruido" y Breaking Change | Promueva un commit `feat(project)!: ...` a `main`, usando `chore(release): ...` para los mensajes de fusión de promoción. | 1. En el nuevo Release PR, el `CHANGELOG.md` **NO CONTIENE** ninguna entrada de `chore(release)`. <br> 2. La versión del componente `project` se incrementa en una versión **MAYOR** (ej., `1.x.x` -> `2.0.0`). |
| **B-5** | Manejo Correcto de `revert` | 1. Promueva un commit `feat(activity): ...` a `main`.<br>2. Inmediatamente después, promueva un commit `revert(activity): ...` a `main`. | En el Release PR resultante: <br>1. La versión del componente `activity` **NO ha cambiado**. <br>2. El `CHANGELOG.md` contiene entradas tanto para el `feat` en la sección "Features" como para el `revert` en una nueva sección "Reverts". |

---
### Parte C: Verificación de la Lógica de Vinculación Commit-Archivo

**Objetivo:** Probar que el sistema está protegido contra lanzamientos "fantasma".

| ID | Escenario | Acciones | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **C-1** | Commit "Fantasma" es IGNORADO | Promueva un commit con el mensaje `fix(payment): phantom commit` a `main`, donde el cambio de código se realiza en un archivo **fuera** del directorio `services/payment/` (ej., en la raíz). | **No se crea ni actualiza ningún Release PR.** El flujo de trabajo `release-please` se ejecuta pero determina correctamente que no hay cambios liberables para el componente `payment`. |
