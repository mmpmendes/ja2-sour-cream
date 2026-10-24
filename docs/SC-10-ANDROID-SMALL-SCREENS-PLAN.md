# SC-10 — Plano faseado de acessibilidade Android em ecrãs pequenos

> Plano de investigação e implementação, 22-10-2026. Os resultados das fases concluídas estão registados abaixo; as restantes fases continuam propostas verificáveis. Este plano detalha SC-10 do [roadmap](SOUR_CREAM_ROADMAP.md) e usa SC-04 apenas para a escala visual comum.

## Contexto guardado para as próximas fases

- Instalação original de Jagged Alliance 2 disponível em `JA2_ORIGINAL_GAME_DIR` (existência da instalação confirmada em 22-10-2026). Os dados são necessários para ensaios locais; não os copiar para o repositório. No Android, transferir uma cópia legítima conforme [guia de dispositivo](ANDROID_BUILD_AND_DEVICE_TEST.md).
- Objetivo: tornar **toda a versão Android jogável por toque em ecrãs pequenos**, com zoom e controlos legíveis, sem perder a funcionalidade original. Implementar em fases pequenas e testar cada uma antes de avançar.
- Limite absoluto: **não mudar a mecânica base**. UI, input, apresentação, configuração e ciclo de vida podem mudar; custos de ações, alcance, linha de visão, probabilidades, IA, mapas, itens, economia, tempo, eventos, progressão e formato de save não mudam. Uma UI que pareça exigir uma alteração de regras deve usar uma alternativa visual ou de input e ser comparada com o mesmo estado em desktop.

## O que o código já permite afirmar

| Evidência | Consequência para o plano |
| --- | --- |
| `src/game/ScreenIDs.h` e `src/game/Screens.cc` enumeram e ligam menu, jogo tático, mapa, laptop, save/load, opções, diálogos, autoresolve, loja, introdução e ecrãs auxiliares. | A auditoria não pode limitar-se ao combate. `HelpScreen.cc`, popups e inventários são sobreposições dentro desses ecrãs. |
| `src/sgp/SGP.cc` converte eventos de rato/dedo para coordenadas lógicas SDL antes de os passar a `Input.cc`; `Video.cc` usa apresentação lógica com letterbox e modos de escala. | Capturas e testes de hit testing precisam de cobrir barras, rotação e combinações de resolução/escala; zoom do mundo exige uma segunda conversão explícita só no viewport. |
| `src/game/UILayout.cc` exige pelo menos 640×480 e mantém posições de inventário e viewport em coordenadas lógicas; a vista tática termina 120 unidades acima do fundo. | Ampliar todo o framebuffer cortaria controlos. Separar tamanho físico, resolução lógica, escala de UI e zoom do mundo. |
| `src/sgp/Input.cc` guarda um `gMainFingerId`; `FingerDown` substitui-o em cada novo dedo, mas `FingerUp` só aceita o identificador atual. `MouseSystem.cc` converte eventos em callbacks e deteta toque duplo. | Antes de reservar pinça ou dois dedos, auditar sequências intercaladas/canceladas e impedir que o dedo libertado execute uma ação. Não concluir ainda que há um bug reproduzido no dispositivo. |
| `src/game/Tactical/Real_Time_Input.cc` e `Turn_Based_Input.cc` usam dois dedos para `PAN_MODE`, toque repetido para postura e toque para preparar ações. `Tactical/Touch_UI.cc` mostra `Do`, `Aim`, `Mode`, `Item`, `Run`, `X` com botões de 32 unidades lógicas quando há toque. | Qualquer novo gesto compete com pan, mira, seleção e confirmação já existentes. Medir dimensão física e descobrir caminhos visíveis para os comandos em falta. |
| `src/game/Laptop/Laptop.h` enumera correio, finanças, IMP, A.I.M., M.E.R.C., Bobby Ray, seguros, florista e outras páginas. `src/game/Strategic/Map_Screen_Interface_Map_Inventory.cc` e `src/game/Tactical/ShopKeeper_Interface.cc` registam muitas regiões pequenas. | Inventário, formulários e lojas precisam de testes de selecionar, transportar, confirmar e cancelar; estes fluxos não são equivalentes a simples botões. |
| `android/app/src/main` tem launcher com páginas Data, Settings e Logs; o manifesto declara as duas atividades e `configChanges` para orientação/tamanho. | A matriz inclui instalação dos dados, permissões, teclado virtual, saída/retoma e rotação; a declaração no manifesto não prova que o estado visual sobreviva à rotação. |
| `LauncherActivity.kt::getRecommendedResolution()` já calcula uma resolução a partir das dimensões do ecrã e do modo de escala. | Usar o cálculo atual como linha de base; validar legibilidade, alvos físicos e layout em vários tamanhos antes de propor outra fórmula. Resolução automática por si só não prova escala de UI acessível. |
| `android/app/build.gradle` declara `compileSdk 37`, enquanto o [guia Android](ANDROID_BUILD_AND_DEVICE_TEST.md) ainda diz `compileSdkVersion 34`/SDK Platform 34. | Antes de repetir a build, reconciliar instruções e configuração num incremento documental próprio; não usar o guia como prova da configuração atual. |
| [Guia Android](ANDROID_BUILD_AND_DEVICE_TEST.md) regista build debug anterior, mas instalação/teste físico ficaram por verificar por ADB indisponível naquela sessão. | Não atribuir cobertura de toque ou zoom à build. Repetir a build e testar APK/atividade em equipamento real. |

**Estado da evidência:** isto é uma leitura estática do código, não uma reprodução de falhas na campanha. Relatos externos do roadmap são apenas casos candidatos. Faltam medidas de densidade, tamanho de alvo, taxa de erros e teste num telemóvel/tablet. As escolhas de layout e limiares de gesto ficam condicionadas aos resultados da fase 0.

## Regra de implementação para todas as fases

1. Antes de editar lógica, fixar commit, dados originais/localização, mods, save quando redistribuível, dispositivo, versão Android, orientação, resolução lógica, escala SDL, dimensões físicas e passos. Reproduzir a lacuna de acesso ou erro de toque; guardar vídeo/captura e resultado esperado/observado. Se não ocorrer, registar `não reproduzido` e não fazer correção hipotética.
2. Inventariar para cada comando: rato, teclado, toque atual, alternativa visível proposta, ação de cancelamento, feedback antes da confirmação e risco de ação irreversível. Usar a mesma função de comando já chamada pela UI original; não replicar cálculo de regras.
3. Um dedo pode selecionar ou arrastar conforme o contexto. Um gesto reservado a pan/zoom só entra em modo de navegação após classificação inequívoca; libertar/cancelar dedo nesse modo não deve gerar clique, tiro, movimento, troca, compra ou passagem de tempo. Após perda de foco, rotação ou nova atividade, limpar o estado transitório de input.
4. Primeiro ampliar a área clicável, ajustar composição, oferecer lente/inspector ou botões contextuais. Se isso não bastar, introduzir zoom **visual** do mundo com transformação direta e inversa testada. Manter painéis e janelas modais fora do zoom do mundo; repor zoom com botão visível. Pinça é opção adicional, nunca único caminho.
5. Quando uma alteração de UI parecer exigir regras novas, parar esse incremento e documentar a alternativa: selecionar primeiro e confirmar depois; oferecer botão que invoque o comando existente; apresentar informação já disponível ao jogador; ampliar apenas a renderização ou alvo de input. Não deslocar entidades, ajustar grade, arredondar alvo para uma quadrícula vizinha ou modificar custos para compensar precisão.
6. Em cada incremento: comparar com rato/teclado em desktop, toque no Android, e pelo menos um save existente quando o percurso o usa. Mesmo estado + mesmo comando devem chegar à mesma rotina e produzir o mesmo resultado, descontada a aleatoriedade normal. Não editar serialização nem prioridades do VFS para resolver UI.

## Matriz de teste que antecede as mudanças — fase 0 (`SC-10.1`)

**SC-10.1a — Preparar referência.** Guardar builds/commits, executar desktop com os dados JA2 locais, fazer capturas de 640×480 e de resolução maior, e montar APK debug segundo o guia. Preparar cenários com campanha nova, tática em tempo real e turnos, mapa, inventário, laptop, loja, saves e diálogos. Não versionar dados originais nem saves que os incorporem.

**SC-10.1a-doc — Pré-requisito de build.** Atualizar a divergência `compileSdk 37`/`34` entre Gradle e guia, verificar SDK instalado e só então documentar o comando que de facto compila o commit escolhido.

**SC-10.1b — Dispositivos.** O alvo físico inicial é um **Pixel 8, ecrã ~6", na horizontal**; o utilizador aceita ambiente virtual. Testar nele e em emuladores de telemóvel menor, maior e tablet, sem rato/teclado **dentro do jogo**; comandos de ADB/emulador servem apenas para preparar e recolher evidência. Registar dimensão em mm/dp, DPI, recorte/insets, Android/API, ABI, orientação, resolução da janela, resolução lógica e escala. O modelo de layout deve adaptar-se automaticamente às dimensões e densidade detetadas, sem constantes exclusivas do Pixel 8. O emulador não substitui ergonomia em tablet físico; registar essa cobertura como não verificada, sem a tornar bloqueio da primeira entrega. Apurar a versão Android do Pixel 8 durante a fase 0.

**SC-10.1c — Linha de base por ecrã.** Para todas as linhas da tabela seguinte, executar a ação e registar `passa`, `falha reproduzida`, `ambígua`, `não testada`; número de toques errados, tempo, necessidade de teclado, área física do alvo e legibilidade. Medir também botões junto às bordas, letterbox, ações perigosas e cancelamento. Distinguir erro de input, hit testing, layout, texto, teclado virtual e regras existentes.

