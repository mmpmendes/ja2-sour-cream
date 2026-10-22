# Convenções de alteração

## C++ e documentação

`CONTRIBUTING.md` pede preservar o estilo local, indentação com tabulações no código C++, includes agrupados/ordenados e alterações pequenas que compilem e possam ser revistas. Documentar hipótese, reprodução, verificação e risco segundo `AGENTS.md` e `docs/SOUR_CREAM_ROADMAP.md`.

## Limites funcionais deste fork

O roadmap prioriza campanha jogável, toque completo e compatibilidade com rato/teclado. Mudanças de saves, localização ou prioridade de VFS exigem testes de compatibilidade; a fase de acessibilidade evita alterar esses formatos e regras de jogo. Usar incrementos `SC-10.x` verificáveis, com linha de base anterior à correção.

## Desconhecidos

- [TODO] Confirmar convenções específicas para testes de UI Android quando forem acrescentados; o guia atual diz que `android/app/src/test` e `src/androidTest` não existem.

## Evidência

- `AGENTS.md`
- `CONTRIBUTING.md`
- `docs/SOUR_CREAM_ROADMAP.md`
- `docs/ANDROID_BUILD_AND_DEVICE_TEST.md`
