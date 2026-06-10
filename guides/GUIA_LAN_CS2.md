# Guia: Servidor LAN CS2 (MR12 estilo Major) + Skins

Guia reproduzível do setup completo: servidor competitivo com MatchZy + skins via WeaponPaints.
Ambiente: **Windows 11**, modo **LAN** (`sv_lan 1`, sem VAC).

> Para atualizar a versão do jogo que o servidor roda (e revalidar os plugins depois),
> ver o guia: [ATUALIZAR_SERVIDOR.md](ATUALIZAR_SERVIDOR.md)

---

## PARTE 1 - Servidor base (CS2 dedicado)

### 1.1 SteamCMD
1. Baixar SteamCMD: https://steamcdn-a.akamaihd.net/client/installer/steamcmd.zip
2. Extrair em `C:\steamcmd`
3. Criar pasta `C:\cs2server`

### 1.2 Baixar o servidor (~60 GB)
```
C:\steamcmd\steamcmd.exe +force_install_dir C:\cs2server +login anonymous +app_update 730 validate +quit
```
Esperar "Success! App '730' fully installed."

### 1.3 Arquivo de inicialização
Criar `C:\cs2server\start.bat` (modo LAN, sem GSLT):
```bat
@echo off
cd /d C:\cs2server\game\bin\win64
cs2.exe -dedicated -console -usercon ^
 +game_type 0 +game_mode 1 ^
 -tickrate 128 ^
 -maxplayers 13 ^
 +exec server.cfg ^
 +map de_mirage
pause
```
- `-maxplayers 13` = 10 jogadores + observador + vaga do CSTV.

### 1.4 server.cfg
Colocar `server.cfg` em `C:\cs2server\game\csgo\cfg\server.cfg` com config MR12:
- mp_maxrounds 24, mp_overtime_maxrounds 6, mp_overtime_startmoney 10000
- mp_freezetime 15, mp_roundtime_defuse 1.92, mp_startmoney 800, mp_maxmoney 16000
- mp_halftime 1, mp_match_can_clinch 1, mp_forcecamera 1, sv_lan 1
- tv_enable 1, tv_port 27020, tv_delay 0 (CSTV para gravar demos)

(NÃO declarar fogo amigo: o gamemode_competitive já aplica.)

### 1.5 Conectar
- Descobrir IP local: `ipconfig` (ex: 192.168.0.107)
- Liberar portas 27015-27020 (TCP/UDP) no Firewall do Windows
- Jogadores no console (~): `connect 192.168.0.107:27015`

---

## PARTE 2 - Stack de plugins (ordem importa!)

A ordem de dependência é de baixo pra cima. Instalar nesta sequência:

