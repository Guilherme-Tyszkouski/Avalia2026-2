# Registro de Validação — P26-2B-273A-9914-8600-DE35

Evidências produzidas pelo agente de teste e validação. Cada entrada diz o que
foi executado, o que contaria como falha, e o que de fato aconteceu.

Um teste que não pode falhar não prova nada — por isso o critério de falha está
escrito antes do resultado, e não depois.

---

## V1 — Restrição de voto único no banco

**Data:** 10/11/2026
**Agente:** `qa-validacao`
**Harness:** Claude (sessão com acesso ao sistema de arquivos da máquina)
**Ambiente:** Cloudflare D1 local (`wrangler d1 execute DB --local`)

### Por que este teste existe

O enunciado exige que cada estudante vote **apenas uma vez** em cada
publicação. A pergunta não é se a interface esconde o botão depois do voto — é
se o sistema recusa o segundo voto quando alguém chama o banco ou o endpoint
diretamente, por fora da tela.

### Preparação

```
INSERT INTO discord_users (user_id, username, avatar) VALUES ('111', 'teste', NULL);
INSERT INTO publicacoes (autor_id, tipo, titulo, conteudo, categoria)
  VALUES ('111', 'snippet', 'Teste', 'console.log(1)', 'JavaScript');
INSERT INTO votos (publicacao_id, usuario_id) VALUES (1, '111');
```

Resultado: `3 commands executed successfully.` — o primeiro voto é aceito, como
deve ser.

### O teste

```
INSERT INTO votos (publicacao_id, usuario_id) VALUES (1, '111');
```

**O que contaria como falha:** o comando ser aceito. Se o segundo voto entrasse,
a restrição não estaria valendo e o requisito não estaria cumprido, mesmo que a
interface escondesse o botão.

### Resultado observado

```
ERROR  UNIQUE constraint failed: votos.publicacao_id, votos.usuario_id:
SQLITE_CONSTRAINT (extended: SQLITE_CONSTRAINT_UNIQUE)
```

**Situação: aprovado.** A recusa vem do banco, na camada mais baixa, e portanto
vale independentemente de qualquer erro nas camadas acima.

Evidência em imagem: [`validacao/v1-voto-unico.png`](./validacao/v1-voto-unico.png)

### O que este teste NÃO cobre

- Não testa o endpoint HTTP de voto: isso é o V-futuro, depois que
  `functions/api/votos` existir. O endpoint precisa capturar este erro e
  responder 409, em vez de deixar vazar um 500.
- Não testa o banco remoto, só o local. A migration ainda não foi aplicada em
  produção.

---

## V2 — Deploy online e conexão com o D1 de produção

**Data:** 10/11/2026
**Agente:** `qa-validacao`
**Harness:** Claude (navegador integrado + sistema de arquivos)
**URL:** https://p26-2b-273a-9914-8600-de35.pages.dev

### Por que este teste existe

O item 1 da rubrica é eliminatório: sem a aplicação no ar e funcionando, os
outros 75 pontos não contam. Além disso, o `validar-local.mjs` busca a URL do
README e confere se a chave aparece no HTML que voltou — ou seja, a verificação
é automática e não aceita a palavra do aluno.

### O que foi verificado

**Chave no rodapé.** Lido do HTML servido pelo site publicado, não da
aparência da tela:

```
Chave Única de Auditoria: P26-2B-273A-9914-8600-DE35 · Turma 2B · Licenciado sob MIT
```

**O que contaria como falha:** a chave estar apenas no arquivo local e não na
resposta do servidor.

**Banco em produção.** `GET /api/status` respondeu:

```json
{"ok":true,"chave":"P26-2B-273A-9914-8600-DE35","projectId":19,
 "d1Connected":true,"discordOAuthConfigured":false,"totalRegistros":0}
```

**O que contaria como falha:** `d1Connected: false`, que indicaria binding
ausente — o erro clássico de aplicar as migrations só no banco local e publicar
sem as tabelas.

