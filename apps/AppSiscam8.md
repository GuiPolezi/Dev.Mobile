# Publicação do App — Câmara Municipal de Paulo de Faria

**Projeto:** AppSiscam8 (arquitetura multi-tenant)
**Tenant:** PauloDeFaria
**Plataformas:** Android (Google Play Store) e iOS (Apple App Store)

---

## 1. Contexto do Projeto

O AppSiscam8 utiliza uma arquitetura **multi-tenant**: um único código-fonte atende a vários clientes (câmaras municipais), cada um configurado como um "tenant" dentro da pasta `src/tenants/`. Cada tenant possui seu próprio arquivo `build.config.json`, com nome, cores, ícones, package name e URL base da API.

Este documento descreve o passo a passo realizado para publicar o app do tenant **PauloDeFaria** nas duas lojas oficiais.

---

## 2. Preparação do Ambiente

### 2.1 Testes locais (Android)

Antes de qualquer build de produção, o app foi testado localmente em modo de desenvolvimento:

```bash
npm run android:tenant PauloDeFaria
```

Esse comando executa, em sequência:
1. `sync-tenants` — sincroniza o registro central de tenants
2. `generate-config` — gera as configurações específicas do tenant
3. `expo prebuild` — gera a pasta nativa `android/`
4. `expo run:android` — compila e instala o app em um emulador

**Resultado:** app funcionando corretamente, com ícone, splash e cores do tenant.

### 2.2 Testes locais (iOS)

Builds nativos de iOS exigem **macOS + Xcode** (limitação da própria Apple). O código-fonte foi obtido via Git (Gitea) em um Mac, com o arquivo `.env` copiado manualmente (fora do controle de versão, por conter credenciais).

```bash
npm run ios:tenant PauloDeFaria
```

Pré-requisitos instalados no Mac:
- Xcode (via App Store)
- CocoaPods (`sudo gem install cocoapods`)

**Resultado:** app funcionando corretamente no simulador iOS.

---

## 3. Ajuste identificado durante os testes

Durante os testes, foi identificado que dois botões do menu principal ("Mesa Diretora" e "Comissões") redirecionavam para URLs incorretas.

**Investigação:**
- Os itens de menu do app **não são fixos no código-fonte** — são carregados dinamicamente via API (`ApiLicenca`, endpoint `Mobile/ListarMenus`), consultando o `idCliente` do tenant.
- A correção, portanto, **não foi feita no código do app**, e sim diretamente no cadastro do backend/banco de dados vinculado ao cliente.

**Ação:** URLs corrigidas na base de dados. Nenhuma alteração de código foi necessária.

---

## 4. Publicação no Android (Google Play Store)

### 4.1 Geração do build de produção (.aab)

Durante a primeira tentativa de build, foi necessário resolver um erro de build do Gradle (`Unable to delete file... classes.jar`), causado por processos Java/Gradle pendurados. Resolvido com:

```bash
cd android
.\gradlew.bat --stop
cd ..
taskkill /F /IM java.exe
```

Após a limpeza, o build de produção do tenant foi gerado com:

```bash
npm run build:tenant PauloDeFaria
```

Esse comando injeta a variável `EXPO_PUBLIC_TENANT=PauloDeFaria` e executa o pipeline completo (sync-tenants → generate-config → prebuild → `gradlew bundleRelease`).

**Arquivo final gerado:**
```
android\app\build\outputs\bundle\release\app-release.aab
```

> Observação: existe também um arquivo `intermediary-bundle.aab` em `intermediates/`, que é apenas um artefato intermediário do processo de build e **não deve ser usado**.

### 4.2 Criação do app no Google Play Console

