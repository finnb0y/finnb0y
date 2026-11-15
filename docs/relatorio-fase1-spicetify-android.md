# Spicetify para Android — Relatório Técnico da Fase 1 (Pesquisa e Viabilidade)

**Data:** 15/11/2026 · **Status:** rascunho para aprovação · **Código escrito:** nenhum

---

## 0. Como ler este relatório (nível de evidência)

Segui sua regra de diferenciar fatos de hipóteses. Cada afirmação relevante leva uma etiqueta:

| Etiqueta | Significado |
|---|---|
| **[C]** Confirmado | Li a fonte nesta sessão (link na seção 11). |
| **[P]** Conhecimento prévio | Fato que conheço de documentação oficial anterior, mas **não consegui reler nesta sessão**. Precisa de nova verificação antes de virar decisão. |
| **[H]** Hipótese | Inferência minha, sem evidência direta. |

**Limitação importante desta pesquisa:** o ambiente em nuvem desta sessão bloqueou o acesso a `developer.spotify.com`, `spicetify.app` e `support.google.com` (erro de DNS do proxy de rede). Consegui ler o GitHub (Spotify SDK, Spicetify, Marketplace, projetos semelhantes) e `developer.android.com`. Por isso, **as regras da Web API e os Termos do Spotify estão marcados como [P]**. A seção 10 lista exatamente o que precisa ser reverificado.

---

## 1. Como o Spicetify funciona no Desktop

### 1.1 Fatos confirmados
- O Spicetify é uma CLI escrita em **Go** (com um helper em Node/pnpm), licença LGPL-2.1, para **Windows, macOS e Linux**. Não há suporte declarado a mobile. **[C]**
- Ele modifica o **cliente oficial** já instalado e oferece quatro coisas: mudar cores, injetar CSS, injetar **Extensions** (JS que "manipula a UI e controla o player") e injetar **Custom Apps** (páginas inteiras novas). **[C]**
- Ele expõe um objeto global `Spicetify` (tipado em `globals.d.ts`) com, entre outros: `Player` (eventos `songchange`, `onplaypause`, `onprogress`; `play`, `seek`, `setVolume`…), `Platform` (descrito como "um vasto conjunto de APIs internas", tipado como `any`), `React`/`ReactDOM` do próprio Spotify, `CosmosAsync` (requisições a serviços internos do cliente), `ContextMenu`, `Menu`, `PopupModal`, `Keyboard`, `LocalStorage`, `URI`, `Topbar`, `Playbar`. **[C]**
- Última versão: **v2.46.7 (12/11/2026)**, compatível com Spotify Desktop 1.2.14 → 1.3.9. Cada atualização do Spotify pode quebrar o Spicetify, exigindo nova versão e reaplicação. **[C]**
- **Marketplace** (custom app): descobre temas/extensões buscando na API do GitHub repositórios com o *topic* `spicetify-extensions` e lendo um `manifest.json` na raiz; snippets CSS vêm de um `snippets.json`; o estado instalado fica em `localStorage`. Avisa que o conteúdo da comunidade **não é validado** pelo Spicetify. **[C]** (wiki de 2022, pode estar desatualizada)
- O repositório `spicetify/spicetify-extensions` foi **arquivado em 06/05/2024** e é só um índice de temas, não de extensões. **[C]**

### 1.2 Mecanismo de base (por que funciona no Desktop)
- O Spotify Desktop é um aplicativo web (HTML/CSS/JS + React) empacotado num runtime Chromium (**CEF**). A interface fica num pacote `xpui.spa` (um zip). O Spicetify faz backup, descompacta, aplica *patches* em JS/CSS, injeta seus scripts e reempacota. **[P]** (não consegui reler a documentação do spicetify.app; o resto é coerente com os fatos [C] acima: React exposto, CSS injetável, `globals.d.ts`)
- **Conceito-chave:** o Spicetify funciona porque a UI do Desktop é **código web interpretado no cliente**, que pode ser alterado e no qual dá para injetar código no mesmo contexto de execução. É isso que dá acesso ao React interno, aos módulos webpack e às APIs internas (`Platform`, `Cosmos`).

