# 🎬 Jellyfin Home Media Stack

Minha "Netflix local": um conjunto de serviços em Docker que baixa, organiza, legenda e transmite filmes e séries — tudo self-hosted em casa.

A stack sobe **automaticamente junto com o PC** e fica rodando em segundo plano. Você só precisa de comando manual quando quiser parar, atualizar ou ver logs.

---

## 📦 Serviços

| Serviço          | Função                                              | Acesso (local)                  |
| ---------------- | --------------------------------------------------- | ------------------------------- |
| **Jellyfin**     | Servidor de mídia (streaming dos filmes/séries)     | <http://localhost:8096>         |
| **Jellyseerr**   | Interface para pedir filmes/séries novos            | <http://localhost:5055>         |
| **Prowlarr**     | Gerenciador de indexadores (trackers)               | <http://localhost:9696>         |
| **Radarr**       | Automação de filmes                                 | <http://localhost:7878>         |
| **Sonarr**       | Automação de séries                                 | <http://localhost:8989>         |
| **Bazarr**       | Download automático de legendas                     | <http://localhost:6767>         |
| **qBittorrent**  | Cliente de download (torrent)                       | <http://localhost:8080>         |
| **FlareSolverr** | Resolve desafios Cloudflare para os indexadores     | (sem interface)                 |
| **NPM**          | Nginx Proxy Manager — proxy reverso + HTTPS         | <http://localhost:81> (painel)  |
| **Cloudflared**  | Túnel Cloudflare para acesso externo seguro         | (sem interface)                 |

> 💡 Para acessar de **outro dispositivo na mesma rede** (celular, TV, notebook), troque `localhost` pelo IP desta máquina: **`192.168.1.10`**.
> Exemplo: <http://192.168.1.10:8096>

---

## 🚀 Início automático no boot

Já está tudo configurado, **não precisa fazer nada**:

- O serviço do Docker está habilitado para iniciar com o sistema (`systemctl enable docker`).
- Cada container usa a política `restart: unless-stopped`, então:
  - 🔁 Ao **ligar/reiniciar o PC**, todos os serviços voltam sozinhos.
  - ✋ Se você **parar manualmente** um serviço, ele continua parado (não volta no boot) até você subir de novo.

---

## 🕹️ Comandos do dia a dia

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

> ⚠️ **`stop` x `down`:** use `stop` para pausar (mantém o auto-start no boot).
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
docker compose logs --tail=100 npm  # Últimas 100 linhas do NPM
```

### Atualizar as imagens

```bash
docker compose pull        # Baixa as versões mais recentes
docker compose up -d       # Recria os containers que mudaram
docker image prune -f      # (opcional) Limpa imagens antigas
```

---

## 📁 Estrutura do projeto

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
    ├── npm/
    └── letsencrypt/        # Certificados do Nginx Proxy Manager
```

- **Mídia:** todos os serviços compartilham `/home/v1cferr/Videos/Jellyfin` como biblioteca.
- **`config/`** guarda bancos de dados, configs e cache — fica fora do git (`.gitignore`).

---

## 🔐 Variáveis de ambiente (`.env`)

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

## 🔗 Links úteis

- Jellyfin — <https://jellyfin.org/> · <https://github.com/jellyfin/jellyfin>
- Jellyseerr — <https://github.com/fallenbagel/jellyseerr>
- Prowlarr — <https://prowlarr.com/>
- Radarr — <https://radarr.video/> · Sonarr — <https://sonarr.tv/>
- Bazarr — <https://www.bazarr.media/>
- qBittorrent — <https://www.qbittorrent.org/>
- Nginx Proxy Manager — <https://nginxproxymanager.com/>
- Cloudflare Tunnel — <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>
