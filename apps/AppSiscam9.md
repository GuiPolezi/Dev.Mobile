# Publicação do App — Câmara Municipal de Santa Rosa de Viterbo

**Projeto:** AppSiscam9 (Siscam Web 9 — arquitetura multi-tenant)
**Tenant:** SantaRosaDeViterbo
**Plataformas:** Android (Google Play Store) e iOS (Apple App Store)

> ✅ **Status atual: PUBLICAÇÃO REALIZADA — apps enviados às lojas em 24/07/26.**
> O bloqueio anterior (backend não provisionado) foi resolvido: o ambiente do Siscam Web 9 para o cliente foi disponibilizado, permitindo a configuração, os testes e o envio do app para as lojas. Android e iOS encontram-se **em análise** pelo Google e pela Apple.

---

## 1. Contexto do Projeto

O AppSiscam9 é o aplicativo móvel oficial do **Sistema de Processo Legislativo Municipal (Siscam Web 9)**, desenvolvido em React Native com Expo. Assim como o AppSiscam8, utiliza uma arquitetura **multi-tenant**: um único código-fonte atende a múltiplos clientes (câmaras municipais), cada um configurado em `src/Tenants/<Nome>/`.

Este documento registra todo o processo de configuração e publicação do tenant **Santa Rosa de Viterbo** — incluindo o histórico da pendência inicial de backend e como ela foi resolvida.

---

## 2. Dados de Cadastro do Tenant

| Campo | Valor |
|---|---|
| **ID Cliente** | 147 |
| **Package Name** | `br.com.sinoinformatica.processolegislativo.santarosadeviterbo` |
| **Cidade / UF** | Santa Rosa de Viterbo / SP |
| **Scheme (deep link)** | santarosadeviterbo |
| **Base URL (API)** | `https://santarosaviterbo9.siscam.com.br/api/` |
| **Visualização de documentos** | `https://santarosaviterbo9.siscam.com.br/` |
| **Política de Privacidade** | `https://sinoinformatica.com.br/politica-de-privacidade/camara-santarosadeviterbo/index.html` |

---

## 3. Histórico — Pendência inicial (resolvida)

Na primeira tentativa de configuração, o app compilava e instalava normalmente, mas falhava no login automático de sistema (`AxiosError: Request failed with status code 400`). A causa raiz era **externa ao app**: o ambiente de backend do Siscam Web 9 para este cliente ainda não havia sido provisionado.

**Resolução:** a equipe responsável disponibilizou o ambiente do cliente. Vale registrar que a URL definitiva do backend ficou **diferente da prevista inicialmente** — o `build.config.json` precisou ser atualizado do subdomínio antigo (`santarosadeviterbo.servicos.siscam.com.br`) para o endereço definitivo (`santarosaviterbo9.siscam.com.br`).

> 💡 **Aprendizado:** antes de configurar o tenant, sempre confirmar com a equipe de infraestrutura a **URL definitiva** do ambiente do cliente. A URL prevista pode mudar no provisionamento.

---

## 4. Configuração da API e do Tenant

- [x] Validação da API via **Postman** no endpoint `.../api/Account/Login`, utilizando as credenciais de sistema (fixas e compartilhadas entre os tenants) → retorno **200 OK**
- [x] Atualização do `build.config.json` do tenant com os dados definitivos (nome, cidade/UF, idCliente, brasão, política de privacidade, package name, cor do tema, `baseUrl` e `baseViewDocumentosUrl`)
- [x] Execução do `npm run sync-tenants`
- [x] `npm run android` → **funcionando**
- [x] Build/execução iOS validada na sequência

---

## 5. Configuração do Banco de Dados (backend)

Ao abrir o app pela primeira vez com o backend no ar, o app ainda apresentava falha: era necessário **criar os registros na tabela `Configurações`** do banco do cliente.

- ✅ Registros criados na tabela `Configurações`
- ⚠️ Após criar os registros, é necessário **reiniciar a aplicação do sistema** (via Postman) para que as configurações sejam recarregadas e validadas