**Saída da fase:** matriz preenchida com evidência para os comandos obrigatórios, capturas comparáveis e uma lista ordenada de lacunas reais. Se o ADB continuar bloqueado, esta fase continua aberta.

## Requisitos técnicos e catálogo de tarefas

Os blocos seguintes descrevem o trabalho por área, sem juntar ecrãs numa fase. O **registo de fases de um ecrã cada** está mais abaixo e determina a ordem de execução. Cada fase começa por observação e só implementa uma mudança quando a lacuna se reproduzir. `P0` impede um percurso completo por toque; `P1` causa erros frequentes; `P2` cobre fluxos ocasionais ou polimento. A prioridade final vem da matriz, não de pressupor defeitos.

### Pré-requisitos técnicos comuns (`SC-10.2`, `SC-10.3`, apoio a `SC-04.2`)

| Incremento | Trabalho verificável | Prova de saída |
| --- | --- | --- |
| A1 — estado dos dedos | Reproduzir sequências 1→2→1→0, 2→1→0, interrupção, toque fora de região, Android background/foreground. Se falharem, introduzir rastreio por ID e estado de gesto com cancelamento; manter mouse e toque único. Ficheiros: `SGP.cc`, `Input.cc`, `MouseSystem.cc`. | Testes de sequência para estado e callbacks; nenhum `UP` perdido nem ação em fim de gesto; rato/teclado iguais. |
| A2 — semântica visível | Definir ações universais `selecionar`, `confirmar`, `voltar/cancelar`, `abrir ações`, `deslocar vista`, `aproximar/afastar/repor`; nunca exigir duplo toque, toque longo ou dois dedos sem botão equivalente. Verificar colisão com pan tático já existente antes de mapear gestos. | Quadro gesto→comando por ecrã, incluindo prioridade de modais e estados de arrasto; revisão com vídeo de execução. |
| A3 — coordenadas | Especificar transformações janela→SDL lógico→UI ou viewport→mundo e inversa; testar extremos, insets, letterbox, rotação e resoluções. Ficheiros: `Video.cc`, `UILayout.cc`, `SGP.cc`, regiões MSYS. | Alvo tocado coincide com o desenhado antes/depois de escala e rotação; nenhuma quadrícula vizinha selecionada. |
| A4 — zoom tático de prova | Prototipar zoom apenas dentro do viewport, ancorado no ponto escolhido, com pan limitado, botões `+`, `−`, `repor`, estado transitório por ecrã. Proibir UI e overlays táticos de entrarem no zoom. Pinça apenas após A1/A2. | Capturas e teste de grid/objeto/soldado/portas nas bordas; confirmar que alcance, percurso, AP e resultado de ação permanecem iguais. Se exigir mexer nas regras, rejeitar protótipo e usar lente/inspector. |
| A5 — preferências | Definir onde guardar escala, zoom inicial e duração do toque se os testes justificarem controlos ajustáveis; expor valores no launcher ou opções. Não persistir em save. | Reabrir app preserva preferência; save permanece legível por build desktop e sem nova informação de jogo. |

## Registo de fases: exatamente um ecrã ou estado visual por fase

Cada linha é uma **fase independente** de SC-10: observar no Pixel 8 e no emulador pertinente, registar todos os comandos desse ecrã, corrigir apenas lacunas reproduzidas e fechar com a verificação indicada. A verificação comum da fase inclui **toque, alternativa visível, cancelar/voltar, escala automática, regressão com rato/teclado e ausência de mudanças mecânicas**. Se um ID do motor não for alcançável no jogo normal, a fase termina com evidência da condição de entrada e a classificação `não alcançável`; não se inventa UI. `F001` etc. são etiquetas deste plano dentro de SC-10, não estados do roadmap. A ordem pode mudar com a linha de base, mas nunca se fundem dois ecrãs na mesma fase.

### Launcher e abertura

| Fase | Ecrã/estado | Trabalho particular e verificação de saída |
| --- | --- | --- |
| F001 | Launcher — Data | Seleção de dados, versão e saves; testar diretório errado, permissão negada e relançamento (`DataTabFragment.kt`). |
| F002 | Launcher — Settings | Resolução automática, escala e teclado dos campos; testar Pixel 8 e perfis menores/maiores (`SettingsFragment.kt`). **Estado: concluída; evidência abaixo.** |
| F003 | Launcher — Logs | Ler/copiar log, voltar e relançar após erro (`LogsTabFragment.kt`). **Estado: concluída; evidência abaixo.** |
| F004 | Splash/inicialização `INIT_SCREEN` | Visibilidade, espera e transição sem toque perdido (`JAScreens.cc`). |
| F005 | Introdução `INTRO_SCREEN` | Verificar reprodução e avanço por toque no fluxo original de novo jogo; confirmar a saída para `INIT_SCREEN` e a continuação da campanha (`Intro.cc`). **Concluída no AVD; validação física pendente.** |
| F006 | **Main Title** `MAINMENU_SCREEN` | Novo jogo, continuar/carregar, opções, créditos e sair; confirmar alvos/voltar (`MainMenuScreen.cc`). **Concluída no AVD; validação física pendente.** |
| F007 | Novo jogo `GAME_INIT_OPTIONS_SCREEN` | Dificuldade e opções iniciais legíveis e selecionáveis (`GameInitOptionsScreen.cc`). **Concluída no AVD; validação física pendente.** |

**SC-10/F007 — reprodução e correção, 22-10-2026:** commit inicial `2609006c488d87bf863ba6f0371b269d2d8787e7`, árvore Git limpa. O código confirma `MAINMENU_SCREEN` → New Game → `GAME_INIT_OPTIONS_SCREEN`: três dificuldades, estilo Realistic/Sci-Fi, armas Reduced/Gun Nut, gravação Save Anywhere/Iron Man/Dead is Dead, Done e Cancel. As caixas selecionam em `POINTER_UP`; Enter executa Done e Esc executa Cancel. A confirmação de gravação precede a de dificuldade; No no aviso de gravação repõe Save Anywhere, No na confirmação de dificuldade mantém F007, Cancel volta ao título e Yes segue o fluxo original. O AVD visível `Medium_Phone_API_37.0`, Android 17/API 37, `x86_64`, tinha 1080×2400 a 420 dpi de base. Com o jogo em paisagem efetiva 2400×1080, toque no texto **Experienced** (aprox. 735,435) deixou **Novice** selecionado; toque na caixa correspondente (1040,435) selecionou **Experienced**. Esperado: a legenda visível também selecionar a opção, com alvo tátil utilizável. A lacuna de hit testing da legenda foi reproduzida antes de alterar código. Capturas antes/depois e registos SDL/ADB foram inspecionados localmente; as capturas temporárias foram removidas após a validação.

**Alteração e validação F007:** apenas em Android, as legendas das quatro famílias de opções receberam áreas de toque que encaminham `POINTER_UP` para a seleção exclusiva das caixas existentes, sem modificar as regras ou o desenho. O APK foi reconstruído com `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug --max-workers=1` (`BUILD SUCCESSFUL`, Gradle 9.8.2, `compileSdk 37`, NDK 29.0.14206865) e instalado no AVD. No perfil original em paisagem 2400×1080@420 dpi, toques nas legendas Novice/Experienced/Expert, Realistic, Tons of Guns e Iron Man mudaram apenas a escolha pretendida. Done em Iron Man mostrou o aviso; No regressou a Save Anywhere. Done em Expert mostrou confirmação; No manteve F007. Cancel e Esc regressaram ao título; Enter e Yes iniciaram uma única transição para a abertura da campanha, verificada apenas à entrada. No perfil compacto 1600×720@320 dpi (override 720×1600), todas as escolhas e Done/Cancel continuaram visíveis e o texto Experienced foi acionável. No perfil tablet virtual 2560×1920@320 dpi (override 1920×2560), as opções também permaneceram distintas; toque em Experienced e Dead Is Dead funcionou, Done abriu o aviso e No regressou a Save Anywhere. Home→Recentes→jogo retomou F007 com a seleção mantida, sem ação repetida. Foi usado ADB para injetar toques no AVD porque a interface de computador desta sessão não expôs a janela Android para interação direta; os pontos foram escolhidos nas capturas do ecrã visível. O AVD ficou ligado; os overrides de tamanho, densidade e rotação foram repostos (1080×2400@420 dpi, rotação livre).

**Limites F007:** uma suspeita de desalinhamento dos alvos no perfil compacto **não se reproduziu** após medir os pontos na captura. Um toque de rato simulado por ADB moveu o cursor mas não confirmou clique, inclusive sobre uma caixa original; assim, rato físico Android continua por validar e não foi inferido defeito específico de F007. O teste de rotação dinâmica não confirmou preservação do mesmo frame; perfis em paisagem foram validados separadamente e a retoma por Home/Recentes passou. Sem Pixel ou tablet físico nesta execução, ergonomia e rotação em hardware ficam pendentes. F007 fecha para a lacuna reproduzida e para a validação no AVD; F005 e F006 permanecem abertas, e SC-10 continua em investigação.

**SC-10/F001 — linha de base e alteração, 22-10-2026:** commit base `ff066c9de2d38b8a5614bf4d59b13f6daf415b79`; `compileSdk 37`, `targetSdk 26`, NDK `29.0.14206865`; wrapper Gradle 9.8.2. Foi reproduzida no Pixel 8 a sobreposição do botão Choose da pasta de saves com o FAB de iniciar jogo ao chegar ao fim da página Data em horizontal. Recusar permissões também deixava apenas um Toast transitório sem ação; a validação existente rejeitou corretamente `/storage/emulated/0/Download` (não contém `Data`) e permitiu cancelar. A pasta original já presente no telefone em `Download/Jagged Alliance 2` foi selecionada e aceite; o caminho persistiu ao regressar ao launcher. Nenhum ficheiro de jogo foi copiado para o repositório.

