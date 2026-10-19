# SC-01.1 — Matriz de gravação e carga

Estado em 19-10-2026: **em investigação**. Referência: `a4adb5113b67346cf52ede15492d2d1adc541bce` (Linux). Este documento separa testes de componentes, ensaios do sistema de ficheiros e cenários completos do jogo. Nenhum ensaio isolado demonstra, por si, que `SaveGame()` falha ou que um save completo volta a carregar.

## Preparação e registo

Usar uma cópia legal dos dados originais de JA2, sem mods inicialmente. Registar para cada execução: commit, sistema/dispositivo, versão e localização dos dados, mods, diretórios de temporários e saves, respetivos sistemas de ficheiros, nome do slot, passos, estado esperado, estado observado, log e resultado (`passou`, `falhou`, `inconclusivo` ou `não executado`). Não publicar dados proprietários nem saves que os contenham. Antes de simular falhas, copiar os saves pessoais para fora da área de teste; usar apenas uma campanha e diretórios descartáveis.

Para verificar uma substituição, calcular `sha256sum` do save **antes** e **depois**. O mesmo hash após uma falha confirma que os bytes do ficheiro anterior não mudaram, mas a capacidade de o carregar tem de ser verificada no jogo. Após uma gravação bem-sucedida, comparar também dia/hora de jogo, setor, equipa, inventário, saldo e estado relevante do cenário. Um hash diferente sozinho não confirma a carga correta.

| ID | Cenário e estado a registar antes de gravar | Verificação após carga ou falha | Estado neste ambiente |
| --- | --- | --- | --- |
| S01 | Campanha nova: dia/hora, setor, equipa e saldo | Voltar ao menu, carregar e comparar os quatro valores | Não executado: faltam dados originais |
| S02 | Combate: turno, posições, vida, munições e inimigos visíveis | Carregar e comparar sem executar nova ação | Não executado: faltam dados originais |
| S03 | Mudança de setor: origem, destino, grupo e hora | Carregar depois da transição e comparar | Não executado: faltam dados originais |
| S04 | Inventário: itens, quantidades e posições de dois mercenários | Mover/trocar itens, gravar, carregar e comparar; confirmar ausência de perda/duplicação | Não executado: faltam dados originais |
| S05 | Contratação: mercenário, contrato, saldo e chegada | Gravar e carregar antes e depois da chegada | Não executado: faltam dados originais |
| S06 | Quicksave e autosave: slot, data/hora e conteúdo anterior | Substituir cada slot e carregar a versão esperada | Não executado: faltam dados originais |
| S07 | Mod ativo: nome, versão e recurso alterado | Carregar com o mesmo mod; testar alteração do conjunto de mods separadamente | Não executado: faltam dados e mod de teste validado |
| S08 | Save antigo: versão, localização, mods e estado conhecido | Carregar e comparar estado observável; os testes de cabeçalho não substituem esta prova | Não executado: faltam dados originais |
| S09 | Android: campanha, save e estado antes de suspender | Suspender, retomar, carregar; repetir após bloquear/desbloquear | Não executado: faltam dispositivo e dados originais |
| S10 | Gravação falhada: hash do save válido anterior | Falta de espaço ou permissão num ambiente descartável; exigir erro, hash anterior igual e carga anterior possível | Não executado no jogo: faltam dados originais |
| S11 | Interrupção: hash do save válido anterior | Interromper em fases controladas da gravação; reiniciar e tentar carregar o anterior | Não executado no jogo: faltam dados originais |
| S12 | Temporário e destino em sistemas de ficheiros distintos | Gravar/carregar e comparar estado; numa cópia interrompida, confirmar hash e carga do save anterior | Teste do percurso de ficheiros passou; fluxo do jogo não executado |

Para S09, seguir [o guia de dispositivo Android](ANDROID_BUILD_AND_DEVICE_TEST.md), registando modelo, API, ABI, resolução, versão dos dados e mods. Não desinstalar uma aplicação que contenha saves sem os copiar primeiro. Para S10 e S11, não provocar falhas sobre um único save válido; a medição tem de acontecer sobre uma cópia descartável.

## Percurso de escrita observado no código

- `SaveGame()` escreve `save/<slot>.sav` através de `GCM->tempFiles()`, fecha o ficheiro e chama `FileMan::moveFile()` para o caminho de `GCM->saveGameFiles()` (`src/game/SaveLoadGame.cc`). Na exceção, apaga a tentativa temporária, devolve `FALSE` e emite mensagem de erro. A UI também apresenta erro quando recebe `FALSE` (`src/game/SaveLoadScreen.cc`). Ainda falta verificar esse comportamento num jogo iniciado.
- `DefaultContentManager` obtém o destino de `EngineOptions_getSaveGameDir()` e cria o temporário através de `TempDir_create()`. Fora do Android, este usa o diretório temporário do sistema; no Android, a cache da aplicação (`rust/stracciatella_c_api/src/c/fs/tempdir.rs`).
- `FileMan::moveFile()` chama `Fs_rename()`. A função Rust tenta `std::fs::rename` e, perante `EXDEV` (ou o equivalente Windows), copia e apaga a origem. A implementação anterior copiava diretamente sobre o destino: uma falha durante essa cópia podia truncar o save anterior. A correção SC-01.2 copia primeiro para um ficheiro provisório no diretório de destino e só o substitui após a cópia completa. Continua a faltar validação dentro do jogo e em Android/Windows.
- `BackupSavedGame()` roda cópias para o modo Dead is Dead antes de chamar `SaveGame()` (`src/game/SaveLoadScreen.cc`). Este modo exige um caso separado em SC-01.2 antes de propor alterações à substituição.

