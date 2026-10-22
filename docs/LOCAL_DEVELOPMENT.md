# Ambiente local de desenvolvimento

Este guia regista a configuração e os comandos verificados nesta máquina para Linux. O fluxo Android completo está em [ANDROID_BUILD_AND_DEVICE_TEST.md](ANDROID_BUILD_AND_DEVICE_TEST.md).

## Linux nesta máquina

Ambiente observado em 20-10-2026:

- Linux x86_64, GCC 16.2.1, CMake 4.4.4 e Ninja;
- Rust/Cargo 1.99.1, acima do mínimo do repositório (1.85.1);
- SDL3 3.4.18; GDB disponível;
- FLTK não instalado: a build funciona sem o launcher, com `BUILD_LAUNCHER=OFF`;
- dados originais instalados em `/home/marcio/Games/Heroic/Jagged Alliance 2`, com `data/` em minúsculas. O VFS do jogo procura esse diretório sem distinguir maiúsculas de minúsculas.

Configurar e compilar em modo debug:

```sh
cmake -S . -B _bin -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DBUILD_LAUNCHER=OFF
cmake --build _bin --parallel 8
```

Executar os testes a partir do diretório de build. O diretório de trabalho importa: os testes C++ carregam `externalized/` relativo ao diretório atual.

```sh
cmake --build _bin --target cargo-test --parallel 8
(cd _bin && ./ja2 -unittests)
```

Arrancar o jogo original e depurar com GDB:

```sh
cd _bin
./ja2 -gamedir "/home/marcio/Games/Heroic/Jagged Alliance 2"
gdb --args ./ja2 -gamedir "/home/marcio/Games/Heroic/Jagged Alliance 2"
```

O jogo precisa de uma sessão gráfica SDL disponível. A primeira execução cria a configuração normal em `~/.ja2/ja2.json`; os saves usam o diretório configurado pelo jogo.

Validação em 20-10-2026: o utilizador confirmou que a build local arrancou e correu corretamente com os dados originais. Num smoke test não interativo anterior, SDL reportou `SDL_CreateTexture ... Parameter 'renderer' is invalid`; esse resultado não se reproduziu na execução interativa do utilizador.

## Android

O projeto requer JDK 17, SDK Platform 34, Build Tools 36.0.0, CMake 3.22.1, NDK 29.0.14206865 e os quatro alvos Rust Android. Seguir [o guia Android](ANDROID_BUILD_AND_DEVICE_TEST.md) para instalar, compilar, instalar e validar o APK.

Validação em 20-10-2026: Gradle 9.6.0 com JDK 17, AGP 9.4.1 e KGP 2.4.20; `GRADLE_USER_HOME=/tmp/ja2-gradle RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` concluiu com sucesso para as quatro ABIs. O APK saiu em `android/app/build/outputs/apk/debug/app-debug.apk` (77 MB). KGP 2.4.20 ainda não declara suporte oficial ao AGP 9.4; a combinação passou a build, mas não está plenamente coberta pela matriz publicada. O executável `adb` está instalado, mas não foi possível iniciar o daemon nesta sessão (`could not install *smartsocket* listener: Operation not permitted`); instalação e teste num dispositivo continuam por validar.

## Depuração

- C++: usar `_bin/compile_commands.json` no editor; o executável debug é `_bin/ja2` e pode ser aberto com GDB.
- Rust: os símbolos debug ficam em `_bin/dependencies/lib-stracciatella/target/x86_64-unknown-linux-gnu/debug/`; definir breakpoints nas crates e arrancar o executável C++ que liga à biblioteca.
- Android: abrir `android/` no Android Studio ou usar o Gradle wrapper; escolher a variante `debug` e ligar um dispositivo com Depuração USB. Recolher logs com `adb logcat`.

Não copiar dados proprietários do JA2 para o repositório. O diretório `_bin/` é ignorado pelo Git.