### 1.3 Por que isso não se transfere para o Android
- O app Android do Spotify é um APK nativo (DEX/Kotlin/Java + bibliotecas `.so`), assinado pelo Spotify. **[P]**
- Modificar o APK exige descompilar, alterar e **reassinar com outra chave**. Isso quebra atualizações pela Play Store e a integridade do app, e é exatamente o modelo de projetos como xManager (que distribuía APKs modificados "Ad-Free" e foi **arquivado em 06/04/2026**). **[C]** quanto ao xManager; **[P]** quanto aos detalhes de assinatura.
- Não existe no Android um mecanismo suportado para injetar código em outro app (sandbox por UID, sem root). **[P]**
- **Conclusão:** um "Spicetify que modifica o Spotify Android" não é viável de forma sustentável nem legítima. O equivalente viável é um **app próprio** que **reimplementa a camada de UI** e usa o Spotify só pelos canais oficiais.

---

## 2. O que dá para reproduzir no Android

| Recurso do Spicetify | No Android (app próprio) | Viabilidade |
|---|---|---|
| Temas / cores / CSS | Motor de temas próprio (tokens → Compose) | **Alta**: controle total, porque a UI é nossa |
| Fontes e ícones | Fontes carregadas de arquivo; ícones como path SVG → `ImageVector` | **Alta** |
| Alterar layout | Layouts declarativos (árvore de *slots* com componentes de uma lista permitida) | **Média/Alta** |
| Extensions (JS) | Plugins JS isolados com API por permissões | **Média**: exige projetar a API com cuidado |
| Custom Apps | Plugins que adicionam "telas" declarativas | **Média** |
| Marketplace | Catálogo via GitHub (topic + manifest), como o do Spicetify | **Alta** (tecnicamente simples) |
| Hooks no React/`Platform` internos do Spotify | **Não reproduzível.** Só temos o que o App Remote e a Web API expõem | — |
| Modificar a UI do app Spotify | **Não** (e fora do escopo) | — |

---

## 3. Integração oficial com Spotify e suas limitações

### 3.1 Spotify Android SDK
O SDK tem duas bibliotecas independentes **[C]**:

**a) App Remote** (versão atual **v0.8.0**, release de 04/07; o ano não aparece na página) **[C]**
- Ainda marcado como **Beta**. **[C]**
- **O app oficial do Spotify precisa estar instalado**, e é preciso conexão com a internet para conectar. **[C]**
- APIs: `PlayerApi` (tocar URI, pausar/retomar, pular, shuffle, assinar `PlayerState`/`PlayerContext`), `UserApi` (capacidades do usuário, adicionar/remover da biblioteca), `ImagesApi` (capas), `ContentApi` (listas de conteúdo), `ConnectApi` (escolher dispositivo). **[C]**
- **Tocar uma faixa específica exige Premium**; o SDK manda consultar a capacidade *on-demand* via `UserApi` antes. **[C]**
- A autenticação embutida do App Remote só pede o escopo `app-remote-control`, sem outros escopos. Para Web API é preciso a biblioteca de Auth. **[C]**
- **Implicação central:** o áudio é tocado **pelo app do Spotify**, não pelo nosso. Nós somos um **controle remoto com UI própria**. Não recebemos o stream de áudio. **[C]** (pelo design da API) / **[P]** (ausência de qualquer API de stream)

**b) Auth Library** (`spotify/android-auth`) **[C]**
- Usa **PKCE** quando o app do Spotify instalado suporta ou no fallback via navegador, e pode devolver *refresh token* quando PKCE é usado (changelog 2.2.0). **[C]**
- Desde a 5.0.0 o app precisa declarar no manifesto o `intent-filter` do `RedirectUriReceiverActivity` (com `autoVerify` para redirect https). minSdk 21. **[C]**
- Há divergência de versão: o release do android-sdk cita Auth 2.1.0, e o changelog do android-auth vai até 5.0.0. É preciso conferir no Maven Central. **[C]** (a divergência em si)

### 3.2 OAuth 2.0 + PKCE (conceito)
- PKCE: o app gera um `code_verifier` aleatório, envia o hash (`code_challenge`) na autorização e depois troca o `code` pelo token provando que tem o verifier. Isso dispensa guardar *client secret* no APK, o que é obrigatório para apps móveis, já que qualquer segredo no APK pode ser extraído. **[P]**
- O Spotify descontinuou o *Implicit Grant* e passou a não aceitar redirect URIs HTTP não-loopback (prazo de migração por volta de 27/11/2025). **[P]**

