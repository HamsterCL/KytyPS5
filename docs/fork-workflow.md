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

Estado en `integrated` (commit `de54c2dd`): **14 de 14 pasan**.

### Caso `kernel_file_system` en Windows

El check "path truncate errors preserve file size and descriptor position" esperaba `ENOTDIR` para
`truncate("/savedata0/archivo.dat/child")`. En Windows `std::filesystem::status` informa "no existe"
y `KernelTruncate` devolvía `ENOENT`. Con la STL de Visual Studio 18 falla también en `origin/main`
sin cambios propios. Arreglo en `integrated` (`de54c2dd`): `KernelTruncate` revisa si algún directorio
padre es un archivo regular y en ese caso devuelve `ENOTDIR`.

## Historial de cambios del fork

| Fecha | Cambio |
|---|---|
| 2026-10-16 | Fork sincronizado con `upstream/main`. Clon en `D:\EmuPS5` y remoto `upstream` agregado. |
| 2026-10-16 | `main` mergeado en `integrated` (`25a0f37b`), 3 conflictos resueltos. |
| 2026-10-16 | Setup de build en Windows y primera compilación completa. |
| 2026-10-16 | Fix de `KernelTruncate` (`ENOTDIR`) en `integrated` (`de54c2dd`). |
