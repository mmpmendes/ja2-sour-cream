# Sour Cream — plano de melhoria

> Estado: proposta inicial, 17-10-2026. Nenhuma tarefa abaixo está marcada como concluída. Ordem por impacto **estimado** para jogadores, não por facilidade de implementação. Reavaliar após reproduções, testes de jogo e feedback.

## Objetivo e regras de execução

O primeiro marco é uma campanha que se consegue **iniciar, continuar e progredir sem bloqueios conhecidos**. **Dispositivos móveis são um foco principal do fork:** todas as ações essenciais devem ser utilizáveis por toque, com zoom e controlos legíveis, mantendo as regras e a funcionalidade original com rato e teclado. Depois vêm clareza tática, ritmo do jogo, áudio, mods e identidade do fork.

- Trabalhar por incrementos `SC-xx.y` que possam ser revistos isoladamente. Uma investigação pode terminar sem mudança de código se a hipótese não se confirmar.
- Para cada bug: guardar passos de reprodução, plataforma, versão/commit, dados originais usados, mods ativos, save de exemplo quando redistribuível, resultado esperado e resultado observado. Não incluir dados proprietários do JA2 no repositório.
- Para cada alteração de regras: comparar com o comportamento atual, documentar o efeito no equilíbrio e, quando fizer sentido, oferecer configuração em vez de alterar todas as campanhas.
- Manter compatibilidade com saves existentes salvo decisão explícita e documentada de migração. Testar pelo menos um save antigo e um novo quando se mexe em serialização, recursos, mods ou regras persistidas.
- Tratar toque como forma de jogar completa, incluindo mapa estratégico, combate, inventário, laptop, opções e saves. Não resolver falta de comandos com um gesto impossível de descobrir ou executar; oferecer controlos visíveis e alternativas para gestos com vários dedos ou pressão prolongada.
- Separar zoom do mundo, escala da interface e resolução interna. Qualquer transformação visual deve manter a correspondência exata entre o ponto tocado e o alvo do jogo, sem alterar simulação, alcance, percurso ou cálculo de combate.
- Reutilizar as verificações existentes: `cmake --build <build-dir> --target cargo-test` e `<build-dir>/ja2 -unittests` (ver caminhos e opções em [COMPILATION.md](../COMPILATION.md) e [.ci/ci-build.sh](../.ci/ci-build.sh)). Não considerar um build ou teste manual executado sem o executar.

### Estados e prioridade

Estados permitidos neste documento: `por iniciar`, `em investigação`, `em implementação`, `em validação`, `concluído`, `descartado`. Cada mudança de estado deve trazer uma ligação para issue/PR ou uma nota de evidência. `P0` bloqueia uma campanha ou arrisca progresso; `P1` afeta repetidamente a experiência; `P2` melhora extensibilidade ou apresentação. O esforço (`S/M/L`) é relativo e precisa de refinamento após investigação.

| Ordem | ID | Prioridade | Esforço | Estado | Dependências |
| --- | --- | --- | --- | --- | --- |
| 1 | SC-01 Saves e recuperação | P0 | L | em investigação | — |
| 2 | SC-02 Primeiro arranque e dados | P0 | M | por iniciar | — |
| 3 | SC-03 Bloqueios de campanha | P0 | M | por iniciar | SC-01.1 para guardar reproduções |
| 4 | SC-10 Experiência móvel, toque e zoom | P1 | L | em investigação | SC-04.1 para medir legibilidade; SC-05.1 para ações táticas |
| 5 | SC-04 Escala e ecrãs largos | P1 | L | por iniciar | — |
| 6 | SC-05 Clareza das ações táticas | P1 | M/L | por iniciar | — |
| 7 | SC-06 IA e ritmo dos turnos | P1 | L | por iniciar | SC-05.1 para distinguir regras de apresentação |
| 8 | SC-07 Áudio multiplataforma | P1 | M | por iniciar | — |
| 9 | SC-08 Diagnóstico de mods | P2 | M | por iniciar | SC-02.1 para mensagens de arranque coerentes |
| 10 | SC-09 Identidade Sour Cream | P2 | S/M | por iniciar | decisão sobre nomes públicos e compatibilidade de caminhos |

## SC-01 — Saves e recuperação

**Investigação iniciada em 19-10-2026 (SC-01.1):** [matriz e evidência](SC-01-SAVE-MATRIX.md). Os testes de cabeçalhos/estados de save passaram (20 testes). O percurso Rust de `EXDEV` copiava diretamente sobre o destino; uma cópia interrompida truncou o ficheiro anterior num ensaio isolado. SC-01.2 passa a usar um provisório no diretório de destino, sujeito a validação no jogo. Ainda não houve gravação/carga de campanha por falta dos dados originais.

**Resultado esperado:** uma falha ao gravar não destrói o último save válido; um save aceite pelo jogo volta a carregar com o estado esperado. Começar em `src/game/SaveLoadGame.cc`, `src/game/SaveLoadScreen.cc`, `src/game/SaveLoadGameStates.cc` e `src/externalized/DefaultContentManager.cc`.

**Facto observado:** `SaveGame()` já escreve num ficheiro temporário e chama `FileMan::moveFile()` no fim. Existem `TODO000F` perto de tamanhos de dados guardados/carregados. Estes factos justificam auditoria; não demonstram, por si, corrupção atual.