### 3.3 Web API: limitações que afetam o projeto
- **27/11/2024:** apps **novos** perderam acesso a: Recommendations, Related Artists, Audio Features, Audio Analysis, Featured Playlists, Category Playlists, `preview_url` (prévias de 30 s) e playlists algorítmicas/editoriais do Spotify. **[P]**
- **Modo de desenvolvimento:** o app só funciona para usuários cadastrados manualmente no dashboard (historicamente até 25; há relatos de regras mais restritas em 2026, **verificar**). **[P]**
- **Extended Quota (desde 15/05/2025):** só para **organizações** (pessoa jurídica) com serviço lançado e cerca de **250 mil MAU**. Indivíduos não conseguem. **[P]**
- Endpoints de **controle de reprodução** da Web API (`/me/player/*`) exigem **Premium**. **[P]**
- **Web Playback SDK:** só navegadores com EME/Widevine; não serve de player embutido em app Android. **[P]**
- **Termos do Desenvolvedor:** proíbem engenharia reversa da plataforma/clientes do Spotify, remover, alterar ou bloquear anúncios e conteúdos, e oferecer produto que substitua o serviço. **[P]** (texto exato a reler)

### 3.4 O que isso significa na prática
1. **O projeto, com Spotify, é na prática um app pessoal ou de grupo pequeno** (modo de desenvolvimento). Distribuição pública com integração Spotify para muita gente **não é alcançável** por um desenvolvedor individual. **[P→H]**
2. Recursos de "descoberta" (recomendações, artistas relacionados, audio features) **não estão disponíveis** para apps novos. Não vamos planejar nada em cima deles.
3. Com conta **Free**, a integração é limitada (sem tocar faixa específica; possivelmente só shuffle de contexto). O MVP precisa degradar de forma elegante.

---

## 4. Publicidade: o que posso e o que não posso fazer

Aqui preciso ser direto, porque parte do item 2.C entra em conflito com a sua própria regra da seção 3 ("sem modificar mecanismos de publicidade ou controles de acesso de terceiros") e com os Termos do Spotify.

**O que não vou fazer:** engenharia reversa dos componentes de anúncio do app Spotify Android, mapear como eles decidem e exibem anúncios para contorná-los, ou implementar bloqueio dos anúncios do Spotify. Isso é contornar o modelo de receita de um terceiro (e o que levou projetos como xManager a viverem de APKs modificados). Também não é necessário para o produto que você descreveu.

**O que é fato e é útil:**
- **No nosso app não existe anúncio para bloquear.** Arquivos locais tocam sem nenhum anúncio. Quando o Spotify é a fonte via App Remote, o áudio (e os anúncios de contas Free) é tocado **pelo app Spotify**. Nossa UI apenas reflete o `PlayerState`. **[C]** (arquitetura do App Remote)
- O caminho legítimo para o usuário sem anúncios no Spotify é o **Premium**, que também é o que destrava o controle completo pelo SDK. **[C]** (requisito do SDK)
- Em nível conceitual e público: anúncios de streaming normalmente são **decididos no servidor** (segmentação, frequência) e o cliente só reproduz o que recebe. **[H]**, não vou aprofundar além disso.

**Alternativas que mantêm o espírito de "filtragem" no nosso app** (todas viáveis):
- **Filtros de conteúdo próprios:** ocultar faixas explícitas, podcasts, artistas ou gêneros na **nossa** biblioteca e nas nossas listas.
- **Regras de fila** em plugins: "pular automaticamente faixas que casem com X" usando `PlayerApi.skipNext()` **em músicas**, respeitando as restrições que o próprio `PlayerState` informa (se o Spotify não permitir pular, não pulamos).
- **Zero telemetria e zero anúncios** como princípio declarado do nosso app.

---

## 5. Engenharia reversa: como aproveitar as ferramentas

JADX (DEX → Java), Apktool (recursos/manifest/smali), MobSF (análise estática/dinâmica de segurança), ADB e Android Studio (APK Analyzer, Profiler) são ótimas para aprender **arquitetura de APKs**. Proposta legítima e com o mesmo valor didático:
1. Analisar **os nossos próprios builds** (ver o que o R8 faz, tamanho de DEX, manifest mesclado, permissões, bibliotecas nativas).
2. Analisar **APKs de projetos open source** (Auxio, Gramophone), comparando o APK com o código-fonte. É o melhor jeito de aprender a ler saída do JADX, porque você tem a resposta certa ao lado.
3. Usar **MobSF no nosso app** como parte do processo de segurança (especialmente do sistema de plugins).

