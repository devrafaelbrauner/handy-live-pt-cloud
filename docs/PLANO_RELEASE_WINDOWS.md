# Plano de release para Windows — handy-live-pt-cloud

> Estado: plano, nada publicado. Base verificada: `main` em `3925ee8` mais o PR #1 (transcrição na nuvem), em 04/10/2026.

## Objetivo

Publicar no GitHub Releases de `devrafaelbrauner/handy-live-pt-cloud` os instaladores Windows da versão com transcrição na nuvem. Eles devem instalar num PC com Windows 10 ou 11 que não tenha nenhuma ferramenta de desenvolvimento.

## Onde estamos

- **O CI já gera os instaladores Windows sem assinatura.** O "Main Branch Build" do `main` ([run 37219339029](https://github.com/devrafaelbrauner/handy-live-pt-cloud/actions/runs/37219339029)) passou nos dois jobs Windows:

  | Job                      | Duração (com cache) | Artefato (expira em 03/11/2026)                                                            |
  | ------------------------ | ------------------- | ------------------------------------------------------------------------------------------ |
  | `windows-latest` (x64)   | 28 min              | `handy-x86_64-pc-windows-msvc`, 56 MB: instalador NSIS `.exe` (20 MB) e MSI `.msi` (39 MB) |
  | `windows-11-arm` (ARM64) | 19 min              | `handy-aarch64-pc-windows-msvc`, 40 MB                                                     |

- **Os pacotes já são auditados no CI** (`.github/workflows/build.yml`, passo "Audit Windows package runtime contents"):
  - o NSIS é instalado em modo portátil silencioso e o MSI é extraído;
  - confere-se a presença do runtime VC++ junto do app, do `onnxruntime.dll` e das DLLs de transcribe/ggml;
  - roda-se `handy.exe --list-devices`.
- **Arquiteturas:** no x64 há aceleração por GPU (Vulkan) e ONNX Runtime; no ARM64 só CPU.
- **Identidade própria:** `productName` "Handy Live PT", `identifier` `com.rafael.handylive`, versão `0.9.6`. O instalador NSIS usa esse nome na pasta, nos atalhos e na chave de desinstalação, então convive com o Handy oficial. A única colisão é o atalho padrão `Ctrl+Space`, se os dois estiverem abertos.
- **Sem assinatura de código:** o PR #6 removeu o `signCommand` do upstream (Azure Trusted Signing) e o fork não tem certificado.

## O que impede o release hoje

1. **`release.yml` usa `sign-binaries: true` e monta as 7 plataformas.** No fork, os jobs macOS falham ao importar um certificado vazio, o mesmo problema que o PR #6 resolveu no "Main Branch Build". O Windows sairia sem assinatura, mas a run terminaria vermelha. O `pr-test-build.yml` tem o mesmo `sign-binaries: true`.
2. **O atualizador aponta para o lugar errado:**
   - `plugins.updater.endpoints` (em `src-tauri/tauri.conf.json`) busca `devrafaelbrauner/handy-live-pt/releases/latest/download/latest.json`, com a chave pública do upstream, e `createUpdaterArtifacts` é `false`. Hoje a verificação falha em silêncio, com erro só no console.
   - `PORTABLE_RELEASES_URL` (`src/components/update-checker/portableInstaller.ts`) manda para `cjpais/Handy/releases/latest`.
3. **Versão:**
   - a tag seria `v0.9.6`, a mesma do upstream;
   - o MSI só aceita sufixo numérico até 65535 (`-N` ou `+N`), regra do `convert_version` do `tauri-bundler`. Algo como `0.9.6-cloud.1` quebra o build MSI.
4. **O PR #1 ainda não está no `main`.** A transcrição na nuvem é o motivo deste release.
5. **README:** o caminho de dados citado para Windows é o do upstream (`com.pais.handy`). Neste fork ele é `%APPDATA%\com.rafael.handylive`.

## Fases

### F0 — Pré-requisitos (você)

1. Configurar o segredo da revisão Kilo, para a pipeline dos PRs chegar à aprovação humana:
   `gh secret set KILO_API_KEY -R devrafaelbrauner/handy-live-pt-cloud`
2. Aprovar e mergear o PR #1.

### F1 — PR "release Windows" (agente; um PR, sem merge)

1. `release.yml`:
   - `sign-binaries: false`;
   - matriz só com `windows-latest` (x64), mais `windows-11-arm` se a decisão 2 incluir ARM64;
   - mantém release em rascunho com `generate_release_notes`.
2. `pr-test-build.yml`: `sign-binaries: false`.
3. Versão `0.9.6-1` em `src-tauri/tauri.conf.json` e `src-tauri/Cargo.toml` (mais `Cargo.lock`).
   - Próximas versões: `0.9.6-2`, `0.9.6-3`…
   - Ao sincronizar com uma versão nova do upstream (ex.: 0.9.7), volta para `0.9.7-1`.
   - O MSI vira `0.9.6.1`.
4. Atualizador, no mínimo:
   - endpoint para `https://github.com/devrafaelbrauner/handy-live-pt-cloud/releases/latest/download/latest.json`;
   - `PORTABLE_RELEASES_URL` para `https://github.com/devrafaelbrauner/handy-live-pt-cloud/releases/latest`.

   O canal próprio completo fica na F4 (decisão 5).

5. README:
   - seção "Instalar no Windows": download, SmartScreen ("Mais informações → Executar assim mesmo"), NSIS ou MSI, pasta de dados, logs, desinstalação, GPU Vulkan ou CPU;
   - correção do caminho de dados.
6. Rodar os gates locais antes de abrir o PR: `bun run lint`, `bun run build`, `bun run format:check`, `bun run check:translations`, `cargo test`.

### F2 — Build de teste e validação num PC Windows real

1. Gerar instaladores do PR da F1:
   `gh workflow run "PR Test Build" -R devrafaelbrauner/handy-live-pt-cloud -f pr_number=<n>`
   e baixar o artefato `handy-pr-<n>-x86_64-pc-windows-msvc`.
2. Roteiro num Windows 10 ou 11 x64 (anotar o resultado de cada item no PR, com prints):
   1. Instalar pelo NSIS como usuário atual, sem admin, passando pelo SmartScreen. Abrir o app.
   2. Liberar o microfone em Configurações → Privacidade → Microfone, incluindo "aplicativos da área de trabalho".
   3. Ditar com `Ctrl+Space`, segurando e tocando, no Bloco de Notas, no Word e no Chrome. O texto precisa ser colado.
   4. Modelo local: baixar o Whisper Small, conferir em Avançado se usa Vulkan (GPU) ou CPU e ditar em pt-BR.
   5. Nuvem: API na Nuvem → OpenRouter ou OpenAI → colar a chave → ligar → ditar. Com uma chave inválida, a notificação "rejected the API key" deve aparecer.
   6. Conferir sobreposição, ícone na bandeja, "iniciar com o Windows", histórico e "tentar novamente".
   7. Com o Windows Defender ligado, nenhum alerta ao instalar e abrir.
   8. Desinstalar pelo Painel de Controle. Se o Handy oficial estiver instalado, ele precisa continuar intacto.
   9. MSI: instalar como admin em outra máquina ou VM, abrir e desinstalar.

### F3 — Release (você aprova cada passo)

1. Mergear o PR da F1 e confirmar o "Main Branch Build" verde.
2. Disparar o rascunho:
   `gh workflow run release.yml -R devrafaelbrauner/handy-live-pt-cloud`
   O workflow cria o rascunho `v0.9.6-1` e anexa o NSIS e o MSI. É um build limpo, sem cache, então demora mais que os 28 min do "Main Branch Build".
3. Conferir os anexos:
   - baixar cada um;
   - gerar o hash com `Get-FileHash -Algorithm SHA256`;
   - instalar o NSIS do rascunho num PC limpo.
4. Notas do release em pt-BR:
   - novidade: transcrição na nuvem com chave de API;
   - aviso do SmartScreen e hashes SHA-256;
   - requisitos: Windows 10/11 x64 e WebView2 (o instalador baixa se faltar, então precisa de internet).
5. Publicar o rascunho.

### F4 — Depois do release (opcional, conforme as decisões)

- **Atualização automática própria:**
  1. gerar o par de chaves com `bun tauri signer generate -w ~/.tauri/handy-live-pt-cloud.key`;
  2. colocar a chave pública em `plugins.updater.pubkey` e ligar `createUpdaterArtifacts`;
  3. cadastrar os segredos `TAURI_SIGNING_PRIVATE_KEY` e `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`;
  4. ajustar o build sem assinatura de código para repassar essas duas variáveis. Hoje ele não repassa nenhuma variável de assinatura.

  A partir daí cada release publica um `latest.json` e o app atualiza sozinho.

- **Assinatura de código:**
  - Certum "Open Source Code Signing in the Cloud": US$ 58 por ano em certum.store, emitido para pessoa física com o prefixo "Open Source Developer", validade de 1 ano, chave na nuvem SimplySign (app no celular e SimplySign Desktop).
  - A assinatura é feita com `signtool` num Windows com SimplySign conectado, como passo manual entre o rascunho e a publicação. A automação fica para depois.
  - A reputação no SmartScreen cresce com os downloads; o aviso pode continuar nas primeiras versões.
- **winget:** só depois de URLs estáveis e, de preferência, com assinatura.

## Decisões suas

1. **Assinatura**
   - (a) Sem assinatura agora, para uso próprio e nas máquinas da unidade. **Recomendado.**
   - (b) Certum Open Source, US$ 58 por ano.
   - O Azure Artifact Signing (antigo Trusted Signing) não atende pessoa física no Brasil: a confiança pública é só para organizações dos EUA, Canadá, União Europeia e Reino Unido.
   - A SignPath Foundation (gratuita) exige reputação pública do projeto e um fork visível no GitHub, e este repositório ainda não tem nenhum dos dois.
2. **Arquiteturas:** só x64 (**recomendado**) ou x64 + ARM64. O ARM64 roda só em CPU.
3. **Instaladores:** publicar NSIS e MSI (**recomendado**) ou só NSIS.
   - O NSIS instala por usuário, sem admin, e oferece modo portátil.
   - O MSI instala para a máquina (exige admin) e serve para implantação gerenciada.
4. **Versão:** esquema `0.9.6-1` (**recomendado**) ou outro sufixo numérico.
5. **Atualização automática:**
   - só redirecionar as URLs agora e montar o canal próprio na F4 (**recomendado**);
   - ou montar o canal já no primeiro release.
6. **WebView2:**
   - padrão: o instalador baixa se faltar (**recomendado**);
   - `offlineInstaller`, cerca de +130 MB, para PCs sem internet.

## Riscos

- **Antivírus:** SmartScreen e Defender podem sinalizar um app sem assinatura que instala hook global de teclado e cola texto simulando o teclado. Se acontecer, enviar o arquivo em https://www.microsoft.com/wdsi/filesubmission.
- **Whisper no Windows:** o README do upstream registra travamentos do Whisper em algumas configurações de Windows. Testar com Vulkan e com CPU; as alternativas são Parakeet (CPU) ou a transcrição na nuvem.
- **Atalho:** `Ctrl+Space` colide com o Handy oficial se os dois estiverem abertos.
- **Prazo dos artefatos:** os artefatos do CI expiram em 30 dias. O que vale para distribuição é o anexo do release.
