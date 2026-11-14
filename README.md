# Copa da API Finanças — Registro de estudo

**Disciplina:** Programação Web Back-End (Node.js)  
**Data do estudo:** 13/11/2026  
**Atividade:** Copa da API Finanças: roteiro de estudo

## 1. Objetivo

Esta atividade foi realizada para revisar as regras mais importantes da API de Controle Financeiro e praticar a explicação das decisões implementadas no projeto. Os assuntos principais são códigos HTTP, validação, segurança, isolamento de dados por usuário, parcelamento de compras e previsão de faturas.

## 2. Rodada 1 — Códigos de status e regras gerais

### Treino 1

1. **POST /categorias com nome que já existe para o mesmo tipo: 409 Conflict.** É um conflito com um registro que o usuário já possui.
2. **PUT /fontes-destinos/12 com tipo "poupanca": 400 Bad Request.** O tipo informado não está entre os valores permitidos: `renda`, `conta` ou `cartao_credito`.
3. **GET /relatorios/categorias sem `inicio` e `fim`: 400 Bad Request.** Os dois parâmetros são obrigatórios para esse relatório.
4. **POST /auth/login com senha incorreta: 401 Unauthorized.** As credenciais não autorizam o acesso. A mensagem deve ser genérica, sem revelar se o e-mail ou a senha está incorreto.
5. **GET /rota-que-nao-existe: 404 Not Found.** Não existe uma rota correspondente à URL solicitada.
6. **POST /lancamentos com dados válidos: 201 Created.** A requisição criou um novo lançamento.

### Regras que precisam ser lembradas

- O código 200 é usado em consultas, atualizações e login bem-sucedidos.
- O código 204 indica exclusão concluída sem conteúdo no corpo da resposta.
- O código 401 é usado quando o token está ausente, inválido ou expirado, ou quando as credenciais de login estão incorretas.
- O código 404 também é usado quando o registro pertence a outro usuário. Não se usa 403 nesse caso, para não confirmar a existência de um registro que não pertence ao solicitante.
- O identificador do usuário deve vir do token autenticado, por meio de `req.auth.id`, e não do corpo, da URL ou da query string.
- E-mails devem ser gravados em minúsculas e sem espaços, e CPFs devem ser armazenados somente com números.
- A senha deve ser protegida com `bcrypt.hash` e verificada com `bcrypt.compare`. O campo `senha_hash` nunca deve aparecer na resposta.
- A chave do JWT deve ficar em uma variável de ambiente, como `JWT_SECRET`, e não diretamente no código. O token é assinado com `jsonwebtoken` e as rotas privadas são protegidas por middleware de autenticação.

## 3. Rodada 2 — Caça ao bug

### Treino 2

O trecho apresentado possui três problemas:

1. O SQL é montado com template string e recebe diretamente `req.params.id`. Isso permite que um dado da requisição seja inserido no comando SQL. A consulta deve usar parâmetros com `?`.
2. A consulta SQL está dentro de `routes/`, mas a organização do projeto determina que os comandos SQL fiquem em `models/`.
3. Depois de enviar a resposta 404, a função continua e tenta enviar uma resposta 204. Isso pode causar o erro `ERR_HTTP_HEADERS_SENT`, pois a mesma requisição recebe duas respostas.

Uma forma de corrigir é deixar a consulta no model e fazer a rota encerrar a execução após o erro:

```javascript
// models/FonteDestino.js
static async excluir(id, usuarioId) {
  const [resultado] = await db.query(
    'DELETE FROM fontes_destinos WHERE id = ? AND usuario_id = ?',
    [id, usuarioId]
  );

  return resultado.affectedRows > 0;
}
```

```javascript
// routes/fontesDestinos.js
router.delete('/:id', async (req, res) => {
  const excluiu = await FonteDestino.excluir(req.params.id, req.auth.id);

  if (!excluiu) {
    return res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
  }

  return res.status(204).end();
});
```

Além disso, antes da exclusão, o projeto precisa verificar se existem lançamentos vinculados. Se houver vínculos que impeçam a operação, a API deve responder com `409 Conflict`.

## 4. Rodada 3 — Parcelas e faturas

### Treino 3 — Divisão de R$ 250,00 em 7 parcelas

O cálculo deve ser feito em centavos inteiros para evitar diferenças causadas por números de ponto flutuante.

- Valor total: R$ 250,00 = 25.000 centavos.
- Divisão inteira: 25.000 ÷ 7 = 3.571 centavos por parcela, com resto de 3 centavos.
- Os 3 centavos restantes são adicionados à primeira parcela.

| Parcela | Valor |
|---|---:|
| 1/7 | R$ 35,74 |
| 2/7 | R$ 35,71 |
| 3/7 | R$ 35,71 |
| 4/7 | R$ 35,71 |
| 5/7 | R$ 35,71 |
| 6/7 | R$ 35,71 |
| 7/7 | R$ 35,71 |
| **Total** | **R$ 250,00** |

A soma fecha exatamente o valor da compra: R$ 35,74 + (6 × R$ 35,71) = R$ 250,00.

Exemplo de implementação:

```javascript
function dividirEmParcelas(valor, quantidade) {
  const totalCentavos = Math.round(valor * 100);
  const base = Math.floor(totalCentavos / quantidade);
  const resto = totalCentavos - base * quantidade;

  return Array.from({ length: quantidade }, (_, indice) =>
    (indice === 0 ? base + resto : base) / 100
  );
}

module.exports = { dividirEmParcelas };
```

### Treino 4 — Fechamento no dia 18, vencimento no dia 3

A compra foi de R$ 250,00 em 2 parcelas, realizada em 18/08/2026. Como a compra ocorreu exatamente no dia do fechamento, a primeira parcela entra na fatura do mês seguinte.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---|---:|---|---|---|
| 1/2 | R$ 125,00 | 18/09/2026 | 03/10/2026 | 2026-10 |
| 2/2 | R$ 125,00 | 18/10/2026 | 03/11/2026 | 2026-11 |

O vencimento ocorre no mês seguinte ao fechamento porque o dia 3 é menor que o dia 18.

### Treino 5 — Fechamento no dia 31, vencimento no dia 8

A compra foi de R$ 120,00 em 2 parcelas, realizada em 31/01/2026. Como a compra foi feita no próprio dia de fechamento, a primeira parcela vai para o fechamento seguinte. Fevereiro de 2026 não possui dia 31, então o fechamento é ajustado para o último dia do mês.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---|---:|---|---|---|
| 1/2 | R$ 60,00 | 28/02/2026 | 08/03/2026 | 2026-03 |
| 2/2 | R$ 60,00 | 31/03/2026 | 08/04/2026 | 2026-04 |

A referência da fatura corresponde ao mês do vencimento, no formato `AAAA-MM`.

### Regra do limite do cartão

O limite utilizado corresponde à soma das parcelas cujas faturas ainda não venceram. O limite disponível é calculado subtraindo esse total do limite total do cartão.

## 5. Rodada 4 — Aposta final

### Treino 6 — Por que usar UNIQUE no banco?

A validação feita somente com um `SELECT` antes do `INSERT` não impede duplicidades em situações simultâneas. Duas requisições podem consultar o banco ao mesmo tempo, ambas não encontrar o e-mail e, em seguida, tentar cadastrar o mesmo valor.

A restrição `UNIQUE` no banco resolve esse problema porque o próprio banco garante que o valor não será repetido, mesmo com requisições concorrentes. Quando ocorre uma duplicidade, o MySQL gera o erro `ER_DUP_ENTRY`, que deve ser tratado pela API e convertido em `409 Conflict`.

### Outros conceitos importantes

- **Transações:** o cadastro que envolve várias inserções deve utilizar `beginTransaction`, `commit` e `rollback`. Se uma etapa falhar, o `rollback` desfaz as alterações da transação e evita deixar o cadastro pela metade. Com pool de conexões, a transação deve usar uma conexão obtida por `pool.getConnection()`.
- **Relatórios:** totais e agrupamentos podem ser calculados no SQL com `SUM` e `GROUP BY`. Como valores retornados pelo `mysql2` podem chegar como texto, é importante convertê-los com `Number()` antes de realizar operações numéricas na aplicação.
- **Organização:** as rotas recebem e validam as requisições; os models concentram as consultas SQL; as consultas devem ser parametrizadas; e os módulos do projeto seguem CommonJS.

## 6. Proposta de carta-bug

**Arquivo:** `models/Lancamento.js`  
**Regra violada:** isolamento dos dados por usuário (RF3).

### Trecho com erro

O método abaixo busca o lançamento somente pelo ID, sem conferir se ele pertence ao usuário autenticado:

```javascript
static async buscarPorId(id) {
  const [linhas] = await db.query(
    'SELECT id, valor, descricao FROM lancamentos WHERE id = ?',
    [id]
  );
  return linhas[0] || null;
}
```

**Erro:** a consulta não filtra pelo campo `usuario_id`. Um usuário poderia tentar consultar um lançamento de outra pessoa caso descobrisse ou informasse o ID.

### Correção

```javascript
static async buscarPorId(id, usuarioId) {
  const [linhas] = await db.query(
    'SELECT id, valor, descricao FROM lancamentos WHERE id = ? AND usuario_id = ?',
    [id, usuarioId]
  );
  return linhas[0] || null;
}
```

A rota deve passar o ID do usuário obtido do token:

```javascript
const lancamento = await Lancamento.buscarPorId(req.params.id, req.auth.id);

if (!lancamento) {
  return res.status(404).json({ erro: 'Lançamento não encontrado.' });
}
```

Assim, a consulta retorna somente um lançamento que pertence ao usuário autenticado. Quando o registro não existe ou pertence a outra pessoa, a resposta é 404.

## 7. Conclusão

Com os exercícios, revisei que uma API não deve apenas retornar os dados esperados: ela também precisa respeitar os códigos HTTP, validar as entradas, proteger informações, limitar o acesso aos registros de cada usuário e manter a consistência dos valores financeiros.

Os pontos que exigem mais atenção no projeto são o uso correto de `req.auth.id`, as consultas parametrizadas, o tratamento de erros, a restrição `UNIQUE` no banco e o cálculo das parcelas e faturas em centavos e datas.

## Referências

- [Copa da API Finanças — roteiro de estudo](https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html)
- [Projeto 2 — API de Controle Financeiro](https://github.com/JoaoLorenzetti/api-controle-financeiro)
