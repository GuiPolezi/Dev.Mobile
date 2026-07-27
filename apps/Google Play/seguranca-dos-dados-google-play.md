# Guia — Segurança dos Dados (Google Play) · Apps de Tenant (Processo Legislativo)

> Guia de preenchimento do formulário **Segurança dos dados** do Play Console para os apps de Câmaras Municipais (base multi-tenant Sino Informática).
> Baseado na análise do código-fonte (React Native/Expo) em julho/2026 — app de referência: **Câmara Municipal Santa Rosa de Viterbo**.
>
> ⚠️ **Não copiar a declaração dos apps antigos** ("apenas Registros de falhas e Diagnóstico"). O código atual **não** possui Crashlytics/Analytics, mas **coleta dados pessoais, localização, fotos e IDs de dispositivo**. Declaração incorreta pode causar rejeição ou remoção do app.

---

## O que o app realmente coleta (mapeado no código)

| Dado | Origem no código | Enviado ao servidor? |
|---|---|---|
| Nome, e-mail, telefone | Cadastro Ouvidoria (`ouvidoriacadastro`), Fale Conosco, Fale com Vereador | Sim |
| Login e senha (conta criada no app) | Cadastro Ouvidoria (campos Login/Senha/Confirmar Senha) + "Esqueceu sua senha" | Sim |
| Localização exata (lat/long) + endereço | `expo-location` (foreground) → demandas da Ouvidoria (`GeolocalizacaoLink`) | Sim |
| Fotos (anexos) | Câmera + galeria (`useFilePicker` — **somente .jpg/.jpeg**, sem vídeos/documentos) | Sim (base64) |
| Texto das demandas/mensagens | Ouvidoria, Fale Conosco, Fale com Vereador | Sim |
| ID do dispositivo + token FCM | `react-native-device-info` (`getUniqueId`) + `@react-native-firebase/messaging` | Sim (registro para push) |

**Não há no código:** Firebase Analytics, Crashlytics, login social (OAuth), biometria/2FA, coleta de vídeos, áudio ou documentos.
**Transmissão:** sempre HTTPS (`baseUrl` https do tenant).

---

## Etapa 2 — Coleta de dados e segurança

| Pergunta | Resposta |
|---|---|
| O app coleta ou compartilha dados obrigatórios? | **Sim** |
| Os dados são criptografados em trânsito? | **Sim** (HTTPS) |
| Métodos de criação de conta | **Nome de usuário e senha** (apenas) |

> O login de vereadores **não** conta como criação de conta — as contas são provisionadas pela Câmara, não criadas no app.

---

## Etapa 3 — Tipos de dados (marcar ✅ / deixar ❌)

**Local**
- ❌ Local aproximado
- ✅ **Local exato**

**Informações pessoais**
- ✅ **Nome**
- ✅ **Endereço de e-mail**
- ✅ **IDs de usuários** (login do cadastro da Ouvidoria)
- ✅ **Endereço**
- ✅ **Número de telefone**
- ❌ Raça e etnia · Posicionamento político · Orientação sexual · Outras informações

**Informações financeiras** — ❌ todas
**Saúde e fitness** — ❌ todas
**Mensagens** — ❌ todas (texto de formulário **não** entra aqui; ver "Atividade em apps")

**Fotos e vídeos**
- ✅ **Fotos**
- ❌ Vídeos (picker filtra apenas .jpg/.jpeg)

**Arquivos de áudio** — ❌ todas
**Arquivos e documentos** — ❌ (não há document picker; se habilitarem anexo PDF/DOC no futuro, marcar)

**Atividade em apps**
- ✅ **Outro conteúdo gerado pelo usuário** (texto das demandas da Ouvidoria, Fale Conosco, Fale com Vereador)
- ❌ Interações no app · Histórico de pesquisa · Apps instalados · Outras ações

**Navegação na Web** — ❌
**Informações e desempenho do app** — ❌ todas (sem Crashlytics/Analytics)

**IDs do dispositivo ou outros IDs**
- ✅ **IDs do dispositivo ou outros IDs** (`getUniqueId` + token FCM para push)

---

## Etapa 4 — Uso e tratamento (para CADA tipo marcado)

| Pergunta | Resposta padrão |
|---|---|
| Esses dados são coletados, compartilhados ou ambos? | **Coletados** (não compartilhados) |
| Processados de forma efêmera? | **Não** |
| Coleta obrigatória ou opcional? | **Opcional** (usuário só fornece se usar Ouvidoria / Fale Conosco / login; navegação não exige dados) |
| Finalidade | **Funcionalidade do app** |

> "Não compartilhados": os dados vão apenas para o servidor da Câmara (siscam.com.br). Não há SDKs de terceiros recebendo dados de usuário além do FCM para entrega de push.

---

## Requisito obrigatório — Exclusão de conta

Como o app **permite criação de conta**, o Google exige uma **URL pública de exclusão de conta e dados**, acessível sem reinstalar o app.

- Padrão sugerido: `https://sinoinformatica.com.br/exclusao-de-conta/camara-<tenant>/`
- Conteúdo mínimo: identificação do app, quais dados são excluídos, como solicitar (formulário ou e-mail), prazo de atendimento.
- A página **precisa estar no ar** antes de enviar para revisão.

---

## Checklist por tenant (antes de enviar para revisão)

- [ ] `PolicyUrl` do `build.config.json` está **no ar** e cobre: nome/e-mail/telefone, localização exata, fotos, ID de dispositivo/push, registro de IP no servidor (`IpUsuario`), direitos LGPD e canal de contato
- [ ] URL de **exclusão de conta** publicada e informada no Play Console
- [ ] Formulário Segurança dos Dados preenchido conforme este guia
- [ ] `packageName`, ícones e screenshots do tenant corretos
- [ ] **iOS:** screenshots capturados em iPhone/iPad (nunca Android — causa rejeição pela Guideline 2.3.10; conferir também "View All Sizes in Media Manager")

---

## Quando este guia fica desatualizado

Revisar a declaração se o app ganhar:
- Crashlytics ou Analytics → marcar "Informações e desempenho do app" (Registros de falhas / Diagnóstico)
- Anexo de documentos (PDF/DOC) → marcar "Arquivos e documentos"
- Anexo de vídeo → marcar "Vídeos"
- Login social / biometria → ajustar "Métodos de criação de conta"
- Qualquer SDK de terceiros que receba dados → reavaliar "compartilhamento"

*Última revisão: 24/07/2026 · Base: src.7z (Santa Rosa de Viterbo)*