1. **SC-01.1 — Matriz de saves.** Definir cenários pequenos e reproduzíveis: campanha nova; combate; mudança de setor; inventário; contratação; mods ativos; quicksave/autosave; save criado numa versão anterior. Incluir gravação/carga após suspender e retomar a aplicação móvel. Registar hash e estado observável antes/depois da carga. Guardar apenas fixtures que possam ser distribuídas legalmente.
2. **SC-01.2 — Auditoria de escrita.** Confirmar se o temporário e o destino estão no mesmo sistema de ficheiros, o que acontece se `moveFile` falhar ou o processo terminar, e se o save anterior continua íntegro. Só acrescentar cópia de segurança, substituição segura ou recuperação se um cenário demonstrar necessidade. Cobrir falta de espaço e interrupção quando possível.
3. **SC-01.3 — Auditoria de leitura.** Verificar limites, versões e campos associados aos `TODO000F`. Se houver defeito, criar um teste que falhe antes da correção e preservar leitura de saves antigos.

**Aceitação:** todos os cenários da matriz passam; falhas simuladas deixam um save anterior legível; erros mostram mensagem útil e não afirmam sucesso; regressões de formato são documentadas. **Risco:** alterar serialização pode invalidar campanhas; evitar mudanças amplas sem fixtures.

## SC-02 — Primeiro arranque e dados

**Resultado esperado:** o jogador chega ao menu com uma instalação válida ou recebe instruções precisas para corrigir a configuração. Começar em `src/launcher`, `rust/stracciatella/src/config`, `rust/stracciatella/src/vfs` e `src/externalized/DefaultContentManager.cc`.