**Alterações F001:** o ScrollView da página Data reserva espaço inferior para o FAB; a recusa agora mostra diálogo persistente com Cancel e Open settings. O guia de build foi corrigido após inspeção de `android/app/build.gradle` e do wrapper. Build atual `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` passou; APK instalado no Pixel e AVD. Pixel 8, Android 17/API 37, 1080×2400 físico, 420 dpi, app 2400×1080 em rotação horizontal: depois de iniciar pelo FAB, apareceu o menu principal do JA2 e a atividade do jogo permaneceu horizontal. A página Data também foi verificada em vertical, conforme requisito do utilizador. Em AVD Android 37, foram aplicados overrides 720×1600@320 dpi (telemóvel pequeno) e 1600×2560@320 dpi (tablet); controlos continuaram visíveis, roláveis e separados do FAB. No AVD verificaram-se seleção de pasta de saves, recusa de permissão, abertura de App settings e Cancel. Capturas locais em `/tmp/f001-scroll.png`, `/tmp/f001-postfix.png`, `/tmp/f001-small.png`, `/tmp/f001-tablet.png`, `/tmp/perm-emulator.png` e `/tmp/f001-final.png`; não versionadas.

**Resultado/limites:** os cenários reproduzidos e a validação de orientação passaram após a correção; cancelamento do seletor e retorno ao launcher foram verificados. A build desktop e o percurso equivalente com rato/teclado não foram repetidos porque nenhuma lógica ou input do jogo foi alterado. A campanha, saves, VFS e formatos não foram modificados. F001 pode ser fechada para a página Data; ficam sem cobertura o tablet físico e as versões de Android fora dos dispositivos usados. A instalação original é privada e não foi incluída no repositório.

**SC-10/F002 — linha de base, reprodução e alteração, 22-10-2026:** commit inicial `7ce3713ed771600a160354081b3796a33aa763bb`, ramo `android-enhancement`, árvore Git limpa. O APK debug já existente foi instalado no Pixel 8, Android 17/API 37, ABI `arm64-v8a`, 1080×2400, 420 dpi; orientação inicial vertical. Em Data, toquei Settings. O resultado esperado era alcançar também o botão de resolução automática; observado: os campos ocupavam a largura disponível e Detect não aparecia na captura nem na árvore `uiautomator` (larguras 550 px e 406 px, mais o separador, preenchiam a linha de 974 px). Ao abrir o campo numérico, o teclado Gboard apareceu; com pouco espaço, o FAB ficava sobre o texto de Scaling Mode. Um gesto de scroll levava ao spinner e a Debug; Voltar fechou o teclado sem sair da página nem alterar os valores. Em horizontal, Detect estava visível e o toque mudou 980×480 para 1070×480. A rotação de volta, a navegação Settings→Logs→Data→Settings e a permanência temporária do valor foram observadas. Um force-stop/reabertura restaurou 980×480 e Debug desativado, sem persistir as alterações transitórias.

**Percurso e limites confirmados no código:** resolução é editada em dois campos numéricos e o botão Detect reutiliza `getRecommendedResolution()`. Conversões `UInt` atualizam imediatamente o `ConfigurationModel`; campo vazio ou texto não convertível não atualiza o modelo e zero é aceite, sem alteração nesta fase. Escala oferece `LINEAR` (Linear Interpolation), `NEAR_PERFECT` (Near perfect with oversampling) e `PERFECT` (Pixel perfect centered); Debug é booleano. `LauncherActivity.startGame()` valida dados, grava `res`, `scaling` e `debug` no `.ja2/ja2.json` e abre `StracciatellaActivity`; `rust/stracciatella/src/config/ja2_json.rs` aplica esses mesmos valores às opções do motor. Não iniciei o jogo nesta fase, para não gravar opções nem alterar dados locais; a aplicação ao motor não foi revalidada em runtime. Detect resultou em 980×480 na orientação vertical e 1070×480 na horizontal deste Pixel; a fórmula existente foi mantida.

**Alteração F002:** apenas `fragment_launcher_settings.xml`: os campos usam largura proporcional (`0dp`/peso igual), preservando espaço adaptativo para Detect; o ScrollView reserva 88dp no fim e não recorta o padding, de modo que o FAB não cobre os controlos quando o teclado está aberto e o conteúdo é rolado. A página Data e o fluxo de Settings não foram alterados. Os mesmos controlos, rótulos, listeners, validação, modelo e persistência continuam em uso.

**Validação F002:** `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` passou (`BUILD SUCCESSFUL`; Gradle usou a cache local fora da sandbox). O APK atualizado foi instalado e verificado no Pixel 8: Detect visível/tátil em vertical e horizontal; toque em horizontal atualizou o modelo; rotação conservou o valor visível; Gboard, scroll até escala/Debug, Voltar e regresso entre Data/Logs/Settings passaram. No AVD `Medium_Phone_API_37.0`, Android 17/API 37, x86_64, o perfil 720×1600@320 dpi mostrou todos os controlos; os três modos de escala foram legíveis/selecionáveis e Debug ligou/desligou por toque. Perfil tablet virtual 1920×2560@320 dpi e a sua rotação horizontal 2560×1920 mantiveram Detect, escala e Debug no layout. Overrides de size, density e rotação foram repostos; Pixel ficou em rotação automática, vertical. Capturas e árvores de UI úteis, não versionadas: `/tmp/f002-settings.png`, `/tmp/f002-keyboard-final.png`, `/tmp/f002-landscape.png`, `/tmp/f002-fixed-portrait.png`, `/tmp/f002-pixel-keyboard-scroll-fixed.png`, `/tmp/f002-small.png`, `/tmp/f002-tablet-portrait.png` e `/tmp/f002-tablet-landscape.png`.

**Resultado/limites:** a falha de visibilidade do botão e a sobreposição com teclado foram reproduzidas e corrigidas; os cenários de launcher aceites passaram nos perfis indicados. `git diff --check` passou. Não alterei opções, formato de configuração, código compartilhado com desktop, regras, saves, VFS ou dados JA2. Não há tablet físico nesta sessão; não repeti o percurso de arranque do jogo, a campanha ou a regressão de rato/teclado desktop porque a alteração é apenas de layout Android. SC-10 mantém-se em investigação: F002 fecha somente a página Settings.

**SC-10/F003 — linha de base, reprodução e alteração, 22-10-2026:** commit inicial `eec5738296186e923135c8afceaba091aa41bccb`, ramo `android-enhancement` (um commit à frente de `origin/android-enhancement`), árvore de trabalho limpa. `android/app/build.gradle` confirma `compileSdk 37`, NDK `29.0.14206865` e ABIs `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`; o wrapper usa Gradle 9.8.2 (AGP 9.4.1/KGP 2.4.20 nas configurações atuais). O ADB do SDK confirmou o Pixel 8 autorizado: Android 17/API 37, `arm64-v8a`, 1080×2400 a 420 dpi, vertical com rotação automática. O APK debug existente foi instalado antes da correção. Passos de reprodução: abrir o launcher, tocar Logs (no Pixel vertical, centro aproximado em 900×350), percorrer o conteúdo com swipes para cima até ao fim e observar as últimas linhas; repetir após rodar para horizontal. O cache continha o `ja2.log` real (33 linhas, 5.370 bytes). Esperado: ler/percorrer todas as linhas sem obstrução e alcançar copiar; observado: o FAB de iniciar cobria as últimas linhas, em vertical e horizontal (2400×1080 a 420 dpi). O botão tinha alvo aproximado de 56 dp e era tocável. Copiar colocou o texto completo na área de transferência e mostrou `Copied logs to clipboard`. Navegação Logs→Settings→Data→Logs e Voltar (que fecha o launcher) seguida de relançamento funcionaram. A linha final do log mostrava que a execução anterior chegou a `GameLoop`/versão; não houve erro de arranque nem exceção recuperável reproduzível, pelo que esse cenário fica **não reproduzido**.

**Alteração F003:** apenas `android/app/src/main/res/layout/fragment_launcher_logs_tab.xml`. O ScrollView passa a ocupar o espaço restante (`0dp` + peso) e reserva 88 dp abaixo do seu viewport para o FAB; o texto mantém largura disponível e altura de conteúdo, permitindo scroll até ao fim sem passar sob o FAB nem pela barra de navegação. Descrição e botão copiar permanecem visíveis. Nenhum log, listener, clipboard, mensagem ou comando foi alterado.

