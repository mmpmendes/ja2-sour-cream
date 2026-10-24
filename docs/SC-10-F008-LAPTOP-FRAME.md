# SC-10/F008 — moldura do laptop

**24-10-2026 — em validação.** A falha de retoma foi corrigida e a matriz abaixo foi executada no AVD. A comparação com rato em desktop e a ergonomia em dispositivos físicos continuam pendentes; F008 não está declarada concluída. SC-10 permanece em investigação.

## Referência e âmbito

- HEAD inicial: `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`. Nesta continuação, antes de editar, a árvore já tinha alterações não staged em `android/app/src/main/AndroidManifest.xml`, `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md`, `docs/SOUR_CREAM_ROADMAP.md` e o relatório novo `docs/SC-10-F008-LAPTOP-FRAME.md`; foram preservadas. Nenhum ficheiro foi staged.
- Documentos de referência: `AGENTS.md`, roadmap, plano Android por ecrã, `COMPILATION.md`, `CONTRIBUTING.md` e guia Android de build/dispositivo.
- AVD visível `Medium_Phone_API_37.0`, `emulator-5554`, Android 17/API 37, `x86_64`, emulator 37.2.12.0, SwiftShader. Base física configurada: 1080×2400@420 dpi. Não se usou modo headless nem Pixel.
- Dados English já selecionados no launcher; nenhuma seleção adicional de mods. O save privado `2026-10-22t21-39-48z-f006test.sav` foi confirmado antes de carregar: dia 1, 01:00, 0 mercenários, $45.000. No fim mantinha 409.643 bytes e a data de 22-10-2026; não se gravou sobre ele.
- F008 abrange a barra lateral, favoritos, minimizar/regressar, Shut Down, indicadores e apresentação da moldura. O fundo `LAPTOP_MODE_NONE` foi apenas destino das transições. Nas apps, verificaram-se entrada e saída pela moldura; não se testou nem alterou a lógica de F009–F032.

## Entrada, saída e inventário confirmado no código

`Screens.cc` liga `LAPTOP_SCREEN` a `LaptopScreenHandle()`. O mapa solicita a entrada em `Map_Screen_Interface_Bottom.cc::HandleExitsFromMapScreen()`; o início de campanha também a solicita em `Strategic/Game_Init.cc`. `EnterLaptop()` pausa o jogo e cria recursos/regiões; `ExitLaptop()` remove-os e retoma o relógio. `LeaveLapTopScreen()` encaminha a saída para o mapa.

| Comando da moldura em `Laptop.cc` | Caminho original e alternativa visível | Resultado observado |
| --- | --- | --- |
| E-mail, Files, History, Personnel, Financial | `CreateLaptopButtons()` e callbacks de cada app; seleção em `POINTER_UP` | Cada botão abriu a app identificada no título. |
| Web / favoritos | `WWWRegionButtonCallbackPrimary()`; tocar novamente em Web mostra/oculta a lista | A lista apresentou A.I.M. e Cancel no save inicial; A.I.M. abriu a página correspondente. |
| Cancelar favoritos | `BookmarkCallBack()`, toque no fundo ou `HandleLapTopESCKey()` | Cancel e fundo ocultaram a lista; Esc também, sem sair do laptop. |
| Minimizar/regressar | X visível ou ícone da barra de título; callbacks de minimizar | Regressou ao último programa aberto ou ao fundo do laptop. O X não representa histórico de páginas web. |
| Voltar ao mapa | Shut Down → `HandleExit()` / `LeaveLapTopScreen()`; Esc sem sobreposição | Uma saída para o mapa; o ícone do mapa permitiu reentrar. |
| Correio/ficheiro novo | Regiões de `CreateDestroyMouseRegionForNewMailIcon()` | Os indicadores abriram o correio/ficheiro correspondente; X permitiu regressar. Verificados nos perfis base e tablet. |
| Teclado | Tab e Ctrl+Tab alternam programas; Esc cancela favoritos ou sai | Tab abriu correio e Ctrl+Tab finanças; Esc regressou ao mapa. Não há percurso de foco entre botões com Tab: o atalho alterna apps. |

`SGP.cc` converte coordenadas SDL da janela para coordenadas lógicas antes de `Input.cc`/`MouseSystem.cc` distribuírem o dedo principal. Os atalhos laterais e as regiões usam o mesmo comando primário por rato/toque. O wrapper primário/secundário de Web conserva a ação secundária original; os percursos necessários testados dispõem de botões visíveis e não exigiram gesto especial. As regiões são criadas/removidas com o laptop e com os favoritos.