**Evidência externa a reproduzir neste fork:** [ja2-stracciatella#2384](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2384) descreve um erro de localização de `externalized/` numa instalação. Confirmar o comportamento no commit atual antes de corrigir.

1. **SC-02.1 — Matriz de instalações.** Executar a partir do diretório de build, de uma instalação com prefixo personalizado, de um pacote e do launcher Android; testar dados originais ausentes, caminho errado, versões/localizações suportadas e `externalized/` ausente. Documentar qual localização é procurada em cada caso.
2. **SC-02.2 — Validação e mensagens.** Distinguir caminho inexistente, dados originais inválidos, recursos do fork ausentes e falta de permissão. Mostrar o caminho procurado e uma ação concreta no launcher ou no erro do executável.
3. **SC-02.3 — Fluxo guiado.** Se a matriz mostrar fricção real, facilitar a seleção da pasta e a confirmação da versão dos dados no launcher, sem exigir edição manual de `ja2.json`.

**Aceitação:** instalações válidas arrancam nos cenários suportados; cada cenário inválido falha com diagnóstico acionável; os testes usam diretórios temporários e não requerem dados proprietários quando a verificação é apenas de caminhos. **Risco:** não assumir que todos os pacotes usam a mesma estrutura de diretórios.

## SC-03 — Bloqueios de campanha

**Resultado esperado:** falhas que impedem progressão têm reprodução e correção isoladas. Começar em `src/game/Strategic`, `src/game/Tactical` e `src/game/SaveLoadGame.cc`, consoante o caso.

1. **SC-03.1 — Triage.** Reproduzir em build atual os relatos que impedem progresso. Primeiro candidato: [ja2-stracciatella#2554](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2554), sobre controlo de setor após desembarque de helicóptero. Verificar `Strategic/Map_Screen_Helicopter.cc`, `Strategic/Assignments.cc` e `Strategic/Player_Command.cc`; não presumir a causa.
2. **SC-03.2 — Correção mínima por caso.** Corrigir uma causa por alteração. Incluir teste de estado estratégico quando a lógica for isolável ou save/cenário manual reproduzível quando depender do motor completo.
3. **SC-03.3 — Registo de regressões.** Manter uma pequena lista de campanhas de teste com etapa, resultado esperado e build em que passou. Acrescentar casos só quando houver defeito confirmado.

**Aceitação:** o caso falha antes e passa depois da correção; save anterior continua a carregar; não há alteração inadvertida no controlo de setores por outros meios. **Risco:** uma correção local pode afetar regras estratégicas noutras entradas no setor.

## SC-10 — Experiência móvel, toque e zoom

**Resultado esperado:** é possível completar uma campanha num telemóvel ou tablet Android sem rato/teclado, com todos os comandos originais acessíveis. O jogador consegue aproximar o mundo para selecionar alvos e ler detalhes, voltar rapidamente à vista normal e usar a interface sem toques acidentais. A interface de desktop e as regras do jogo conservam o comportamento original. Android é a primeira plataforma móvel porque já existe `android/app`; a viabilidade de iOS será avaliada separadamente.

**Base existente:** `src/sgp/Input.cc` encaminha eventos de dedo a partir de SDL e acompanha um dedo principal; `src/sgp/MouseSystem.cc` distribui toques a regiões; `src/game/Tactical/Touch_UI.cc` já oferece botões contextuais táticos de 32 unidades lógicas; `android/app` contém o launcher e a atividade do jogo. O zoom por gesto ainda precisa de investigação e protótipo. Relatos candidatos para reprodução: [troca de posições por toque #2489](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2489), [toque nas preferências #2485](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2485) e [atalhos indisponíveis em Android #1883](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/1883). Não presumir que estes casos se reproduzem no fork.

1. **SC-10.1 — Matriz de equivalência.** Inventariar ações de rato/teclado e o respetivo caminho por toque: selecionar e cancelar; mover, correr, mudar postura e disparar; apontar e alternar modo de fogo; abrir menus de contexto; arrastar, largar e trocar itens; percorrer o mapa estratégico; contratar no laptop; usar opções e saves. Marcar ações ausentes, ambíguas ou que exigem precisão irrealista. Testar com dedo, sem periféricos, num telemóvel e num tablet. Registar orientação, densidade, tamanho físico e resolução lógica.
2. **SC-10.2 — Modelo de gestos.** Definir e prototipar seleção, confirmação, toque prolongado, arrasto do mapa e interação com inventário por contexto. Investigar a identificação de múltiplos dedos em `Input.cc` antes de reservar pinça ou gesto de dois dedos. Gestos não devem disparar movimentos, tiros ou trocas de itens ao começar, terminar ou ser interrompidos; deve existir cancelamento claro. Preservar entrada por rato/teclado e controlador quando já suportada.
3. **SC-10.3 — Zoom do mundo.** Prototipar zoom centrado no ponto tocado para a vista tática; avaliar depois o mapa estratégico. Manter painéis e botões utilizáveis e distinguir esta função da escala global de UI de SC-04. A pinça pode controlar o zoom quando for fiável, mas incluir botões visíveis de aproximar, afastar e repor para utilizadores que não conseguem fazer pinça. Definir limites, deslocamento da vista e comportamento ao mudar de ecrã. Verificar conversão de coordenadas para selecionar soldados, quadrículas, objetos e regiões de UI depois de zoom e pan; nenhum comando pode atingir um alvo diferente do mostrado.
4. **SC-10.4 — Alvos e legibilidade.** Medir a dimensão física e a taxa de erros dos controlos táticos, inventário, mapa, laptop, diálogos e opções. Aumentar área de toque quando possível sem ampliar indevidamente o desenho; para ações densas, oferecer seleção ampliada ou confirmação contextual. Rever contraste, texto, feedback de seleção, margens do sistema e disposição em telemóvel/tablet. Não esconder comandos originais atrás de um único gesto.
5. **SC-10.5 — Preferências acessíveis.** Permitir ajustar sensibilidade de gestos, duração do toque prolongado e escala/zoom onde a implementação o justificar. Guardar preferências sem modificar saves ou regras. Documentar gestos no jogo e fornecer alternativa visível às ações críticas.
6. **SC-10.6 — Validação em dispositivos.** Executar percursos completos em telemóvel e tablet: iniciar campanha, contratar, deslocar esquadrão, combater, gerir inventário, guardar/carregar, suspender/retomar e mudar orientação quando suportado. Medir erros de toque, legibilidade, tempo para concluir ações e desempenho do zoom; comparar com a mesma campanha em desktop. Separar falhas do launcher Android, SDL/input, UI do jogo e renderização.
7. **SC-10.7 — Alcance de plataformas móveis.** Verificar instalação, permissões, armazenamento, ciclo de vida e desempenho nas versões Android que se pretende suportar. Investigar requisitos de build, distribuição e acesso aos dados originais em iOS antes de prometer uma versão; criar uma tarefa de portabilidade própria se a viabilidade e os testes em dispositivo o justificarem. Reutilizar o modelo de input/zoom comum sempre que possível.

**Investigação local em 20-10-2026 (SC-10.7):** compilador e dependências de desktop estão presentes; a configuração debug e a build Linux passaram, assim como 165 testes C++ e os testes Rust. Os dados JA2 estão disponíveis em `JA2_ORIGINAL_GAME_DIR` (`data/`, minúsculas; VFS confirma resolução sem distinguir maiúsculas/minúsculas). O utilizador confirmou que a build local arrancou e correu corretamente com os dados originais. O Android usa Gradle 9.6.0, AGP 9.4.1 e KGP 2.4.20; `:app:assembleDebug` passou para as quatro ABIs com esta combinação e gerou `android/app/build/outputs/apk/debug/app-debug.apk` (77 MB). A matriz oficial do Kotlin ainda não lista AGP 9.4 como totalmente suportado por KGP 2.4.20. O JDK 17, SDK/NDK e alvos Rust necessários estavam disponíveis. Não foi possível iniciar o daemon ADB nesta sessão (`Operation not permitted`), portanto instalação, arranque do launcher e campanha em dispositivo continuam por validar. O guia e a evidência local estão em [ANDROID_BUILD_AND_DEVICE_TEST.md](ANDROID_BUILD_AND_DEVICE_TEST.md) e [LOCAL_DEVELOPMENT.md](LOCAL_DEVELOPMENT.md). O DSL legado fica restrito ao módulo por `android.newDsl.optOut=:app`; o plugin Kotlin atual mantém opt-out temporário e produz aviso de descontinuação.

**Planeamento em 22-10-2026 (SC-10.1):** [plano detalhado por ecrã](SC-10-ANDROID-SMALL-SCREENS-PLAN.md) criado após inventário estático de IDs de ecrã, modos do laptop, input SDL, layout e launcher Android. O utilizador definiu uma fase para cada ecrã/página, Pixel 8 (~6") na horizontal como alvo físico inicial e emuladores para outros tamanhos; a UI deverá escalar automaticamente. Nenhuma lacuna de toque ou zoom foi ainda reproduzida em dispositivo nesta etapa; a matriz física e a validação de campanha permanecem pendentes. O código atual declara `compileSdk 37` em `android/app/build.gradle`, mas o guia Android ainda refere 34; reconciliar antes da próxima build.

**SC-10/F001 concluída em 22-10-2026:** reproduzida sobreposição do botão de escolher pasta de saves com o FAB no Pixel 8 horizontal e feedback transitório após recusa de permissão. A página foi ajustada para reservar margem inferior ao FAB e mostrar diálogo acionável para abrir as definições de permissões. Build debug passou; Pixel 8/API 37 e AVD/API 37 validaram os fluxos e orientações documentados, incluindo menu do jogo horizontal. Sem alterações a regras, saves, VFS ou dados proprietários. [Evidência, medidas e cobertura](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F002 concluída em 22-10-2026:** no Pixel 8/API 37 reproduziu-se Detect fora da linha de resolução em vertical e sobreposição do FAB ao texto com teclado aberto. Campos proporcionais e margem inferior do scroll corrigiram a apresentação; build debug, Pixel em vertical/horizontal, AVD API 37 em 720×1600@320 dpi e perfil tablet virtual 1920×2560@320 dpi nas duas orientações passaram os cenários de Settings. Valores, listeners e formato `.ja2/ja2.json` não foram alterados; overrides do AVD foram repostos. Sem validação de tablet físico ou runtime do jogo nesta fase. SC-10 permanece em investigação; F002 fecha apenas Settings. [Reprodução, evidência e cobertura](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F003 concluída em 22-10-2026:** no Pixel 8/API 37 reproduziu-se o FAB a cobrir as últimas linhas do `ja2.log` em vertical e horizontal. O ScrollView passou a reservar uma faixa inferior fora do viewport rolável; build debug e Pixel após correção confirmaram scroll completo, cópia/Toast, navegação, Voltar, relançamento e rotação. O AVD Medium Phone API 37 cobriu perfil 720×1600@320 dpi e tablet virtual 1920×2560@320 dpi nas duas orientações; sem log/dados no AVD, validou-se apenas o layout vazio. O cenário de retorno após erro de arranque ficou não reproduzido. Só o layout Android Logs foi alterado; Data, Settings, jogo, saves, VFS e configuração ficaram fora do diff. SC-10 continua em investigação; F003 fecha apenas Logs. A preferência é executar primeiro no AVD e usar Pixel apenas se o AVD não cobrir o cenário. [Reprodução, evidência e cobertura](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F004 concluída em 22-10-2026:** reproduziu-se no AVD Medium Phone API 37 a permanência do splash/menu em retrato, apesar de o jogo dever iniciar em paisagem. `SGP.cc` agora pede `LandscapeLeft` no Android antes de inicializar SDL; build debug passou e o AVD confirmou a rotação da versão final para 2560×1920 no perfil tablet virtual. Perfis de telefone 720×1600@320 dpi em ambas as orientações tinham sido verificados na build anterior de F004. A política padrão de tablet do Android API 37 pode ignorar o pedido da app; foi temporariamente desativada para validar e depois restaurada. F004 fecha apenas `INIT_SCREEN`; SC-10 permanece em investigação. [Reprodução, validação e limites](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F005 — avaliação anterior em 22-10-2026:** no AVD Android 17/API 37, `x86_64`, o toque no fundo executou a ação original de fechar o clipe atual e avançar; sem interação, a sequência terminou e prosseguiu a inicialização/cinemática de novo jogo. `Intro.cc` mostra que essa saída retorna `INIT_SCREEN`, cujo caminho inicializa a campanha, não `MAINMENU_SCREEN`. O destino Main Title escrito no plano não corresponde a este fluxo. Não houve alteração de código nem de ações; F005 continua aberta para alinhar o ecrã/destino pretendido. Home/Recentes com mudança forçada de perfil levou à recriação da atividade e ao splash; não se confirmou repetição da INTRO nem recuperação do mesmo frame. Cobertura: AVD em retrato 1080×2400@420 dpi e paisagem efetiva 1600×720@320 dpi; sem Pixel, rato físico ou tablet. [Reprodução, destino real e limites](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F006 — avaliação anterior em 22-10-2026:** Main Title executado no AVD Android 17/API 37, `x86_64`, em retrato 1080×2400@420 dpi, paisagem compacta 1600×720@320 dpi e tablet virtual paisagem 2560×1920@320 dpi. Toque validou Start New Game/Cancel, Preferences/Done e Quit; a confirmação de Quit em Options respondeu corretamente a No e Yes. Continue/Load estava desativado sem saves. `O` abriu Options por teclado; suspensão/retoma voltou ao título sem input duplicado. Uma suspeita inicial de hit testing deslocado no perfil compacto não se reproduziu usando coordenadas medidas na captura. Sem rato físico, save existente, rotação dinâmica ou validação física; nenhum defeito F006 foi reproduzido e nenhum código foi alterado. F006 permanece aberta, assim como SC-10 em investigação; F005 permanece aberta e o destino de INTRO não foi redirecionado. [Reprodução, evidência e limites](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F007 concluída no AVD em 22-10-2026:** commit inicial `2609006c488d87bf863ba6f0371b269d2d8787e7`, árvore limpa. No `GAME_INIT_OPTIONS_SCREEN`, a legenda de Experienced não selecionava a opção ao toque embora a caixa a selecionasse. A correção Android restringe-se a áreas de toque nas legendas das quatro famílias de opções, mantendo seleção exclusiva e ações originais. Build debug passou; no AVD visível Android 17/API 37, `x86_64`, validaram-se paisagem original 2400×1080@420 dpi, perfil compacto 1600×720@320 dpi e tablet virtual 2560×1920@320 dpi. Legendas, caixas, Done/Cancel, avisos, Yes/No, Esc/Enter e retoma por Home/Recentes responderam; Yes entrou uma vez na abertura da campanha. A suspeita de desalinhamento no perfil compacto não se reproduziu. Rato físico, Pixel/tablet físicos e rotação dinâmica do mesmo frame ficam pendentes. Overrides restaurados e AVD deixado ligado. F005 e F006 permanecem abertas; SC-10 continua em investigação. [Reprodução, validação e limites](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F005 e F006 concluídas no AVD em 22-10-2026:** em paisagem 2400×1080@420 dpi no AVD visível Android 17/API 37, `x86_64`, a intro de novo jogo avançou por toque, retomou após Home/Recentes e terminou no laptop/mapa da campanha. A antiga expectativa de regresso ao Main Title foi corrigida no plano: `INTRO_BEGINING` sai para `INIT_SCREEN` e continua a campanha; não houve redirecionamento. No título, New Game, Preferences/Done, Credits, Quit e Home/Recentes responderam. Continue desativado sem save não transitou; após criar um save local no AVD, Continue abriu Load Game, Cancel voltou ao título e Load entrou no mapa guardado. A suspeita de desalinhamento no perfil compacto continuou não reproduzida. Não foi alterado código para F005/F006 nem executada nova build; usou-se o APK debug validado em F007. Rato físico, hardware Pixel/tablet e preservação exata do frame durante rotação dinâmica ficam pendentes. SC-10 continua em investigação. [Percursos, comandos e limites](SC-10-ANDROID-SMALL-SCREENS-PLAN.md).

**SC-10/F008 — correção validada no AVD em 24-10-2026; fase em validação:** HEAD inicial `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`. Home/Recentes com o laptop aberto provocou recriação por `screenLayout`, quit SDL e perda do estado carregado. Acrescentar apenas `screenLayout` à atividade do jogo no manifesto preservou a página e o PID na repetição. Build debug e instalação passaram; navegação da moldura, favoritos/Cancel, minimizar, Shut Down/reentrada, Esc e retoma foram verificados em paisagem 2400×1080@420 dpi, 1600×720@320 dpi e 2560×1920@320 dpi. Rotação base/tablet preservou o conteúdo. Não se reproduziram bloqueio de entrada, desaparecimento da barra ou desalinhamento de alvos com coordenadas corretas; `Laptop.cc` e as páginas internas ficaram sem alterações. **Continuação em 24-10-2026:** ADB encontrou apenas o AVD `emulator-5554`, Android 17/API 37, 1080×2400@420 dpi e launcher em primeiro plano; save F006test (409.643 bytes) e dados JA2 no AVD foram confirmados intactos. Não havia Pixel nem janela desktop no CUA. A build atual falhou no sandbox (cache Cargo somente de leitura/rustc abortado) e fora dele (rustup falhou ao instalar `rust-src`), por isso não se obteve janela atual e rato/teclado não foram testados. Não houve novo teste de renderer, não se reproduziu falha F008 nesta execução e não se alterou código; a tentativa desktop anterior com build antiga já está documentada no relatório. Ergonomia e hardware Pixel/tablet permanecem pendentes; F008 não está concluída e SC-10 continua em investigação. [Inventário, passos, medidas e evidência](SC-10-F008-LAPTOP-FRAME.md).

**Confirmação adicional, 24-10-2026:** estado inicial repetido no mesmo HEAD/branch e mesmas alterações sem stage. ADB falhou ao iniciar dentro da sandbox, mas funcionou fora; enumerou apenas `emulator-5554` (Android 17/API 37, `x86_64`), já em 1080×2400@420 dpi, rotação automática e launcher. O save privado continuou com 409.643 bytes e `Data/` original presente com 178 ficheiros. CUA mostrou `apps: []`; não havia janela desktop operável. Nova build de `_bin/ja2` parou em `cargo-build-debug` (exit 101), por banco do cache Cargo somente de leitura e impossibilidade de rustup criar temporário; não houve teste de desktop nem nova execução da matriz no AVD, e nenhuma falha F008 foi reproduzida. F008 permanece em validação; rato/teclado desktop e ergonomia em Pixel físico continuam pendentes. [Evidência desta execução](SC-10-F008-LAPTOP-FRAME.md).

**Continuação, 24-10-2026:** a confirmação atual repetiu HEAD `2df20b6eca6ff683b7b13b2b4a5c8da08e93b02d`, branch `android-enhancement`, sem stage. ADB fora da sandbox encontrou apenas o AVD Android 17/API 37, `x86_64`, 1080×2400@420 dpi no launcher; save com 409.643 bytes e `Data/` foram preservados. CUA não ofereceu inventário de apps/janelas. A build atual desktop falhou tanto na sandbox (cache Cargo somente leitura e rustup sem temporário) como fora dela (exit 101 por artefactos Cargo incompatíveis com `rustc 1.101.0-nightly`). Não se repetiu a matriz AVD, não se reproduziu nova falha e rato/teclado desktop não foram testados; os testes AVD dos três perfis continuam sendo evidência anterior. Pixel físico e ergonomia continuam pendentes; F008 fica em validação e SC-10 em investigação. [Evidência detalhada](SC-10-F008-LAPTOP-FRAME.md).

**Verificação atual, 24-10-2026:** HEAD/branch e alterações unstaged mantiveram-se; nenhum ficheiro foi staged. ADB fora da sandbox confirmou apenas o AVD visível `emulator-5554`, Android 17/API 37, `x86_64`, 1080×2400@420 dpi em retrato com launcher em primeiro plano. O save manteve 409.643 bytes e `Data/` continuava presente (178 ficheiros). A CUA não encontrou apps ou tabs e não expõe `cua.listWindows()`; a build atual voltou a falhar em Cargo (`E0514`, artefactos incompatíveis com `rustc 1.101.0-nightly`). Não se abriu build antiga, não se iniciou o jogo no AVD, não se repetiu a matriz nem se reproduziu nova falha. Desktop e Pixel físico continuam sem validação; F008 permanece em validação e SC-10 em investigação. [Registo detalhado](SC-10-F008-LAPTOP-FRAME.md).

**Nova execução, 24-10-2026:** o estado inicial era HEAD `97d6301d3373e3819c6d84de50b6c890ae69d6dd`, branch `android-enhancement`, árvore limpa (sem stage, alterações ou não rastreados). ADB falhou dentro da sandbox e funcionou fora dela; o único dispositivo era o AVD visível `emulator-5554`, Android 17/API 37, `x86_64`, 1080×2400@420 dpi em retrato com launcher em primeiro plano. Save F006test manteve 409.643 bytes e `Data/` estava presente; nenhum dos dois foi aberto ou alterado. A CUA mostrou `apps: []`, browsers sem tabs e não disponibilizou `cua.listApps()`/`cua.listWindows()`. A build inicial falhou em `cargo-build-debug` (exit 101; cache Cargo somente leitura e metadados rejeitados com `E0514` por `rustc 1.101.0-nightly`). A tentativa com `RUSTUP_TOOLCHAIN=1.85.1` compilou a biblioteca Rust, mas falhou no C++ em `src/sgp/SGP.cc:420` porque `shouldStartInFullScreen` não está declarado (declaração comentada na linha 337); esse bloqueio fora de F008 não foi alterado. Não houve janela atual, interação desktop, repetição da matriz AVD ou reprodução nova de F008. Desktop e Pixel físico continuam não testados; **F008 fica registada como incompleta, em validação**, e SC-10 em investigação. [Registo detalhado](SC-10-F008-LAPTOP-FRAME.md).

**Desfecho da build, 24-10-2026:** a declaração comentada em `SGP.cc:337` foi restaurada como correção mínima: `EngineOptions_shouldStartInFullscreen(params.get())` é a API que abastece a variável já usada em `:420`, preservando o comportamento de `--fullscreen` e `ja2.json`. A build `RUSTUP_TOOLCHAIN=1.85.1 cmake --build _bin --target ja2 -j1` passou; reutilizou a biblioteca Rust já concluída na tentativa anterior, recompilou C++ e ligou `ja2`. A CUA ainda não tinha apps nem janela desktop selecionável, então nenhum atalho ou comando da moldura foi exercitado no desktop; não iniciei o binário porque isso abriria os dados originais. A matriz dos perfis paisagem é evidência anterior e não foi repetida. No fim, apenas metadados confirmaram o AVD Android 17/API 37 em 1080×2400@420 dpi no launcher, a presença de `/sdcard/Download/JA2/Data` e o save de 409.643 bytes. Não havia Pixel físico. **F008 permanece incompleta, em validação; SC-10 continua em investigação.** [Registo detalhado](SC-10-F008-LAPTOP-FRAME.md).

**Regressão F008 agendada para o fim das fases do laptop:** depois de F009–F032, executar a lista de cliques/atalhos desktop via CUA e a validação física de alvos, área segura, retoma e rotação registadas no relatório F008. Até lá, F008 continua em validação; esta nota não fecha F008 nem SC-10.

**SC-10/F009 — verificação de 24-10-2026:** HEAD `b11ff300fd901f28ccfd3febedc4db967a869b78`, branch `android-enhancement`, com as alterações documentais anteriores preservadas e sem stage. A leitura estática de `Laptop.cc`/`Laptop.h` confirmou a entrada em `LAPTOP_MODE_NONE`, desenho do desktop e callbacks dos ícones, mas não validou recursos, input nem regresso em runtime. CUA não expôs janela do jogo; o AVD atual era Android 17/API 37, `x86_64`, 1080×2400@420 dpi no launcher. O jogo não foi iniciado porque alcançar o laptop exige abrir dados/campanha JA2 proibidos nesta execução; `F006test` e os dados originais não foram acessados. Entrada, ícones, apps e regresso ficam **não testados**; nenhuma falha foi reproduzida nem classificada como não reproduzida. [Relatório F009](SC-10-F009-LAPTOP-MODE-NONE.md). F009 permanece aberta; F008 continua em validação e SC-10 em investigação.

**Nova tentativa F009, 24-10-2026:** nova consulta à CUA continuou a mostrar `apps: []` e sem janelas; `cua.listWindows()` não está disponível. A revisão estática do callback genérico confirmou os eventos de chamada primário/secundário, mas não encontrou causa reproduzível para corrigir. Sem uma janela observável e sem iniciar uma campanha que leia dados protegidos, não há um percurso seguro de runtime nesta sessão; F009 permanece aberta e não testada.

**Reexecução F009 no AVD, 24-10-2026:** após autorização para uma campanha nova não gravada e leitura dos dados JA2, usei o AVD visível `emulator-5554` (Android 17/API 37, `x86_64`, 1080×2400@420 dpi, retrato) com toques de ecrã e capturas ADB. O desktop inicial do laptop apareceu; E-mail, Web, Files, History, Personnel e Financial abriram o ecrã esperado e o X de cada app regressou ao desktop sem bloquear nem executar outra transição. Não foram abertas páginas internas; a campanha não foi gravada e `F006test` não foi acedido. Mouse/teclado desktop, hardware físico, ergonomia e entrada a partir do mapa continuam pendentes. [Relatório F009](SC-10-F009-LAPTOP-MODE-NONE.md). F009 continua aberta; F008 permanece em validação com regressão adiada e SC-10 em investigação.

**Aceitação:** a matriz SC-10.1 não tem comandos essenciais sem via tátil; pinça e pan não executam ações de jogo acidentalmente; zoom e reposição deixam seleção e hit testing alinhados com a imagem; os controlos têm alternativa visível aos gestos; a campanha de validação decorre sem teclado/rato; o mesmo save mantém as mesmas regras e resultados ao passar de móvel para desktop. **Riscos:** a UI original foi desenhada para rato e 640×480; zoom de todo o framebuffer pode tornar painéis inacessíveis, e gestos podem colidir com arrasto de itens. Resolver por ecrã e com testes físicos, não apenas em emulador.

## SC-04 — Escala e ecrãs largos

**Resultado esperado:** texto e controlos legíveis em HiDPI, incluindo ecrãs móveis, e painel tático funcional em proporções largas. Esta tarefa trata resolução interna e escala da interface; o zoom e os gestos do mundo ficam em SC-10. Começar em `src/sgp/Video.cc`, `src/game/UILayout.cc` e `src/game/Tactical/Interface*`.

**Referências de necessidades:** [HiDPI/Retina #2534](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2534) e [layout tático largo #2400](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2400). O jogo já oferece modos de escala SDL; medir a legibilidade antes de propor outro mecanismo.

1. **SC-04.1 — Capturas de referência.** Comparar 640×480, 1080p, ecrã HiDPI, telemóvel e tablet; janela e ecrã inteiro quando aplicável; escala linear, quase perfeita e perfeita. Identificar texto ilegível, alvos de toque/clique pequenos, áreas vazias e elementos cortados.
2. **SC-04.2 — Modelo de escala.** Prototipar separação entre resolução lógica, tamanho da janela e escala de interface. Definir como coordenadas de rato/toque são convertidas e como a escolha fica guardada.
3. **SC-04.3 — Layout tático.** Ajustar a colocação do painel para formatos largos, mantendo acesso a informação e comandos e sem tapar conteúdo essencial. Testar também o tamanho mínimo suportado.

**Aceitação:** capturas comparáveis antes/depois, nenhum controlo essencial cortado, input alinhado com o que se vê e definição persistente. **Risco:** mudanças globais de coordenadas podem afetar todos os ecrãs; implementar por etapas.

## SC-05 — Clareza das ações táticas

**Resultado esperado:** antes de confirmar uma ação, o jogador entende o custo e os principais motivos de impossibilidade. Começar em `src/game/Tactical/Handle_UI.cc`, `src/game/Tactical/PathAI.cc`, `src/game/Tactical/Weapons.cc` e `src/game/Tactical/LOS.cc`.

1. **SC-05.1 — Inventário de previsões.** Comparar o custo em pontos de ação e o percurso mostrados pela UI com o custo efetivo para movimento, mudança de postura e disparo. Investigar também a [dúvida sobre custo fixo de disparo #2545](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2545). Enumerar diferenças comprovadas.
2. **SC-05.2 — Uma fonte de cálculo.** Para cada diferença, reutilizar a regra efetiva na pré-visualização; evitar uma segunda fórmula mantida apenas para a UI.
3. **SC-05.3 — Apresentação.** Indicar ação impossível, pontos insuficientes, percurso bloqueado e linha de visão de forma consistente, também após seleção por toque ou com zoom. Evitar revelar informação que as regras do jogo escondem ao jogador.

**Aceitação:** uma tabela de ações representativas mostra previsão igual ao resultado; testes cobrem limites de pontos de ação; feedback permanece legível nas resoluções suportadas. **Risco:** mostrar probabilidade ou informação oculta pode alterar a experiência tática.

## SC-06 — IA e ritmo dos turnos

**Resultado esperado:** adversários fazem escolhas mais coerentes sem turnos excessivamente longos nem mudanças bruscas de dificuldade. Começar em `src/game/TacticalAI/{AIMain,DecideAction,Movement,Attacks,Knowledge}.cc`.

1. **SC-06.1 — Medição.** Instrumentar tempo por turno e contar ações nulas/repetidas em cenários fixos. Guardar semente aleatória, mapa, unidades e dificuldade; estabelecer linha de base.
2. **SC-06.2 — Casos prioritários.** Selecionar dois ou três comportamentos comprovadamente problemáticos por frequência ou gravidade. Definir a decisão esperada antes de mudar heurísticas.
3. **SC-06.3 — Correções e comparação.** Corrigir cada caso isoladamente; comparar tempo de turno, resultado tático e comportamento em dificuldades diferentes. Alterações de equilíbrio merecem notas de versão.

**Aceitação:** casos escolhidos melhoram de forma reproduzível e o tempo de turno não piora de modo relevante; execução sem regressões de IA óbvias. **Risco:** uma heurística local pode produzir comportamento inesperado noutras situações.

## SC-07 — Áudio multiplataforma

**Resultado esperado:** música e efeitos sem cortes ou distorção nas plataformas suportadas. Começar em `src/sgp/SoundMan.cc` e `src/game/Utils/Music_Control.cc`.

1. **SC-07.1 — Reprodução.** Confirmar [glitches de música em Windows/SDL3 #2527](https://github.com/ja2-stracciatella/ja2-stracciatella/issues/2527) numa build deste fork; anotar dispositivo, formato, transições e duração.
2. **SC-07.2 — Diagnóstico.** Medir estado de buffers, mudanças de faixa, pausa e retoma. Relacionar cada falha com uma causa observada antes de mudar o fluxo de áudio.
3. **SC-07.3 — Correção e comparação.** Verificar música, efeitos, vídeos e volume em Windows e pelo menos outra plataforma.

**Aceitação:** cenário original reproduzido antes e ausente depois, sem regressão audível nos cenários comparados. **Risco:** áudio depende do dispositivo e do backend; registar ambiente de teste.

## SC-08 — Diagnóstico de mods

**Resultado esperado:** um mod inválido indica ficheiro e causa, e a ordem de sobreposição é compreensível. Começar em `rust/stracciatella/src/{mods,vfs,schemas}`, `src/externalized/ModPackContentManager.cc` e `src/externalized/DefaultContentManager.cc`.

**Facto observado:** existem esquemas JSON e validação de vários modelos; o gestor de mods já identifica mods ativados mas ausentes. A tarefa é localizar lacunas de diagnóstico, não criar validação genérica do zero.

1. **SC-08.1 — Casos de erro.** Testar manifesto malformado, JSON inválido, referência em falta, dois mods com o mesmo recurso e save com conjunto diferente de mods.
2. **SC-08.2 — Mensagens.** Para falhas confirmadas, incluir mod, caminho virtual/físico, campo JSON quando disponível, prioridade de camada e ação sugerida. Evitar despejar rastos técnicos na interface normal.
3. **SC-08.3 — Verificação antecipada.** Se o custo de arranque permitir, validar recursos relevantes antes de entrar na campanha; preservar modo de diagnóstico detalhado no log.

**Aceitação:** cada fixture inválida aponta para o mod e ficheiro corretos; mods válidos continuam a carregar; a ordem de sobreposição observada corresponde à documentada. **Risco:** validação demasiado rígida pode rejeitar mods existentes.

## SC-09 — Identidade Sour Cream

**Resultado esperado:** nome e apresentação coerentes sem perder saves, configuração ou associação correta ao projeto de origem. Começar em `README.md`, `CMakeLists.txt`, `src/game/GameVersion.cc`, `src/launcher`, `assets/icons` e documentação de distribuição.

1. **SC-09.1 — Inventário de nomes.** Separar nomes públicos (janela, launcher, pacote, documentação) de identificadores técnicos (executáveis, diretórios de configuração, saves, API, nomes de biblioteca).
2. **SC-09.2 — Decisão de compatibilidade.** Documentar quais identificadores continuam `ja2`/`stracciatella` e quais mudam. Se mudar caminhos de configuração ou saves, especificar descoberta/migração e retorno seguro.
3. **SC-09.3 — Aplicação.** Atualizar marca, instruções de instalação e atribuições/licenças; verificar nomes em Linux, Windows, macOS e Android conforme os pacotes afetados.

**Aceitação:** jogador vê uma identidade coerente; instalações e saves anteriores são encontrados ou recebem migração explícita; atribuições ao JA2 e Stracciatella permanecem corretas. **Risco:** renomear identificadores internos cedo pode criar incompatibilidades sem benefício direto para o jogador.

## Marcos de entrega

| Marco | Conteúdo | Condição de saída |
| --- | --- | --- |
| M1 — Campanha fiável | SC-01, SC-02, SC-03 | Iniciar, gravar, carregar e progredir nos cenários documentados; sem bloqueio confirmado por resolver no conjunto escolhido. |
| M2 — Android jogável | SC-10, SC-04 | Campanha de validação completada por toque em telemóvel e tablet Android; zoom e alvos alinhados; comandos originais preservados em desktop; decisão de viabilidade iOS registada. |
| M3 — Jogo claro | SC-05 | Previsões de ações consistentes em toque e rato, com texto e controlos legíveis nas resoluções escolhidas. |
| M4 — Polimento | SC-06, SC-07, SC-08 | Comparações de IA e áudio concluídas; erros de mods acionáveis. |
| M5 — Fork público | SC-09 | Nome, documentação, pacotes e migração de dados verificados. |

## Como manter este documento

Ao iniciar um item, acrescentar sob ele uma nota datada com a hipótese e a reprodução planeada. Ao concluir, registar commit/PR, cenários verificados e limitações. Reordenar a tabela apenas com nova evidência de impacto; não marcar tarefas como concluídas porque existe código relacionado ou porque o upstream fechou uma issue.
