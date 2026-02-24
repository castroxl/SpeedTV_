# 📚 SpeedTV API Documentation

Este documento detalha todos os endpoints disponíveis na API do SpeedTV para integração com aplicativos móveis, Smart TVs e sistemas externos.

---

## 📺 1. API Xtream Codes (Padrão IPTV Universal)

Esta é a API recomendada para criar ou integrar aplicativos nativos (Smarters, XCIPTV, etc). O SpeedTV implementa totalmente os endpoints essenciais.

**URL Base:** `http://speedtv.x44bet.com`

### Autenticação & Informações do Servidor
```http
GET /player_api.php?username={usuario}&password={senha}
```
**Resposta:** Informações da conta, limites de conexões simultâneas locais e status de validade, junto às informações do fuso e domínios do servidor web.

### Obter Categorias de TV Ao Vivo
```http
GET /player_api.php?username={usuario}&password={senha}&action=get_live_categories
```
**Resposta:** Array com nomes e IDs de categorias de canais.

### Obter Canais Ao Vivo (Streams)
```http
GET /player_api.php?username={usuario}&password={senha}&action=get_live_streams
```
**Resposta:** Array de canais com IDs, categorias mapeadas e logos.

### Reproduzir um Canal de TV (M3U8)
```http
GET /{usuario}/{senha}/{stream_id}.m3u8
```
> **Nota de Segurança:** Esta rota de stream é convertida internamente pelo backend num Proxy HLS exclusivo de sessão atrelado ao IP, invalidando tentativas de interceptar segmentos externos!

---

## 🔐 2. Autenticação & Sessões Web (Frontend API)

Para aplicativos feitos com requisições Web ou PWAs, você pode preferir gerenciar usuários programaticamente e buscar streams baseados em Token (mais seguro que expor senha no M3U).

### Efetuar Login e Obter Permissões
```http
GET /api/user/info?user={usuario}&pass={senha}
```
**Resposta (200 OK):**
```json
{
  "user": "joao123",
  "name": "João da Silva",
  "plan": "premium",
  "avatar": "joao123.jpg",
  "maxConnections": 3,
  "filmes": true,
  "subscription": {
    "status": "active",
    "expiresAt": "2026-12-31T23:59:59Z"
  }
}
```

### Resetar Senha / Recuperar via WhatsApp
```http
POST /api/user/forgot-password
Content-Type: application/json

{
  "user": "joao123"
}
```
**Resposta:** JSON com `whatsappUrl` para redirecionamento.

### Gerar arquivo `.m3u` completo na hora (Download Direto)
```http
GET /api/user/m3u?user={usuario}&pass={senha}
```

---

##  🎬 3. Dados Dinâmicos VOD (Filmes, Séries, EPG)

### Listar Filmes
```http
GET /api/filmes?page=1&limit=50&categoria=Acao&busca=Matrix
```
*Autenticação Web (`stw_user`/`stw_pass` via query) ou Sessão de Origem necessária.*

### Detalhar / Reproduzir um Filme
```http
GET /api/filme/:id
```
**Resposta:** Retorna metadados e URL para HLS mascarado (`stream_directo` ou `m3u8` gerado pelo FFmpeg).

### Listar Jogos (Futebol/Eventos) ao Vivo
```http
GET /api/jogos
```
**Resposta:** Retorna eventos ao vivo com os IDs dos canais atrelados (ex: "Premiere 1", "ESPN").

### Baixar Guia de Programação (EPG)
```http
GET /api/epgs
```

---

## ⚙️ 4. Painel de Controle (Admins)

Rotas de uso exclusivo para usuários cujo log retorne `plan: "admin"`.

- **Criar/Editar Vários Usuários:**
  `POST /api/register`
- **Excluir Usuário:**
  `DELETE /adm/users/EXCLUIR_USER`
- **Listar Conexões Simutâneas Ao Vivo:**
  `GET /adm/connections`
- **Atualizar e Forçar Scrapper de Filmes/Canais:**
  `POST /adm/force-filmes-refresh` / `POST /adm/force-weekly-scrapper`

---

### Exemplo Básico de App Android: Fazendo Login (Kotlin/Java HTTP)
Para entrar na plataforma direto por seu app, envie uma requisição ao `/player_api.php`. Se o "status" retornado for `"Active"`, está logado! Armazene o usuário/senha no `SharedPreferences` do dispositivo Android. Na hora em que o usuário clicar no Canal (ID=102), basta passar a URL pro reprodutor EXOPlayer do Android:
`http://speedtv.x44bet.com/joao/123senha/102.m3u8`

---

## 🧭 5. Páginas Existentes no Sistema

O sistema conta com rotas otimizadas para front-end nativo, web ou apps PWA (WebView):

