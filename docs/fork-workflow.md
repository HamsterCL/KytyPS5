# Flujo de trabajo del fork (HamsterCL/KytyPS5)

Guía de cómo se mantiene este fork, cómo se compila en Windows y cuál es el estado de los tests.
Última actualización: 2026-10-16.

## Remotos y ramas

| Remoto | URL | Uso |
|---|---|---|
| `origin` | https://github.com/HamsterCL/KytyPS5.git | Fork personal |
| `upstream` | https://github.com/KytyPS5/KytyPS5.git | Repo original |

- `main`: espejo de `upstream/main`. No se le hacen commits propios.
- `integrated`: rama con cambios propios (40 commits sobre `main` en el último merge).
  Se mantiene al día haciendo **merge** de `main` (sin rebase, para no necesitar force push).

## Sincronizar el fork con upstream

Solo en GitHub (sin clon local):

```
gh repo sync HamsterCL/KytyPS5 --source KytyPS5/KytyPS5 --branch main
```

Desde el clon local:

```
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

## Llevar `main` a `integrated`

```
git fetch origin
git checkout integrated
git merge origin/main
# resolver conflictos, git add <archivos>, git commit
git push origin integrated
```

### Conflictos conocidos (merge del 2026-10-16)

| Archivo | Resolución |
|---|---|
| `.github/workflows/build.yml` | Unión de las listas de tests (`host_input` de `main` + `kernel_file_system` de `integrated`). |
| `src/graphics/shader/recompiler/backend/spirv/spirvEmitterFlow.cpp` | Se usó `Result(inst)` de `main` en `EmitConditionRef`. Se descartó el `const_cast` de `integrated`; conviene vigilar que no reaparezca el problema de la condición constante `true`. |
| `src/kernel/fileSystem.cpp` | Ambas ramas implementaban `Writev`/`Pwritev` y `Truncate`. Se conservó la versión de `main` y los comentarios de `integrated`. |

## Pull requests hacia upstream

- Cada PR sale de una **rama aparte basada en `upstream/main`**, nunca de `integrated` (arrastraría los
  commits propios del fork).
- Flujo usado para el fix de `KernelTruncate` (PR #1577):

```
git fetch upstream
git checkout -b fix/<tema> upstream/main
git cherry-pick <commit>          # o commitear directo
git push -u origin fix/<tema>
gh pr create -R KytyPS5/KytyPS5 --base main --head HamsterCL:fix/<tema> --title "..." --body-file pr.md
```

- Commits y descripciones de PR van solo con el nombre del autor, sin trailers `Co-Authored-By` ni
  menciones a la IA.
- El aviso de GitHub "`<rama>` had recent pushes ... Compare & pull request" aparece solo para ramas con
  pushes de la última hora; no es un PR, no se puede descartar y desaparece solo. No pulsarlo en `integrated`.

### PR abiertos

| PR | Rama | Estado |
|---|---|---|
| [#1577](https://github.com/KytyPS5/KytyPS5/pull/1577) | `fix/truncate-enotdir` | Abierto. `KernelTruncate` devuelve `ENOTDIR` si un padre es un archivo regular. Con este toolchain `kernel_file_system` sigue fallando en `main` en otro check ("Net unknown socket options return protocol-option errors"), ajeno al PR. |

## Compilar en Windows

### Requisitos instalados en esta máquina

| Herramienta | Ubicación |
|---|---|
| Visual Studio 18 Community (CMake y Ninja incluidos) | `C:\Program Files\Microsoft Visual Studio\18\Community` |
| LLVM / clang-cl 23.1.3 | `C:\Program Files\LLVM\bin` (winget `LLVM.LLVM`) |
| Qt 6.10.3 `msvc2022_64` | `C:\Qt\6.10.3\msvc2022_64` (instalado con `aqtinstall`) |
| glslang 16.6.0 | `D:\glslang\x\bin` (solo trae `glslang.exe`; el CMake acepta ese nombre) |
| Submódulos | `git submodule update --init --recursive --depth 1 --jobs 4` |

CMake, Ninja y clang-cl no están en el PATH global: hay que entrar por el entorno de VS
(`vcvars64.bat`) y anteponer sus carpetas al PATH.

### Comandos

```
cmake -S . -B _Build/windows -G Ninja -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl ^
  -DCMAKE_PREFIX_PATH=C:/Qt/6.10.3/msvc2022_64