> 💡 **Aprendizado:** um tenant novo no Siscam 9 exige configuração em **duas frentes**: o app (build.config.json) e o **banco de dados do cliente** (tabela `Configurações`). Sem os registros no banco, o app não abre corretamente mesmo com o backend no ar.

---

## 6. Configuração dos Menus do Aplicativo

O app iniciou com apenas 2 botões. Os demais foram criados tomando como **referência a Câmara de Itatiba** (também Siscam 9, já publicada):

**Botões de referência:** Proposituras, Votações, Sessões, Vereadores, Fale com Vereador, Comissões, Legislações, Documentos de Sessão, Assinar Digitalmente.

### Problemas encontrados e tratativas

| Problema | Tratativa | Resultado |
|---|---|---|
| **Vereadores**, **Comissões** e **Fale com Vereador** sem resultados (suspeita: `api/autores` ou configuração do banco) | Ajuste realizado no **banco de dados** (com apoio de outro dev) | ✅ Funcionando |
| **Documentos de Sessão** sem resultados | Tentativa de correção sem sucesso; menu **removido** do app (custo/benefício não compensava) | ✅ Resolvido por remoção |
| **Assinar Digitalmente** não exibe o assinante (consequência do problema de autores) | Mantido como está por enquanto — acompanhar | ⚠️ Pendência conhecida |

> 💡 **Aprendizado:** problemas de menus "vazios" no app geralmente não são do código mobile, e sim de **dados/configuração no banco do cliente**. Validar o banco antes de mexer no app — e, quando um menu não é essencial e o custo de correção é alto, **remover temporariamente** é uma decisão válida.

---

## 7. Publicação — Android (Google Play)

- [x] Criação do app no **Google Play Console**
- [x] Configuração do app:
  - URL da política de privacidade informada
  - Declaração de área restrita: **SIM** (app possui login) — credenciais de teste de revisão fornecidas ao Google *(não versionadas neste repositório)*
- [x] Preenchimento dos detalhes do app (README.md de apoio gerado com as informações solicitadas pelo console)
- [x] Criação da **versão de teste interno** (geração do `.aab`)
- [x] Aguardada a **propagação do teste interno (~1 hora)** antes de testar via link do Play Console
- [x] Teste interno validado — **funcionando**
- [x] **Aplicativo enviado para disponibilização em 24/07/26**

> 💡 **Aprendizado:** o teste interno do Google Play leva cerca de **1 hora para propagar** após o envio do `.aab`. Não adianta tentar instalar imediatamente.

---

## 8. Publicação — iOS (App Store)

- [x] Criação do **Bundle ID** em developer.apple.com
- [x] Criação do app em **App Store Connect**
- [x] Obtenção das **capturas de tela** (iPhone e iPad)
- [x] Configuração do app e geração do build
- [x] **Teste interno realizado — funcionando**
- [x] **Aplicativo enviado para publicação em 24/07/26**

---

## 9. Resumo do Status

| Etapa | Status |
|---|---|
| Criação do tenant (app) | ✅ Concluída |
| Assets e Firebase | ✅ Concluída |
| Provisionamento do backend (dependência externa) | ✅ Resolvido |
| Configuração do banco (tabela `Configurações`) | ✅ Concluída |
| Configuração dos menus do app | ✅ Concluída (Documentos de Sessão removido) |
| Testes locais (Android e iOS) | ✅ Validados |
| Teste interno (Play Console / TestFlight) | ✅ Funcionando |
| Envio Android (Google Play) | 🟡 Enviado em 24/07/26 — **em análise** |
| Envio iOS (App Store) | 🟡 Enviado em 24/07/26 — **em análise** |

### Pendências conhecidas (pós-publicação)

- ⚠️ "Assinar Digitalmente" não exibe o assinante — acompanhar com os devs
- ⚠️ Menu "Documentos de Sessão" removido — reavaliar reinclusão futuramente

---

*Documento de registro e acompanhamento interno. Credenciais de sistema e de revisão das lojas não são versionadas neste repositório.*