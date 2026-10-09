# Cinnamon cai ao conectar um fone Bluetooth

Conectar um fone Bluetooth derruba o Cinnamon no Ubuntu 26.04. O X continua de pé. O que some é o painel e o gerenciador de janelas. A mensagem na tela fala em modo de emergência e em applet, mas o applet de Bluetooth não é a causa.

Autor: Gemayel Lira

## O que aparece

O `cinnamon-launcher` tenta subir o Metacity como reserva. No Ubuntu esse programa não está instalado, então a sessão fica sem janelas. O log mostra:

```
FileNotFoundError: [Errno 2] No such file or directory
```

em `os.execvp("metacity", ...)`.

## Causa

A biblioteca é a `libcvc` (pacote fonte `cinnamon-desktop`, binário `libcvc0t64` 6.4.2-1). Ela acompanha a porta ativa de cada stream de áudio.

A partir do PipeWire 1.5.84 a lista de portas pode mudar enquanto o BlueZ troca o perfil do fone. Em 6.4.2, `gvc_mixer_stream_get_port()` responde a isso com `g_assert_not_reached()` e aborta o processo (`signal 6`). O Cinnamon e o `csd-media-keys` morrem juntos.

No `master` do Linux Mint o aborto já foi trocado por `return NULL` (issue [linuxmint/cinnamon-desktop#267](https://github.com/linuxmint/cinnamon-desktop/issues/267), fechada em março de 2026). Três chamadores em `libcvc/gvc-mixer-control.c` continuam usando o ponteiro sem testar:

- `gvc_mixer_control_lookup_device_from_stream`
- `gvc_mixer_control_change_output`
- `gvc_mixer_control_change_input`

`update_sink` e `update_source` já testam `NULL`. Os três acima não. Tirar só o `assert` troca o aborto por `segfault at 0` dentro de `libcvc.so.0`.

## Correção

Tratar porta ausente como troca de porta: não usar o ponteiro na busca, e chamar `gvc_mixer_stream_change_port()` quando `active_port` é `NULL`.

- Upstream, em cima do `master`: [linuxmint/cinnamon-desktop#278](https://github.com/linuxmint/cinnamon-desktop/pull/278)
- Ubuntu 26.04, pacote `6.4.2-1ubuntu1`: [bugs.launchpad.net/bugs/2170365](https://bugs.launchpad.net/bugs/2170365) e `patches/cinnamon-desktop_6.4.2-1ubuntu1.debdiff`

O patch do Ubuntu junta as duas partes, porque a 6.4.2 ainda tem o `assert`. O pull request do Mint só cobre os três chamadores, porque o `master` já devolve `NULL`.

## Como reproduzir

1. Ubuntu 26.04, Cinnamon 6.4.13, `libcvc0t64` 6.4.2-1, PipeWire 1.6.2, WirePlumber 0.5.13.
2. Conectar um fone Bluetooth (A2DP).

Trechos do journal estão em `logs/`. Endereços de dispositivo foram omitidos.

## Verificação

Com as três checagens, o mesmo fone foi desconectado e conectado três vezes. O PID do `cinnamon` não mudou e o journal não registrou `segfault` nem `code should not be reached`. Ver `logs/validation.txt`.

## Ambiente

| Peça | Versão |
| --- | --- |
| Ubuntu | 26.04.1 LTS |
| Kernel | 7.0.0-38-generic |
| Cinnamon | 6.4.13-2ubuntu0.2 |
| libcvc0t64 | 6.4.2-1 |
| PipeWire | 1.6.2-1ubuntu1.2 |
| WirePlumber | 0.5.13-1ubuntu1 |