Não recomendo aplicar essas ferramentas ao APK do Spotify: os Termos do Desenvolvedor proíbem engenharia reversa **[P]**, e uma violação pode resultar em bloqueio do *client ID*, o que mataria a integração oficial da qual o projeto depende.

---

## 6. Como funcionaria o app independente

```mermaid
flowchart LR
  subgraph Nosso App
    UI[UI Compose\n(temas + layouts)] --> VM[ViewModels]
    VM --> PB{PlaybackBackend}
    PB --> LOCAL[LocalBackend\nMedia3/ExoPlayer]
    PB --> SPOT[SpotifyBackend\nApp Remote]
    VM --> LIB[(Room: biblioteca,\nplaylists, histórico)]
    VM --> WAPI[Spotify Web API client]
    PLG[Plugin Host] -->|API com permissões| VM
  end
  SPOT <-->|IPC do SDK| SAPP[App Spotify oficial\n(toca o áudio)]
  WAPI <-->|HTTPS + token PKCE| SWEB[api.spotify.com]
  LOCAL --> MS[MediaStore\n(arquivos locais)]
```

- **Arquivos locais:** `MediaStore` (permissão `READ_MEDIA_AUDIO`) → indexação no Room → reprodução por **Media3 ExoPlayer** dentro de um `MediaSessionService`/`MediaLibraryService` (notificação, tela de bloqueio, Bluetooth, Android Auto). **[C]** (Media3)
- **Spotify:** biblioteca/playlists do usuário via Web API (leitura) + reprodução e estado via App Remote (o app Spotify toca).
- **Abstração `PlaybackBackend`:** a UI nunca sabe "quem" toca. Isso isola o maior risco do projeto (mudanças do Spotify) num único módulo.
- **Histórico:** registrado localmente por nós a partir dos eventos dos dois backends. Não depende do endpoint de "recently played".
- **Playlists:** locais (nossas, no Room) e Spotify (lidas pela Web API; edição usa endpoints de playlist, **verificar escopos**). Uma playlist local pode misturar fontes. **[H]** de UX a validar: misturar fontes numa fila exige trocar de backend entre faixas.

---

## 7. Tecnologias: avaliação e recomendação

| Área | Opções avaliadas | Recomendação e motivo |
|---|---|---|
| Linguagem | Kotlin, Java, Flutter/Dart, React Native | **Kotlin.** Os SDKs do Spotify e o Media3 são nativos Android; Flutter/RN adicionariam uma ponte para cada SDK. |
| UI | Jetpack Compose, Views/XML | **Compose.** Decisivo para temas: no sistema de Views os temas são recursos **compilados** no APK e difíceis de trocar em runtime; no Compose o tema é **estado** (`CompositionLocal`) e um tema carregado de JSON muda a UI inteira na hora. |
| Reprodução local | Media3, MediaPlayer, libVLC | **Media3 (ExoPlayer + MediaSession).** Oficial, integra notificação/Auto/Bluetooth, usado por Auxio e Gramophone **[C]**. libVLC só se precisarmos de codecs exóticos. |
| Banco | Room, SQLDelight, Realm | **Room.** Dados relacionais (faixas × playlists × histórico), Flow, migrações. SQLDelight seria a escolha se quiséssemos Kotlin Multiplatform, o que não é objetivo. |
| Preferências | DataStore, SharedPreferences | **DataStore (Preferences)** para configurações e tema ativo. Os temas em si ficam como **arquivos** (pacotes), não no DataStore. |
| HTTP / JSON | OkHttp + kotlinx.serialization, Retrofit, Ktor | **OkHttp + kotlinx.serialization**, com cliente fino para a Web API. Menos dependências; Retrofit pode entrar depois se o número de endpoints crescer. |
| Imagens | Coil, Glide | **Coil** (feito para Compose). Única dependência "de conveniência" justificável no MVP. |
| DI | Hilt, Koin, manual | **Manual (AppContainer) no MVP.** Poucas dependências; migrar para Hilt se o grafo crescer. |
| Plugins (script) | ver 8.3 | **`androidx.javascriptengine` (JavaScriptSandbox)** **[P]**, a validar num protótipo. |
| minSdk | — | **26** (proposta; ajustar depois de validar os requisitos do JavaScriptSandbox). |

