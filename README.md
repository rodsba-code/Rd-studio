# RD Studio

PWA de gerenciamento do projeto Restored Desire e geração de imagens no PixAI.
Tudo roda no navegador: um único `index.html`, sem build e sem dependência de CDN
de CSS. Os dois únicos scripts externos são os do Google (login e API do Drive),
e o app continua funcionando se eles não carregarem.

## Publicar no GitHub Pages

1. Crie um repositório e suba **o conteúdo desta pasta na raiz** (não a pasta inteira):
   `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`,
   `icon-512-maskable.png`, `.nojekyll`.
2. Settings → Pages → Source: `Deploy from a branch`, branch `main`, pasta `/ (root)`.
3. A URL fica como `https://SEU_USUARIO.github.io/NOME_DO_REPO/`.

Ao publicar uma alteração no `index.html`, suba o número da versão em `sw.js`
(`const CACHE = 'rd-studio-v2'`), senão o service worker segue servindo a versão antiga.

## Google Cloud (login e leitura do Drive)

1. Ative a **Google Drive API** no projeto.
2. Crie uma credencial **OAuth client ID** do tipo *Web application*.
3. Em **Authorized JavaScript origins**, adicione:
   - `https://SEU_USUARIO.github.io`
   - `http://localhost:8000` (para testes locais)
4. Se a tela de consentimento estiver em modo de teste, cadastre sua conta como usuário de teste.
5. O escopo usado é só `drive.readonly`. O token fica na memória da aba, nunca no `localStorage`.

O login não funciona abrindo o arquivo direto (`file://`). Para testar local:
`python3 -m http.server 8000`.

## Configurações do app (aba Config)

A última aba da barra inferior tem os quatro campos, salvos no `localStorage`
deste aparelho (o ícone de engrenagem no topo leva para a mesma aba):

| Chave | O que é |
|---|---|
| `gas_url` | URL do Apps Script publicado (termina em `/exec`) |
| `gas_secret` | Senha que o proxy exige |
| `gcp_client_id` | Client ID do Google Cloud |
| `tracker_file_id` | ID do arquivo Progress_Tracker no Drive |

Progress_Tracker_ver27.txt: `1t5JtcVdvu4L2Ye544H3JhuwN9aWUHyyg`
(cada nova versão é um arquivo novo, com ID novo).

## Contrato do proxy (Google Apps Script)

Publique como app da web, **executando como você** e com acesso para **qualquer pessoa**.
O app envia POST com o corpo em `text/plain` (evita o preflight de CORS):

```json
{ "secret": "...", "action": "create", "payload": {
    "prompt": "...", "negative_prompt": "...", "aspectRatio": "2:3",
    "mode": "standard", "modelVersionId": "...", "batchSize": 1, "seed": 42 } }
```

```json
{ "secret": "...", "action": "check", "taskId": "..." }
```

O `doPost` deve ler `e.postData.contents`, conferir o `secret` e responder JSON
via `ContentService`. Campos vazios (`negative_prompt`, `seed`) não são enviados.

O que o proxy precisa traduzir para a API do PixAI:

- `negative_prompt` → `negativePrompt` (a API usa camelCase)
- `POST https://api.pixai.art/v2/image/create` com `Authorization: Bearer <token>`
- `GET https://api.pixai.art/v1/task/{id}` para o status
- se a tarefa concluída trouxer `outputs.mediaIds`, baixe cada imagem em
  `GET https://api.pixai.art/v1/media/{mediaId}/image` (com o token) e devolva em
  `outputs.mediaUrls`, como link https ou `data:image/...;base64`

O app aceita `taskId` ou `id`, na raiz ou dentro de `task`/`data`, e trata os
status `waiting`, `running` e `completed`.

## Modelos do PixAI

| Modelo | modelVersionId |
|---|---|
| Tsubaki.2 | 1983308862240288769 |
| Haruka v2 | 1861558740588989558 |
| Hoshino v2 | 1954632828118619567 |

Modos: fast, standard, quality, ultra. Lote: 1 ou 4.

## Onde mexer no código

Tudo está no `index.html`, em blocos comentados:

- `BASE` e `LOOKS`: descrições base das personagens e os looks do Estúdio
- `MODELS`, `MODES`, `RATIOS`: opções do PixAI
- `DAYS`, `CAST`, `IDEAS_SEED`: dados de exemplo das abas Capítulos, Elenco e Ideias
- `lerTrackerReal()`: leitura do Progress_Tracker no Drive
