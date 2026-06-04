# Jellyfin

Agora, para gerenciar o seu homelab centralizado, basta entrar na pasta do projeto e usar os comandos nativos do Compose:

- Subir tudo em background: docker compose up -d
- Parar todos os serviços: docker compose down
- Ver os logs unificados: docker compose logs -f

Para ajudar no planejamento de como expandir essa estrutura ou customizar caminhos e variáveis de ambiente adicionais, utilize o simulador de arquitetura abaixo para visualizar e gerar o mapeamento estático correto antes de aplicar no terminal.

## URLs

- Site oficial: <https://jellyfin.org/>
- Repositório: <https://github.com/jellyfin/jellyfin>

    /home/v1cferr/Projects/Local/jellyfin/
    ├── docker-compose.yml
    └── config/
        ├── jellyfin/
        └── qbittorrent/
