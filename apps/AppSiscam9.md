# Publicação do App — Câmara Municipal de Santa Rosa de Viterbo

**Projeto:** AppSiscam9 (Siscam Web 9 — arquitetura multi-tenant)
**Tenant:** SantaRosaDeViterbo
**Plataformas previstas:** Android (Google Play Store) e iOS (Apple App Store)

> ⚠️ **Status atual: EM PENDÊNCIA — publicação bloqueada.**
> Este documento registra o progresso realizado até o momento e o motivo específico que impede a continuidade do processo.

---

## 1. Contexto do Projeto

O AppSiscam9 é o aplicativo móvel oficial do **Sistema de Processo Legislativo Municipal (Siscam Web 9)**, desenvolvido em React Native com Expo. Assim como o AppSiscam8, utiliza uma arquitetura **multi-tenant**: um único código-fonte atende a múltiplos clientes (câmaras municipais), cada um configurado em `src/Tenants/<Nome>/`.

Este documento trata da configuração do tenant **Santa Rosa de Viterbo**, ainda não publicado.

---

## 2. Dados de Cadastro do Tenant

| Campo | Valor |
|---|---|
| **ID Cliente** | 147 |
| **Package Name** | `br.com.sinoinformatica.processolegislativo.santarosadeviterbo` |
| **Cidade / UF** | Santa Rosa de Viterbo / SP |
| **Scheme (deep link)** | santarosadeviterbo |

---

## 3. Etapas já concluídas

As seguintes etapas do processo de configuração do tenant foram realizadas com sucesso:

- [x] Criação do tenant via wizard (`npm run create-tenant`), gerando `src/Tenants/SantaRosaDeViterbo/build.config.json`
- [x] Adição das 4 imagens obrigatórias na pasta `Assets/` do tenant (ícone, ícone adaptativo, splash, brasão — 1024x1024)
- [x] Adição dos arquivos de configuração do Firebase:
  - `google-services.json` (Android)
  - `GoogleService-Info.plist` (iOS)

  > O app Android foi registrado no mesmo projeto Firebase compartilhado entre os tenants (`appsiscam9`), com `package_name` correspondente ao novo tenant.

- [x] Execução do `npm run sync-tenants`, atualizando o registro central de tenants
- [x] Geração do pré-build (`npx expo prebuild --clean`)
- [x] Build local (Android) compilado e instalado com sucesso no emulador, após resolução de um erro pontual de arquivo travado no Gradle (`Unable to delete file... classes.jar`), comum no ambiente Windows

---

## 4. ⚠️ Problema Atual — Motivo da Pendência

Ao abrir o app já instalado no emulador, ocorre a seguinte falha:

```
ERROR  Erro ao realizar login: [AxiosError: Request failed with status code 400]
```

### Diagnóstico

O app realiza um **login automático de sistema** ao iniciar (usuário/senha fixos, compartilhados entre todos os tenants), contra o endpoint:

```
{baseUrl}/Account/Login
```

Onde `{baseUrl}` é o endereço específico do cliente, definido no `build.config.json` do tenant:

```
https://santarosadeviterbo.servicos.siscam.com.br/
```

**Causa raiz identificada:** esse servidor/subdomínio ainda **não está provisionado**. Diferente do login em si (que usa credenciais genéricas de sistema, iguais para todos os clientes) ou da estrutura do tenant no app (que está correta e completa), o problema está **na infraestrutura do backend**: o ambiente do Siscam Web 9 para este cliente específico ainda não foi disponibilizado pela equipe responsável.

Em outras palavras: **o app mobile está pronto do lado do frontend**, mas não há, ainda, um sistema (backend/API) ativo para ele se conectar.

### Por que não é possível contornar isso agora

- Não é um erro de configuração do tenant (build.config.json, assets e Firebase estão corretos)
- Não é um erro de credenciais de login (usuário/senha de sistema são fixos e válidos)
- É uma **dependência externa de outra equipe** (provisionamento do ambiente Siscam Web 9 para o cliente Santa Rosa de Viterbo)

---

## 5. Próximos Passos (assim que o bloqueio for resolvido)

1. Confirmar com a equipe responsável que o ambiente `https://santarosadeviterbo.servicos.siscam.com.br/` está no ar e respondendo
2. Testar novamente o login localmente (Android e, em seguida, iOS)
3. Validar as demais funcionalidades do app (menus, navegação, integração com API)
4. Seguir para o fluxo de build de produção e publicação nas lojas (mesmo processo já validado com sucesso no tenant Paulo de Faria)

---

## 6. Resumo do Status

| Etapa | Status |
|---|---|
| Criação do tenant (app) | ✅ Concluída |
| Assets e Firebase | ✅ Concluída |
| Build local (Android) | ✅ Compilado com sucesso |
| Teste funcional (login/API) | ❌ **Bloqueado — servidor do cliente não provisionado** |
| Build de produção | ⏸️ Aguardando desbloqueio |
| Publicação nas lojas | ⏸️ Aguardando desbloqueio |

---

*Documento gerado para fins de registro e acompanhamento interno da pendência.*
