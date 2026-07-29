# Correção do envio de e-mail — Fale com o Vereador (e e-SIC)

**Data:** 29/07/2026
**Pacote afetado:** `Sino.Site.UI`
**Sintoma:** o formulário "Fale com o Vereador" (app SISCAM 9 → `POST api/falecom/vereadoravancado`) era registrado com sucesso no banco (`site_falecomvereadoravancado`), retornava `200 — "Formulário registrado e e-mail enviado com sucesso!"`, mas o e-mail **nunca chegava ao vereador selecionado** — caía sempre em `suporte@sinoinformatica.com.br`.

---

## 1. Diagnóstico

A investigação descartou, nesta ordem:

1. **O app mobile (appsiscam9)** — envia `VereadorEmail` corretamente no form-data.
2. **O controller da API** (`FalecomController.VereadorAvancadoAsync`) — valida o `VereadorEmail`, persiste no banco e repassa o destinatário correto ao serviço de e-mail.
3. **O banco** — a coluna `VereadorEmail` é gravada com o e-mail certo.

A causa estava no último elo: a classe `Sino.Site.UI/Extensions/EmailDestinoHelper.cs`, um **stub de teste que foi parar na versão publicada do pacote** (9.0.1.10x):

```csharp
// CÓDIGO COM DEFEITO (removido)
public static class EmailDestinoHelper
{
    public const string EmailTeste = "suporte@sinoinformatica.com.br";

    public static MailAddress CriarDestinatario(string displayName) =>
        new(EmailTeste, displayName);

    // Recebe o destinatário real e o DESCARTA (repare no parâmetro "_")
    public static string ResolverDestinatario(string? _) => EmailTeste;
}
```

E o `EmailServiceSite.SendEmailAsync` — usado por **todos** os envios — chamava o helper na véspera de montar a mensagem:

```csharp
to = EmailDestinoHelper.ResolverDestinatario(to);  // trocava qualquer destino por suporte@
```

Por isso o fluxo inteiro parecia saudável (banco OK, resposta 200, SMTP disparando de verdade), mas o campo `To` era substituído no último passo.

**Impacto além do Fale com Vereador:** o mesmo helper era usado diretamente pelas páginas do **e-SIC** — notificações à ouvidoria, confirmações de protocolo ao cidadão, prazos e trâmites também caíam todos no suporte.

---

## 2. O que foi desenvolvido (a correção)

### 2.1 `Sino.Site.UI/Services/EmailService.cs` — **a correção principal**

No método privado `SendEmailAsync` da classe `EmailServiceSite`, a linha que desviava o destinatário foi substituída por um **desvio opcional, controlado por configuração**:

```csharp
// Desvio global de e-mails APENAS para testes/homologação: defina
// 'EmailSettings:RedirectAllTo' no ambiente de teste para capturar
// todos os envios. Em produção a chave deve ficar ausente — o
// destinatário real (ex.: o vereador selecionado no formulário) é usado.
var redirecionarPara = _configuration["EmailSettings:RedirectAllTo"];
if (!string.IsNullOrWhiteSpace(redirecionarPara))
    to = redirecionarPara;
```

Comportamento resultante:

| Ambiente | Configuração | Destinatário efetivo |
|---|---|---|
| **Produção** | `EmailSettings:RedirectAllTo` **ausente** | O destinatário real (o vereador selecionado no formulário) |
| Homologação/testes | `EmailSettings:RedirectAllTo` = ex.: `suporte@sinoinformatica.com.br` | Todos os e-mails desviados para o endereço configurado |

Isso preserva a intenção original do stub (testar sem disparar e-mails a vereadores reais), mas de forma segura: o desvio agora é uma decisão explícita de configuração de ambiente, não um comportamento embutido no código.

### 2.2 Páginas do e-SIC — destinatários reais

As 5 páginas que usavam `EmailDestinoHelper.CriarDestinatario(...)` passaram a usar os destinatários corretos (as variáveis já eram calculadas nas páginas, mas não eram usadas). Um `e-mail` inválido/ausente lança exceção dentro do `try` já existente e cai no mesmo tratamento de erro de antes — comportamento preservado.

| Arquivo | Notificação → | Confirmação → |
|---|---|---|
| `Areas/Site/Pages/Esic/RegistrarManifestacao.cshtml.cs` | Ouvidoria | E-mail do cidadão |
| `Areas/Admin/Pages/Esic/Create.cshtml.cs` | Ouvidoria | E-mail do cidadão |
| `Areas/Admin/Pages/Esic/Details.cshtml.cs` | Ouvidoria (e-SIC) | E-mail do cidadão |
| `Areas/Admin/Pages/Esic/Prazos/Create.cshtml.cs` | Ouvidoria (e-SIC) | E-mail do cidadão |
| `Areas/Admin/Pages/Esic/Ocorrencias/Create.cshtml.cs` | Ouvidoria (e-SIC) | E-mail do cidadão |

