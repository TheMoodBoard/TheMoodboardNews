# Painel de publicação — The Moodboard

Agora as edições **04 em diante** e os destaques da home (capa + grade "Últimas
Edições") são gerados automaticamente a partir do arquivo **`edicoes.json`**.
Ninguém precisa mexer no `index.html` para publicar uma edição nova.

Há duas formas de publicar:

1. **Pelo painel visual** em `/admin/` (recomendado — preenche um formulário e
   faz upload do PDF). Precisa de uma configuração inicial única (abaixo).
2. **Editando o `edicoes.json`** direto no GitHub (funciona na hora, sem
   configuração).

---

## Como o site usa o `edicoes.json`

- A edição com o **maior número** vira automaticamente a **capa da home** e a
  primeira da grade "Últimas Edições".
- As edições **01, 02 e 03** continuam escritas à mão no `index.html` (elas têm
  leitores de matéria antigos) — deixe `"estatico": true` nelas e não as apague.
- Toda edição nova deve ter `"estatico": false` (ou nem incluir o campo).
- O PDF pode ser um **arquivo no repositório** (campo `pdf`) **ou** um **link do
  Google Drive** (campo `pdfLink`). Use o Drive para não pesar o repositório com
  PDFs grandes.

### Campos de cada edição

| Campo       | Obrigatório | Exemplo                          |
|-------------|-------------|----------------------------------|
| `num`       | sim         | `"07"` (sempre 2 dígitos)        |
| `titulo`    | sim         | `"Título da edição"`             |
| `label`     | sim         | `"Sétima edição"`                |
| `data`      | sim         | `"Outubro 2026"`                 |
| `dataCurta` | sim         | `"Out 2026"`                     |
| `cor`       | sim         | `"#2874A6"` (hex)                |
| `tags`      | sim         | `"Moda · Cultura"`               |
| `resumo`    | sim         | frase curta                      |
| `descricao` | não         | texto do card do PDF             |
| `materias`  | sim         | `4`                              |
| `tempo`     | sim         | `"~25 min"`                      |
| `pdf`       | não*        | `"uploads/edicao-07.pdf"`        |
| `pdfLink`   | não*        | link do Google Drive             |
| `estatico`  | —           | `false` para edições novas       |
| `home`      | não         | destaque na home (ver JSON)      |

\* Informe **um** dos dois: `pdf` **ou** `pdfLink`.

---

## Opção 2 (sem configuração): publicar editando o JSON no GitHub

1. Suba o PDF para o Google Drive e deixe-o **"qualquer pessoa com o link"**.
   Copie o link (algo como `https://drive.google.com/file/d/XXXX/view`).
2. No GitHub, abra `edicoes.json` → botão de editar (lápis).
3. Copie o primeiro bloco `{ ... }` e cole no topo da lista, ajustando os campos.
   Preencha `pdfLink` com o link do Drive (deixe `pdf` como `""`).
4. "Commit changes" na branch `main`. Em ~1 minuto o site atualiza sozinho.

---

## Opção 1: ligar o painel visual em `/admin/`

O GitHub Pages é um site estático (sem servidor), então o login com GitHub
precisa de um pequeno **proxy OAuth** gratuito. Passo único:

### A) Criar um GitHub OAuth App
1. GitHub → **Settings** da organização **TheMoodBoard** → **Developer settings**
   → **OAuth Apps** → **New OAuth App**.
2. Preencha:
   - **Application name:** `The Moodboard CMS`
   - **Homepage URL:** a URL do site (ex.: `https://themoodboard.github.io/TheMoodboardNews/`)
   - **Authorization callback URL:** `https://<seu-proxy>.workers.dev/callback`
     (você terá essa URL no passo B — pode voltar e ajustar depois).
3. Guarde o **Client ID** e gere um **Client Secret**.

### B) Subir o proxy OAuth (Cloudflare Workers, grátis)
Use o template pronto da comunidade (busque por **"decap-cms cloudflare oauth
worker"**, ex.: `sterlingwes/decap-proxy` ou `i40west/netlify-cms-oauth`):
1. Crie conta em cloudflare.com → **Workers & Pages** → **Create Worker** →
   dê o nome `moodboard-oauth` → **Deploy** → **Edit code**.
2. Apague o código de exemplo e cole **exatamente** este:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname === "/auth") {
      const gh = new URL("https://github.com/login/oauth/authorize");
      gh.searchParams.set("client_id", env.GITHUB_CLIENT_ID);
      gh.searchParams.set("redirect_uri", `${url.origin}/callback`);
      gh.searchParams.set("scope", "repo,user");
      gh.searchParams.set("state", crypto.randomUUID());
      return Response.redirect(gh.toString(), 302);
    }

    if (url.pathname === "/callback") {
      const code = url.searchParams.get("code");
      const res = await fetch("https://github.com/login/oauth/access_token", {
        method: "POST",
        headers: { "Content-Type": "application/json", Accept: "application/json" },
        body: JSON.stringify({
          client_id: env.GITHUB_CLIENT_ID,
          client_secret: env.GITHUB_CLIENT_SECRET,
          code,
        }),
      });
      const data = await res.json();
      const status = data.access_token ? "success" : "error";
      const content = data.access_token
        ? { token: data.access_token, provider: "github" }
        : { error: data.error || "no token" };
      const msg = `authorization:github:${status}:${JSON.stringify(content)}`;
      const html = `<!doctype html><meta charset="utf-8"><body><script>
        (function () {
          function receive(e){
            if (window.opener) window.opener.postMessage(${JSON.stringify(msg)}, e.origin);
            window.removeEventListener('message', receive, false);
          }
          window.addEventListener('message', receive, false);
          if (window.opener) window.opener.postMessage('authorizing:github', '*');
        })();
      </script>Pode fechar esta janela.</body>`;
      return new Response(html, { headers: { "Content-Type": "text/html; charset=utf-8" } });
    }

    return new Response("Moodboard OAuth OK", { status: 200 });
  },
};
```

3. **Deploy**. Depois em **Settings → Variables and Secrets** adicione dois
   **Secrets**:
   - `GITHUB_CLIENT_ID` = Client ID do passo A
   - `GITHUB_CLIENT_SECRET` = Client Secret do passo A
   (Deploy de novo após salvar as variáveis.)
4. A URL do worker será algo como `https://moodboard-oauth.SEU-USER.workers.dev`.
   Volte ao passo **A.2** e confirme o **callback** como
   `https://moodboard-oauth.SEU-USER.workers.dev/callback`.

### C) Apontar o painel para o proxy
Em `admin/config.yml`, troque a linha `base_url:` pela URL do seu worker:
```yaml
base_url: https://moodboard-oauth.SEU-USER.workers.dev
```
Faça commit. Pronto: acesse `https://<seu-site>/admin/`, clique em
**"Login with GitHub"** e publique.

### Quem pode publicar
Qualquer pessoa que seja **colaboradora do repositório** `TheMoodBoard/
TheMoodboardNews` com permissão de escrita. Adicione/remova pessoas em
**Settings → Collaborators** do repositório. Sem acesso de escrita, o login
funciona mas a publicação é bloqueada pelo GitHub.

---

## Testar o painel localmente (opcional)
Sem precisar do proxy:
```bash
npx decap-server
```
Descomente `local_backend: true` no `admin/config.yml`, sirva a pasta
(`py -m http.server`) e acesse `http://localhost:8000/admin/`.
Comente a linha de novo antes de publicar.
