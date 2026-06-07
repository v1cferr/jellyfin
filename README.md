# Jellyfin Home Media Stack

Minha "Netflix local": um conjunto de serviços em Docker que baixa, organiza, legenda e transmite filmes e séries — tudo self-hosted em casa.

A stack sobe **automaticamente junto com o PC** e fica rodando em segundo plano. Você só precisa de comando manual quando quiser parar, atualizar ou ver logs.

---

## Serviços

O acesso "bonitão" é pelos domínios `*.v1cferr.dev` (HTTPS), que funcionam **só na rede local** —
o DNS desses subdomínios aponta para `192.168.1.10` (IP privado, não roteável pela internet).
O reverse proxy é o **Caddy** (via systemd no host, fora do Docker), com certificado curinga
emitido pela Let's Encrypt via DNS-challenge da Cloudflare.

| Serviço          | Função                                             | Acesso local               | Domínio (na LAN)                 |
| ---------------- | -------------------------------------------------- | -------------------------- | -------------------------------- |
| **Jellyfin**     | Servidor de mídia (streaming + DLNA)               | <http://192.168.1.10:8096> | <https://jellyfin.v1cferr.dev>   |
| **Jellyseerr**   | Interface para pedir filmes/séries novos           | <http://localhost:5055>    | <https://jellyseerr.v1cferr.dev> |
| **Prowlarr**     | Gerenciador de indexadores (trackers)              | <http://localhost:9696>    | <https://prowlarr.v1cferr.dev>   |
| **Radarr**       | Automação de filmes                                | <http://localhost:7878>    | <https://radarr.v1cferr.dev>     |
| **Sonarr**       | Automação de séries/anime                          | <http://localhost:8989>    | <https://sonarr.v1cferr.dev>     |
| **Bazarr**       | Download automático de legendas                    | <http://localhost:6767>    | <https://bazarr.v1cferr.dev>     |
| **qBittorrent**  | Cliente de download (torrent)                      | <http://localhost:8080>    | <https://torrent.v1cferr.dev>    |
| **FlareSolverr** | Resolve desafios Cloudflare (helper do Prowlarr)   | (sem interface)            | — (interno, não exposto)         |
| **Cloudflared**  | Túnel Cloudflare (ingress no painel da Cloudflare) | (sem interface)            | —                                |

> O **Jellyfin** fica bindado na IP da LAN (`192.168.1.10:8096`), **não** no `localhost` — use o
> IP ou o domínio. Os demais serviços respondem em `localhost` nesta máquina.
>
> O **NPM (Nginx Proxy Manager)** foi **desativado**; o reverse proxy agora é o **Caddy**
> (`/etc/caddy/Caddyfile`, fora deste repo). As pastas `config/npm` e `config/letsencrypt` ficam
> apenas para eventual rollback.

---

## Início automático no boot

Já está tudo configurado, **não precisa fazer nada**:

- O serviço do Docker está habilitado para iniciar com o sistema (`systemctl enable docker`).
- Cada container usa a política `restart: unless-stopped`, então:
  - Ao **ligar/reiniciar o PC**, todos os serviços voltam sozinhos.
  - Se você **parar manualmente** um serviço, ele continua parado (não volta no boot) até você subir de novo.

---

## Comandos do dia a dia

Sempre rode dentro da pasta do projeto (`~/Projects/Local/jellyfin`).

### Tudo de uma vez

```bash
docker compose up -d        # Sobe/atualiza toda a stack em segundo plano
docker compose stop         # Para tudo (volta com 'start'; NÃO remove nada)
docker compose start        # Religa o que foi parado
docker compose restart      # Reinicia todos os serviços
docker compose down         # Para e REMOVE os containers (config preservada)
docker compose ps           # Mostra o que está rodando
```

> **`stop` x `down`:** use `stop` para pausar (mantém o auto-start no boot).
> O `down` remove os containers — eles só voltam quando você rodar `up -d` de novo.

### Um serviço específico

```bash
docker compose restart jellyfin     # Reinicia só o Jellyfin
docker compose stop qbittorrent     # Para só o qBittorrent
docker compose start qbittorrent    # Religa só o qBittorrent
```

### Logs (diagnóstico)

```bash
docker compose logs -f              # Logs de tudo, ao vivo
docker compose logs -f sonarr       # Logs só do Sonarr
docker compose logs --tail=100 qbittorrent  # Últimas 100 linhas do qBittorrent
```

### Atualizar as imagens