**Validação F003:** `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` passou (`BUILD SUCCESSFUL`; Gradle 9.8.2), APK instalado no Pixel 8 e no AVD `Medium_Phone_API_37.0`. No Pixel, o scroll até à última linha foi repetido após a correção: a linha final ficou acima do FAB e da navegação do sistema; em horizontal, a faixa do FAB ficou fora da área rolável. Foram confirmados toque em copiar, clip de 5.370 bytes/UID da app e Toast; troca Logs/Settings/Data/Logs, Voltar, relançamento e rotação sem perder o conteúdo observado. A rotação automática do Pixel foi reposta. No AVD Android 17/API 37, ABI `x86_64`, perfil original 1080×2400@420 dpi; overrides 720×1600@320 dpi (vertical/horizontal) e tablet virtual 1920×2560@320 dpi (vertical/horizontal) mantiveram descrição e botão visíveis, ScrollView separado do FAB e navegação utilizável. O AVD não tinha dados originais nem `ja2.log`, portanto só validou layout vazio, toque, navegação e orientação, não o scroll de log real. Size/density/rotação foram repostos; o Pixel não estava ligado ao terminar a limpeza, depois de a rotação automática ter sido reposta. Capturas e cópia local do log foram removidas após registar os resultados; nenhum dado de JA2 foi copiado para o repositório. Não executei testes automatizados: a alteração é somente de layout. `git diff --check` passou. Não alterei Data, Settings, código partilhado com desktop, rato/teclado, arranque do jogo, regras, saves, VFS ou formatos de configuração; por isso não repeti desktop/campanha. **Preferência para as fases seguintes:** testar primeiro no AVD; usar o Pixel só se o AVD não cobrir o cenário e, se o Pixel estiver inacessível, pedir ao utilizador/dev que o ligue.

**Resultado/limites:** F003 fecha apenas Launcher — Logs; a lacuna reproduzida foi corrigida e os cenários de apresentação, cópia, navegação e orientação passaram. Recuperação após erro de arranque ficou não reproduzida porque não havia erro disponível; não se provocou falha artificial. Sem tablet físico e sem validação em versões Android fora da API 37. SC-10 continua em investigação.


**SC-10/F004 — Splash/inicialização `INIT_SCREEN`, 22-10-2026:** commit inicial `509df901c443273bdce5bff52a66c3a29cfc17ca`, ramo `android-enhancement`, árvore limpa. Build debug atual executada com `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug --max-workers=1` (Gradle wrapper 9.8.2; `compileSdk 37`, NDK 29.0.14206865); `BUILD SUCCESSFUL`.

**Reprodução antes da alteração:** AVD `Medium_Phone_API_37.0`, Android 17/API 37, ABI `x86_64`, perfil inicial 1080×2400@420 dpi, retrato; dados originais selecionados no launcher. Passos: abrir o launcher, iniciar o jogo e aguardar o splash/logo e menu. Esperado: launcher utilizável em retrato e atividade do jogo em paisagem. Observado: `INIT_SCREEN`/menu permaneciam em retrato; SDL registou pedido com `requestedOrientation=13` e o ecrã continuou 1080×2400 até rotação manual. Um toque durante o splash não iniciou campanha e o menu apareceu normalmente. Voltar (`KEYCODE_BACK`) durante o arranque/menu não cancelou nem fechou o jogo; não se observou perda de estado. Home suspendeu a atividade; retomar pela tarefa recente regressou ao menu. Não foi testado rato físico.

**Alteração e validação:** `src/sgp/SGP.cc` define `SDL_HINT_ORIENTATIONS=LandscapeLeft` apenas em Android, antes de `SDL_Init(SDL_INIT_VIDEO)`. A build final pediu `requestedOrientation=0` (`LandscapeLeft`) e, no perfil tablet virtual 1920×2560@320 dpi/API 37, rodou a janela para 2560×1920. O AVD aplica por omissão `ignoreOrientationRequest=true` a ecrãs grandes; para validar a resposta do código, essa política foi temporariamente desativada, depois reposta. A validação de perfis de telefone 720×1600@320 dpi em retrato e 1600×720@320 dpi em paisagem, incluindo launcher e menu, já tinha sido feita na build com hint de orientação nas duas direções; o perfil tablet confirmou o comportamento da versão final com orientação fixa. Capturas e dados copiados para o AVD foram removidos. Overrides de tamanho, densidade e rotação foram repostos, assim como `ignoreOrientationRequest=true`; AVD deixado ligado. Sem teste Pixel (o AVD reproduziu e permitiu validar o pedido de orientação) e sem teste desktop (alteração Android-only). `git diff --check` passou; sem testes automatizados, pois não foi alterada lógica de regras.

**Resultado/limites:** F004 fecha apenas a inicialização `INIT_SCREEN`. A lacuna reproduzida de arranque em retrato foi corrigida; toques acidentais no splash e cancelamento por Voltar não produziram transição inesperada. No AVD API 37 com política padrão de tablet, o Android pode ignorar o pedido de orientação; o comportamento de orientação fixa foi validado com essa política temporariamente desativada. Não há validação em hardware físico nem noutras versões Android. SC-10 continua em investigação.

**SC-10/F005 — reprodução e percurso real, 22-10-2026; fase aberta:** commit inicial `19e918293dac930abf98ef0c8fbb5ea410fbcc5e`, ramo `android-enhancement`, árvore de trabalho limpa. F004 está concluída (evidência acima) e SC-10 permanece em investigação. `android/app/build.gradle` confirmou `compileSdk 37`, NDK `29.0.14206865`, ABIs `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`; wrapper `android/gradle/wrapper/gradle-wrapper.properties` confirmou Gradle 9.8.2. A build de referência `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug --max-workers=1` passou (`BUILD SUCCESSFUL`; cache Gradle exigiu executar fora da sandbox). APK debug instalado no AVD.

**Dispositivo e reprodução:** AVD `emulator-5554`, Android 17/API 37, ABI `x86_64`. Perfil original 1080×2400 a 420 dpi em retrato; perfil pequeno usado com overrides `wm size 720x1600`, `wm density 320`, orientação Android 90°, resultando em 1600×720 efetivos (paisagem). Os dados originais foram transferidos da variável `JA2_ORIGINAL_GAME_DIR` apenas para `/sdcard/Download` do AVD; nenhum dado foi copiado para o repositório. Passos: abrir launcher, iniciar, tocar em **Start New Game**, aceitar opções iniciais e confirmar o aviso de dificuldade. Resultado esperado segundo a linha F005 anterior: poder ver/avançar a intro e regressar ao Main Title. Observado: `INTRO_SCREEN` toca a sequência de quatro vídeos de introdução do novo jogo; em retrato e paisagem o primeiro vídeo/legenda foi apresentado. Sem interação, os vídeos progrediram e a saída entrou na inicialização/cinemática da campanha.

**Descoberta:** no fluxo `INTRO_BEGINING`, tocar no fundo fecha apenas o clipe atual e a sequência natural termina na inicialização/cinemática da campanha, não no Main Title.

**Input e destino confirmados no código/runtime:** `GameInitOptionsScreen.cc` define `INTRO_BEGINING` e sai para `INTRO_SCREEN`; `Intro.cc` cria uma região de fundo de ecrã inteiro. Um toque nessa região fechou apenas o clipe atual e a sequência avançou para o seguinte. `Espaço` também chama `SmkCloseFlic`; `Esc` chama `PrepareToExitIntroScreen`. Não existe botão/indicação de skip no vídeo. O clique de fundo é o mecanismo original, corresponde ao toque recebido e funcionou; não se adicionou uma ação nova nem se mudou o código. Voltar (`adb shell input keyevent 4`) não cancelou a sequência no ensaio; a reprodução prosseguiu. Não houve rato físico; não se alterou o percurso de rato/teclado partilhado.

**Divergência de escopo:** para `INTRO_BEGINING`, `PrepareToExitIntroScreen()` retorna `INIT_SCREEN`. `InitScreenHandle()` segue por `InitializeJA2()`/`InitNewGame()` e a espera natural mostrou a cinemática de campanha, não `MAINMENU_SCREEN`. Assim, “saltar para Main Title” não descreve o comportamento original deste `INTRO_SCREEN`. Redirecionar essa saída mudaria a sequência de novo jogo; não foi feito. F005 permanece aberta até se confirmar qual introdução/destino deve ser objeto desta fase.

**SC-10/F006 — reprodução no Main Title, 22-10-2026; fase aberta:** commit inicial `56c98ec117a8a367c25100098db09336dd504d89`, ramo `android-enhancement` (um commit à frente de `origin/android-enhancement` no início), árvore inicial limpa. F004 está concluída; SC-10 permanece em investigação e F005 continua aberta porque `INTRO_BEGINING` regressa a `INIT_SCREEN`/campanha, não a `MAINMENU_SCREEN`. Não se alterou esse percurso. `android/app/build.gradle`: compileSdk 37, NDK `29.0.14206865`, ABIs `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`; wrapper Gradle 9.8.2. Build atual executada com `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug --max-workers=1`, `BUILD SUCCESSFUL`; instalada com `adb install -r`. O primeiro ADB na sandbox falhou ao criar socket (`Operation not permitted`); fora da sandbox, `adb devices -l` encontrou `emulator-5554` ativo. O diretório original continha `data/` em minúsculas; os ficheiros originais foram copiados apenas para `/sdcard/Download/JA2` no AVD, nunca para o repositório.

**Dispositivo e perfis:** AVD visível `emulator-5554`, Android 17/API 37, ABI `x86_64`. Perfis usados: retrato 1080×2400@420 dpi (sem overrides); paisagem compacta efetiva 1600×720@320 dpi; tablet virtual paisagem 2560×1920@320 dpi (override 1920×2560 antes de rodar). A rotação forçada para paisagem foi usada apenas durante os dois últimos perfis. Ao terminar foram repostos `wm size reset`, `wm density reset`, `accelerometer_rotation=1` e `user_rotation=0`; AVD deixado ligado. Capturas de cada ecrã e logcat SDL foram inspecionados em `/tmp/f006-*` e removidos após o registo. Sem Pixel nem rato físico; rato CLI não foi aceite como teste de clique e não conta como validação. Teclado Android: `O` abriu Options no perfil retrato e o caminho regressou ao título.

