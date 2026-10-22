# Sour Cream: instruções para agentes

Este repositório é um fork de JA2 Stracciatella. O nome pretendido para o fork é **Sour Cream**. O código e vários documentos ainda usam o nome original; não interpretes isso como uma migração já concluída. **A experiência em dispositivos móveis é um dos focos principais do fork**, com acessibilidade por toque e zoom sem perder a funcionalidade original.

## Antes de trabalhar

1. Lê [docs/SOUR_CREAM_ROADMAP.md](docs/SOUR_CREAM_ROADMAP.md). É o plano de produto, ordenado por impacto. Escolhe um identificador `SC-xx` e limita a alteração a um incremento verificável desse item.
2. Confirma o comportamento atual no código e, para bugs, reproduz o problema antes de alterar a lógica. Uma nota `TODO`, um relato externo ou uma hipótese do plano não são prova de um defeito neste fork.
3. Lê [COMPILATION.md](COMPILATION.md) e as convenções de [CONTRIBUTING.md](CONTRIBUTING.md). O projeto combina C++/CMake e Rust/Cargo; `ja2-launcher` é opcional.
4. Preserva dados de campanha, compatibilidade de saves e suporte aos dados originais do JA2. Alterações a formatos, localização ou prioridades do VFS exigem testes de compatibilidade.
5. Em alterações de interface ou input, verifica rato/teclado e toque. Cada comando necessário para completar uma campanha deve continuar acessível sem rato ou teclado num dispositivo móvel. Gestos devem ter alternativa visível para quem não os consegue executar; zoom e escala visual não podem alterar regras, posição de objetos ou o resultado das ações.

## Mapa rápido

- Arranque e ciclo SDL: `src/sgp/SGP.cc`; gestão de ecrãs: `src/game/GameLoop.cc` e `src/game/Screens.cc`.
- Toque e zoom móvel: `src/sgp/Input.cc`, `src/sgp/MouseSystem.cc`, `src/game/Tactical/Touch_UI.cc`, `src/sgp/Video.cc`, `src/game/UILayout.cc` e `android/app`.
- Jogo: `src/game/{Strategic,Tactical,TacticalAI,TileEngine,Laptop,Editor}`.
- Recursos e mods: `src/externalized`, `assets/externalized`, `assets/mods`, `rust/stracciatella/src/{vfs,mods,schemas}`.
- Configuração: `rust/stracciatella/src/config` e `src/launcher`.
- Build e verificações existentes: `CMakeLists.txt`, `dependencies/lib-stracciatella/CMakeLists.txt`, `.ci/ci-build.sh`.
- Build e testes num dispositivo Android: `docs/ANDROID_BUILD_AND_DEVICE_TEST.md`.

## ADB e dispositivos Android

Se o daemon ADB não conseguir iniciar dentro da sandbox (por exemplo, erro `Operation not permitted` ao abrir o socket local), executa os comandos ADB fora da sandbox. Não concluas que o dispositivo ou AVD está inacessível antes de tentar nesse contexto. Regista o dispositivo/AVD, Android/API, orientação, resolução e densidade usados na reprodução.

Não encerres o AVD quando terminares de o usar; deixa-o ligado para a próxima sessão, salvo pedido explícito em contrário.

Os dados originais do JA2 estão disponíveis em `JA2_ORIGINAL_GAME_DIR`. Usa sempre esta variável para consultar ou copiar os ficheiros originais para um dispositivo ou AVD. Não registes o valor privado do caminho neste ficheiro nem copies dados proprietários para o repositório.

## Ao concluir uma tarefa

Regista no resumo da alteração: `SC-xx`, cenário reproduzido ou requisito observado, ficheiros alterados, verificações executadas, resultado e riscos que ficaram. Se não houver dados originais do JA2 disponíveis para teste manual, diz explicitamente quais cenários ficaram por verificar. Atualiza o estado e a evidência no roadmap apenas depois de a alteração estar concluída.
