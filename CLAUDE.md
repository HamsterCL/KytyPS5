# CLAUDE.md

Contexto de este repo para futuras sesiones. Detalle completo en `docs/fork-workflow.md`.

## Qué es

Fork personal (`HamsterCL/KytyPS5`) del emulador de PS5 `KytyPS5/KytyPS5`. Clon local en `D:\EmuPS5`,
Windows 11, shell PowerShell. Idioma de trabajo con el usuario: español.

## Remotos y ramas

- `origin` = fork (HamsterCL/KytyPS5), `upstream` = KytyPS5/KytyPS5.
- `main` es espejo de `upstream/main`: no se le hacen commits propios.
- `integrated` tiene los cambios propios. Se actualiza con **merge** de `origin/main` (no rebase).
- Estado a 2026-10-16: `integrated` en `113b44f5`, publicado en `origin/integrated`.

## Lo realizado hasta ahora

1. Se sincronizó `main` del fork con upstream usando `gh repo sync`.
2. Se clonó el fork en `D:\EmuPS5` y se agregó el remoto `upstream`.
3. Se mergeó `origin/main` en `integrated` (`d7cc0793`). Conflictos resueltos en
   `.github/workflows/build.yml`, `spirvEmitterFlow.cpp` y `src/kernel/fileSystem.cpp`
   (detalle en `docs/fork-workflow.md`).
4. Se instaló el toolchain de build y se compiló `launcher` más los targets de test.
5. Se verificó que `kernel_file_system` fallaba también en `origin/main` con este toolchain, y se
   arregló `KernelTruncate` para devolver `ENOTDIR` cuando un padre es un archivo regular (`113b44f5`).
6. Resultado: 14 de 14 tests pasan en `integrated`.
7. Primera prueba con Astro Bot: crash en el hilo `SceSndzAudioOutMain` (puntero corrupto con `1.0f` en la
   mitad alta) y ~13 fps. Se dejó un build limpio de `upstream/main` en `D:\EmuPS5_main` para el A/B.
8. Se actualizaron 8 submódulos de `3rdparty/` a su último release estable (`b66dd6c7`, rama `deps/update`
   mergeada en `integrated` y publicada). Compila y 14/14 tests; sin probar en ejecución.

## Para retomar

Leer primero "Estado actual y pendientes" en `docs/fork-workflow.md`. Resumen: falta que el usuario compare
Astro Bot en `D:\EmuPS5_main\_Build\windows\install` (upstream limpio) contra
`D:\EmuPS5\_Build\windows\install` (integrated, build anterior a las dependencias nuevas) y reporte crash
y fps. Según eso: bisecar commits de AJM en `integrated`, reportar a upstream, o probar un build con LTO.

## Compilar y probar (Windows)

- CMake y Ninja vienen con VS 18 Community y **no están en el PATH**. Hay que cargar
  `VC\Auxiliary\Build\vcvars64.bat` y anteponer al PATH:
  `...\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin`, `...\CMake\Ninja`,
  `C:\Program Files\LLVM\bin`, `D:\glslang\x\bin`.
- Qt: `C:\Qt\6.10.3\msvc2022_64` (pasar en `CMAKE_PREFIX_PATH`). Compilador: `clang-cl`.
- Directorio de build: `_Build/windows` (ignorado por git). El comando completo está en
  `docs/fork-workflow.md`; el CI de referencia es `.github/workflows/build.yml`.
- Un build limpio de todos los targets tarda varios minutos; compilar un solo target de test es rápido.

## Cosas a tener en cuenta

- Antes de hacer `git push`, confirmar con el usuario; los pushes a `origin integrated` se hicieron
  solo cuando lo pidió.
- En PowerShell 5.1, `git push` imprime progreso por stderr y se ve como error aunque haya funcionado:
  revisar la línea `a..b rama -> rama` y `git status -sb`.
- Los submódulos de `3rdparty/` hay que inicializarlos en cada clon o worktree nuevo.
- `integrated` ya está en uso en `D:\EmuPS5`, así que un worktree de esa rama debe ir con `--detach`.
- Vigilar en futuros merges la resolución de `EmitConditionRef` en `spirvEmitterFlow.cpp`
  (se usó `Result(inst)` de `main` en vez del `const_cast` de `integrated`).
- PR del fix de `KernelTruncate` abierto: https://github.com/KytyPS5/KytyPS5/pull/1577 (rama `fix/truncate-enotdir`, parte de `upstream/main`).
  Con este toolchain `kernel_file_system` sigue fallando en `main` en otro check ("Net unknown socket options...").
- Los PR a upstream salen siempre de una rama aparte basada en `upstream/main`, nunca de `integrated`.
- Commits y PR solo con el nombre del usuario: sin trailers `Co-Authored-By` ni menciones a la IA.
- El aviso de GitHub "had recent pushes" en `integrated` no es un PR ni se puede cerrar; se ignora (ver `docs/fork-workflow.md`).
- Worktree auxiliar vigente: `D:\EmuPS5_main` (detached en `upstream/main`, para el A/B). Borrar con
  `git worktree remove --force D:\EmuPS5_main` al terminar. `D:\EmuPS5_integrated` ya se borró.