**Resultados reproduzidos por toque:** no retrato 1080×2400, Start New Game abriu Initial Game Settings, Cancel regressou ao título, Preferences abriu Options e Done regressou; Continue/Load não acionou transição porque não havia save, mantendo-se desativado. Quit do Main Title saiu para o launcher, sem confirmação; Options > Quit mostrou confirmação, No manteve Options e Yes regressou ao Main Title, conforme a semântica original observada. Na paisagem compacta 1600×720, toques medidos diretamente no screenshot abriram Start New Game e Preferences, e Cancel regressou. Uma primeira tentativa tinha usado coordenadas estimadas incorretamente e foi descartada; com coordenadas corretas, o alegado desalinhamento não se reproduziu. No tablet virtual, Start New Game abriu Initial Game Settings, Cancel regressou, Preferences abriu Options e Done regressou. Continue/Load com save existente não foi validável sem saves de teste. Não se entrou em campanha: no novo jogo confirmou-se apenas a entrada em Initial Game Settings.

**Ciclo de vida e limites:** no perfil compacto, Home suspendeu a atividade; ao retomar a tarefa por Recentes, `SDL.onResume()` e `surfaceCreated()` foram registados e o Main Title voltou visível, sem bloqueio. Não se observou repetição de ação. Não foi possível completar uma rotação de dispositivo no perfil de telefone sem impor manualmente os overrides; perfis retrato e paisagem foram validados separadamente. Não foi testado save/carregamento com save existente, mouse físico nem hardware Pixel. Não se reproduziu lacuna em `MAINMENU_SCREEN`; nenhum código foi alterado. F006 continua aberta até validação dessas lacunas e de rotação/lifecycle em condições representativas; SC-10 permanece em investigação.

**Fecho F005/F006 no AVD, 22-10-2026:** novo ponto de partida no commit `2609006c488d87bf863ba6f0371b269d2d8787e7`; estavam presentes apenas as três alterações não staged de F007 em `GameInitOptionsScreen.cc` e nos dois documentos SC-10. O APK debug já compilado e instalado na validação F007 foi reutilizado; não houve alteração de código nem nova build. AVD visível `Medium_Phone_API_37.0` com SwiftShader, Android 17/API 37, `x86_64`, base 1080×2400@420 dpi, execução prioritária em paisagem efetiva 2400×1080@420 dpi. Os dados originais já selecionados em `/sdcard/Download/JA2` permaneceram apenas no AVD; não foram copiados para o repositório. Capturas de cada transição foram inspecionadas localmente. Como a interface de computador não expôs a janela Android operável, foram injetados toques por ADB em coordenadas medidas nas capturas do AVD visível.

**F005, percurso observado:** Launcher → Start New Game → Initial Game Settings → Ok → Yes iniciou `INTRO_SCREEN` em paisagem. A legenda Rebel base in Omerta e o vídeo de Prague apareceram; um toque no fundo avançou o clipe em reprodução. Home → Recentes → cartão do jogo retomou a sequência, sem voltar ao título ou reiniciar a campanha. A sequência terminou na abertura do laptop da campanha (dia 1, 01:00); após Shut Down, o mapa inicial apareceu. Isso confirma em runtime o destino `INTRO_BEGINING` → `INIT_SCREEN` → campanha mostrado em `Intro.cc`/`JAScreens.cc`. A referência anterior a regressar ao Main Title era uma expectativa incorreta do plano e foi corrigida nesta linha, sem redirecionar o código. `Espaço` e `Esc` continuam disponíveis pelo código original; não se acrescentou gesto nem botão de skip. A rotação com mudança forçada de perfil já tinha recriado a atividade no ensaio anterior; preservação do mesmo frame nessa condição e rato físico permanecem sem validação. A introdução normal por toque não exige skip para prosseguir. **F005 concluída para o percurso original no AVD.**

**F006, comandos e regressos observados:** antes de existir save, Continue estava desativado e um toque não causou transição. Em paisagem, Preferences → Options → Done regressou ao título; Credits abriu o ecrã correspondente e Esc regressou ao título; Quit do título regressou ao launcher. Home → Recentes retomou o mesmo título, sem ação residual. A mudança forçada de retrato para paisagem recriou o splash e voltou ao título, sem transição duplicada. New Game abriu Initial Game Settings, já validado em F007. Para testar Continue, a campanha recém-iniciada foi guardada via mapa → Options → Save Game; a mensagem `GAME SAVED` apareceu e o slot `F006test` mostrou dia 1, 01:00, 0 mercenários e $45.000. Options → Quit mostrou a confirmação: No manteve Options, Yes regressou ao título. Continue ficou ativo, abriu Load Game com esse slot, Cancel regressou ao título; nova entrada e Load Game retornaram ao mapa guardado. Os perfis compactos e tablet virtuais em paisagem, e o retrato base, já constam da validação F006 anterior. Não se reproduziu falha de alvo no título; não se mudou a sua lógica. Rato físico, Pixel/tablet físicos e preservação exata de frame numa rotação dinâmica continuam pendentes. **F006 concluída para os comandos do título no AVD; SC-10 permanece em investigação.**

**Ciclo de vida e cobertura:** apresentação e toque para fechar clipe validados em 1080×2400@420 dpi (retrato), e apresentação/espera natural validadas no perfil pequeno em 1600×720@320 dpi (paisagem); launcher e título foram usados apenas como entrada/destino. `Espaço` foi enviado durante vídeo. Home/Recentes foi inconclusivo: após voltar de Recentes, o processo tinha saído com `EXIT_SELF`, a atividade SDL foi recriada e mostrou o splash de arranque; não há evidência de que `INTRO_SCREEN` se repetiu, nem foi possível confirmar recuperação do mesmo frame. Mudança forçada de resolução/rotação durante essa tentativa interferiu na leitura do resultado. Voltar, rotação durante um vídeo e suspensão/retoma sem alteração manual de perfil continuam por validar. Não foi necessário usar Pixel: o AVD reproduziu toque e espera. Sem rato físico, outras APIs Android ou tablet físico; risco principal é a incompatibilidade entre o destino pedido (Main Title) e o fluxo de campanha observado. As capturas e os logs temporários foram removidos após o registo desta evidência; nenhum dado proprietário entrou no repositório.

### Computador: moldura e **uma fase por página**

Os nomes abaixo correspondem a `LaptopMode` em `Laptop.h`; modos transitórios ou sem handler visível continuam com fase própria para verificar alcance/saída. O ficheiro de entrada e mudança de modo é `Laptop.cc`; a página usa o ficheiro da família indicado na coluna. Não alterar preços, contratos, geração de personagem, eventos ou tempo de jogo.

