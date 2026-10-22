# Estrutura relevante para a UI Android

## Entradas e diretórios

- `android/app/src/main/java/io/github/ja2stracciatella/`: atividades, modelo de configuração e fragmentos Data, Settings e Logs do launcher.
- `src/sgp/`: ciclo SDL (`SGP.cc`), input (`Input.cc`), regiões de ponteiro (`MouseSystem.cc`) e vídeo (`Video.cc`).
- `src/game/`: ciclo e IDs de ecrã, layout, menu, opções, saves e ajuda.
- `src/game/Tactical/`, `Strategic/`, `Laptop/`, `TileEngine/`: ecrãs e painéis jogáveis de combate, mapa, laptop e viewport.
- `rust/stracciatella/src/{config,vfs,mods}/`: configuração, montagem de recursos e mods.
- `assets/`: recursos empacotados; os dados originais proprietários são externos.

## Mapa completo de ecrãs

`src/game/ScreenIDs.h` define IDs; `src/game/Screens.cc` associa handlers. Para a matriz de ações por ecrã e subecrã, ver `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md`. `src/game/Laptop/Laptop.h` enumera as páginas internas do laptop que partilham um ID principal.

## Evidência

- `src/game/ScreenIDs.h`
- `src/game/Screens.cc`
- `src/game/Laptop/Laptop.h`
- `src/sgp/SGP.cc`
- `android/app/src/main/AndroidManifest.xml`