> **Atenção:** a partir do deploy dessa correção, os e-mails do e-SIC passam a chegar de verdade à ouvidoria e aos cidadãos. Coordenar com a equipe que atende a ouvidoria.

### 2.3 `Sino.Site.UI/Extensions/EmailDestinoHelper.cs` — **excluído**

O stub de teste foi removido do projeto (nenhuma referência restante), para que não volte a ser usado por engano.

### Resumo dos arquivos

| Ação | Arquivo |
|---|---|
| **Modificado** | `Sino.Site.UI/Services/EmailService.cs` *(correção principal)* |
| Modificado | `Sino.Site.UI/Areas/Site/Pages/Esic/RegistrarManifestacao.cshtml.cs` |
| Modificado | `Sino.Site.UI/Areas/Admin/Pages/Esic/Create.cshtml.cs` |
| Modificado | `Sino.Site.UI/Areas/Admin/Pages/Esic/Details.cshtml.cs` |
| Modificado | `Sino.Site.UI/Areas/Admin/Pages/Esic/Prazos/Create.cshtml.cs` |
| Modificado | `Sino.Site.UI/Areas/Admin/Pages/Esic/Ocorrencias/Create.cshtml.cs` |
| **Excluído** | `Sino.Site.UI/Extensions/EmailDestinoHelper.cs` |

Build do `Sino.Site.UI` validado após as alterações: **0 erros**.

---

## 3. Como rodar local para testar

O site de Itatiba (`itatiba-9`) consome o `Sino.Site.UI` via NuGet. Para testar a correção **sem publicar pacote**, o `Sino.Sistema.Itatiba.Web.csproj` foi ajustado para, em **Debug**, referenciar o projeto local corrigido (em Release continua usando o pacote do feed):

```xml
<!-- Em Release usa o pacote publicado; em Debug usa o projeto local
     (Site-9-master) para testar alterações do Sino.Site.UI sem publicar. -->
<PackageReference Include="Sino.Site.UI" Version="9.0.1.108" Condition="'$(Configuration)' == 'Release'" />
<ProjectReference Include="..\..\Site-9-master\site-9\Sino.Site.UI\Sino.Site.UI.csproj" Condition="'$(Configuration)' != 'Release'" />
```

> O caminho relativo pressupõe as pastas `itatiba-9/` e `Site-9-master/` lado a lado. Ajuste se a estrutura for outra. **Não commitar esse condicional** no repositório do Itatiba — é um artifício de teste local.

Também foi adicionada ao `appsettings.Development.json` a seção `EmailSettings` (a que o `EmailServiceSite` realmente lê — a seção `Email` existente é usada por outro serviço):

```json
"EmailSettings": {
  "SmtpHost": "smtp.sinoinformatica.com.br",
  "SmtpPort": "587",
  "SmtpUser": "noreply-cmitatiba@sinoinformatica.com.br",
  "SmtpPass": "<senha do SMTP>",
  "FromName": "Câmara Municipal de Itatiba"
}
```

### Pré-requisitos

- Acesso à rede interna da Sino (ou VPN): o banco de desenvolvimento é o MySQL `192.168.10.43` (`sistema9_cmitatiba`) — o site **não sobe** sem conseguir conectar.
- Acesso ao SMTP `smtp.sinoinformatica.com.br`.

### Subir o site

```bash
cd itatiba-9/Sino.Sistema.Itatiba.Web
dotnet run --launch-profile http
# sobe em http://localhost:5158 (Debug → usa o Sino.Site.UI local corrigido)
```

### Teste 1 — Postman (o teste decisivo)

- `POST http://localhost:5158/api/falecom/vereadoravancado`
- Body **form-data**:
  - `VereadorEmail` = *seu próprio e-mail* (para você receber)
  - `Nome`, `Email`, `Assunto`, `Mensagem` = valores de teste

**Resultado esperado:** `200` + o e-mail chega **na sua caixa** — e não mais em `suporte@sinoinformatica.com.br`. Registro correspondente criado em `site_falecomvereadoravancado`.

### Teste 2 — ponta a ponta com o app (opcional)

1. Em `appsiscam9/src/Tenants/Itatiba/build.config.json`, troque temporariamente:
   `"faleComUrl": "http://localhost:5158/"`
2. `TENANT=Itatiba npm run web` e envie pelo formulário **Fale com o Vereador**.
3. Ao final, restaure `"faleComUrl": "https://www.camaraitatiba.sp.gov.br/"`.

### Teste 3 — desvio de homologação (opcional)

Adicione em `EmailSettings` do `appsettings.Development.json`:

```json
"RedirectAllTo": "suporte@sinoinformatica.com.br"
```

Repita o envio: tudo deve voltar a cair no suporte (validando o mecanismo de teste). **Remova a chave depois** — e garanta que ela **nunca** exista em produção.