| Fase | Página/estado único | Trabalho particular e verificação de saída |
| --- | --- | --- |
| F008 | Moldura `LAPTOP_SCREEN` | Navegação, favoritos, voltar/fechar, área segura e legibilidade (`Laptop.cc`). **Incompleta, em validação, 24-10-2026:** retoma corrigida; matriz AVD executada anteriormente. Nesta verificação, ADB encontrou só `emulator-5554`, em 1080×2400@420 dpi no launcher; save (409.643 bytes) e presença de `Data/` foram confirmados sem alteração. CUA mostrou `apps: []`, browsers sem tabs e não disponibilizou `cua.listWindows()`. `cmake --build _bin --target ja2 -j1` falhou primeiro com exit 101 (`E0514`, artefactos incompatíveis com `rustc 1.101.0-nightly`). **Nova execução no HEAD `97d6301d3373e3819c6d84de50b6c890ae69d6dd`:** `RUSTUP_TOOLCHAIN=1.85.1 cmake --build _bin --target ja2 -j1` reproduziu a falha C++ em `SGP.cc:420`; a linha comentada em `:337` era a leitura existente de `EngineOptions_shouldStartInFullscreen`. Restaurar apenas essa declaração preservou o comportamento de `--fullscreen`/`ja2.json` e a build passou. A CUA continuou sem apps/janela selecionável; não iniciei o executável para respeitar a proibição de abrir dados originais. Assim, comandos desktop não foram testados; a matriz AVD não foi repetida nem surgiu falha F008 nova. O AVD permaneceu ligado no launcher, save (409.643 bytes) e `Data/` só tiveram metadados confirmados; não havia dispositivo físico. Ergonomia física e comandos desktop permanecem pendentes. [Inventário, reprodução e resultados](SC-10-F008-LAPTOP-FRAME.md). |
| F009 | `LAPTOP_MODE_NONE` | Desktop interno: ícones e entrada nas apps, regressar sem bloqueio (`Laptop.cc`). |
| F010 | `LAPTOP_MODE_FINANCES` | Folhear movimentos e ler valores (`Finances.cc`). |
| F011 | `LAPTOP_MODE_PERSONNEL` | Selecionar perfis e navegar separadores (`Personnel.cc`). |
| F012 | `LAPTOP_MODE_HISTORY` | Folhear cronologia, datas e voltar (`History.cc`). |
| F013 | `LAPTOP_MODE_FILES` | Lista de ficheiros legível e selecionável (`Files.cc`). |
| F014 | `LAPTOP_MODE_FILES_ENRICO` | Abrir, percorrer e fechar o ficheiro; confirmar alcance em `Files.cc`. |
| F015 | `LAPTOP_MODE_FILES_PLANS` | Abrir, percorrer e fechar planos; confirmar alcance em `Files.cc`. |
| F016 | `LAPTOP_MODE_EMAIL` | Lista, seleção, navegação e avisos (`EMail.cc`). |
| F017 | `LAPTOP_MODE_EMAIL_NEW` | Estado de correio novo: confirmar alcance, leitura e retorno (`EMail.cc`). |
| F018 | `LAPTOP_MODE_EMAIL_VIEW` | Mensagem aberta, anexos/scroll e voltar (`EMail.cc`). |
| F019 | `LAPTOP_MODE_WWW` | Entrada web, favoritos e retorno (`Laptop.cc`). |
| F020 | `LAPTOP_MODE_AIM` | Página inicial A.I.M., links e navegação (`AIM.cc`). |
| F021 | `LAPTOP_MODE_AIM_MEMBERS` | Perfil, duração de contrato e confirmação tátil (`AIMMembers.cc`). |
| F022 | `LAPTOP_MODE_AIM_MEMBERS_FACIAL_INDEX` | Retratos/índice e seleção precisa (`AIMFacialIndex.cc`). |
| F023 | `LAPTOP_MODE_AIM_MEMBERS_SORTED_FILES` | Ordenação e lista, sem alterar critério de ordenação (`AIMSort.cc`). |
| F024 | `LAPTOP_MODE_AIM_MEMBERS_SORTED_FILES_VIDEO` | Estado de vídeo: leitura/saída ou prova de não alcance (`Laptop.cc`, `AIMSort.cc`). |
| F025 | `LAPTOP_MODE_AIM_MEMBERS_ARCHIVES` | Arquivo, página anterior/seguinte e retorno (`AIMArchives.cc`). |
| F026 | `LAPTOP_MODE_AIM_POLICIES` | Texto, scroll e links (`AIMPolicies.cc`). |
| F027 | `LAPTOP_MODE_AIM_HISTORY` | História A.I.M., scroll e retorno (`AIMHistory.cc`). |
| F028 | `LAPTOP_MODE_AIM_LINKS` | Links e voltar sem toque na ligação adjacente (`AIMLinks.cc`). |
| F029 | `LAPTOP_MODE_MERC` | Entrada M.E.R.C. e navegação (`Mercs.cc`). |
| F030 | `LAPTOP_MODE_MERC_ACCOUNT` | Conta, botões e confirmação (`Mercs_Account.cc`). |
| F031 | `LAPTOP_MODE_MERC_NO_ACCOUNT` | Criação/aviso de conta e retorno (`Mercs_No_Account.cc`). |
| F032 | `LAPTOP_MODE_MERC_FILES` | Perfis e contratação por toque (`Mercs_Files.cc`). |
| F033 | `LAPTOP_MODE_BOBBY_R` | Entrada da loja, categorias e retorno (`BobbyR.cc`). |
| F034 | `LAPTOP_MODE_BOBBY_R_GUNS` | Lista/quantidade; testar sem compra acidental (`BobbyRGuns.cc`). |
| F035 | `LAPTOP_MODE_BOBBY_R_AMMO` | Lista, filtros e quantidade (`BobbyRAmmo.cc`). |
| F036 | `LAPTOP_MODE_BOBBY_R_ARMOR` | Lista e seleção exata (`BobbyRArmour.cc`). |
| F037 | `LAPTOP_MODE_BOBBY_R_MISC` | Lista e seleção exata (`BobbyRMisc.cc`). |
| F038 | `LAPTOP_MODE_BOBBY_R_USED` | Lista/estado do item e seleção (`BobbyRUsed.cc`). |
| F039 | `LAPTOP_MODE_BOBBY_R_MAILORDER` | Cesto, morada, confirmar/cancelar sem cobrar no cancelamento (`BobbyRMailOrder.cc`). |
| F040 | `LAPTOP_MODE_CHAR_PROFILE` | Entrada IMP e navegação (`CharProfile.cc`). |
| F041 | `LAPTOP_MODE_CHAR_PROFILE_QUESTIONAIRE` | Estado do questionário: provar alcance e toque ou classificar (`CharProfile.cc`, `Laptop.cc`). |
| F042 | `LAPTOP_MODE_FLORIST` | Entrada e links (`Florist.cc`). |
| F043 | `LAPTOP_MODE_FLORIST_FLOWER_GALLERY` | Galeria e escolha (`Florist_Gallery.cc`). |
| F044 | `LAPTOP_MODE_FLORIST_ORDERFORM` | Campos/teclado virtual, confirmar/cancelar encomenda (`Florist_Order_Form.cc`). |
| F045 | `LAPTOP_MODE_FLORIST_CARD_GALLERY` | Cartões e seleção (`Florist_Cards.cc`). |
| F046 | `LAPTOP_MODE_INSURANCE` | Entrada, opções e retorno (`Insurance.cc`). |
| F047 | `LAPTOP_MODE_INSURANCE_INFO` | Informação e scroll (`Insurance_Info.cc`). |
| F048 | `LAPTOP_MODE_INSURANCE_CONTRACT` | Campos, termos e confirmar/cancelar (`Insurance_Contract.cc`). |
| F049 | `LAPTOP_MODE_INSURANCE_COMMENTS` | Comentários, scroll e retorno (`Insurance_Comments.cc`). |
| F050 | `LAPTOP_MODE_FUNERAL` | Página/links e regresso (`Funeral.cc`). |
| F051 | `LAPTOP_MODE_BROKEN_LINK` | Erro de ligação com saída visível (`BrokenLink.cc`). |
| F052 | `LAPTOP_MODE_BOBBYR_SHIPMENTS` | Estado de encomendas, páginas e retorno (`BobbyRShipments.cc`). |
| F053 | IMP — entrada `IMP_Begin_Screen` | Iniciar/criar perfil; auditar como subecrã próprio (`IMP_Begin_Screen.cc`). |
| F054 | IMP — questionário `IMP_Personality_Quiz` | Respostas, avançar/voltar e seleção legível (`IMP_Personality_Quiz.cc`). |
| F055 | IMP — atributos `IMP_Attribute_Selection` | Distribuir pontos com controlo fino e confirmação; regras iguais (`IMP_Attribute_Selection.cc`). |
| F056 | IMP — retrato `IMP_Portraits` | Escolher retrato e voltar (`IMP_Portraits.cc`). |
| F057 | IMP — voz `IMP_Voices` | Pré-escutar/escolher e voltar (`IMP_Voices.cc`). |
| F058 | IMP — confirmação `IMP_Confirm` | Rever personagem, confirmar ou cancelar sem gerar perfil no cancelamento (`IMP_Confirm.cc`). |

Além das seis fases IMP representativas, **cada estado visual distinto** dos restantes `IMP_*.cc` (`IMP_MainPage`, `IMP_HomePage`, `IMP_AboutUs`, `IMP_Personality_Entrance`, `IMP_Personality_Finish`, `IMP_Attribute_Entrance`, `IMP_Attribute_Finish`, `IMP_SkillTraits`, `IMP_Compile_Character`, `IMP_Finish`) recebe uma fase própria `F059`–`F068` pela ordem da tabela seguinte. Se um ficheiro só prepara dados e não apresenta ecrã, fechar a fase com prova de ausência de UI.

| Fase | Estado IMP único | Verificação particular |
| --- | --- | --- |
| F059 | `IMP_MainPage` | Navegação e entrada no fluxo. |
| F060 | `IMP_HomePage` | Links e retorno. |
| F061 | `IMP_AboutUs` | Texto/scroll e retorno. |
| F062 | `IMP_Personality_Entrance` | Iniciar questionário sem toque errado. |
| F063 | `IMP_Personality_Finish` | Rever resultado e avançar/voltar. |
| F064 | `IMP_Attribute_Entrance` | Iniciar atributos e voltar. |
| F065 | `IMP_Attribute_Finish` | Rever atributos sem alterar pontos. |
| F066 | `IMP_SkillTraits` | Selecionar traços, ler estado e cancelar. |
| F067 | `IMP_Compile_Character` | Espera/resultado e retorno, sem mudar geração. |
| F068 | `IMP_Finish` | Confirmação final e saída para computador. |

### Jogo tático, combate e respetivas janelas

| Fase | Ecrã/estado único | Trabalho particular e verificação de saída |
| --- | --- | --- |
| F069 | **Ecrã tático em tempo real** `GAME_SCREEN` | Seleção, movimento, pan, interação e saída; zoom só no mundo (`Real_Time_Input.cc`, `Handle_UI.cc`). |
| F070 | **Ecrã de combate por turnos** `GAME_SCREEN` | Alvo, mira, modo de fogo, postura, AP e fim de turno; ação só após confirmação (`Turn_Based_Input.cc`, `Touch_UI.cc`). |
| F071 | Painel de equipa tático | Retratos, seleção e comandos (`Interface_Panels.cc`). |
| F072 | Radar tático | Visibilidade, toque para navegar e coordenadas (`Radar_Screen.cc`). |
| F073 | Inventário tático | Equipar, trocar, combinar, soltar e cancelar (`Interface_Items.cc`). |
| F074 | Seleção de item | Subpainel de itens, quantidades e alternativa a arrastar (`Inventory_Choosing.cc`). |
| F075 | Diálogo tático | Falas, opções, avançar e sair (`Interface_Dialogue.cc`). |
| F076 | Interação com portas/objetos | Menu contextual e ação exata (`Handle_Doors.cc`, `Handle_Items.cc`). |
| F077 | Mapa tático sobreposto | Inspeção e regresso sem ação no mundo (`Overhead_Map.cc`). |
| F078 | Posicionamento inicial no setor | Escolher posição, confirmar e cancelar (`Tactical_Placement_GUI.cc`). |
| F079 | Comerciante `SHOPKEEPER_SCREEN` | Ofertas, reparar, comprar/vender, confirmar/cancelar (`ShopKeeper_Interface.cc`). |
| F080 | Pré-batalha | Opções de entrada/retirada, cancelar (`PreBattle_Interface.cc`). |
| F081 | Autoresolve `AUTORESOLVE_SCREEN` | Iniciar/parar, ler resultado e continuar (`Auto_Resolve.cc`). |