**Situação: aprovado** nos dois pontos.

### Configuração registrada

- Projeto Pages: `p26-2b-273a-9914-8600-de35`
- Production branch: `main`
- Root directory: `P26-2B-273A-9914-8600-DE35`
- Build output directory: `public`
- Build command: vazio
- Banco D1 `db-p26-2b-273a-9914-8600-de35`, binding `DB`, migrations 0001 e
  0002 aplicadas em `--local` e `--remote`

### O que este teste NÃO cobre

- `discordOAuthConfigured: false` — o login do Discord ainda não existe. O
  perfil "Discord: Guilherme (discord_de35)" que aparece na tela é o placeholder
  do scaffold, não uma sessão real.
- Nenhuma funcionalidade do Projeto #19 foi implementada ainda: o que está no ar
  é o painel de diagnóstico gerado pelo `iniciar.mjs`.

---

## V3 — Login com Discord OAuth2 de ponta a ponta

**Data:** 10/11/2026
**Agente:** `auth-discord` implementou; `qa-validacao` verificou
**Harness:** Claude (sistema de arquivos + navegador integrado para configurar
o Discord Developer Portal); execução do servidor local pelo Guilherme
**Ambiente:** `wrangler pages dev` em `http://localhost:8788`

### Por que este teste existe

O item 2 da rubrica exige login **exclusivo** via Discord OAuth2 importando
`user_id`, nome e avatar. Chegar na tela inicial depois de autorizar não prova
nada: o redirect pode acontecer sem que a sessão tenha sido criada. A prova é
o servidor reconhecer a pessoa numa requisição seguinte.

### O teste

`GET /api/auth/eu` após completar o fluxo de autorização.

**O que contaria como falha:** `{"autenticado": false}`, ou ausência de
`avatarUrl` — que indicaria que o hash do avatar não foi convertido em URL.

### Resultado observado

```json
{
  "autenticado": true,
  "userId": "6990363395421xxxxx",
  "nome": "Guilherme Tyszkouski",
  "avatarUrl": "https://cdn.discordapp.com/avatars/6990363395421xxxxx/cd741ffc...png?size=128"
}
```

(identificador parcialmente omitido: o repositório é público)

**Situação: aprovado.** Uma única resposta prova cinco etapas: validação do
`state`, troca do `code` por `access_token`, segunda chamada a
`GET /users/@me`, gravação em `discord_users` e leitura do cookie assinado.

### Decisões de segurança verificadas no código

- `state` aleatório gerado no início e conferido no retorno, contra CSRF de
  login. A documentação do Discord o trata como opcional e recomendado.
- Troca do código feita servidor-a-servidor; o Client Secret nunca chega ao
  navegador.
- Corpo da troca em `application/x-www-form-urlencoded` — a documentação do
  Discord rejeita JSON nesse endpoint.
- Escopo `identify` apenas. `email` não foi solicitado: o projeto não precisa,
  e cada escopo a mais é dado pessoal sob nossa responsabilidade.
- Cookie `HttpOnly`, `SameSite=Lax`, `Secure` em HTTPS, com expiração explícita.
- Chave do usuário é o `id` do Discord, não o nome: `global_name` e `username`
  mudam quando a pessoa troca o apelido, e a autoria dos resumos se perderia.
- `/api/auth/sair` é POST, não GET: com GET, uma tag `<img>` em qualquer site
  deslogaria quem visitasse a página.

### O que este teste NÃO cobre

- Produção: os segredos ainda não foram enviados ao Cloudflare Pages, então o
  login só funciona em `localhost`. `GET /api/status` em produção continua
  reportando `discordOAuthConfigured: false`.
- Expiração da sessão após sete dias não foi testada.
- A interface ainda mostra o perfil fictício do scaffold; ela não consome
  `/api/auth/eu` ainda.
