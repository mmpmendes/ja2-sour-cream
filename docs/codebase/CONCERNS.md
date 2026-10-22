# Riscos e lacunas verificáveis para SC-10

## Código observado

- `Input.cc` mantém um identificador de dedo principal que é substituído em `FingerDown`, enquanto `FingerUp` ignora outro ID. Os callbacks táticos em `Real_Time_Input.cc` e `Turn_Based_Input.cc` contam toques para pan. Risco de conflito com pinça, sujeito a reprodução.
- `Touch_UI.cc` constrói botões táticos de 32 unidades lógicas; a dimensão física em dispositivo ainda não foi medida.
- `UILayout.cc` exige 640×480 lógicos, e `Video.cc` apresenta esse espaço com letterbox/escala. Zoom do framebuffer inteiro pode cortar painéis; precisa de protótipo limitado ao viewport.
- `SaveLoadScreen.cc` tem callbacks de ponteiro e alguns de botão direito; verificar via tátil para funções essenciais.
- `android/app/build.gradle` diz `compileSdk 37`; `docs/ANDROID_BUILD_AND_DEVICE_TEST.md` ainda documenta 34.

## Validação em falta

- [TODO] Reproduzir falhas de toque em equipamento real; a leitura estática não prova defeitos de campanha.
- O utilizador definiu Pixel 8 (~6") na horizontal como alvo físico inicial, complementado por emuladores. [TODO] Medir densidade/insets e apurar versão Android em dispositivo.
- O utilizador pediu uma fase por ecrã, incluindo cada página do laptop; o plano enumera essas fases em `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md`.

## Alteração recente

Na amostra dos 12 commits mais recentes, `docs/SOUR_CREAM_ROADMAP.md` apareceu 3 vezes e `docs/ANDROID_BUILD_AND_DEVICE_TEST.md` 2 vezes; isto identifica documentação em revisão, não demonstra fragilidade do código de UI.

## Evidência

- `src/sgp/Input.cc`
- `src/game/Tactical/Real_Time_Input.cc`
- `src/game/Tactical/Turn_Based_Input.cc`
- `src/game/Tactical/Touch_UI.cc`
- `src/game/UILayout.cc`
- `src/game/SaveLoadScreen.cc`
- `android/app/build.gradle`
- `docs/ANDROID_BUILD_AND_DEVICE_TEST.md`
