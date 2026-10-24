# SC-10/F009 — desktop interno do laptop (`LAPTOP_MODE_NONE`)

**24-10-2026 — execução inicial bloqueada; reexecução por toque no AVD concluída, fase ainda aberta.** A revisão estática confirma que a entrada inicializa `LAPTOP_MODE_NONE`, desenha o desktop e regista os seis ícones laterais. Na reexecução autorizada, todas as seis apps abriram e regressaram ao desktop por toque no AVD, sem bloqueio observado. F009 permanece aberta por falta de cobertura de mouse/teclado desktop e hardware físico.

## Estado inicial e limites

- HEAD: `b11ff300fd901f28ccfd3febedc4db967a869b78`; branch `android-enhancement`, alinhada com `origin/android-enhancement`.
- Antes desta execução já existiam alterações unstaged em `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md`, `docs/SC-10-F008-LAPTOP-FRAME.md` e `docs/SOUR_CREAM_ROADMAP.md`. Foram preservadas. Não havia alterações staged.
- Foram lidos integralmente `AGENTS.md`, roadmap, plano SC-10, relatório F008, `COMPILATION.md`, `CONTRIBUTING.md` e guia Android.
- A declaração `shouldStartInFullScreen` permanece em `src/sgp/SGP.cc:337`; não foi alterada. A build não foi repetida nesta fase.
- A CUA reportou `apps: []` e browsers sem tabs. Não havia janela do jogo observável para entrada real por rato/teclado.
- ADB dentro da sandbox não conseguiu iniciar o daemon (`Operation not permitted` ao abrir o socket local). Fora da sandbox, `adb devices -l` encontrou apenas `emulator-5554`, um AVD visível. Propriedades lidas: Android 17/API 37, ABI `x86_64`, 1080×2400 a 420 dpi, retrato, rotação automática ligada e `LauncherActivity` em primeiro plano. O AVD não foi manipulado nem encerrado.
- Não se abriu, carregou, copiou, sobrescreveu nem alterou `F006test` ou quaisquer dados originais JA2.

## Requisito e confirmação estática do código

O cenário F009 exige entrar no laptop, observar `LAPTOP_MODE_NONE`, testar os ícones/entrada nas apps e regressar ao desktop sem bloqueio. A revisão de `src/game/Laptop/Laptop.cc` e `src/game/Laptop/Laptop.h` encontrou:

- `LaptopScreenHandle()` chama `EnterLaptop()` e cria os botões ao entrar;
- `EnterLaptop()` inicializa `guiCurrentLaptopMode` como `LAPTOP_MODE_NONE`, cria regiões, carrega/desenha o fundo do desktop e prepara os estados das apps;
- `RenderLaptop()` encaminha `LAPTOP_MODE_NONE` para `DrawDeskTopBackground()`;
- `CreateLaptopButtons()` cria botões visíveis para e-mail, web, ficheiros, pessoal, histórico, finanças e saída;
- os callbacks das apps mudam o modo no evento `MSYS_CALLBACK_REASON_POINTER_UP`; a sequência de transição/maximização e retorno é processada pelo handler do laptop;
- o modo `LAPTOP_MODE_NONE` não desenha moldura de programa; os mecanismos de minimizar/regressar e `HandleLapTopESCKey()` têm tratamento distinto do modo desktop.

Isto confirma a estrutura implementada, mas não demonstra que os recursos carregam, que os alvos recebem input ou que a transição conclui em runtime.

## Resultado e cobertura desta execução

Não foi iniciada a aplicação no AVD nem o executável desktop: para alcançar o laptop é necessário iniciar/continuar uma campanha com recursos do jogo, e abrir esses dados está expressamente proibido nesta fase. O save de teste também está proibido. A janela visível do AVD estava no launcher e não havia uma janela de jogo selecionável pela CUA. Não foram enviados cliques, teclas nem toques ao jogo; não se usou `adb input mouse tap`.

| Cenário F009 | Resultado nesta execução |
| --- | --- |
| Entrada no laptop e apresentação do desktop `LAPTOP_MODE_NONE` | Não testado: runtime depende dos dados/campanha proibidos. |
| Ícones laterais e entrada em cada app | Não testado: sem janela de jogo e sem campanha permitida. |
| Regresso de cada app ao desktop sem bloqueio/ação duplicada | Não testado: depende dos mesmos percursos runtime. |
| Rato, teclado e toque observáveis | Não testados. |
| Falha de runtime reproduzida | Nenhuma; o runtime não foi iniciado, portanto não se classifica qualquer hipótese como não reproduzida. |
| Revisão estática de entrada, desenho e callbacks | Feita; confirma os caminhos descritos acima, sem validar comportamento visual ou input. |