## Falha reproduzida e correção

1. No perfil base em paisagem, carregar F006test, entrar no laptop e abrir Financial.
2. Premir Home, abrir Recentes e tocar no cartão do jogo.
3. Esperado: retomar o mesmo laptop, sem perder a campanha carregada ou repetir a entrada.
4. Antes: a mudança para a orientação do ecrã inicial levou a `onDestroy()`. Os eventos Android registaram `wm_relaunch_resume_activity` com alterações `0x500`; o PID passou de `4488` para `5107`, com `EXIT_SELF`, e iniciou-se novamente o jogo.

O manifesto tratava `orientation|screenSize`. Na API instalada, `CONFIG_SCREEN_LAYOUT=0x100` e `CONFIG_SCREEN_SIZE=0x400`; faltava tratar a primeira componente de `0x500`. O exemplo Android do SDL incluído também declara `screenLayout`. `SDLActivity.onDestroy()` envia quit ao motor, enquanto `onConfigurationChanged()` já está implementado.

**Alteração:** acrescentou-se apenas `screenLayout` a `configChanges` de `StracciatellaActivity` em `android/app/src/main/AndroidManifest.xml`. Não foi necessário alterar `Laptop.cc`, callbacks, desenho ou páginas internas. O launcher conserva a sua declaração anterior.

**Depois:** a mesma sequência base manteve Financial e o PID `5371`; os logs mostraram `onConfigurationChanged()`, `onResume()` e `nativeResume()`, sem destruição/reentrada. A rotação base retrato → paisagem preservou o mesmo processo e conteúdo. No compacto, Home/Recentes preservou Mail Box e o PID `3329`; no tablet, preservou A.I.M., favoritos abertos e o PID `3718`. Cancel e os controlos seguintes continuaram a responder após a retoma.

## Perfis e resultados

Resolução lógica mantida em **1038×480**, escala **LINEAR**, sem alterar configuração ou regras para compensar o tamanho do dispositivo. Coordenadas de toque foram medidas no PNG original, considerando letterbox e o centro dos botões; o controlo nativo de computador não expôs a janela Android, pelo que se injetou toque por ADB no AVD visível.

| Perfil efetivo em paisagem | Preparação | Escala lógica → píxel | Cobertura e resultado |
| --- | --- | --- | --- |
| 2400×1080@420 dpi | Sem override de tamanho/densidade; rotação 1 | 2,25×; origem do painel 640×480 em (480, 0) | Seis apps, favoritos/A.I.M., Cancel/fundo/Esc, X/ícone, retorno à app anterior, indicadores, arrasto para fora do botão, Shut Down/reentrada, Tab/Ctrl+Tab, Home/Recentes e rotação: passaram após a correção. |
| 1600×720@320 dpi | `wm size 720x1600`, `wm density 320`, rotação 1 | 1,5×; origem do painel em (320, 0) | Seis apps, favoritos/A.I.M./Cancel, X/ícone, Shut Down/reentrada/Esc e Home/Recentes: passaram. |
| 2560×1920@320 dpi | `wm size 1920x2560`, `wm density 320`, rotação 1 | ≈2,466×; origem do painel em (490,8, 368,1) | Seis apps, favoritos/A.I.M./Cancel, X/ícone, indicadores, Shut Down/reentrada/Esc, Tab/Ctrl+Tab, Home/Recentes com favoritos e rotação: passaram. |

Também se observou a apresentação em retrato base 1080×2400@420 dpi e tablet 1920×2560@320 dpi durante a rotação: conteúdo preservado com letterbox, sem reiniciar o processo. Não se atribui a esses ensaios cobertura de precisão tátil ou ergonomia de retrato. No tablet, Android Back enviado por ADB manteve Financial visível; Shut Down e Esc são os caminhos de saída efetivamente validados.

Nos três perfis em paisagem, os comandos da moldura e as legendas ficaram dentro da área apresentada, sem corte por barras do sistema; AVD sem recorte de câmara. As regiões de favoritos têm 100×18 unidades lógicas: altura nominal ≈15,4 dp na base, 13,5 dp no compacto e 22,2 dp no tablet. O ícone de minimizar na barra tem região 20×20: ≈17,1 / 15 / 24,7 dp. Estes alvos são pequenos; a precisão das injeções e a leitura das capturas não demonstram conforto com dedo ou leitura física. A medição ergonómica e o recorte do Pixel continuam pendentes.

