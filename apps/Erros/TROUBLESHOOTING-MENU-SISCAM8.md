# 🔴 SISCAM8 — Troubleshooting: Erro 500 ao carregar Menu no App Mobile

> **⚠️ ESTE DOCUMENTO SE APLICA AOS APLICATIVOS MOBILE DO SISCAM8**
> (Apps React Native/Expo das Câmaras Municipais — tenant ou PLD multi-cidade)

**Caso real resolvido:** Câmara Municipal de Santa Rita do Passa Quatro (cliente 562) — Julho/2026

---

## 1. Sintoma

Ao abrir o aplicativo **SISCAM8**, o menu não carrega e o console exibe:

```
Falha ao carregar menu: {
  "ExceptionMessage": "Object reference not set to an instance of an object.",
  "ExceptionType": "System.NullReferenceException",
  "Message": "Ocorreu um erro.",
  "StackTrace": "at lambda_method(Closure, Object[])
    at NHibernate.Loader.Hql.QueryLoader.GetResultList(...)
    at NHibernate.Loader.Hql.QueryLoader.List(...)
    ..."
}
```

- Acontece **apenas em um cliente específico**; os demais apps funcionam normalmente.
- O erro aparece na chamada `getMenuClient` (arquivo `app/(auth)/index.tsx` → `src/functions/getMenuClient.ts`).

---

## 2. Entendendo a arquitetura (ESSENCIAL para não errar o alvo)

O erro **NÃO é do aplicativo** e **NÃO é do servidor da cidade**. O fluxo do menu é:

```
App SISCAM8
   │
   │  POST /api/Mobile/ListarMenus  (body: idCliente)
   ▼
SERVIDOR DE LICENÇAS (central, compartilhado por todos os clientes)
https://licenca.sinoinformatica.com.br
   │
   │  lê as tabelas do BANCO DE LICENÇAS
   ▼
Banco de Licenças: cliente / configuracao / licenca / menu / sistema / tipocliente / usuario
```

O servidor da cidade (`<cidade>.servicos.siscam.com.br`) só é usado **depois**, para
login, vereadores, sessões etc. **O menu nunca passa pelo banco da cidade.**

> ❌ **Erro comum:** inserir configurações de menu (`Mobile:Menu:...`) na tabela
> `configuracao` do banco **da cidade**. Isso não tem nenhum efeito sobre este erro.

- Stack trace com `NHibernate` = código C#/.NET rodando **no servidor**, nunca no app.
- `at lambda_method(Closure, Object[])` dentro de `GetResultList` = o erro ocorre na
  **materialização dos resultados** da query → forte indício de **registro com campos
  NULL** em colunas que o código mapeia como não anuláveis (`int`, `bool`).

---

## 3. Diagnóstico passo a passo

### Passo 1 — Confirmar que o problema é o servidor de licenças

1. No console do app (modo dev), localize a linha:
   `API Licença URL completa: https://licenca.sinoinformatica.com.br/api/Mobile/ListarMenus`
2. Descubra o `idCliente` do app com problema (em `Constants.idCliente` para apps tenant,
   ou no `cityStore` para o app PLD).
3. Reproduza no **Postman/curl**, fora do app:

```bash
# Cliente com problema (esperado: 500)
curl -X POST "https://licenca.sinoinformatica.com.br/api/Mobile/ListarMenus" \
  -H "Content-Type: application/json" \
  -d "562"

# Cliente que funciona (esperado: 200 com a lista de menus)
curl -X POST "https://licenca.sinoinformatica.com.br/api/Mobile/ListarMenus" \
  -H "Content-Type: application/json" \
  -d "146"
```

Se o padrão for **id com problema → 500** e **id funcional → 200**, está provado:
o app está correto e o problema é **dados do cliente no banco de licenças**.

### Passo 2 — Comparar o cliente quebrado com um cliente funcional

Método que resolveu o caso: **comparar tabela por tabela** o cliente com erro contra um
cliente que funciona (no caso, 562 vs. 146 — Paulo de Faria).

No banco de licenças (HeidiSQL):

```sql
-- Cadastro base (verificar campos NULL e Ativo = 1)
SELECT * FROM cliente WHERE id IN (146, 562);

-- Configurações por chave/valor (kit padrão: MobileAtivo, EndPoint, BrasaoUri,
-- UserName, UserPassword, Legislacaodigital, Cor, Ouvidoria, DesativarMenusAutoSiscam)
SELECT chave, valor FROM configuracao WHERE cliente = 562 ORDER BY chave;
SELECT chave, valor FROM configuracao WHERE cliente = 146 ORDER BY chave;

-- Licenças/módulos ativos
SELECT * FROM licenca WHERE cliente IN (146, 562);

-- ⭐ MENUS CUSTOMIZADOS — principal suspeito
SELECT * FROM menu WHERE cliente = 562;
SELECT * FROM menu WHERE cliente = 146;
```

