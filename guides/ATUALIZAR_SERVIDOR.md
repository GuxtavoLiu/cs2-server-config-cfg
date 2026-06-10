# Guia: Atualizar a versão do CS2 no servidor + revalidar plugins

Quando a Valve solta update do CS2, os clientes atualizam sozinhos e NÃO conseguem mais
conectar num servidor desatualizado (erro de versão incompatível). Este guia cobre o update
do servidor e TODAS as verificações dos plugins depois, na ordem certa.

Regra de ouro: **nunca atualizar no dia do evento**. Atualizar e testar 1-2 dias antes,
porque Metamod/CSSharp/WeaponPaints podem quebrar e os fixes dos devs levam de horas a dias.

---

## PARTE 1 - Antes de atualizar (backup)

Com o servidor DESLIGADO, fazer backup do que NÃO vem do Steam (o update pode sobrescrever):

```bat
mkdir C:\cs2backup
xcopy /E /I /Y C:\cs2server\game\csgo\addons C:\cs2backup\addons
xcopy /E /I /Y C:\cs2server\game\csgo\cfg C:\cs2backup\cfg
copy /Y C:\cs2server\game\csgo\gameinfo.gi C:\cs2backup\gameinfo.gi
```

Backup do banco de skins (MariaDB no Docker):
```
docker exec cs2-mariadb mariadb-dump -u cs2user -pcs2pass123 weaponpaints > C:\cs2backup\weaponpaints.sql
```