## Hipóteses não reproduzidas e limites

- **Entrada no laptop bloqueada: não reproduzido.** Os toques anteriores atingiam margens/outros controlos do mapa. O centro do ícone abriu o laptop; na base, o ponto usado foi aproximadamente (1545, 960).
- **Barra lateral apagada no compacto: não reproduzido.** A inspeção dos PNGs originais encontrou a barra presente; os recortes da barra nas várias apps e nas capturas espaçadas tinham píxeis idênticos. Não se alterou o redesenho.
- **Alvos desalinhados, cancelamento preso ou transições duplicadas: não reproduzidos** nos percursos registados. Um arrasto iniciado em History e terminado no fundo manteve Financial. Após retoma, o primeiro comando produziu a ação esperada. Isto não cobre multitouch nem interrupção com dedo ainda pressionado.
- `input mouse tap` produziu cursor/ajuda de hover, mas não um clique de navegação verificável; não conta como validação de rato. Não havia dispositivo físico ligado nem superfície desktop operável. Rato/teclado em desktop, Pixel/tablet físicos, outras APIs e ergonomia permanecem pendentes.
- F005–F007 conservaram código e estado. As páginas internas, saves, VFS e formatos não foram alterados. Não se afirma validação de ciclo de vida de todos os outros ecrãs por esta correção.

## Build, evidência e reposição

Comando executado em `android/`:

```sh
RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug --max-workers=1
```

`BUILD SUCCESSFUL` para as quatro ABIs; `adb install -r` devolveu `Success`. Versões reais: Gradle 9.8.2, AGP 9.4.1, Kotlin 2.4.21 pelo catálogo `libs.versions.toml` (a variável antiga do topo ainda diz 2.4.20), compileSdk 37, targetSdk 26, NDK 29.0.14206865. O APK inicial tinha metadados CMake antigos `b7d03d92a`; após regeneração do CMake, o título e o log mostraram `v0.23.0-git+2df20b6ec`. SHA-256 final do APK: `83400b7a91dbd71a3608b25b42b7abf68a5830db92deb918f22a32febafc4e42`.

Capturas antes/depois e de cada entrada/saída foram inspecionadas em temporários locais; logs SDL, eventos de recriação e `dumpsys activity exit-info` fundamentam o antes/depois acima. Não se acrescentaram testes automatizados: a alteração é um atributo de ciclo de vida Android, verificado pela build, manifesto e reprodução manual. Não se executaram testes Rust/C++ sem lógica alterada a verificar.

Foram executados `wm size reset`, `wm density reset`, `accelerometer_rotation=1` e `user_rotation=0`. A leitura final confirmou 1080×2400, 420 dpi, rotação 0 e launcher em primeiro plano. As capturas temporárias desta execução foram removidas; o log do processo do emulador em uso foi conservado em `/tmp`. O AVD permanece ligado e visível. Os dados originais e o save ficam apenas no dispositivo.

## Nova tentativa de validação, 24-10-2026

Nesta execução, `adb devices -l` encontrou apenas `emulator-5554` (`sdk_gphone64_x86_64`): Android 17/API 37, `x86_64`, 1080×2400@420 dpi, rotação automática ligada, rotação atual 0 e LauncherActivity em primeiro plano. Não havia Pixel físico. `JA2_ORIGINAL_GAME_DIR` estava disponível e os dados originais continuavam em `/sdcard/Download/JA2/Data` no AVD; os ficheiros não foram copiados nem alterados. O save privado `files/.ja2/SavedGames/2026-10-22t21-39-48z-f006test.sav` foi apenas localizado e listado: 409.643 bytes, timestamp 22-10-2026 21:39; não foi aberto, carregado ou gravado.

O inventário CUA mostrou `apps: []` e nenhum separador de browser; `cua.listWindows()` não estava disponível nesta sessão. Assim, não existia uma janela desktop operável na interface de controlo. A build Linux existente em `_bin` era anterior e não foi usada para declarar cobertura. Tentou-se reconstruir o alvo atual `ja2` com `cmake --build _bin --target ja2 -j2`; na sandbox, Cargo reportou a base de dados do cache como somente de leitura e processos `rustc` abortaram. A repetição autorizada fora da sandbox com `-j1` também falhou: rustup não conseguiu renomear o download parcial do componente `rust-src` (`No such file or directory`), e os scripts de build `quote`/Cargo terminaram com erro. Não foi possível obter um executável atual para abrir e capturar; por isso nenhum clique real nem tecla desktop foi enviado. Os comandos da moldura desktop desta execução ficam **não testados**, não “passaram” nem “não reproduzidos”. Não houve nova tentativa de renderer nem interação com a build antiga.

