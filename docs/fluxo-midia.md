# Fluxo de mídia (baixar e organizar)

Como adicionar mídia para que organização, metadados, legenda e remoção do torrent
aconteçam sozinhos. Versão rápida também no dash: <https://dash.v1cferr.dev>

## Passo a passo

1. Cadastre o título ANTES no *arr:
   - Série/anime: Sonarr (<https://sonarr.v1cferr.dev>). Anime: marque Series Type = Anime.
   - Filme: Radarr (<https://radarr.v1cferr.dev>).
2. Baixe o torrent no qBittorrent do Docker (<https://torrent.v1cferr.dev>) com a Categoria:
   - `Series` (séries e anime) ou `Filmes`.
3. Não recorte/cole os arquivos. Deixe onde o qBittorrent salvou.
4. Pronto. Sonarr/Radarr importam (hardlink) para `media/`, o Jellyfin pega os metadados,
   e o torrent é removido automaticamente (não seedamos).

## Regras que fazem funcionar

- O título precisa estar cadastrado no *arr, senão o download fica parado sem importar.
- A categoria do torrent precisa ser `Series` ou `Filmes`.
- Para escolher a release na mão sem risco: use o Interactive Search dentro do Sonarr/Radarr
  (o Nyaa.si já está configurado para anime).
- Legenda pt-BR: o Bazarr baixa sozinho se o arquivo não tiver embutida (perfil pt-BR já criado).
  Muitos rips já trazem pt-BR embutida, é só selecionar no player do Jellyfin.

## Convenção de nomes (só se for organizar na mão)

```text
media/tv/Nome da Série (Ano)/Season 01/Nome da Série S01E01.mkv
media/movies/Nome do Filme (Ano)/Nome do Filme (Ano).mkv
```

Nome limpo (sem tag de fansub no início), pasta `Season 0X` e o padrão `SxxEyy` no arquivo.
