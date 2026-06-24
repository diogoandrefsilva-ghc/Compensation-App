# Compensation-App (App Remuneração) — guia para o assistente

App pessoal mobile para consultar/gerir a evolução salarial.
**Sem build, sem npm.** Site estático (GitHub Pages), PWA. Dados **locais** (`localStorage`) com import/export; lê um modelo de referência via `raw.githubusercontent`.

## Estrutura
- `index.html` — **ficheiro único** (~1521 linhas): markup + CSS + JS tudo dentro. (Ainda NÃO dividido como o SplitBill/FestasBV.)
- `sw.js` — service worker (cache PWA).
- `MODELO.md` — documento do modelo de dados (referência; não é código).
- Não mexer: `apple-touch-icon.png`, `manifest.json`.

## Como NÃO gastar tokens à toa
- Lê só o troço relevante do `index.html`, não o ficheiro todo. Para localizar um botão/campo, procura o `id` no markup e salta para o handler no `<script>`.
- Faz **edições cirúrgicas** (diffs pequenos). **Nunca reescrevas o ficheiro inteiro.**
- Se isto crescer, vale a pena dividir em `index.html` + `app.js` + `style.css` (como já fiz no SplitBill e FestasBV).

## Regras técnicas (não partir a app)
- O JS está inline e há handlers `onclick="…"` → as funções têm de ser **globais**. Se algum dia extraíres para `app.js`, carrega-o como `<script src>` normal (NÃO module).
- **PWA/cache:** se alterares o HTML/CSS/JS, **sobe a versão do CACHE no `sw.js`**.
- Sem backend nem chaves: os dados vivem no `localStorage` do dispositivo (backup por import/export de JSON).

## Deploy
GitHub Pages a partir de `main`. Um push para `main` publica.
