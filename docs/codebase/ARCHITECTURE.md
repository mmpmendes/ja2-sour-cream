# Fluxo de UI e input

## Caminho de input

O ciclo em `src/sgp/SGP.cc` converte coordenadas de eventos de rato e dedo para o sistema lógico do renderer SDL e despacha os eventos para `Input.cc`. `MouseSystem.cc` transforma eventos de ponteiro em callbacks de regiões. Os ecrãs em `Screens.cc` processam esses callbacks; a vista tática tem handlers distintos em tempo real e por turnos.

## Caminho de apresentação

`Video.cc` cria uma apresentação lógica SDL e escolhe o modo de escala. `UILayout.cc` define o tamanho mínimo lógico de 640×480 e calcula posições de viewport, painéis e inventário. O zoom proposto em `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md` ainda não existe como camada independente confirmada; precisa de transformação direta/inversa própria antes da entrada nos handlers táticos.

## Limites de estado

O launcher Android define dados originais e configuração; o jogo usa Rust para configuração/VFS. A UI de toque tática chama comandos de jogo existentes em `Touch_UI.cc`. [TODO] Confirmar em dispositivo o comportamento de todos os overlays após pausa e rotação.

## Evidência

- `src/sgp/SGP.cc`
- `src/sgp/Input.cc`
- `src/sgp/MouseSystem.cc`
- `src/sgp/Video.cc`
- `src/game/UILayout.cc`
- `src/game/Tactical/Touch_UI.cc`
