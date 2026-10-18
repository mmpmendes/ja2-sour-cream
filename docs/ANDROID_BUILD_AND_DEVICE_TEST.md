# Build e testes do Sour Cream num dispositivo Android

Este guia descreve uma **build de depuração local** do estado atual do repositório e a sua validação num telemóvel ou tablet Android. Os comandos de terminal assumem Linux ou macOS; no Windows, abrir o projeto `android/` no Android Studio ou usar `gradlew.bat` e adaptar os caminhos. O nome visível e o identificador da aplicação ainda são os de JA2 Stracciatella.

## 1. Preparar o computador e o dispositivo

1. Obter uma instalação **original e não modificada** de Jagged Alliance 2. O APK contém o código e os recursos de `assets/`, mas não os ficheiros proprietários do jogo. A pasta original escolhida mais tarde no launcher deve conter `Data/` diretamente dentro dela. O launcher avisa se detetar dados de JA2 1.13.
2. Instalar Android Studio ou o Android SDK Command-line Tools, o **JDK 17**, Android SDK Platform **34**, Platform Tools (`adb`), SDK Build Tools **35.0.0**, CMake e Android NDK **29.0.14206865**. No Android Studio, usar **SDK Manager** para instalar esses componentes. Se usar as ferramentas de linha de comandos, com `sdkmanager` no `PATH`:

   ```sh
   sdkmanager "platforms;android-34" "platform-tools" "build-tools;35.0.0" "cmake;3.22.1" "ndk;29.0.14206865"
   sdkmanager --licenses
   ```

   O projeto fixa Gradle **8.13**, Android Gradle Plugin **8.13.2**, `compileSdkVersion 34` e a versão do NDK em `android/app/build.gradle`. O JDK 17 e o mínimo dos Build Tools são requisitos publicados para AGP 8.13. Consultar [compatibilidade oficial do AGP 8.13](https://developer.android.com/build/releases/agp-8-13-0-release-notes) e [instalação do NDK e CMake](https://developer.android.com/studio/projects/install-ndk).
3. Confirmar que o Gradle encontra o SDK. Usar o SDK configurado pelo Android Studio ou criar `android/local.properties` com `sdk.dir=/caminho/absoluto/para/Android/Sdk` (ficheiro ignorado pelo Git). Confirmar que `java -version` indica JDK 17 e que `adb version` funciona.
4. Instalar Rust com `rustup`. A versão mínima indicada em `min-rust-version` é **1.85.1**. Para reproduzir a configuração da CI, instalar essa toolchain e os quatro alvos usados pelo APK:

   ```sh
   rustup toolchain install 1.85.1
   rustup +1.85.1 target add armv7-linux-androideabi aarch64-linux-android i686-linux-android x86_64-linux-android
   rustup +1.85.1 target list --installed
   ```

5. No dispositivo, ativar **Opções de programador** e **Depuração USB**, ligar por USB, aceitar a chave RSA mostrada no ecrã e confirmar:

   ```sh
   adb devices -l
   adb shell getprop ro.product.cpu.abilist
   adb shell getprop ro.build.version.sdk
   ```

   Na linha correspondente ao dispositivo, o estado a seguir ao número de série deve ser `device`. Se aparecer `unauthorized`, aceitar o pedido no Android. Se houver vários dispositivos, acrescentar `-s <serial>` a cada comando `adb`. Ver [preparação de um dispositivo físico](https://developer.android.com/studio/run/device) e [ADB](https://developer.android.com/tools/adb).

## 2. Fazer a build de depuração

Na raiz do repositório:

```sh
cd android
RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug
```

O Gradle chama CMake para o código C++, Cargo para a biblioteca Rust e empacota os quatro ABIs definidos em `android/app/build.gradle`: `armeabi-v7a`, `arm64-v8a`, `x86` e `x86_64`. O primeiro build precisa de acesso aos repositórios de dependências e pode demorar. O resultado esperado é `BUILD SUCCESSFUL` e o ficheiro `android/app/build/outputs/apk/debug/app-debug.apk`, relativo à raiz do repositório. O APK de debug é assinado automaticamente para testes locais; **não** é necessário o keystore de release. Ver também [build Android na linha de comando](https://developer.android.com/build/building-cmdline).

Para repetir a build depois de alterações, executar novamente `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` dentro de `android/`. Não usar `assembleRelease` para este fluxo: `android/app/build.gradle` exige um keystore e a variável `ANDROID_KEYSTORE_PASSWORD` para a variante release. Não alterar `abiFilters` ou versões de ferramentas apenas para fazer passar uma build sem registar a razão.

## 3. Instalar o APK

Com o dispositivo listado pelo ADB, abrir um terminal na raiz do repositório (ou executar `cd ..` se ainda estiver em `android/`):

```sh
adb install -r android/app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n io.github.ja2stracciatella/.LauncherActivity
```

`Success` confirma a instalação; o segundo comando deve abrir o launcher. O `applicationId` atual é `io.github.ja2stracciatella`. Se o Android recusar uma atualização por **assinatura diferente**, exportar ou copiar primeiro os saves que importam antes de remover qualquer instalação anterior: desinstalar a aplicação pode apagar os seus dados privados.

## 4. Copiar e selecionar os dados originais do JA2

Copiar a pasta da instalação original para armazenamento partilhado do dispositivo. Este exemplo assume que a pasta local se chama `JA2` e que contém `Data/`:

```sh
adb push "/caminho/absoluto/para/JA2" /sdcard/Download/
adb shell ls /sdcard/Download/JA2
```

O resultado da segunda linha deve incluir `Data`. `/sdcard/Download/JA2` é um exemplo; pode transferir os ficheiros por outro meio e escolher essa pasta no launcher. Não colocar apenas `Data/` na pasta escolhida: selecionar a **raiz da instalação** que contém `Data/`.

No ecrã **Data** do launcher:

1. Em **Original Jagged Alliance 2 Game Directory**, tocar em **Choose** e selecionar `/storage/emulated/0/Download/JA2` (equivalente habitual de `/sdcard/Download/JA2`). Autorizar o acesso a ficheiros quando o Android pedir.
2. Em **Original Jagged Alliance 2 Game Version**, escolher a edição/localização correspondente aos ficheiros copiados.
3. Deixar **Save Game Directory** no valor padrão, salvo se estiver a testar explicitamente outra pasta. Na página **Settings**, começar com a resolução detetada automaticamente e um modo de escala legível.
4. Tocar no botão de iniciar jogo. Confirmar que aparece o menu principal, sem mensagem sobre `Data` em falta ou recursos inválidos.

## 5. Fazer os testes no dispositivo

Registar **modelo, versão Android/API, ABI, tamanho do ecrã, commit, versão dos dados JA2 e mods ativos**. Executar primeiro sem mods. Fazer os passos só com toque, sem rato/teclado, e anotar resultado esperado/observado:

| Área | Ação | Resultado esperado |
| --- | --- | --- |
| Arranque | Iniciar pelo launcher e entrar numa campanha nova | O menu e a vista tática abrem sem falha; texto e botões são visíveis. |
| Combate | Selecionar mercenário, mover, cancelar, apontar e disparar | O alvo e a ação correspondem ao ponto tocado; não há ação acidental. |
| Inventário | Abrir, escolher, mover e trocar itens | Item e destino corretos; nenhuma perda ou duplicação. |
| Estratégia e laptop | Abrir mapa, deslocar grupo, contratar mercenário | As ações essenciais são alcançáveis por toque. |
| Progresso | Guardar, voltar ao menu e carregar | Setor, equipa, inventário e estado da campanha mantêm-se. |
| Ciclo de vida | Suspender a app, voltar, bloquear/desbloquear e reabrir | O jogo retoma ou recupera sem perder um save válido. |
| Apresentação | Testar modo de escala, orientação suportada e áudio | Input continua alinhado com a imagem, sem controlos cortados. |

Repetir os cenários mais importantes num **telemóvel e num tablet** quando disponíveis. Comparar ações críticas com a versão desktop para detetar funcionalidade original sem equivalente tátil. O zoom por pinça e os controlos de zoom estão planeados em `SC-10` no [roadmap](SOUR_CREAM_ROADMAP.md); a sua ausência no código atual deve ser registada como trabalho pendente, não como falha do processo de build.

### Testes automatizados existentes

O projeto Android **não tem atualmente ficheiros `src/test` ou `src/androidTest`**. A variante Android também define `WITH_UNITTESTS=OFF`, e a CI faz build Android com `RUN_TESTS=false`. Assim, `assembleDebug` valida a compilação e o empacotamento, mas não substitui a matriz manual acima. Quando forem acrescentados testes Android, executar as tarefas Gradle correspondentes e atualizar esta secção. Os testes Rust/C++ executados em desktop na CI verificam lógica partilhada, mas não validam toque ou ciclo de vida Android.

## 6. Guardar evidência e diagnosticar falhas

O launcher tem uma página **Logs** com opção de copiar os registos da última execução. Para recolher também o registo do sistema, no computador:

```sh
adb logcat -d -v time > android-logcat.txt
```

Anexar ao relato o trecho relevante do log, os passos de reprodução e uma captura ou vídeo quando ajudar. Rever os logs antes de os partilhar: podem conter caminhos locais. Não incluir ficheiros proprietários do JA2 no repositório.

Problemas comuns:

- **`JAVA_HOME` ou versão Java incorreta:** usar JDK 17 para o Gradle deste projeto.
- **`SDK location not found` / NDK ausente:** confirmar `android/local.properties` ou a configuração do Android Studio e instalar exatamente o NDK fixado em `android/app/build.gradle`.
- **Alvo Rust ausente:** repetir `rustup +1.85.1 target add ...` para o ABI indicado no erro.
- **`adb devices` vazio ou `unauthorized`:** verificar cabo/Depuração USB e aceitar a autorização no dispositivo.
- **Instalação incompatível:** confirmar ABI com `ro.product.cpu.abilist` e assinatura da instalação anterior; proteger saves antes de qualquer desinstalação.
- **Jogo não arranca após o launcher:** verificar a pasta escolhida, a presença direta de `Data/`, a versão dos dados e os logs. Uma pasta modificada por JA2 1.13 não é uma base adequada para este teste.

## Registo de execução

Ao testar uma alteração do roadmap, preencher este bloco no relatório ou na issue:

```text
Tarefa SC-xx:
Commit:
Dispositivo / Android API / ABI / resolução:
Dados JA2 e localização / mods:
Build: comando, resultado e caminho do APK
Instalação: resultado de adb install
Cenários testados: passos, esperado, observado
Logs/capturas:
Testes não executados e motivo:
Regressões em rato/teclado ou saves:
```