### Mapa estratégico e cada painel

| Fase | Ecrã/estado único | Trabalho particular e verificação de saída |
| --- | --- | --- |
| F082 | Mapa estratégico `MAP_SCREEN` | Setor, nível, rota e zoom visual opcional sem alterar destino (`MapScreen.cc`, `Map_Screen_Interface_Map.cc`). |
| F083 | Painel de equipa do mapa | Seleção de mercenário/esquadrão, estado e voltar (`Map_Screen_Interface.cc`). |
| F084 | Menu de movimento/viagem | Rota, veículo, confirmação/cancelamento (`Map_Screen_Interface.cc`). |
| F085 | Menu de atribuições | Trabalho/descanso/treino, escolha exata (`Assignments.cc`). |
| F086 | Helicóptero | Selecionar destino e cancelar (`Map_Screen_Helicopter.cc`). |
| F087 | Inventário estratégico | Item entre mercenários, confirmar/cancelar (`Map_Screen_Interface_Map_Inventory.cc`). |
| F088 | Pool de itens do setor | Páginas, transferência, sair sem perda (`Map_Screen_Interface_Map_Inventory.cc`). |
| F089 | Painel de mina/cidade | Texto, navegação, botão pequeno (`Map_Screen_Interface_TownMine_Info.cc`). |
| F090 | Gestão de milícia | Escolhas, confirmação, sem mudar regras (`Map_Screen_Interface_Map.cc`). |
| F091 | Contrato de mercenário | Renovar/cancelar e conferir valores (`Merc_Contract.cc`). |
| F092 | Painel inferior/tempo | Avanço, pausa, mensagens e retorno (`Map_Screen_Interface_Bottom.cc`). |

### Restantes ecrãs, modais e IDs do motor

| Fase | Ecrã/estado único | Trabalho particular e verificação de saída |
| --- | --- | --- |
| F093 | Opções `OPTIONS_SCREEN` | Alternâncias, sliders, texto, voltar (`Options_Screen.cc`). |
| F094 | Gravar/carregar `SAVE_LOAD_SCREEN` | Slot, nome, sobrescrever, load e cancelar (`SaveLoadScreen.cc`). |
| F095 | Estado `LOADSAVE_SCREEN` | Auditar handler/alcance próprio e saída (`Screens.cc`, `SaveLoadScreen.cc`). |
| F096 | Ajuda | Scroll, abas e fecho visível (`HelpScreen.cc`). |
| F097 | Mensagem `MSG_BOX_SCREEN` | Ler, escolher e cancelar sem teclado (`MessageBoxScreen.cc`). |
| F098 | Popup genérico | Escolha/fecho, teste com texto longo (`Utils/PopUpBox.cc`). |
| F099 | Erro `ERROR_SCREEN` | Substituir instrução exclusiva de ESC por saída tátil visível (`JAScreens.cc`). |
| F100 | Transição `FADE_SCREEN` | Verificar que não retém input nem dispara toque em ecrã seguinte (`Fade_Screen.cc`). |
| F101 | Crédito `CREDIT_SCREEN` | Scroll/saída por toque (`Screens.cc`). |
| F102 | Animação `SEX_SCREEN` | Confirmar transição/saída; nenhuma mecânica do evento alterada (`JAScreens.cc`). |
| F103 | `DEMO_EXIT_SCREEN` | Auditar alcance/handler e classificar se legado (`Screens.cc`). |
| F104 | `MAPUTILITY_SCREEN` | Auditar alcance/handler e classificar (`Screens.cc`). |
| F105 | Editor `EDIT_SCREEN` | Inspecionar comandos e declarar cobertura própria; não confundir com campanha (`Editor/EditScreen.cc`). |
| F106 | Paleta `PALEDIT_SCREEN` | Auditar acesso/saída tátil ou classificar como ferramenta de desenvolvimento (`JAScreens.cc`). |
| F107 | Debug `DEBUG_SCREEN` | Auditar acesso/saída tátil ou classificar como ferramenta de desenvolvimento (`JAScreens.cc`). |
| F108 | Debug de quests `QUEST_DEBUG_SCREEN` | Auditar acesso/saída tátil ou classificar como ferramenta de desenvolvimento (`Quest_Debug_System.cc`). |
| F109 | IMP — sistema de texto | Auditar se `IMP_Text_System.cc` apresenta estado visual próprio; se sim, rever leitura/avanço/retorno; se não, fechar com prova de não ser ecrã. |

`MAX_SCREENS` e `NO_PENDING_SCREEN` são contadores/sentinela, não ecrãs; `ANIEDIT_SCREEN` consta apenas de um comentário como removido em `ScreenIDs.h`. `GAME_SCREEN` não é duplicado por engano: F069 e F070 separam explicitamente os estados de tempo real e combate por turnos. `LAPTOP_MODE_END` é limite do enum, não uma página. Qualquer modal visual encontrado na fase 0 que não conste acima ganha **nova linha e fase própria** antes de a matriz ser considerada completa.

### Catálogo — launcher, arranque, menu e gestão do jogo (`SC-10.4a`, `SC-10.5a`)

| Ecrã | Inventário e proposta de incremento | Verificação |
| --- | --- | --- |
| Launcher Android — Data | Escolher diretório original, versão/localização dos dados e diretório de saves; diagnosticar caminho errado/permissão e permitir regressar do seletor. `DataTabFragment.kt`, `GameDir.kt`, layouts Android. | Iniciar com cópia válida; testar pasta errada, permissão negada e retorno sem perder configuração; sem teclado físico. |
| Launcher — Settings | Medir largura dos campos de resolução e controlos de escala, teclado numérico, orientação e botão de resolução recomendada. `SettingsFragment.kt`, `ConfigurationModel.kt`, layout. | Configuração legível e aplicável no telemóvel pequeno/tablet; regressar ao jogo com escala coerente. |
| Launcher — Logs/arranque | Abrir/partilhar logs e recuperar de erro de arranque; identificar se UI Android mantém botões visíveis com sistema de navegação e teclado virtual. `LauncherActivity.kt`, `LogsTabFragment.kt`. | Mensagem de erro acionável, voltar ao launcher e relançar. |
| Introdução, créditos e menu principal | Testar saltar/continuar, `New Game`, `Load`, `Options`, `Exit`, botões e confirmação. `MainMenuScreen.cc`, `JAScreens.cc`, `Screens.cc`. | Todos os caminhos executáveis por toque, inclusive regressar do jogo ao menu; foco/letterbox alinhados. |
| Opções da campanha inicial | Selecionar dificuldade e opções sem tocar num valor adjacente. `GameInitOptionsScreen.cc`. | Iniciar campanha com escolhas desejadas e ler o estado antes de confirmar. |
| Opções, sliders e texto | Medir alternâncias, sliders, listas, scrolling, campos e cancelamento. `Options_Screen.cc`, `Utils/Slider.cc`, `Utils/Text_Input.cc`. | Alterar e repor cada opção visível; manter seleção após retoma; desktop intacto. |
| Gravar/carregar | Selecionar slot, escrever nome via teclado virtual, sobrescrever/cancelar, navegar páginas, load/quick save quando exposto. `SaveLoadScreen.cc`, `SaveLoadGame.cc`. | Save antigo e novo abrem; cancelar não substitui save; nenhuma alteração de formato. Integração com SC-01 sem misturar correção de serialização. |

### Catálogo — ecrã tático e sobreposições de combate (`SC-10.4b`, `SC-10.3a`)

| Sub-ecrã/estado | Incrementos de UI a investigar e implementar se a linha de base falhar | Verificação |
| --- | --- | --- |
| Vista tática, tempo real | Selecionar soldado/alvo, caminhar/correr, cancelar, pan/scroll, portas, interação com objetos, saída do setor; distinguir toque de confirmação de navegação. `Real_Time_Input.cc`, `Handle_UI.cc`, `Touch_UI.cc`, `TileEngine/Overhead_Map.cc`. | Seleção, deslocamento, interação e saída por toque; nenhum movimento ao terminar pan/zoom. |
| Vista tática, turnos | Mirar, ciclo de modo de fogo, postura, subir/descer nível, mudar arma, usar item, passar/terminar turno e cancelar; verificar botões contextuais de 32 unidades lógicas e estados de AP. `Turn_Based_Input.cc`, `Touch_UI.cc`, `Interface_Panels.cc`. | Série de ações com alvo/custo igual ao desktop; toque que escolhe alvo não dispara sozinho. |
| Painel de equipa e radar | Retratos, seleção, mensagens, relógio, botões, radar e alternância de painel; separar controlos do zoom do mundo. `Interface.cc`, `Interface_Panels.cc`, `Radar_Screen.cc`, `UILayout.cc`. | Cada controlo dentro da área visível e com alvo físico mensurável; informação essencial legível. |
| Inventário tático | Escolher, mover, trocar, combinar, anexar, dividir pilhas, equipar, soltar no chão e cancelar item em mão; oferecer modo de seleção em dois passos/inspector quando o arrasto não for fiável. `Interface_Items.cc`, `Inventory_Choosing.cc`, `Handle_Items.cc`. | Sem perda/duplicação; item e destino corretos; mesmo resultado e AP que a ação original. |
| Interação contextual | Conversa, diálogo, NPC, menus de ação, portas/fechaduras, objetos e pistas de cursor; tornar alternativas ao botão direito/toque longo visíveis. `Interface_Dialogue.cc`, `Dialogue_Control.cc`, `Handle_Doors.cc`, `Utils/PopUpBox.cc`. | Entrar, escolher e cancelar cada diálogo sem teclado, com estado preservado. |
| Entrada em setor e combate especial | Posicionamento inicial, mapa sobreposto, pré-batalha e autoresolve; inspecionar controlos temporizados e confirmação. `Tactical_Placement_GUI.cc`, `Overhead_Map.cc`, `PreBattle_Interface.cc`, `Auto_Resolve.cc`. | Posicionar/confirmar/cancelar e voltar à campanha por toque. |
| Comerciante no setor | Inventários do jogador e lojista, ofertas, avaliação, reparar, pagar, comprar e cancelar, sem alterar preços ou stock. `ShopKeeper_Interface.cc`. | Resultado monetário e de inventário igual ao desktop para a mesma transação; nenhum pagamento durante pan/zoom. |