## Evidência executada neste ambiente

1. `GTEST_FILTER='SaveLoadGameTest.*:SaveLoadGameStatesTest.*' ./ja2 -unittests`: **20 testes passaram**, incluindo parsing de cabeçalhos de fixtures Linux, Windows, macOS e vanilla russa. Executado em `/workspace/build-ja2` com `HOME=/workspace/sc01-home` e `LD_LIBRARY_PATH=/workspace/sdl-install/lib`. Uma tentativa inicial com `--gtest_filter` foi rejeitada pelo parser de argumentos do jogo; a variável `GTEST_FILTER` é a forma funcional.
   A suite nativa completa também passou (**165 testes**). Após acrescentar os quatro testes do percurso de substituição, `cargo test --locked --workspace --quiet`, em `rust/`, terminou com **113 testes passados, 1 ignorado**.
2. Ensaio descartável de `os.rename` com origem em `/tmp` e destino em `/workspace`: devolveu `EXDEV` (`errno 18`), o que aciona a cópia de `Fs_rename`. Uma reprodução da cópia antiga com limite de tamanho de ficheiro (`ulimit -f 1`) falhou após escrever 1024 bytes no destino anterior: o ficheiro original tinha 19 bytes e deixou de estar íntegro. Quatro testes Rust da correção verificam cópia completa, falha a meio, substituição real entre sistemas de ficheiros e proteção do destino anterior quando a cópia real falha. Isto confirma um defeito no percurso de ficheiros, sem ainda confirmar um cenário completo de `SaveGame()`.
3. As quatro fixtures de save em `assets/unittests/saves` têm hashes SHA-256 que podem servir de referência de integridade dos ficheiros de teste. Os testes existentes verificam os cabeçalhos, não uma carga completa de campanha.

| Fixture | SHA-256 |
| --- | --- |
| `strac-linux/SaveGame01.sav` | `e28a486a55504b3e9e500239ea506dbbc6895807bf96a2eeebde59d5d47d08ae` |
| `strac-macos/SaveGame09.sav` | `43fd7adcf34872532548383b7b9cae9f7f6c9dcc12c29d2bb6869948669764a0` |
| `strac-win/SaveGame09.sav` | `a65dac8a34ebf7b76b242170aeb3909c3d3ae299c84e339a25d2d958ad057049` |
| `vanilla-russian/SaveGame06.sav` | `6777265a55eb5a600f6a4650e83472ac4cbb2ff733c0e631ae7ac0459f7abf1f` |

Para repetir o ensaio de sistema de ficheiros sem tocar em saves reais:

```sh
python3 - <<'PY'
from pathlib import Path
from tempfile import TemporaryDirectory
import errno
import hashlib
import os

with TemporaryDirectory(dir='/tmp') as temp_dir, TemporaryDirectory(dir='/workspace') as save_dir:
    destination = Path(save_dir) / 'slot.sav'
    destination.write_bytes(b'previous-valid-save')
    previous_hash = hashlib.sha256(destination.read_bytes()).hexdigest()
    staged = Path(temp_dir) / 'slot.sav'
    staged.write_bytes(b'new-save')
    try:
        os.rename(staged, destination)
    except OSError as error:
        assert error.errno == errno.EXDEV, error
        assert hashlib.sha256(destination.read_bytes()).hexdigest() == previous_hash
        print('EXDEV; destino anterior inalterado')
    else:
        raise AssertionError('origem e destino estão no mesmo sistema de ficheiros')
PY
```

Este ensaio mostra apenas a primeira tentativa de `rename`; o código Rust tem um percurso de cópia para `EXDEV`. Requer que `/tmp` e `/workspace` estejam em sistemas de ficheiros distintos; confirmar com `df -T /tmp /workspace` antes de o aplicar noutra máquina.

## Critério para avançar

SC-01.1 fica aberto até haver dados originais e, para S09, um dispositivo Android acessível. A primeira execução com dados deve começar por S01 e S10/S12, pois estabelece carga básica e proteção do save anterior. A correção SC-01.2 do percurso de ficheiros precisa de validação em jogo e nas plataformas afetadas antes de ser considerada concluída. O formato do save e as regras não foram alterados.
