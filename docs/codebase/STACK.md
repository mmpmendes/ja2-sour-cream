# Stack relevante para a UI Android

## Produção

- Motor e UI do jogo: C++ compilado por CMake; renderização e input SDL3. A versão mínima SDL indicada em `COMPILATION.md` é 3.0.0.
- Configuração, VFS e mods: Rust em `rust/stracciatella`; há ponte C API em `rust/stracciatella_c_api`.
- Android: módulo Gradle `android/app`, launcher Kotlin/Android Views e jogo SDL/NDK. `android/app/build.gradle` chama o `CMakeLists.txt` da raiz e empacota recursos de `assets/`.
- O launcher FLTK de desktop é opcional; não é o launcher Kotlin usado no Android.

## Ferramentas e versões

- `COMPILATION.md` documenta CMake, Cargo/Rust e os quatro alvos Android. `android/app/build.gradle` fixa NDK `29.0.14206865`, `compileSdk 37`, `minSdk 23` e `targetSdk 26` no estado atual.
- [TODO] Confirmar em dispositivo o conjunto de SDK/Gradle realmente usado no próximo build; `docs/ANDROID_BUILD_AND_DEVICE_TEST.md` ainda cita `compileSdkVersion 34`.

## Evidência

- `COMPILATION.md`
- `CMakeLists.txt`
- `android/app/build.gradle`
- `rust/stracciatella/Cargo.toml`
- `rust/stracciatella_c_api/Cargo.toml`
- `src/sgp/Video.cc`
