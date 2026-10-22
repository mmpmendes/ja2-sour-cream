# Verificações existentes e lacunas

## Automatizadas

`CMakeLists.txt` ativa testes C++ por padrão; `.ci/ci-build.sh` executa o alvo `cargo-test` e `ja2 -unittests` quando `RUN_TESTS=true`. O build Android usa `WITH_UNITTESTS=OFF` em `android/app/build.gradle`. O guia Android diz que ainda não há diretórios de testes Android `src/test` ou `src/androidTest`.

## Manuais necessárias

O guia `docs/ANDROID_BUILD_AND_DEVICE_TEST.md` propõe cenários de arranque, combate, inventário, estratégia/laptop, saves, ciclo de vida e apresentação. O registo anterior no roadmap confirma build Android anterior, mas não ensaio físico por ADB indisponível. [TODO] Executar a matriz de `docs/SC-10-ANDROID-SMALL-SCREENS-PLAN.md` em telemóvel e tablet com dados originais.

## Critério de teste de UI

Para cada mudança: reproduzir primeiro a lacuna, confirmar toque e rato/teclado, verificar seleção/hit testing com escala/zoom e comparar mesma ação/estado com desktop. Build sem dispositivo não valida esses comportamentos.

## Evidência

- `CMakeLists.txt`
- `.ci/ci-build.sh`
- `android/app/build.gradle`
- `docs/ANDROID_BUILD_AND_DEVICE_TEST.md`
- `docs/SOUR_CREAM_ROADMAP.md`