cmake --build _Build/windows --target launcher ^
  audio_out2_port_tests pad_haptics_tests controller_settings_tests host_input_tests ^
  ime_dialog_tests lru_cache_tests virtual_memory_allocation_tests ^
  shader_recompiler_compute_tests resource_tracking_tests scalar_provenance_tests ^
  resource_materialization_tests shader_vertex_metadata_tests archive_file_tests ^
  avplayer_file_tests kernel_file_system_tests --parallel
```

Salida: `_Build/windows/launcher/kyty_launcher.exe` y `_Build/windows/kyty_emulator.exe`.
La config de CI de referencia es `.github/workflows/build.yml`.

## Tests

```
ctest --test-dir _Build/windows --output-on-failure -R "^(audio_out2_port|pad_haptics|controller_settings|host_input|ime_dialog|lru_cache|resource_tracking|scalar_provenance|resource_materialization|shader_vertex_metadata|archive_file|avplayer_file|kernel_file_system|virtual_memory_allocation)$"
```

Estado en `integrated` (commit `113b44f5`): **14 de 14 pasan**.

### Caso `kernel_file_system` en Windows

El check "path truncate errors preserve file size and descriptor position" esperaba `ENOTDIR` para
`truncate("/savedata0/archivo.dat/child")`. En Windows `std::filesystem::status` informa "no existe"
y `KernelTruncate` devolvía `ENOENT`. Con la STL de Visual Studio 18 falla también en `origin/main`
sin cambios propios. Arreglo en `integrated` (`113b44f5`): `KernelTruncate` revisa si algún directorio
padre es un archivo regular y en ese caso devuelve `ENOTDIR`.

## Estado actual y pendientes (para retomar)

Última actualización: 2026-10-17. `integrated` en `b66dd6c7`, publicado en `origin/integrated`.

### Pendiente principal: crash de audio y 13 fps en Astro Bot

Al probar Astro Bot con el build de `integrated` (`438256e`, instalado en `D:\EmuPS5\_Build\windows\install`):

- **Crash:** excepción de host en `runtimeLinker.cpp:711`, tipo 1, código 3221225477 (acceso inválido),
  `address=0xffffffffffffffff`. Hilo: `SceSndzAudioOutMain` (hilo del propio juego, no del emulador).
  La instrucción que falla es `mov edx,[rbx]` con `rbx=0x3f80000040008222`: un puntero pisado cuya mitad
  alta es `0x3f800000` (`1.0f`), es decir, datos float escritos sobre un objeto del hilo de audio.
  Shaders compilados en ese momento: VS 23, PS 37, CS 48, GS 4.
- **Rendimiento:** unos 13 fps constantes. GPU: RTX 3060 Ti (no es el límite). El build es Release
  **sin LTO**; el CI de upstream compila con `-DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON`.
- **Hipótesis (sin confirmar):** algún commit propio de `integrated` cambia lo que recibe el juego,
  sobre todo los de AJM (`fad25e12` rechaza buffers divididos que desbordan; `153b3d89` valida entradas
  de AJM/sockets) y el de `SO_RCVTIMEO` (`1c6f5973`).
- **A/B preparado:** build limpio de `upstream/main` (`42bb0da5`, sin commits propios) en el worktree
  `D:\EmuPS5_main`, instalado en `D:\EmuPS5_main\_Build\windows\install`. Falta que el usuario pruebe
  Astro Bot con ese build y con el de `integrated`, con la misma configuración, y compare crash y fps.
- **Siguientes pasos según el resultado:**
  - Si `upstream/main` no se cae: bisecar los commits propios de `integrated` (empezando por AJM).
  - Si también se cae: es un bug de upstream; recoger el log y reportarlo.
  - Si ambos dan ~13 fps: probar un build con LTO y medir de nuevo.
- **Ojo:** el build instalado en `D:\EmuPS5\_Build\windows\install` es anterior a la actualización de
  dependencias. El build con dependencias nuevas está compilado en `_Build/windows` pero **no instalado**
  (hacer `cmake --install _Build/windows --prefix _Build/windows/install` para probarlo).

### Otros pendientes

- Decidir si se proponen las actualizaciones de dependencias a upstream (habría que rehacerlas desde
  `upstream/main` en una rama aparte).
- Borrar el worktree `D:\EmuPS5_main` cuando termine el A/B (`git worktree remove --force D:\EmuPS5_main`).
- PR #1577 abierto en upstream, sin revisión todavía.

## Dependencias (submódulos)

Actualizadas el 2026-10-17 a su último release estable (rama `deps/update`, ya mergeada en
`integrated`, commit `b66dd6c7`). Compilan y los 14 tests pasan; **no se probó en ejecución con ningún
juego**.

| Submódulo | Antes | Ahora |
|---|---|---|
| SDL3 | 3.4.16 | 3.4.18 (sigue en la rama `release-3.4.x`) |
| xxHash | commit sin tag | v0.8.4 |
| Vulkan-Headers | SDK 1.4.335.0 | SDK 1.4.363.0 |
| SPIRV-Headers | SDK 1.4.328.1 | SDK 1.4.363.0 |
| SPIRV-Tools | SDK 1.4.328.1 | SDK 1.4.363.0 |
| VulkanMemoryAllocator | commit sin tag | v3.4.0 |
| fmt | commit sin tag | 12.2.0 |
| imgui | v1.92.9b | v1.93.0 |

Sin cambios a propósito: tracy (v0.14.1) y magic_enum (v0.9.8) ya estaban al día; spdlog y
nlohmann_json apuntan a commits de desarrollo posteriores a su último release (bajarlos sería un
downgrade); ffmpeg-core y LibAtrac9 son forks del proyecto. Qt 6.10.3 y las GitHub Actions no se tocaron.
Para actualizar un submódulo: `git -C 3rdparty/<m> fetch --depth 1 origin tag <tag>`,
`git -C 3rdparty/<m> checkout <tag>`, y comprobar con `gh api repos/<o>/<r>/compare/<sha>...<tag>` que el
destino no esté por detrás.

## Historial de cambios del fork

| Fecha | Cambio |
|---|---|
| 2026-10-16 | Fork sincronizado con `upstream/main`. Clon en `D:\EmuPS5` y remoto `upstream` agregado. |
| 2026-10-16 | `main` mergeado en `integrated` (`d7cc0793`), 3 conflictos resueltos. |
| 2026-10-16 | Setup de build en Windows y primera compilación completa. |
| 2026-10-16 | Fix de `KernelTruncate` (`ENOTDIR`) en `integrated` (`113b44f5`). |
| 2026-10-16 | PR del fix a upstream: https://github.com/KytyPS5/KytyPS5/pull/1577 (rama `fix/truncate-enotdir`). Historia de `integrated` reescrita para quitar trailers de atribución; los hashes actuales son los de esta tabla y de `CLAUDE.md`. |
| 2026-10-16 | Se aclaró el flujo de PR hacia upstream y el aviso "had recent pushes" de GitHub. |
| 2026-10-17 | Primera prueba con Astro Bot: crash en `SceSndzAudioOutMain` y ~13 fps. Se preparó un build limpio de `upstream/main` para el A/B. |
| 2026-10-17 | Dependencias actualizadas a su último release estable y mergeadas en `integrated` (`b66dd6c7`). |