### 2.1 Metamod:Source (dev build)
1. Baixar Windows dev build: https://www.sourcemm.net/downloads.php?branch=dev
2. Copiar a pasta `addons` para `C:\cs2server\game\csgo\`
3. Editar `C:\cs2server\game\csgo\gameinfo.gi`, adicionar (com TAB, acima de `Game csgo`):
   ```
   Game	csgo/addons/metamod
   ```
4. Confirmar com `meta version` no console do servidor.

### 2.2 CounterStrikeSharp (with runtime)
1. Baixar a release "with-runtime" Windows: https://github.com/roflmuffin/CounterStrikeSharp/releases
2. Copiar a pasta `addons` para `C:\cs2server\game\csgo\` (mesclar)
3. Confirmar com `meta list` (deve aparecer CounterStrikeSharp sem erro).

> IMPORTANTE (Windows 11): se a DLL for bloqueada com erro `0xc0e90002` / "política de
> Controle de Aplicativo", DESLIGAR o **Smart App Control** (Segurança do Windows > Controle
> de Aplicativos e Navegador > Configurações do Controle Inteligente de Aplicativos > Desativado).
> Atenção: uma vez desligado, não pode religar sem reinstalar o Windows.

### 2.3 MatchZy (gerenciador de partida)
1. Baixar: https://github.com/shobhit-pathak/MatchZy/releases (pacote with-cssharp)
2. Copiar `addons` e `cfg` para `C:\cs2server\game\csgo\`
3. Confirmar com `css_plugins list` (deve aparecer MatchZy).

### 2.4 admins.json (para comandos de admin)
Criar `C:\cs2server\game\csgo\addons\counterstrikesharp\configs\admins.json`:
```json
{
  "MeuNome": {
    "identity": "SEU_STEAMID64",
    "flags": ["@css/generic","@css/config","@css/map","@css/rcon","@css/admin","@css/root"]
  }
}
```

---

## PARTE 3 - Skins (WeaponPaints + cadeia de dependências)

A cadeia COMPLETA é: **WeaponPaints → MenuManager → PlayerSettings → AnyBaseLib** (+ banco MySQL).
Instalar TODAS senão dá erro `menu:nfcore` ou `GetCurrentPlayerMenu null`.

### 3.1 Banco de dados (MariaDB via Docker)
```
docker run -d --name cs2-mariadb -e MARIADB_ROOT_PASSWORD=rootpass123 -e MARIADB_DATABASE=weaponpaints -e MARIADB_USER=cs2user -e MARIADB_PASSWORD=cs2pass123 -p 3306:3306 --restart unless-stopped mariadb:latest
```

### 3.2 Instalar a cadeia (todos em addons\counterstrikesharp\)
1. **AnyBaseLibCS2**: https://github.com/NickFox007/AnyBaseLibCS2/releases → copiar `addons`
2. **PlayerSettingsCS2**: https://github.com/NickFox007/PlayerSettingsCS2/releases → copiar `addons`
3. **MenuManagerCS2**: https://github.com/NickFox007/MenuManagerCS2/releases → copiar `addons`
   (instala MenuManagerCore em plugins/ e MenuManagerApi em shared/)
4. **WeaponPaints**: https://github.com/Nereziel/cs2-WeaponPaints/releases
   - Copiar a pasta do plugin (com TODAS as DLLs) para `plugins\WeaponPaints\`
   - Copiar `gamedata\weaponpaints.json` para `addons\counterstrikesharp\gamedata\`

### 3.3 Configurar
1. `addons\counterstrikesharp\configs\core.json` → setar `FollowCS2ServerGuidelines: false`
2. Subir o servidor 1x para gerar o config do WeaponPaints
3. Editar `addons\counterstrikesharp\configs\plugins\WeaponPaints\WeaponPaints.json`:
   ```json
   "DatabaseHost": "127.0.0.1",
   "DatabasePort": 3306,
   "DatabaseUser": "cs2user",
   "DatabasePassword": "cs2pass123",
   "DatabaseName": "weaponpaints",
   ```
4. Reiniciar. Confirmar com `css_plugins list` (MatchZy, WeaponPaints, MenuManagerCore, etc).

### 3.4 Comandos no jogo
- `!knife` → menu de facas
- `!skins` → menu de skins de arma
- `!gloves`, `!agents`, `!music`, `!pins`
- `!wp` → sincroniza as escolhas do banco para o jogo
- `!changemenu chat` → mudar tipo de menu (se necessário)

---

## PARTE 4 - Site de skins (interface visual)

Repo (fork próprio): https://github.com/GuxtavoLiu/cs2-WeaponPaints-Website
(baseado em SwaggyMacro/cs2-WeaponPaints-Website, Node.js)

### 4.1 Rodar
```
git clone <repo> cs2-skins-site
cd cs2-skins-site
npm install
copy src\config.example.json src\config.json
```

### 4.2 config.json
```json
{
  "name": "LAN Skins - CS2",
  "lang": "en",
  "DB": { "DB_HOST":"127.0.0.1","DB_USER":"cs2user","DB_PASS":"cs2pass123","DB_DB":"weaponpaints","DB_PORT":"3306" },
  "HOST": "192.168.0.107",
  "PROTOCOL": "http",
  "PORT": 27075,
  "STEAMAPIKEY": "SUA_STEAM_WEB_API_KEY",
  "connect": { "show": true, "url": "steam://connect/192.168.0.107:27015" }
}
```
- Steam API Key: gerar em https://steamcommunity.com/dev/apikey
- **LAN física**: HOST = IP local, PROTOCOL = http, acessar por http://192.168.0.107:27075
- **ngrok / Radmin VPN**: HOST = IP-radmin ou domínio ngrok (SEM https://), PROTOCOL conforme
  o caso (https para ngrok, http para Radmin). ngrok/Radmin só são necessários para acesso
  remoto; em LAN física não precisa.

### 4.3 Rodar
```
npm start
```
Acessar, logar com Steam, escolher skin/faca, ajustar Float + Pattern (seed), clicar "Change".
Depois no jogo: `!wp` para aplicar.

> CUIDADO no config: HOST sem `https://`; o protocolo vai no campo PROTOCOL separado.
> Se misturar, gera URL quebrada tipo `https//...`.

