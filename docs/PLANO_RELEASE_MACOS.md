# Plano de release para macOS — handy-live-pt-cloud

> Estado: plano guardado para depois do release Windows; nada publicado. Base verificada em 05/10/2026: `main` em `3925ee8`, PR #1 (transcrição na nuvem) e PR #3 (release Windows).

## Objetivo

Publicar no GitHub Releases de `devrafaelbrauner/handy-live-pt-cloud` os DMGs macOS (Apple Silicon e Intel) da versão com transcrição na nuvem. Eles devem instalar num Mac sem nenhuma ferramenta de desenvolvimento.

## Onde estamos

- **O CI já gera DMGs sem assinatura.** O "Main Branch Build" do `main` ([run 37219339029](https://github.com/devrafaelbrauner/handy-live-pt-cloud/actions/runs/37219339029)) passou nos dois jobs macOS:

  | Job                                               | Artefato (expira em 03/11/2026)                                                 |
  | ------------------------------------------------- | ------------------------------------------------------------------------------- |
  | `macos-26` (Apple Silicon)                        | `handy-aarch64-apple-darwin`, 36 MB: `Handy Live PT_0.9.6_aarch64.dmg` e `.app` |
  | `macos-latest` (Intel, compilado em runner arm64) | `handy-x86_64-apple-darwin`, 41 MB                                              |

- **Assinatura ad-hoc, sem notarização.** O log do job arm64 mostra `Signature=adhoc` e "skipping app notarization, no APPLE_ID & APPLE_PASSWORD & APPLE_TEAM_ID…".
- **Biblioteca do ONNX Runtime.** No build sem assinatura, o passo "Disable library validation (unsigned macOS)" do `build.yml` acrescenta `com.apple.security.cs.disable-library-validation` a uma cópia do `Entitlements.plist`. Sem isso, o x86_64 não carrega o `libonnxruntime` empacotado. O hardened runtime continua ligado.
- **O CI só prova que o binário abre.** O passo "Verify macOS app launches (unsigned)" roda `handy --help`, que sai antes de o Tauri iniciar. Interface, microfone, Acessibilidade e colagem nunca foram testados num Mac.
- **O caminho assinado já existe.** O `build.yml` herdou do upstream a importação do certificado Developer ID, `APPLE_SIGNING_IDENTITY` e a notarização. Falta só a credencial.
- **Permissões usadas:** Microfone (`NSMicrophoneUsageDescription`) e Acessibilidade (atalhos globais e colagem).
- **Atalho padrão no macOS:** `Option+Space`.

## O que impede o release hoje

1. **Assinatura.**
   - Sem Developer ID, o Gatekeeper bloqueia a primeira abertura.
   - Desde o macOS 15, o Control-clique não libera mais o app ([Apple, ago/2024](https://developer.apple.com/news?id=saqachfa)). O usuário precisa ir em **Ajustes → Privacidade e Segurança → Abrir Mesmo Assim**, ou rodar `xattr -dr com.apple.quarantine "/Applications/Handy Live PT.app"`.
2. **No ad-hoc, a Acessibilidade se perde a cada versão.**
   - A identidade de um app ad-hoc é o cdhash, que muda em todo build.
   - A permissão continua aparecendo ligada, mas não vale mais, e o app fica em "Waiting…" (`BUILD.md`, "macOS Accessibility remains enabled after a local rebuild").
   - Cada atualização exige `tccutil reset Accessibility com.rafael.handylive` e conceder de novo. Com Developer ID isso não acontece.
3. **Matriz do `release.yml`.** O PR #3 deixou só `windows-latest`. Para o Mac, voltar `macos-26` (arm64) e `macos-latest` (x86_64) e decidir `sign-binaries`:
   - `false`: ad-hoc, o mesmo caminho do "Main Branch Build";
   - `true`: exige os segredos da M1. O flag vale para a matriz toda; nos jobs Windows ele só instala o `trusted-signing-cli` à toa, porque o fork não tem `signCommand`.
4. **Texto do pedido de microfone.** `src-tauri/Info.plist` diz "Request microphone access to transcribe audio locally". Isso deixa de ser verdade quando a nuvem está ligada.
5. **Documentação macOS herdada do upstream.**
   - `BUILD.md`: `osascript` e `tccutil` usam `com.pais.handy` e `/Applications/Handy.app`. No fork são `com.rafael.handylive` e `/Applications/Handy Live PT.app`.
   - README: cita `/Applications/Handy.app/Contents/MacOS/Handy`. No fork é `/Applications/Handy Live PT.app/Contents/MacOS/handy`.
6. **Versão no `Info.plist`.**
   - O `tauri-bundler` copia a versão `0.9.6-1` para `CFBundleShortVersionString` e `CFBundleVersion` (`macos/app.rs`). Esse formato foge dos inteiros separados por ponto que a Apple espera.
   - Não deve bloquear a distribuição fora da App Store (não verificado).
   - Para evitar, definir `bundle.macOS.bundleVersion` numérico (ex.: `0.9.6.1`).
7. **Endpoint personalizado na rede local (não verificado).** No macOS 15 ou mais novo, conectar a um endereço da LAN exige a permissão de Rede Local. O `Info.plist` não declara `NSLocalNetworkUsageDescription`. Só afeta quem usa um servidor de transcrição próprio na rede local.

## Fases

### M0 — Decisão de assinatura (você)

- **(a) Ad-hoc:** grátis. Aviso do Gatekeeper na primeira abertura e Acessibilidade de novo a cada versão. Serve para uso próprio.
- **(b) Apple Developer Program:** US$ 99 por ano, aceita pessoa física. Certificado "Developer ID Application" e notarização. Sem aviso de malware e sem perder permissões ao atualizar. **Recomendado** se o app for usado por outras pessoas.

### M1 — Credenciais (só na opção b)

1. Inscrever-se no Apple Developer Program.
2. Criar o certificado **Developer ID Application** e exportar o `.p12` com senha.
3. Criar uma senha de app para o Apple ID em appleid.apple.com.
4. Cadastrar os segredos:

   ```bash
   base64 -i developer-id.p12 | gh secret set APPLE_CERTIFICATE -R devrafaelbrauner/handy-live-pt-cloud
   gh secret set APPLE_CERTIFICATE_PASSWORD -R devrafaelbrauner/handy-live-pt-cloud
   gh secret set KEYCHAIN_PASSWORD -R devrafaelbrauner/handy-live-pt-cloud   # qualquer senha forte
   gh secret set APPLE_ID -R devrafaelbrauner/handy-live-pt-cloud
   gh secret set APPLE_PASSWORD -R devrafaelbrauner/handy-live-pt-cloud      # a senha de app
   gh secret set APPLE_TEAM_ID -R devrafaelbrauner/handy-live-pt-cloud
   ```

### M2 — PR "release macOS" (agente; um PR, sem merge)

1. `release.yml`: voltar `macos-26` e `macos-latest` à matriz; `sign-binaries` conforme a M0.
2. Na opção (b): `pr-test-build.yml` também com `sign-binaries: true`, para o teste usar o mesmo caminho do release.
3. `Info.plist`: novo texto do microfone, por exemplo "Handy Live PT uses the microphone to transcribe your speech. Audio stays on this Mac unless you turn on cloud transcription."
4. Opcional: `bundle.macOS.bundleVersion` numérico.
5. `BUILD.md` e README:
   - identificador e caminhos do fork;
   - seção "Instalar no macOS": qual DMG baixar (`aarch64` para Apple Silicon, `x64` para Intel), Gatekeeper, Microfone e Acessibilidade;
   - na opção (a), o `tccutil` depois de cada atualização.
6. Rodar os gates locais antes de abrir o PR: `bun run lint`, `bun run build`, `bun run format:check`, `bun run check:translations`, `cargo test`.

### M3 — Teste num Mac real

1. Gerar os DMGs do PR da M2:
   `gh workflow run "PR Test Build" -R devrafaelbrauner/handy-live-pt-cloud --ref <branch> -f pr_number=<n>`
2. Roteiro (anotar o resultado de cada item no PR, com prints):
   1. Baixar o DMG certo e arrastar o app para Aplicativos.
   2. Primeira abertura: na opção (a), "Abrir Mesmo Assim"; na (b), só o aviso de app baixado da internet.
   3. Conceder Microfone e Acessibilidade no onboarding.
   4. Ditar com `Option+Space`, segurando e tocando, no TextEdit, no Notas e no Chrome. O texto precisa ser colado.
   5. Modelo local: baixar o Whisper Small, conferir qual dispositivo de computação é usado (Metal ou CPU) e ditar em pt-BR.
   6. Nuvem: colar uma chave válida e ditar; depois uma chave inválida, com a janela fechada. A janela precisa abrir com o erro (PR #3).
   7. Barra de menus, sobreposição, "abrir ao iniciar sessão", histórico e "tentar novamente".
   8. Instalar uma versão nova por cima e conferir se a Acessibilidade continua valendo. Na opção (a), o esperado é falhar; documentar o `tccutil`.
   9. Se houver um Mac Intel, repetir 1 a 4 nele. O CI só testa o x86_64 com `--help`, via Rosetta.

### M4 — Release (você aprova cada passo)

1. Mergear o PR da M2 e confirmar o "Main Branch Build" verde.
2. `gh workflow run release.yml -R devrafaelbrauner/handy-live-pt-cloud` e conferir os DMGs no rascunho, com `shasum -a 256`.
3. Notas do release em pt-BR:
   - qual DMG baixar;
   - requisitos: Apple Silicon exige macOS 11 ou mais novo. O `minimumSystemVersion` declara 10.15, mas o Intel nunca foi testado nessa versão; conferir antes de prometer;
   - instruções do Gatekeeper e hashes SHA-256.
4. Publicar o rascunho.

### M5 — Depois (opcional)

- **Atualização automática:** só com Developer ID. Com ad-hoc, cada atualização quebraria a Acessibilidade em silêncio. Os passos são os da F4 do plano Windows: chave do atualizador, `createUpdaterArtifacts` e os segredos `TAURI_SIGNING_PRIVATE_KEY` e `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`.
- **Homebrew cask próprio:** só depois de URLs estáveis e assinatura.

## Decisões suas

1. **Assinatura:** ad-hoc (uso próprio) ou Developer ID (US$ 99 por ano; **recomendado** se o app sair das suas máquinas).
2. **Arquiteturas:** Apple Silicon e Intel (o que o CI já monta, **recomendado**) ou só Apple Silicon.
3. **Atualização automática no Mac:** só depois do Developer ID (**recomendado**).

## Riscos

- O x86_64 é compilado num runner arm64 e só testado com `--help` via Rosetta.
- No build ad-hoc, `disable-library-validation` aceita bibliotecas de qualquer assinatura. O hardened runtime continua bloqueando injeção por `DYLD_*`.
- Microfone de headset Bluetooth derruba a qualidade do áudio enquanto grava (README upstream).
- Atalhos com `fn`/Globe só funcionam em teclado Apple (README upstream).