---

## 8. Proposta de arquitetura

### 8.1 Módulos Gradle

```
:app                         -> composição, navegação, AppContainer
:core:model                  -> entidades puras (Track, Album, Playlist, PlayerState)
:core:data                   -> Room, DataStore, repositórios
:core:designsystem           -> componentes base que leem tokens do tema
:theme:engine                -> parser/validador de pacotes de tema, ThemeProvider
:playback:api                -> interface PlaybackBackend + PlaybackController
:playback:local              -> Media3 (ExoPlayer, MediaLibraryService)
:playback:spotify            -> App Remote (isolado; único módulo com o SDK)
:integration:spotify-auth    -> Auth Library + armazenamento seguro de tokens
:integration:spotify-webapi  -> cliente HTTP fino e mapeamento para :core:model
:plugin:api                  -> contratos estáveis: manifesto, permissões, pontos de extensão
:plugin:host                 -> runtime JS isolado + ponte de mensagens
:plugin:manager              -> instalar/atualizar/remover, verificação de integridade
:feature:library | :feature:player | :feature:settings | :feature:extensions
```
**Regra de dependência:** `feature` → `core`/`playback:api`/`plugin:api`; nunca `feature` → `playback:spotify` diretamente. Assim o Spotify pode ser removido ou substituído sem tocar na UI.

### 8.2 Motor de temas

**Pacote de tema** (`.zip`): `theme.json` + `fonts/` + `images/`.
```
theme.json
  schemaVersion, id, name, author, version
  colors.light / colors.dark  -> mapeiam para ColorScheme + tokens extras (ex.: playerBackground)
  typography                  -> família (arquivo em fonts/), tamanhos, pesos
  shapes, spacing, elevation
  icons                       -> nome lógico -> path SVG (convertido em ImageVector)
  components                  -> overrides por componente (ex.: "trackRow.height", "playButton.style")
  layouts (opcional)          -> árvore de slots por tela
```
- **Conceito:** *design tokens*. A UI nunca usa cores fixas; todo componente lê tokens. O tema é só dados (sem código), então é seguro de compartilhar.
- **Layouts:** cada tela define *slots* (ex.: `nowPlaying = [artwork, title, progress, controls, lyricsSlot]`). O tema pode reordenar, ocultar ou trocar variantes **de componentes registrados**. É o equivalente seguro do "CSS que muda layout".
- **Robustez:** validação de schema, versionamento, fallback por token (valor ausente → padrão), reversão automática ao tema padrão se o tema falhar ao carregar.
- Fontes de arquivo e `ImageVector` montado a partir de path data são suportados pelo Compose **[P]**. Não há suporte a carregar `VectorDrawable` XML em runtime, por isso o formato SVG-path. **[P]**

### 8.3 Sistema de plugins

**Alternativas avaliadas:**

| Opção | Prós | Contras | Veredito |
|---|---|---|---|
| Código Kotlin/DEX baixado (`DexClassLoader`) | Poder total | Acesso total ao processo; viola a política da Play contra baixar código executável **[P]** | **Rejeitada** |
| **Zipline** (QuickJS, Cash App) | Kotlin/JS, usado pelo **Spotube** para plugins **[C]** | O próprio README diz que **não oferece sandbox e não deve executar código não confiável** **[C]** | Só para plugins confiáveis |
| **`androidx.javascriptengine`** | JS em **processo isolado** (V8 do WebView), sem acesso a APIs Android **[P]** | Depende da versão do WebView; comunicação assíncrona por mensagens | **Recomendada** (validar em protótipo) |
| WebAssembly (Chicory) | Isolamento forte, multi-linguagem | O README não declara suporte a Android **[C]** | Acompanhar |
| Só declarativo (JSON) | Seguro e simples | Sem lógica | **Camada 1** |

**Desenho em duas camadas:**
1. **Camada declarativa (MVP):** temas, layouts e "snippets" (pequenos overrides de tema). Só dados.
2. **Camada de script (fase posterior):** plugins JS que **não desenham UI arbitrária**. Eles contribuem em **pontos de extensão** e por meio de uma API por **permissões**:
   - Eventos: `onTrackChange`, `onPlayPause`, `onQueueChange`.
   - Ações: `player.skip()`, `queue.add()`, `library.query()` (cada uma com permissão própria).
   - Contribuições de UI **declarativas**: item de menu de contexto, ação no player, painel feito de componentes registrados.
   - Armazenamento próprio isolado por plugin.