1. Criação do app no [Google Play Console](https://play.google.com/console)
2. Nome de pacote (`packageName`) definido conforme o padrão da empresa
3. Preenchimento da ficha da loja (nome, descrição, ícone, screenshots)
4. Vinculação da política de privacidade do tenant
5. Preenchimento da classificação de conteúdo e da seção de segurança de dados

**Atenção:** o nome de pacote (`packageName`) é **definitivo** após a criação do app — não pode ser alterado. Na primeira tentativa, o nome foi digitado incorretamente e o app precisou ser excluído e recriado antes do primeiro upload.

### 4.3 Upload da versão de Teste Interno

1. Upload do arquivo `app-release.aab` na faixa **Teste Interno**
2. Configuração da lista de testadores (e-mails) na aba **Testadores**
3. Acesso via link de opt-in (`https://play.google.com/apps/internaltest/...`)

**Resultado:** app instalado e testado com sucesso via Teste Interno.

### 4.4 Promoção para Produção

Após validação em Teste Interno e conclusão da ficha da loja, a versão foi promovida para a faixa de **Produção**, com envio automático para análise do Google.

**Data de envio:** 20/07/2026
**Status:** em análise (prazo médio informado: ~2 dias úteis)

---

## 5. Publicação no iOS (Apple App Store)

### 5.1 Cadastros prévios

1. Registro do **Bundle ID** no [Apple Developer Portal](https://developer.apple.com/account), com o mesmo identificador usado no Android (padrão de nomenclatura da empresa)
2. Criação do app no [App Store Connect](https://appstoreconnect.apple.com), vinculando o Bundle ID registrado e definindo um SKU interno

### 5.2 Geração do build de produção (via Xcode — processo manual)

O guia do repositório previa um script automatizado (Fastlane) para build e publicação. Optou-se por realizar o processo **manualmente pelo Xcode** na primeira publicação, para maior controle sobre erros e sobre o envio à revisão (o script automatizado submete o app diretamente para revisão, sem etapa intermediária de validação).

**Passos realizados:**

1. Geração da pasta nativa iOS:
   ```bash
   npm run sync-tenants
   EXPO_PUBLIC_TENANT=PauloDeFaria npm run generate-config
   EXPO_PUBLIC_TENANT=PauloDeFaria npx expo prebuild --platform ios --clean
   ```
2. Instalação das dependências nativas:
   ```bash
   cd ios && pod install
   ```
3. Abertura do projeto no Xcode:
   ```bash
   open ios/*.xcworkspace
   ```
4. Configuração de assinatura (**Signing & Capabilities**):
   - Ativação de "Automatically manage signing"
   - Seleção do Team (conta Apple Developer da empresa)
   - Confirmação do Bundle Identifier
5. Alteração do destino de build para **Any iOS Device (arm64)**
6. Geração do Archive: `Product → Archive`

**Erro encontrado durante o Archive:** falha no script de build do módulo `expo-constants`, por ausência da variável `EXPO_PUBLIC_TENANT` no ambiente de execução do Xcode (essa variável, por padrão, é definida dinamicamente pelos scripts npm, não pelo `.env`).

**Solução aplicada:** inclusão temporária da variável `EXPO_PUBLIC_TENANT=PauloDeFaria` no arquivo `.env` durante o processo de build manual, removida posteriormente para não impactar builds de outros tenants.

### 5.3 Upload para o App Store Connect

Após o Archive concluído com sucesso:

1. `Distribute App` → `App Store Connect` → `Upload`
2. Assinatura automática (`Automatically manage signing`)
3. Upload concluído com sucesso

**Aviso não bloqueante identificado:** "Upload symbols failed" — ausência do arquivo dSYM do `hermes.framework`. Trata-se de um problema comum em projetos Expo/React Native com Hermes habilitado, que **não impacta o funcionamento do app**, afetando apenas a legibilidade de relatórios de crash relacionados ao motor JavaScript. O envio de símbolos foi desativado para não bloquear o upload.

### 5.4 Testes via TestFlight

1. Processamento automático do build pelo App Store Connect
2. Resposta ao questionário de conformidade de exportação (uso padrão de criptografia/HTTPS)
3. Configuração de testadores internos na aba **TestFlight**
4. Instalação e teste via aplicativo TestFlight no iPhone

**Resultado:** app funcionando corretamente via TestFlight.

### 5.5 Preenchimento da ficha da App Store e envio para revisão

Preenchimento das seções obrigatórias no App Store Connect:

- Informações gerais do app (nome, subtítulo, categoria)
- Preço e disponibilidade (gratuito)
- Screenshots (gerados via Simulador do Xcode, nos tamanhos exigidos — iPhone e iPad, 10 capturas)
- Descrição, palavras-chave e URL de suporte
- Política de privacidade
- Informações de contato para revisão
- Classificação etária
- Seleção do build testado via TestFlight

> Observação: como o app **não exige login** para acesso às funcionalidades principais, não foi necessário fornecer conta de demonstração (*demo account*) para o revisor.

**Documento de autorização:** foi anexado o documento assinado de autorização de publicação na seção **App Review Information → Attachment**, antecipando possíveis questionamentos da Apple quanto à legitimidade da publicação, por se tratar de um aplicativo institucional de órgão público.

**Data de envio para revisão:** 21/07/2026
**Status:** aguardando revisão (prazo médio informado: ~2 dias úteis)

---

## 6. Resumo do Status Final

| Plataforma | Etapa concluída | Data de envio | Status |
|---|---|---|---|
| **Android** | Publicação em Produção enviada | 20/07/2026 | Em análise |
| **iOS** | Publicação enviada para revisão | 21/07/2026 | Aguardando revisão |

---

## 7. Lições Aprendidas / Pontos de Atenção para Próximas Publicações

1. **Nome de pacote (Android) é definitivo** — conferir com atenção antes da criação do app no Play Console.
2. **URLs de menu vêm do backend**, não do código-fonte — ajustes de link devem ser feitos no cadastro do cliente, não no app.
3. **`EXPO_PUBLIC_TENANT` não fica fixo no `.env`** — é necessário defini-la manualmente ao rodar processos fora dos scripts npm padrão (ex: Archive direto pelo Xcode).
4. **Builds Android podem falhar por arquivos travados no Windows** — resolvido parando o daemon do Gradle e finalizando processos Java pendentes; evitar manter o projeto dentro de pastas sincronizadas por serviços de nuvem (ex: OneDrive).
5. **Aviso de dSYM do Hermes no iOS é normal** e não impede a publicação — apenas reduz a legibilidade de relatórios de crash relacionados ao motor JavaScript.
6. **Recomenda-se sempre testar em faixa intermediária antes de produção**: Teste Interno (Android) e TestFlight (iOS) evitam publicar uma versão com erros diretamente ao público.
7. **Documento de autorização institucional** deve ser anexado preventivamente na revisão da Apple para apps de órgãos públicos, evitando ciclos de rejeição.

---

*Documento gerado para fins de registro e referência de processo interno.*