```bash
docker compose pull        # Baixa as versões mais recentes
docker compose up -d       # Recria os containers que mudaram
docker image prune -f      # (opcional) Limpa imagens antigas
```

---

## Estrutura do projeto

```text
~/Projects/Local/jellyfin/
├── docker-compose.yml      # Definição de todos os serviços
├── .env                    # Variáveis e segredos (NÃO versionado)
├── README.md
└── config/                 # Config persistente de cada serviço (NÃO versionado)
    ├── jellyfin/
    ├── jellyseerr/
    ├── prowlarr/
    ├── radarr/
    ├── sonarr/
    ├── bazarr/
    ├── qbittorrent/
    ├── npm/                # legado — NPM desativado (mantido p/ rollback)
    └── letsencrypt/        # legado — certificados do NPM antigo
```

- **Mídia:** todos os serviços compartilham `/home/v1cferr/Videos/Jellyfin` como biblioteca.
- **`config/`** guarda bancos de dados, configs e cache — fica fora do git (`.gitignore`).

---

## Como adicionar mídia (fluxo automático)

A organização (renomear, mover, baixar pôster/sinopse) é feita pelo **Sonarr** (séries/anime)
e pelo **Radarr** (filmes). Você só escolhe o que baixar — o resto é automático.

### Fluxo recomendado (baixando manual)

1. **Adicione a série/filme no Sonarr/Radarr** (uma vez). Para anime, marque
   **Series Type = Anime** — assim ele entende numeração tipo `- 01` e converte para `S01E01`.
2. Baixe o torrent no **qBittorrent do Docker** (<http://192.168.1.10:8080> / `torrent.v1cferr.dev`)
   — **nunca** no qBittorrent do sistema (Arch), pois os *arr não enxergam aquele.
3. Ao adicionar o torrent, defina a **Categoria**:
   - `Series` → séries e anime (salva em `/data/torrents/tv`)
   - `Filmes` → filmes (salva em `/data/torrents/movies`)
4. Pronto. Ao concluir, Sonarr/Radarr renomeiam, movem (via **hardlink**, sem duplicar espaço)
   para `media/` e o Jellyfin escaneia + busca os metadados sozinho.

> **Versão sem navegador:** use o **Interactive Search** dentro do Sonarr/Radarr. Você ainda
> escolhe a release manualmente, mas a importação fica 100% garantida. O indexer **Nyaa.si**
> já está configurado para anime.

### Por que importa a categoria

Os *arr só importam automaticamente o que está na categoria que esperam
(`Series` no Sonarr, `Filmes` no Radarr) **e** que corresponde a algo já adicionado na biblioteca
deles. Sem categoria ou sem a série cadastrada, o download fica parado sem ser organizado.

### Convenção de nomes (se for organizar manualmente no disco)

O Jellyfin identifica pelo nome da pasta/arquivo. Tags de fansub no **início**
(ex.: `[Erai-raws] ...`) quebram a identificação. Use:

```text
media/tv/Nome da Série (Ano)/
└── Season 01/
    ├── Nome da Série S01E01.mkv
    └── Nome da Série S01E02.mkv

media/movies/Nome do Filme (Ano)/
└── Nome do Filme (Ano).mkv
```

Regras: nome limpo, pasta `Season 0X` e o padrão **`SxxEyy`** no arquivo.
Tags **depois** do `SxxEyy` são ignoradas (ex.: `Nome S01E01 [1080p].mkv` funciona).

---

## Variáveis de ambiente (`.env`)

O arquivo `.env` (ignorado pelo git) centraliza configuração e segredos:

```bash
# Configuração comum dos containers
PUID=1000
PGID=1000
TZ=America/Sao_Paulo

# API keys
JELLYFIN_API_KEY=...
SONARR_API_KEY=...
RADARR_API_KEY=...

# Cloudflare Tunnel
TUNNEL_TOKEN=...
```

> Nunca commite o `.env` — ele contém o token do túnel e as API keys.

---

## Links úteis

- Jellyfin — <https://jellyfin.org/> · <https://github.com/jellyfin/jellyfin>
- Jellyseerr — <https://github.com/fallenbagel/jellyseerr>
- Prowlarr — <https://prowlarr.com/>
- Radarr — <https://radarr.video/> · Sonarr — <https://sonarr.tv/>
- Bazarr — <https://www.bazarr.media/>
- qBittorrent — <https://www.qbittorrent.org/>
- Caddy — <https://caddyserver.com/>
- Cloudflare Tunnel — <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>