**Manifesto de plugin:** `id`, `version`, `apiVersion` (compatibilidade), `permissions[]`, `entry`, `contributes{}` e hash/assinatura. **Ciclo de vida:** instalar (mostrar permissões) → habilitar → atualizar (nova permissão exige novo consentimento) → desabilitar/remover (apaga os dados do plugin).

**Distribuição pela comunidade:** igual ao Marketplace do Spicetify **[C]**: repositórios GitHub com um *topic* e um `manifest.json`, mais um índice curado opcional. Atualização por comparação de `version` + verificação de hash.

### 8.4 Comunicação com serviços externos
- Tokens: Auth Library (PKCE) → access/refresh token guardados criptografados (Keystore). Refresh centralizado num interceptor do OkHttp.
- Web API: cache local no Room (playlists/álbuns) para reduzir chamadas e respeitar *rate limits*; tratamento de 429 com `Retry-After`. **[P]**
- App Remote: conexão gerida pelo `SpotifyBackend` com reconexão; estado exposto como `Flow<PlayerState>` normalizado para `:core:model`.
- Plugins **nunca** recebem tokens; pedem dados ao host, que decide pelas permissões.

```mermaid
flowchart TB
  P[Plugin JS\n(processo isolado)] -- mensagens JSON --> B[Bridge no Plugin Host]
  B -- checa permissões --> API[Plugin API v1]
  API --> PC[PlaybackController]
  API --> REPO[Repositórios]
  API --> UIX[Registro de contribuições de UI]
```

---

## 9. MVP realista (v0.1)

**Objetivo:** provar a arquitetura de ponta a ponta, não ter muitos recursos.

**Dentro do escopo**
1. **Player local:** varredura do MediaStore → Room; tocar/pausar/pular/seek; fila; Media3 com notificação.
2. **Spotify (oficial):** login PKCE; listar playlists e músicas salvas do usuário (Web API, só leitura); conectar via App Remote; "Tocando agora" com capa, controles e degradação para conta Free.
3. **Motor de temas v1:** `theme.json` com cores, tipografia e formas; 3 temas embutidos; importar tema de arquivo; troca instantânea; fallback seguro.
4. **Layout v1:** só a tela "Tocando agora" com slots reordenáveis/ocultáveis pelo tema.
5. **Histórico local** de reprodução (das duas fontes).
6. **Plugins (só arquitetura):** módulo `:plugin:api` com manifesto e modelo de permissões; 1–2 recursos internos (ex.: timer de sono, filtro de conteúdo) escritos **contra essa API**, para validar o contrato antes de abrir para terceiros.

**Fora do MVP (próximas fases):** runtime JS para terceiros, marketplace, editor visual de temas, pacotes de ícones, letras, playlists de fontes mistas, widgets.

**Critério de pronto:** trocar de tema sem reiniciar; tocar arquivo local e faixa Spotify pela mesma tela; remover o módulo `:playback:spotify` e o app continuar compilando e funcionando com arquivos locais.

---

## 10. Principais dificuldades e riscos

| # | Risco | Impacto | Mitigação |
|---|---|---|---|
| 1 | Spotify muda ou restringe a API (como em 2024–2026) | Alto | Spotify isolado num módulo; núcleo local independente |
| 2 | Modo de desenvolvimento limita usuários; Extended Quota inacessível para indivíduos | Alto (distribuição) | Assumir uso pessoal ou de grupo pequeno com Spotify; publicação ampla só da parte local |
| 3 | App Remote ainda é Beta e depende do app Spotify instalado | Médio | Tratar desconexões; feature flags |
| 4 | Recursos limitados para conta Free | Médio | Ler capacidades via `UserApi` e adaptar a UI |
| 5 | Segurança de plugins da comunidade | Alto | Processo isolado, permissões, sem tokens para plugins, revisão no índice |
| 6 | Política da Play sobre código baixado | Médio | Só JS em sandbox; nada de DEX/.so dinâmico |
| 7 | Compatibilidade de temas entre versões | Médio | `schemaVersion`, migrações, fallback por token |
| 8 | Escopo grande para uma pessoa | Alto | MVP pequeno; fases com critério de pronto |
| 9 | Violação de Termos (engenharia reversa/anúncios) | Alto (perda do client ID) | Fora do escopo por decisão (seção 4–5) |