---

## PARTE 5 - Aplicar skin/pattern manualmente pelo banco (controle exato)

Útil para garantir um pattern específico (ex: Karambit Case Hardened "Blue Gem" = seed 661).

### 5.1 Aplicar a skin base no jogo (`!skins`/`!knife`) para criar a linha, depois editar:
```
docker exec -it cs2-mariadb mariadb -u cs2user -pcs2pass123 weaponpaints -e "UPDATE wp_player_skins SET weapon_paint_id = 44, weapon_seed = 661, weapon_wear = 0.01 WHERE steamid = 'SEU_STEAMID64' AND weapon_defindex = 507 AND weapon_team = 2;"
```
- `weapon_defindex` = arma/faca (507 = karambit)
- `weapon_paint_id` = skin (44 = Case Hardened)
- `weapon_seed` = pattern (661 = Blue Gem)
- `weapon_team` = 2 (TR) ou 3 (CT), skins são por lado!
- No jogo: `!wp` e, se preciso, morrer/renascer para a faca atualizar.

### 5.2 Consultar o banco
```
docker exec -it cs2-mariadb mariadb -u cs2user -pcs2pass123 weaponpaints -e "SELECT * FROM wp_player_skins;"
```
Tabelas: wp_player_skins, wp_player_knife, wp_player_gloves, wp_player_agents, wp_player_music, wp_player_pins
Achar defindex/paint/seed das skins: usar cs2inspects.com ou cs2locker.com

---

## PARTE 6 - Rodar uma partida (MatchZy, modo PUG)

1. Jogadores conectam (`connect IP:27015`), entram 5 CT + 5 TR
2. Admin: `.map de_xxxx` (veto na voz: capitães banem, sobra 1 mapa)
3. (Opcional) `!team1 NomeA` / `!team2 NomeB`
4. Admin: `.readyrequired 10`
5. Todos: `.ready`
6. Faca → quem ganha escolhe lado (`.stay` / `.switch`) → LIVE (demo grava sozinha)

Comandos admin úteis: `.pause`, `.unpause`, `.tac` (timeout tático), `.tech` (timeout técnico),
`.stop` (restaura o round atual), `.restore <round>` (volta para um round específico),
`.forceready`, `.rcon <cmd>`

---

## NOTAS / ARMADILHAS

- **Ordem dos plugins é tudo.** Faltou AnyBaseLib → `GetCurrentPlayerMenu null`. Faltou
  MenuManager certo (NickFox, não schwarper) → `menu:nfcore not found`.
- **Smart App Control (Win11)** bloqueia DLLs não assinadas → desligar.
- **WeaponPaints quebra a cada update do CS2** (mexe na memória). Testar 1-2 dias antes do evento.
  Após qualquer update, seguir o checklist do [ATUALIZAR_SERVIDOR.md](ATUALIZAR_SERVIDOR.md).
- **Skins são por lado (team 2=TR, 3=CT).**
- **cvars antigos do CS:GO** (sv_pure, mp_bomb_defuse_time) dão "Unknown command" no CS2: remover.
- **Sempre revogar/regenerar a Steam API Key se ela vazar.**
- **Jogar via Radmin VPN** (fora da LAN física): `sv_lan 0` + GSLT no start.bat
  (`+sv_setsteamaccount TOKEN`, gerar em https://steamcommunity.com/dev/managegameservers
  com App ID 730).