O AVD não foi alterado durante esta tentativa: manteve 1080×2400@420 dpi, retrato, rotação automática e launcher em primeiro plano. O save manteve o tamanho observado; dados originais e save foram preservados. Esta execução não reproduziu nem introduziu falha F008 e não alterou código Android ou lógica do laptop. Em conjunto com a matriz AVD já registada acima — navegação, favoritos/Cancel, minimizar/regressar, Shut Down/reentrada, Esc, Tab/Ctrl+Tab, retomada e três perfis paisagem — continuam pendentes os cliques/atalhos com rato e teclado desktop reais, bem como legibilidade, precisão dos alvos, áreas seguras, retoma e rotação num Pixel físico. F008 permanece em validação; SC-10 continua em investigação.

## Confirmação adicional, 24-10-2026

Na execução seguinte, o estado inicial do Git continuava em HEAD `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`, com as mesmas três alterações tracked e o relatório F008 não tracked; nada foi staged. A interface CUA voltou a devolver `apps: []` e os browsers visíveis não tinham tabs. A chamada documentada para listar janelas não estava disponível (`cua.listWindows is not a function`), portanto não havia sessão desktop operável. O caminho local indicado por `JA2_ORIGINAL_GAME_DIR` estava definido e era diretório; o valor não foi impresso.

ADB falhou dentro da sandbox ao iniciar o daemon (`could not install *smartsocket* listener: Operation not permitted`). Fora dela, `adb devices -l` listou apenas `emulator-5554` (`sdk_gphone64_x86_64`, `device`): Android 17/API 37, ABI `x86_64`, 1080×2400 a 420 dpi, rotação automática `1`, rotação atual `0`; `topResumedActivity` era `LauncherActivity`. O AVD já estava nesse estado e não foi alterado. Por `run-as`, `ls -ln` confirmou o save privado F006test com 409.643 bytes e timestamp 2026-10-22 21:39; não foi aberto, carregado ou gravado. `/sdcard/Download/JA2/Data` continuava presente, com 178 ficheiros. Nenhum ficheiro original foi copiado ou alterado.

Tentou-se reconstruir o alvo desktop atual com `cmake --build _bin --target ja2 -j1`. A build parou no alvo Rust `cargo-build-debug` (exit 101): Cargo não conseguiu atualizar a base de dados do cache por estar somente de leitura; rustup tentou sincronizar toolchains e obter `rust-src`, mas falhou ao criar temporário em `/home/marcio/.rustup/tmp` (`Read-only file system`). Não foi obtido nem aberto um executável atual, nem se usou uma build anterior. Não se enviaram cliques, teclas ou entradas por ADB ao jogo. A moldura não foi exercitada novamente no AVD nesta execução e não surgiu nova reprodução F008. Mantêm-se válidos como evidência anterior, sem serem repetidos agora, os três perfis paisagem e os percursos da matriz acima. Não há Pixel físico; legibilidade física, precisão dos alvos, área segura, retoma e rotação num Pixel continuam por testar. F008 fica em validação e SC-10 em investigação.

## Continuação, 24-10-2026

Estado inicial confirmado de novo: HEAD `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`; as alterações anteriores permaneciam em `AndroidManifest.xml`, neste plano, roadmap e neste relatório não staged. Nenhuma alteração foi staged ou descartada. O primeiro `adb devices -l` na sandbox falhou ao abrir o socket local (`Operation not permitted`); fora da sandbox listou apenas `emulator-5554` (`sdk_gphone64_x86_64`, Android 17/API 37, ABI `x86_64`). O AVD visível manteve 1080×2400@420 dpi, orientação atual 0, rotação automática ligada e `LauncherActivity` em primeiro plano; não foi alterado nem encerrado. Não havia Pixel físico. O save F006test foi apenas listado via `run-as` e continuava com 409.643 bytes; não foi aberto, carregado, copiado ou gravado. A pasta `Data/` permanecia no AVD; `JA2_ORIGINAL_GAME_DIR` estava definido e apontava para uma pasta disponível, sem revelar o seu valor nem copiar dados.