**Pendências de verificação** (bloqueadas pela rede desta sessão):
- Limite atual de usuários do modo de desenvolvimento e regras de 2026 (developer.spotify.com/documentation/web-api/concepts/quota-modes e o blog).
- Lista exata de endpoints restritos desde 27/11/2024.
- Texto dos Termos do Desenvolvedor (engenharia reversa, anúncios, "substituto do serviço").
- Requisito de Premium nos endpoints `/me/player`.
- Política da Play "Device and Network Abuse" (exceção para código interpretado).
- Requisitos e minSdk do `androidx.javascriptengine`.

---

## 11. Referências

**Lidas nesta sessão [C]**
- Spicetify CLI: https://github.com/spicetify/cli · `globals.d.ts`: https://raw.githubusercontent.com/spicetify/cli/main/globals.d.ts · release: https://github.com/spicetify/cli/releases/latest
- Spicetify Marketplace: https://github.com/spicetify/marketplace · wiki: https://github.com/spicetify/marketplace/wiki
- spicetify-extensions (arquivado): https://github.com/spicetify/spicetify-extensions
- Spotify Android SDK: https://github.com/spotify/android-sdk · App Remote README: https://github.com/spotify/android-sdk/blob/master/app-remote-lib/README.md · releases: https://github.com/spotify/android-sdk/releases
- Spotify Android Auth: https://github.com/spotify/android-auth · changelog: https://github.com/spotify/android-auth/blob/master/CHANGELOG.md
- AndroidX Media3: https://developer.android.com/media/media3 · UAMP: https://github.com/android/uamp
- Zipline: https://github.com/cashapp/zipline · Chicory: https://github.com/dylibso/chicory
- Auxio (Kotlin, Media3, GPL-3.0): https://github.com/OxygenCobalt/Auxio
- Gramophone (Kotlin, Media3, Material 3): https://github.com/FoedusProgramme/Gramophone
- Spotube (Compose Multiplatform, plugins via Zipline, AGPL-3.0): https://github.com/team-spotube/spotube
- xManager (arquivado em 06/04/2026; caso de estudo do que **não** fazer): https://github.com/Team-xManager/xManager

**A reverificar (inacessíveis nesta sessão) [P]**
- https://developer.spotify.com/documentation/android
- https://developer.spotify.com/documentation/web-api
- https://developer.spotify.com/documentation/web-api/tutorials/code-pkce-flow
- https://developer.spotify.com/documentation/web-api/concepts/quota-modes
- https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api
- https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access
- https://developer.spotify.com/terms
- https://developer.spotify.com/documentation/web-playback-sdk
- https://spicetify.app/docs
- https://support.google.com/googleplay/android-developer/answer/9888379
- https://developer.android.com/jetpack/androidx/releases/javascriptengine
- Ferramentas: https://github.com/skylot/jadx · https://apktool.org/ · https://github.com/MobSF/Mobile-Security-Framework-MobSF

**Projetos para estudar, por objetivo:** temas e extensões → Spicetify + Marketplace; player local → Auxio, Gramophone, UAMP; sistema de plugins → Spotube (Zipline) + `androidx.javascriptengine`.

---

## 12. Decisões que preciso de você antes da Fase 5
1. **Onde o projeto vai morar:** este repositório (`finnb0y/finnb0y`) é o README do seu perfil no GitHub. Sugiro criar um repositório novo (ex.: `spicedroid`).
2. **Escopo do Spotify:** você aceita que, com Spotify, o app fique limitado a você e a alguns usuários cadastrados (modo de desenvolvimento)?
3. **Conta Premium** disponível para desenvolvimento e testes do App Remote?
4. **Publicidade:** você concorda em substituir o item "bloqueio de anúncios do Spotify" pelos filtros de conteúdo do nosso app (seção 4)?
5. **Liberar acesso de rede** a `developer.spotify.com`, `spicetify.app` e `support.google.com` para eu fechar as pendências [P].

*Aguardando aprovação. Nenhuma implementação foi iniciada.*