### Catálogo — mapa estratégico e painéis (`SC-10.4c`, `SC-10.3b`)

| Sub-ecrã/estado | Incrementos de UI a investigar e implementar se necessário | Verificação |
| --- | --- | --- |
| Mapa e navegação | Escolher setor, ler informação, alternar níveis, deslocar mapa e abrir vista tática; estudar zoom **só do mapa** depois do tático, com botões visíveis, se alvos continuarem pequenos. `MapScreen.cc`, `Map_Screen_Interface_Map.cc`, `Map_Screen_Interface_Border.cc`. | Setor tocado corresponde ao destacado; zoom não muda sector/rota nem aciona comando acidental. |
| Equipa, viagem e tempo | Seleção de mercenários, esquadrões, rota, veículo/helicóptero, atribuições, descanso, avanço de tempo, interromper e confirmar. `Map_Screen_Interface.cc`, `Map_Screen_Helicopter.cc`, `Assignments.cc`, `Map_Screen_Interface_Bottom.cc`. | Concluir deslocação e gestão de grupo por toque; rotas/tempo/custos iguais ao desktop. |
| Inventário estratégico e pool de setor | Selecionar mercenário, mover/trocar itens, navegar páginas, retirar/devolver item, fechar; substituto visível para ação secundária de rato. `Map_Screen_Interface_Map_Inventory.cc`, `UILayout.cc`. | Mesmo item transferido, nenhuma perda/duplicação; cancelar mantém o estado. |
| Painéis de minas, cidades, milícia e contratos | Controlos, listas, pequenos ícones e popups; não alterar regras de treino, salários, lealdade ou contratos. `Map_Screen_Interface_Map.cc`, `Map_Screen_Interface_TownMine_Info.cc`, `Merc_Contract.cc`. | Entrar/voltar, ler e executar escolhas pela UI existente; resultado estratégico igual ao desktop. |

### Catálogo — computador, formulários e economia (`SC-10.4d`)

| Família de páginas | Incrementos de UI a investigar e implementar se necessário | Verificação |
| --- | --- | --- |
| Moldura, navegação, correio e ficheiros | Abas/ícones, favoritos, back, scrolling, correio, anexos, histórico e ficheiros; medir textos e alvos. `Laptop.cc`, `EMail.cc`, `Files.cc`, `History.cc`. | Navegar, ler e regressar por toque sem perder posição; nenhum link errado. |
| A.I.M. e M.E.R.C. | Listas, perfis, contratos, seleção de duração, confirmação/cancelamento e vídeos. `AIM*.cc`, `Mercs*.cc`. | Contratar e rever condições só por toque; valores e contratação iguais ao desktop. |
| IMP | Nome, género, respostas, atributos, retrato, voz, revisão e confirmação; teclado virtual e sliders precisam de área suficiente. `IMP_*.cc`, `Utils/Text_Input.cc`, `Utils/Slider.cc`. | Criar personagem completa sem teclado físico; pontos e regras de criação iguais aos originais. |
| Bobby Ray e expedições | Categorias, quantidade (inclui comportamento tátil próprio em `BobbyRGuns.cc`), cesto, morada, confirmar/cancelar encomenda e consultar entrega. `BobbyR*.cc`, `Store_Inventory.cc`. | Compra exata e cancelamento sem cobrança; quantidade igual à selecionada. |
| Seguros, florista e outras páginas | Contratos, formulários, galerias, pedidos, links partidos/funeral, finanças e pessoal. `Insurance*.cc`, `Florist*.cc`, `Funeral.cc`, `Finances.cc`, `Personnel.cc`. | Cada página alcançável, legível e regressável; transações com confirmação clara. A prioridade temporal depende da decisão de âmbito, mas entram na cobertura final. |

### Catálogo — ajuda, avisos, acessibilidade geral e ciclo de vida (`SC-10.4e`, `SC-10.5`, `SC-10.7`)

| Ecrã/estado | Incremento | Verificação |
| --- | --- | --- |
| Ajuda, caixas de mensagem e popups | `HelpScreen.cc`, `MessageBoxScreen.cc`, `Utils/PopUpBox.cc`: rolar, escolher, cancelar; saída visível para erros que hoje só dizem `ESC` (`JAScreens.cc`). | Nenhum modal prende a campanha sem teclado; botões não ficam fora do viewport. |
| Feedback e descoberta | Mostrar ação selecionada, alvo e custo/informação já conhecida; explicar gestos e botões no próprio jogo, com alternativa visível. Rever contraste, tamanho de letra e idioma sem alterar localização dos dados originais. | Novato encontra comandos essenciais sem memorizar gestos; layout não tapa estado crítico. |
| Ciclo de vida Android | Pausa/retoma, bloqueio, rotação suportada, teclado virtual, insets e retorno do launcher. `StracciatellaActivity.kt`, `SDLActivity.java`, `SGP.cc`, manifesto. | Sem clique residual ou estado de arrasto pendente após retoma; save anterior continua íntegro. |
| Ecrãs auxiliares | Introdução, créditos, erro, saída e, numa etapa própria, editor/debug/paleta do motor. `Screens.cc`, `JAScreens.cc`, `Editor/`. | Percurso de campanha não bloqueado. Editor/debug ficam assinalados fora do marco «campanha Android», mas nunca se afirma equivalência completa do port até serem auditados. |

### Validação transversal após as fases de ecrã (`SC-10.6`)

1. Executar campanha de referência sem periféricos: configurar dados, novo jogo, IMP/contratação, viajar, entrar em setor, combate em tempo real e turnos, inventário, loja, laptop, mapa, diálogos, guardar, suspender, retomar e carregar. Cobrir as páginas opcionais da fase E antes de declarar cobertura de toda a UI.
2. Repetir no telemóvel físico e nos perfis de emulador definidos na fase 0, nas orientações suportadas. Repetir um percurso comparável com rato/teclado em desktop e usar o mesmo save para confirmar compatibilidade. Verificar foco, Android Back, permissões e interrupção durante gesto. Não afirmar ergonomia de tablet físico sem o testar.
3. Para cada ecrã, publicar matriz final de comandos, capturas antes/depois, dimensões físicas dos alvos, erros observados, frames/performance do zoom e casos não testados. O critério é **zero comandos essenciais sem caminho tátil visível**, **zero ação de jogo desencadeada pelo fim de pan/zoom**, **zero discrepância de alvo por transformação**, e **nenhuma mudança de regra ou save**.
4. Executar build e verificações relevantes: `RUSTUP_TOOLCHAIN=1.85.1 ./gradlew :app:assembleDebug` em `android/`; CMake/C++ e `cargo-test` quando houver alteração partilhada; `ja2 -unittests` para testes C++. Uma build verde não substitui ensaio físico. Reportar sempre comando, resultado e cenários manuais.

## Registo de cada incremento

```text
ID SC-10.x / ecrã / commit:
Hipótese e reprodução anterior à mudança:
Dispositivo, Android/API, ABI, mm/dp/DPI, orientação, resolução, escala:
Dados JA2/localização, mods, save de referência:
Comando original / rota por toque / alternativa visível:
Antes: passos, esperado, observado, captura/vídeo:
Depois: passos, esperado, observado, alvo físico, erros:
Comparação desktop e impacto nas regras/saves:
Build/testes executados e resultado:
Riscos e cenários não verificados:
```

## Decisões e dados por recolher

1. **Decidido pelo utilizador:** uma fase por ecrã, incluindo Main Title, computador, cada página do computador, tática, combate e todos os restantes ecrãs. O alvo físico inicial é Pixel 8 (~6") na horizontal, complementado por emuladores. A UI deve adaptar-se automaticamente a outros tamanhos.
2. **[TODO]** Apurar versão Android, dimensões/densidade reais e insets do Pixel 8; preencher medidas e bugs reproduzidos em dispositivo na fase 0. Só então escolher dimensões de alvos, duração de gesto, limites de zoom e layout final. A dimensão mínima suportada requer ensaio nos perfis de emulador, não um valor inventado.

## Fontes locais principais

`docs/SOUR_CREAM_ROADMAP.md`; `COMPILATION.md`; `CONTRIBUTING.md`; `docs/ANDROID_BUILD_AND_DEVICE_TEST.md`; `src/game/ScreenIDs.h`; `src/game/Screens.cc`; `src/sgp/{SGP,Input,MouseSystem,Video}.cc`; `src/game/UILayout.cc`; `src/game/Tactical/{Real_Time_Input,Turn_Based_Input,Touch_UI,Interface_Items,ShopKeeper_Interface}.cc`; `src/game/Strategic/{MapScreen,Map_Screen_Interface,Map_Screen_Interface_Map_Inventory}.cc`; `src/game/Laptop/Laptop.h`; `android/app/src/main/`.