| Rota / URL | Descrição | Nível de Acesso |
|---|---|---|
| `/` | Landing page pública, apresentação do plano e modalidades | Público |
| `/auth` | Tela de Login e Esqueceu a senha (WhatsApp) | Público |
| `/usuario` (ou `/iptv`) | Painel do Usuário (Foto, Nome, Senha, Info M3U e Assinatura) | Logado |
| `/player-web` | Web Player Oficial (Netflix-like) com VOD e Canais Ao Vivo | Logado |
| `/admin` | Painel de Controle para gerenciar Usuários e Streams | **Admin** |
| `/admin-oct` | Tela de login escondida para acesso ao Painel Admin | Público |

---

## 🎓 Tutoriais Práticos Avançados

### Tutorial: Como usar a API de Canais em um App Customizado

Para buscar e abrir um canal ao vivo sem usar o padrão estrito do Xtream Codes, você deve se comunicar com o backend via requisições REST autenticadas (`GET`), e depois passar a stream gerada ao seu player final:

**1. Buscando a lista de canais disponíveis**
Faça uma requisição para a rota `GET /api/channels?user=joao&pass=123` e você receberá um objeto contendo a matriz de `.channels` (todos os canais extraídos com imagem e ID) e `.categories`.

**2. Listando e Filtrando no Android/Web**
Renderize essa lista num layout Grid ou RecyclerView. Cada item da matriz contém um campo `id` (ex: `"102"`).

**3. Gerando a Sessão de Proxy para tocar**
Quando o cliente clicar no canal do Grid, NÃO toque a URL imediatamente. Ao invés disso, chame a API de resolução de stream do SpeedTV:
```http
GET /api/channel/102?user=joao&pass=123
```

O servidor SpeedTV fará uma de três coisas:
- Barrar o acesso se o plano venceu ou se o limite de conexões ativas for ultrapassado (429 Too Many Requests).
- Bloquear caso o canal inexista (404 Not Found).
- Gerar na hora um IP-Locked Token com duração de 8 horas e retornar a HLS URL segura.

**Resposta de Sucesso:**
```json
{
  "m3u8": "http://speedtv.x44bet.com/joao/TOKEN_GIGANTE_GERADO/nome_do_canal.m3u8",
  "stream": "http://speedtv.x44bet.com/joao/TOKEN_GIGANTE_GERADO/nome_do_canal.m3u8"
}
```

Nesta etapa, instrua seu reprodutor (ExoPlayer, HLS.js, VLC) a dar Play no conteúdo de `"m3u8"`. Tudo já está traduzido pelo proxy no lado do servidor. O Web Player Oficial faz esse exato fluxo (vide a tela `/player-web`).

---

### Tutorial: Como usar a API de Filmes (VOD) em um App

O VOD difere dos canais, pois as URLs bases são recuperadas dinamicamente e retransmitidas nativamente via pacote FFmpeg. Seu App também recebe URLs amigáveis.

**1. Listando os Filmes Atuais e Buscando Categorias**
Os dados são pré-cacheados em memória no SpeedTV por causa do TMDB. Para listar:
```http
// Paginar e Buscar:
GET /api/filmes?user=joao&pass=123&page=1&limit=20
```
Você pode passar campos `categoria` e `busca` de forma independente. O retorno json entrega array de objetos com Poster, Descrição, ID do TMDB, Duração e Título de cada filme.

**2. Retornar dados para Reprodução**
Assim que o usuário clicar no filme (Ex: ID `8232` no TMDB do filme), você invoca:
```http
GET /api/filme/8232?user=joao&pass=123
```

O processo aqui é um pouco diferente: o SpeedTV vai conectar no Scraper Headless (Playwright), achar a stream do filme sem anúncios, invocar um spawn do FFmpeg caso seja HLS, ou rotear a MP4 se for servidor direto. E vai te retornar a resposta:

**Resposta de Sucesso:**
```json
{
  "success": true,
  "streamUrl": "http://speedtv.x44bet.com/stream/filme/TOKEN_EXCLUSIVO/filme.m3u8",
  "streamType": "hls", 
  "movie": { "title": "Avatar 2", "overview": "Um filme legal..." }
}
```

Novamente, injete essa `streamUrl` no Player nativo.

> **Importante para Apps Nativos**: Diferente dos Canais (que matam o proxy com 5min inativos), o FFmpeg de Filmes tem um timeout global maior. Caso o Player Nativo pare (stop), certifique-se de chamar a rota silenciosa `POST /api/stream/close` passando `token` no body JSON, para não inflar a CPU do servidor com os transcoders zumbis de filmes pausados/fechados! O player-web implementa esse evento de "close" ao fechar sobreposição de vídeo.
