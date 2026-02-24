# SpeedTV Backend & IPTV Panel 🚀

SpeedTV é um painel completo e servidor IPTV construído em Node.js. Ele atua como um Proxy HLS transparente, fornecendo canais de TV ao vivo com proteção de IP e token, gerenciamento de usuários, sistema VOD (Filmes e Séries via TMDB) e suporte nativo à API Xtream Codes para uso em aplicativos de terceiros (como IPTV Smarters, XCIPTV, SS IPTV, etc).

## 🌟 Principais Funcionalidades

- **Proxy HLS Transparente:** Os canais ao vivo são retransmitidos via proxy sem o uso intensivo de CPU (não requer FFmpeg para os canais, apenas para a conversão VOD), usando mascaramento de hashes.
- **API Xtream Codes Completa:** Compatibilidade nativa com os principais players do mercado.
- **Web Player Integrado:** Player responsivo com EPG (guia de programação) e categorias, acessível via navegador móvel, smart tv ou desktop.
- **Gerenciamento de VOD Avançado:** Filmes e séries capturados dinamicamente e retransmitidos via FFmpeg.
- **Painel de Usuário:** Tela para trocar foto, nome, senha e baixar a própria lista `.m3u` (`/usuario`).
- **Autenticação Unificada:** Sessões persistentes (`/auth`) integradas com limites de conexões simultâneas e painel web.
- **Painel Administrativo:** Gere usuários, rotas, veja conexões ao vivo, assine e edite permissões em `/admin`.

---

## 🐧 Como Instalar e Rodar no Ubuntu Server

Como o sistema lida com proxy de vídeos, streaming e scrapers headless (Playwright), ele roda perfeitamente no Ubuntu Server, mas exige algumas dependências do sistema operacional.

### 1. Atualizar o sistema e instalar dependências básicas
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl dirmngr apt-transport-https lsb-release ca-certificates -y
```

### 2. Instalar o Node.js (Recomendado v20)
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### 3. Instalar o FFmpeg
O FFmpeg é utilizado **apenas** para a conversão e retransmissão do VOD (Filmes e Séries).
```bash
sudo apt install -y ffmpeg
```

### 4. Clonar e Instalar o Projeto
```bash
# Vá para o diretório web
cd /var/www/
# Clone ou envie os arquivos para uma pasta, exemplo "speedtv"
cd speedtv

# Instalar as dependências do Node
npm install
```

### 5. Instalar as dependências do Playwright (Scraper de Filmes)
O Playwright precisa baixar seus navegadores nativos e as bibliotecas gráficas do Ubuntu para poder rodar os navegadores headless (invisíveis).
```bash
npx playwright install chromium
npx playwright install-deps
```

### 6. Subir o Servidor com PM2 (Recomendado na Produção)
Para garantir que o painel inicie junto com o Ubuntu e caso haja alguma falha ele reinicie sozinho, usaremos o PM2.
```bash
sudo npm install -g pm2
pm2 start server.js --name speedtv
pm2 startup
pm2 save
```

### 7. Configuração Mínima do Nginx (Proxy Reverso para a porta 80/443)
Configure seu Nginx para bater na porta `3000` do SpeedTV.
```nginx
server {
    listen 80;
    server_name speedtv.x44bet.com; # Substitua pelo seu domínio

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Fundamental para manter playlists e streams rodando corretamente
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 📡 Como usar como API (Xtream Codes & IPTV)

Este painel implementou um servidor falso completo da Xtream Codes (`player_api.php`). Portanto, qualquer cliente IPTV padrão funcionará desde que o usuário esteja com o pagamento "Active".

### Parâmetros para os Apps Clientes (Smart TV, TV Box, etc):
- **URL do Servidor / Host:** `http://seu-dominio.com` (Ex: http://speedtv.x44bet.com)
- **Porta:** Pode omitir. Ou `80` se for HTTP, `443` se for HTTPS SSL.
- **Nome de Usuário:** `O usuário criado no admin`
- **Senha:** `A senha do usuário`

**URL M3U Direta:**  
O acesso M3U completo de todos os canais também via Xtream Codes está disponível na clássica URL:
`http://seu-dominio.com/get.php?username=USUARIO&password=SENHA&type=m3u` 
*E também pela nossa nova rota unificada proxy na URL do painel*: `http://seu-dominio.com/api/user/m3u?user=USUARIO&pass=SENHA`

> O sistema Xtream Codes reescreve as rotas e injeta tokens únicos para garantir uso simultâneo regulado pelas conexões configuradas para o usuário no admin.

---

## 📱 Como Fazer um App Android (Web-to-App)

Dado que a interface `/player-web` e toda a experiência visual da plataforma SpeedTV é muito rica e já é responsiva no estilo Netflix/Premium, a forma mais rápida, segura e elegante de criar um app Android é envelopar a página usando um **WebView** (Transformar o Site em um App Nativo).

Aqui estão 3 opções de como fazer isso:

### Método 1: Capacitor ou Apache Cordova (Recomendado & Fácil)
Com o Node instalado, você pode criar uma "casca" android rapidamente:
1. Instale o Capacitor CLI: `npm install -g @capacitor/cli`
2. Crie o app: `npx @capacitor/cli create` (Insira o nome SpeedTV)
3. Na pasta do app, altere o `capacitor.config.json` para definir a URL do seu site no parâmetro `server`. Exemplo:
   ```json
   {
     "appId": "com.speedtv.app",
     "appName": "SpeedTV",
     "webDir": "www",
     "server": {
       "url": "https://speedtv.x44bet.com/auth"
     }
   }
   ```
4. Adicione o Android: `npm install @capacitor/android && npx cap add android`
5. Abra e compile o `.apk` utilizando o Android Studio: `npx cap open android`.

### Método 2: PWABuilder (Apenas Web, sem programar nada)
1. Vá até o site [PWA Builder](https://www.pwabuilder.com/).
2. Cole a URL do seu Player Web (`https://seu-dominio.com`).
3. Clique em **Build** para Android e baixe seu arquivo `AAB` ou `APK` já compilado.
*(Nota: Para uma PWA perfeita, edite o `manifest.json` presente na sua pasta `/public/` do servidor com seus ícones e cores desejadas antes de compilar).*

### Método 3: Android Studio puro (Para ter total controle / Nativo)
Se você sabe o básico do Android Studio, basta utilizar um componente `WebView`:
1. Crie um **"Empty Views Activity"** no Android Studio.
2. No seu arquivo de permissionamento (`AndroidManifest.xml`), libere acesso à internet:
   `<uses-permission android:name="android.permission.INTERNET" />`
3. Troque a tela original por um WebView em `activity_main.xml`.
4. Carregue sua URL e habilite os componentes nativos de Javascript no Java/Kotlin:
   ```java
   WebView myWebView = (WebView) findViewById(R.id.webview);
   WebSettings webSettings = myWebView.getSettings();
   webSettings.setJavaScriptEnabled(true);
   webSettings.setDomStorageEnabled(true);
   webSettings.setMediaPlaybackRequiresUserGesture(false); // Para tocar os canais automaticamente
   myWebView.loadUrl("https://speedtv.x44bet.com/auth");
   ```

A opção nativa de WebView do **Android Studio** ou **Capacitor** lhe garante compatibilidade total de vídeo/HLS sem depender que o usuário tenha um determinado navegador instalado na TV Box / Celular.