A CUA devolveu `apps: []` e browsers sem tabs; a interface não disponibilizou `cua.listApps()` nem `cua.listWindows()`. Não foi encontrada janela desktop interativa. `cmake --build _bin --target ja2 -j1` falhou primeiro dentro da sandbox (exit 101), no alvo `cargo-build-debug`: cache Cargo SQLite somente de leitura e rustup sem conseguir criar temporário em `.rustup`. A repetição fora da sandbox também terminou em exit 101 no mesmo alvo; rustup sincronizou para `rustc 1.101.0-nightly`, cujos artefactos Cargo existentes foram rejeitados com erros `E0514` de crates compiladas por versão incompatível. Não se limpou o cache nem os artefactos. Como não se produziu uma build desktop atual, não se abriu binário antigo, não se enviaram cliques/teclas e os comandos da moldura desktop ficaram **não testados**.

Nesta execução não se iniciou o jogo no AVD, não se repetiu a matriz AVD e não se reproduziu uma nova falha F008; portanto, não houve alteração de comportamento ou código. Os resultados anteriores permanecem evidência anterior: moldura, favoritos/Cancel, minimizar/regressar, Shut Down/reentrada, Esc, Tab/Ctrl+Tab, retoma e três perfis paisagem (2400×1080@420 dpi, 1600×720@320 dpi e 2560×1920@320 dpi). A precisão tátil/ergonomia, área segura, legibilidade, rotação e retoma continuam por verificar num Pixel físico; rato/teclado com build atual continuam por verificar em desktop. Dados JA2 e save foram preservados. F008 permanece em validação; SC-10 permanece em investigação.

## Verificação atual, 24-10-2026

O estado inicial do repositório foi confirmado em HEAD `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`, sincronizada com `origin/android-enhancement`. Permaneciam exatamente as alterações tracked em `android/app/src/main/AndroidManifest.xml`, neste plano e no roadmap, além deste relatório não tracked; não havia alterações staged. `JA2_ORIGINAL_GAME_DIR` estava definido e era um diretório; o seu valor não foi impresso.

O inventário CUA atual mostrou `apps: []` e apenas browsers sem tabs. `cua.listWindows()` não está disponível nesta sessão, portanto não foi possível selecionar uma janela desktop interativa. O executável `_bin/ja2` presente tem timestamp de 20-10-2026 e não foi aberto nem usado como evidência. A tentativa atual de `cmake --build _bin --target ja2 -j1` falhou dentro da sandbox com exit 101 no alvo `cargo-build-debug`: Cargo encontrou o cache SQLite somente de leitura e rustup sincronizou os toolchains. Repetida fora da sandbox, a build voltou a falhar com exit 101 porque `rustc 1.101.0-nightly` rejeitou metadados de dependências Rust existentes (`E0514`). Não se limpou cache nem artefactos. Sem build desktop atual e sem janela, os comandos da moldura por rato/teclado ficaram **não testados**; nenhuma hipótese de defeito desktop foi reproduzida.

ADB não conseguiu iniciar o daemon dentro da sandbox (`could not install *smartsocket* listener: Operation not permitted`), mas funcionou fora dela. Enumerou somente `emulator-5554` (`sdk_gphone64_x86_64`, Android 17/API 37, ABI `x86_64`). O AVD visível manteve 1080×2400@420 dpi, orientação atual 0/retrato, rotação automática ligada e `LauncherActivity` em primeiro plano. Não foi alterado nem encerrado. Por `run-as`, uma listagem de metadados confirmou que F006test continua com 409.643 bytes; o save não foi aberto, carregado, copiado ou gravado. `/sdcard/Download/JA2/Data` continuava presente com 178 ficheiros. A pasta indicada por `JA2_ORIGINAL_GAME_DIR` continuava disponível. Não há Pixel físico enumerado.

Esta execução não iniciou o jogo no AVD, não repetiu a matriz da moldura e não reproduziu nova falha F008; nenhum input foi enviado ao jogo. Os resultados AVD nos três perfis paisagem permanecem evidência **anterior**, não repetida: 2400×1080@420 dpi, 1600×720@320 dpi e 2560×1920@320 dpi, com navegação lateral, favoritos/Cancel, minimizar/regressar, Shut Down/reentrada, Esc, Tab/Ctrl+Tab e retoma. Continuam por verificar rato/teclado desktop numa build atual e legibilidade, precisão de alvos, áreas seguras, retoma e rotação num Pixel físico; tablet físico também não foi testado. Nenhum código ou comportamento foi alterado. F008 permanece em validação; SC-10 permanece em investigação.
