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
| 4 | SC-10 Experiência móvel, toque e zoom | P1 | L | por iniciar | SC-04.1 para medir legibilidade; SC-05.1 para ações táticas |
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