Não houve alteração de lógica nem de páginas internas. Não foram executados testes automatizados nem build, por se tratar de uma investigação documental/estática sem alterações de código. `git diff --check` e revisão dos diffs foram executados no encerramento. Nenhum ficheiro foi staged ou removido.

## Pendências

Para validar F009, é necessário um cenário de runtime permitido que não abra `F006test` nem dados JA2 originais, ou autorização/instrução futura que altere essas restrições. Até então, a entrada, os ícones, as transições e o regresso permanecem pendentes. A regressão F008 continua adiada para depois de F009–F032; F008 permanece incompleta/em validação e SC-10 permanece em investigação.

## Nova tentativa, 24-10-2026

Após pedido para tentar novamente, consultei outra vez a CUA: `apps: []`, browsers sem tabs; `cua.listWindows()` continua indisponível (`cua.listWindows is not a function`). Portanto, o AVD não pode ser ligado a uma superfície de input/observação CUA nesta sessão. Também confirmei no código de `Button_System`/`MouseSystem` que `ButtonCallbackPrimarySecondary` só chama a ação primária em libertação do botão esquerdo ou toque curto, e a secundária em botão direito ou repetição de toque. Isto reduz uma hipótese estática de callback de Web disparar em eventos arbitrários, mas não valida os ícones nem a transição F009 em runtime.

Não usei captura de ecrã nem input ADB como substituto da observação CUA, não iniciei o jogo e não abri os dados ou saves. Não há defeito reproduzido que justifique uma alteração de código; a solução segura disponível nesta sessão continua a ser aguardar uma janela de jogo exposta à CUA e um percurso de campanha autorizado. F009 permanece aberta e não testada em runtime.

## Execução autorizada no AVD, 24-10-2026

Após o utilizador autorizar uma campanha nova não gravada, leitura dos dados JA2 e ADB como alternativa à CUA, confirmei o dispositivo `emulator-5554`: Android 17/API 37, ABI `x86_64`, resolução 1080×2400, densidade 420 dpi, retrato, rotação automática ligada. O AVD estava visível e foi mantido ligado. O launcher já tinha a instalação original selecionada como English; não copiei nem alterei dados JA2. Não abri, carreguei, copiei ou alterei `F006test`. Não usei Continue/Load, não gravei a campanha e não alterei o diretório de saves.

O botão Play abriu a atividade de jogo. Start New Game, as opções predefinidas e a confirmação iniciaram uma campanha nova; a introdução foi avançada apenas para alcançar a campanha. A primeira entrada no laptop mostrou a ajuda inicial e, depois de a dispensar, surgiu um aviso de correio novo; ambos foram fechados sem navegar nas respetivas páginas. O desktop `LAPTOP_MODE_NONE` ficou visível, com os ícones laterais. Os toques foram enviados como `adb shell input touchscreen tap`, após inspecionar cada captura atual; cada mudança foi confirmada por uma captura ADB seguinte. Não se usou `adb input mouse tap`.

| Ícone | Ecrã de destino observado | Regresso observado |
| --- | --- | --- |
| E-mail | Mail Box | X da app voltou ao desktop. |
| Web | sir-FER 4.0, com a lista de bookmarks de primeira entrada | X voltou ao desktop; não foram abertos bookmarks. |
| Files | File Viewer | X voltou ao desktop; não se abriu o ficheiro listado. |
| History | History Log | X voltou ao desktop; não se folheou o histórico. |
| Personnel | Personnel Manager | X voltou ao desktop; não foram abertos separadores/perfis. |
| Financial | Bookkeeper Plus | X voltou ao desktop; não foram folheadas páginas nem alterados valores. |

Cada ícone produziu uma transição visível única; cada X regressou ao fundo do desktop sem bloquear. Não se observou clique residual nem mudança para outra app após o regresso. Isto valida a via tátil básica de entrada/retorno e a apresentação dos ecrãs de destino neste perfil AVD; não valida os conteúdos das páginas internas.

**Cobertura ainda pendente:** não foi possível testar mouse/teclado numa janela desktop CUA nem num hardware físico. Não se testou o regresso ao mapa, minimização por outro mecanismo, os indicadores, nem a ergonomia de alvos. A regressão F008 continua adiada para depois de F009–F032. Não se reproduziu bloqueio nesta cobertura tátil, mas F009 não é declarada concluída devido à cobertura pendente. Nenhum código foi alterado; build e testes automatizados não foram necessários. As capturas ficaram em `/tmp/f009-*.png`, fora do repositório.
