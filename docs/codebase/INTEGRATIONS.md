# Integrações relevantes

## Dados e plataforma

- O jogo requer dados externos da instalação original. Neste ambiente foi confirmada a pasta `/home/marcio/Games/Heroic/Jagged Alliance 2/`; não incorporar nem redistribuir os ficheiros proprietários.
- `android/app/src/main` contém atividades Android e o seletor de pasta no `DataTabFragment.kt`. `android/app/build.gradle` inclui `assets/` no APK e liga o motor via CMake/NDK.
- O motor obtém configuração e recursos por Rust em `rust/stracciatella/src/config` e `src/vfs`; `src/externalized` e `assets/externalized` contêm recursos próprios do port.

## Serviços externos

[TODO] Não foi feita auditoria a rede, telemetria ou contas externas; não há dependência dessas integrações no plano de UI.

## Evidência

- `README.md`
- `android/app/build.gradle`
- `android/app/src/main/java/io/github/ja2stracciatella/ui/main/DataTabFragment.kt`
- `rust/stracciatella/src/config/mod.rs`
- `rust/stracciatella/src/vfs/mod.rs`
