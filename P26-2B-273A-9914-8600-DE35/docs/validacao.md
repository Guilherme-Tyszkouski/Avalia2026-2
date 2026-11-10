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