### Passo 3 — Procurar registros com campos NULL na tabela `menu`

**Esta foi a causa raiz do caso real:** o cliente 562 tinha um registro de menu
(id 2963, "Instagram") com as colunas **`Ordem`, `IsLink` e `IdTipoDocumento` = NULL**
(cadastro salvo pela metade). Um único registro assim derruba a lista inteira com
`NullReferenceException`.

Query direta para encontrar o culpado:

```sql
SELECT * FROM menu
WHERE cliente = 562
  AND (Ordem IS NULL
    OR IsLink IS NULL
    OR IdTipoDocumento IS NULL
    OR LayoutLinha IS NULL
    OR LayoutLinhaSpan IS NULL
    OR LayoutColuna IS NULL
    OR LayoutColunaSpan IS NULL
    OR LayoutNumeroEstilo IS NULL);
```

---

## 4. Correção

### Opção A — Completar o registro quebrado (padrão dos demais itens do cliente)

```sql
UPDATE menu
SET Ordem = 13,              -- próxima ordem livre do cliente
    IsLink = 1,              -- 1 se a Rota for URL externa
    IdTipoDocumento = 0,
    LayoutLinha = 0,
    LayoutLinhaSpan = 0,
    LayoutColuna = 0,
    LayoutColunaSpan = 0,
    LayoutNumeroEstilo = 0
WHERE id = 2963;             -- id do registro quebrado
```

### Opção B — Remover o registro quebrado

```sql
DELETE FROM menu WHERE id = 2963;
```

### Depois da correção

1. Se durante o diagnóstico a flag `DesativarMenusAutoSiscam` foi alterada, **reverter**:

```sql
UPDATE configuracao
SET valor = 'true'
WHERE cliente = 562 AND chave = 'DesativarMenusAutoSiscam';
```

2. Testar novamente no Postman (`ListarMenus` com o idCliente). Se ainda der 500,
   **reiniciar a aplicação/app pool do servidor de licenças** (a configuração pode
   estar cacheada em memória) e testar de novo.
3. Testar o app de ponta a ponta: login → menu → navegação nas telas.

---

## 5. Referência — estrutura do banco de licenças

Tabelas: `cliente`, `configuracao`, `licenca`, `menu`, `sistema`, `tipocliente`, `usuario`

### Tabela `menu` (formato retornado pelo ListarMenus)

| Coluna | Exemplo | Observação |
|---|---|---|
| Id | 2890 | |
| Cliente | 146 | vínculo com o cliente |
| Nome | Mesa Diretora | |
| Descricao | (vazio) | |
| Rota | `vereadores` ou URL completa | rota interna do app OU link externo |
| Modulo | 0 | |
| Classe | users-line | ícone (FontAwesome) |
| Ordem | 1 | **não pode ser NULL** |
| IsLink | 1 | 1 = abre URL externa; **não pode ser NULL** |
| IdTipoDocumento | 0 | **não pode ser NULL** |
| Layout* | 0 | colunas de layout, **não podem ser NULL** |

### Kit padrão da tabela `configuracao` por cliente

`MobileAtivo`, `EndPoint`, `BrasaoUri`, `UserName`, `UserPassword`,
`Legislacaodigital`, `Cor`, `Ouvidoria`, `ConsultaOnLine` (opcional),
`DesativarMenusAutoSiscam` (`true` = usa os menus customizados da tabela `menu`).

---

## 6. Prevenção (recomendações)

1. **Banco de licenças:** aplicar `NOT NULL DEFAULT 0` nas colunas `Ordem`, `IsLink`,
   `IdTipoDocumento` e `Layout*` da tabela `menu` — impede que um cadastro salvo pela
   metade derrube o menu de um cliente.
2. **Tela de cadastro de menus:** validar campos obrigatórios antes de salvar.
3. **Backend `ListarMenus`:** tratar registro inválido sem estourar a lista inteira
   (ignorar + logar o id do registro problemático) e retornar lista vazia/404 claro
   quando o cliente não tiver menus — em vez de `NullReferenceException` genérico.
4. **App:** exibir mensagem amigável para erros 5xx do menu (hoje o JSON cru do
   servidor chega ao LogBox).

---

## 7. Resumo em uma frase

> **Erro 500 `NullReferenceException` no `ListarMenus` do SISCAM8 = dado quebrado do
> cliente no BANCO DE LICENÇAS (`licenca.sinoinformatica.com.br`) — quase sempre um
> registro da tabela `menu` com colunas NULL. Compare com um cliente funcional,
> encontre o registro incompleto, corrija (UPDATE) ou remova (DELETE) e reinicie a
> aplicação de licenças se necessário.**