O que importa no backup:
- `addons\` = Metamod + CounterStrikeSharp + todos os plugins e configs (admins.json, WeaponPaints.json etc)
- `cfg\` = server.cfg e configs do MatchZy
- `gameinfo.gi` = tem a linha do Metamod que o update APAGA (ver Parte 3)

---

## PARTE 2 - Atualizar o CS2

1. Fechar o servidor (Ctrl+C ou fechar a janela do start.bat).
2. Rodar o mesmo comando da instalação:
```
C:\steamcmd\steamcmd.exe +force_install_dir C:\cs2server +login anonymous +app_update 730 +quit
```
3. Esperar "Success! App '730' fully installed."

Notas:
- Sem `validate` o update é incremental (mais rápido) e geralmente basta.
- Com `validate` ele confere TODOS os arquivos (demora bem mais) e restaura qualquer
  arquivo original modificado. Usar só se o servidor estiver corrompido/bugado.
- Em AMBOS os casos o `gameinfo.gi` costuma ser sobrescrito pelo update. Sempre conferir (Parte 3).

---

## PARTE 3 - Checklist pós-update (ordem importa!)

A ordem de verificação segue a cadeia de dependência, de baixo pra cima:
gameinfo.gi → Metamod → CounterStrikeSharp → MatchZy → cadeia de skins.
Se um nível de baixo está quebrado, tudo acima dele some junto, então não adianta
olhar o WeaponPaints antes de confirmar que o Metamod carregou.

### 3.1 gameinfo.gi (SEMPRE quebra)
Abrir `C:\cs2server\game\csgo\gameinfo.gi` e conferir se a linha do Metamod ainda existe
(com TAB, acima de `Game csgo`):
```
Game	csgo/addons/metamod
```
Se sumiu, adicionar de novo (ou restaurar do backup). Sem essa linha o Metamod nem carrega
e NENHUM plugin funciona, sem mensagem de erro nenhuma.

### 3.2 Metamod
1. Subir o servidor com o start.bat normal.
2. No console: `meta version`
   - OK → seguir para 3.3
   - "Unknown command" → o update do CS2 quebrou o Metamod (ou o gameinfo.gi, ver 3.1).
     Baixar o dev build mais recente em https://www.sourcemm.net/downloads.php?branch=dev
     e copiar `addons` por cima.

### 3.3 CounterStrikeSharp
1. No console: `meta list`
   - Deve listar CounterStrikeSharp SEM erro.
   - Se aparecer erro de load ou nem aparecer: baixar a release "with-runtime" mais recente
     em https://github.com/roflmuffin/CounterStrikeSharp/releases e copiar `addons` por cima.
   - Atenção: depois de update grande do CS2, o CSSharp às vezes demora 1-2 dias para sair
     versão compatível. Acompanhar a página de releases.
2. Lembrete Win11: se trocar DLLs e der erro `0xc0e90002`, é o Smart App Control de novo
   (ver guia principal, Parte 2.2).

### 3.4 Plugins (MatchZy + cadeia de skins)
1. No console: `css_plugins list`
2. Conferir que TODOS aparecem com status carregado:
   - MatchZy
   - WeaponPaints
   - MenuManagerCore
   - PlayerSettings
   - (AnyBaseLib é lib compartilhada, aparece via dependências)
3. Se algum estiver "Error"/faltando, olhar o log de erro no console na subida do servidor
   e atualizar o plugin específico:
   - MatchZy: https://github.com/shobhit-pathak/MatchZy/releases
   - WeaponPaints: https://github.com/Nereziel/cs2-WeaponPaints/releases
     (copiar TAMBÉM o `gamedata\weaponpaints.json` novo, é ele que guarda os offsets
     de memória que o update do CS2 invalida)
   - MenuManager/PlayerSettings/AnyBaseLib: releases do NickFox007 no GitHub

### 3.5 Banco de dados
1. Conferir que o container está de pé: `docker ps` (deve listar cs2-mariadb como Up).
   Se não: `docker start cs2-mariadb`
2. O update do CS2 não mexe no banco; só conferir conexão no log do WeaponPaints na subida
   (não pode ter erro de "Unable to connect").

### 3.6 Teste funcional em jogo (o teste que vale)
Entrar no servidor com um cliente atualizado e conferir:
1. Conectou normal (`connect IP:27015`), sem erro de versão.
2. `!knife` e `!skins` abrem o menu (prova que a cadeia MenuManager/PlayerSettings/AnyBaseLib está ok).
3. `!wp` aplica as skins do banco (prova que WeaponPaints + MariaDB estão ok).
4. Skin aparece na arma/faca (se a skin some mas o menu abre, o gamedata do WeaponPaints
   está desatualizado, ver 3.4).
5. `.ready` responde no chat (prova que o MatchZy está ok).
6. Comando de admin (ex: `.map de_mirage`) funciona (prova que o admins.json sobreviveu).

### Resumo rápido (cola)
| Verificação | Comando | Prova que |
|---|---|---|
| Linha no gameinfo.gi | (abrir o arquivo) | Metamod vai carregar |
| `meta version` | console servidor | Metamod ok |
| `meta list` | console servidor | CSSharp ok |
| `css_plugins list` | console servidor | plugins carregados |
| `docker ps` | terminal | banco de pé |
| `!knife` / `!wp` | em jogo | cadeia de skins inteira ok |
| `.ready` / `.map` | em jogo | MatchZy + admin ok |

---

## PARTE 4 - Se algo quebrou e o evento é hoje

- O CS2 não tem rollback de versão pelo SteamCMD anônimo: não dá pra "voltar" o servidor.
- Se o que quebrou foi só o WeaponPaints (caso mais comum): jogar sem skins. Remover/renomear
  a pasta `plugins\WeaponPaints\` e subir o servidor; MatchZy e a partida funcionam normal.
- Se o CSSharp quebrou: sem MatchZy também. Dá pra rodar a partida "na mão" com o server.cfg
  MR12 + `mp_warmup_end` no rcon, e gravar demo via CSTV que já fica no server.cfg.
- Restaurar configs perdidos a partir de `C:\cs2backup`.

---

## NOTAS

- A causa nº 1 de "instalei tudo e nada funciona" depois de update é o `gameinfo.gi`
  sobrescrito. Conferir ele PRIMEIRO, antes de sair baixando plugin novo.
- O `validate` não apaga a pasta `addons\` (ela não faz parte do depot), mas restaura
  arquivos originais do jogo que foram modificados, incluindo o `gameinfo.gi`.
- WeaponPaints depende de offsets de memória do processo do jogo: é o plugin mais frágil
  a updates. Se o CS2 atualizou e os devs ainda não soltaram release nova, esperar ou
  jogar sem skins.
- Manter o `C:\cs2backup` de cada update que deu certo; é o jeito mais rápido de recuperar
  config perdida.
