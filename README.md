# AudioBridge

Mesa de monitoração ao vivo em software:
- junta várias interfaces de áudio num único dispositivo agregado;
- monta um mix de fone por músico, no estilo do Mixing Station;
- roda plugins VST3/AU por canal.

A DAW continua gravando em paralelo, sem bagunçar a sessão.

- **Design (fonte da verdade):** [`docs/superpowers/specs/2026-11-09-audiobridge-design.md`](docs/superpowers/specs/2026-11-09-audiobridge-design.md)
- **Plano da fase atual:** [`docs/superpowers/plans/2026-11-09-fase0-fundacao.md`](docs/superpowers/plans/2026-11-09-fase0-fundacao.md)
- **Status:** Fase 0 (fundação) em andamento.
- **Licença:** AGPLv3. O projeto usa o JUCE, que tem licença dupla AGPLv3/comercial. Uma versão comercial de código fechado pode exigir licença comercial do JUCE.

---

## Como colaborar

Dois papéis, os dois trabalhando com o Claude Code:

| Quem | Máquina | Papel |
|---|---|---|
| **Gabriel** | Windows | Desenvolvedor principal e dono do `main`. Revisa e faz merge dos PRs, publica releases. |
| **Usuário piloto** | Mac | Decide o produto e testa com as placas reais. Compila local, corrige o que é específico de Mac e abre PRs. |

### Regras

1. **Nunca commitar direto no `main`.** Toda mudança vai numa branch e entra por Pull Request.
   - Nomes de branch: `feat/fase<N>-<assunto>`, `fix/<assunto>`, `docs/<assunto>`.
2. **Uma tarefa do plano por PR.** O título segue o formato `Fase N · Tarefa M — <resumo>`. Combinem antes quem pega qual tarefa, para dois Claudes não mexerem no mesmo arquivo ao mesmo tempo.
3. **Comece sempre pelo `main` atualizado.** Rode `git checkout main && git pull` e só então crie a branch.
4. **A spec e o plano são a fonte da verdade.** Uma decisão nova de produto ou de arquitetura entra na spec, no mesmo PR ou num PR `docs/`. Decisão de negócio é do usuário piloto; decisão técnica é combinada entre os dois.
5. **Nada é "pronto" sem evidência.** A descrição do PR precisa dizer:
   - o que foi testado;
   - em qual máquina: modelo do Mac, versão do macOS, Intel ou Apple Silicon, quais placas;
   - o resultado: trecho do `AudioBridge.log`, print ou saída dos testes.
6. **O repositório é público.** Não commitar:
   - nomes de pessoas (usar "usuário piloto");
   - vídeos ou áudios (o `.gitignore` já bloqueia `*.mp4` e `*.mov`);
   - senhas, tokens ou logs com dados pessoais.
7. **Releases e tags (`v*`) só o Gabriel publica.**
8. **Idioma:** documentação, commits, PRs e textos da UI em português (pt-BR). Identificadores no código, em inglês.

### Fluxo de um bug encontrado no Mac

1. Reproduzir com o app compilado localmente e guardar o `AudioBridge.log` (botão "Abrir log").
2. Pedir ao Claude para investigar com o skill `superpowers:systematic-debugging`.
3. Corrigir numa branch `fix/...`, compilar, rodar os testes e testar de novo com as placas.
4. Abrir o PR com a evidência e avisar o Gabriel.

---

## Setup no Mac (usuário piloto)

### Parte 1: o humano faz (uma vez só)

Coisas que pedem senha, cliques ou login e que o Claude não consegue fazer sozinho:

1. **Conta no GitHub.** Crie uma conta em github.com, se ainda não tiver, e mande o seu nome de usuário para o Gabriel. Ele vai te adicionar como colaborador do repositório. Aceite o convite que chega por e-mail.
2. **Instalar o Claude Code.** No app **Terminal**:
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```
   Feche e abra o Terminal de novo. Rode `claude` e faça login com a sua conta Claude.
3. **Ferramentas de desenvolvedor da Apple.** No Terminal:
   ```bash
   xcode-select --install
   ```
   Clique em **Instalar** na janela que aparecer e aguarde. Se o comando disser que já estão instaladas, siga em frente.
4. **Homebrew** (gerenciador de pacotes; pede a senha do Mac). No Terminal:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
   No final, o instalador mostra 2 ou 3 comandos em "Next steps" (`echo ... >> ~/.zprofile` e `eval ...`). Copie e rode esses comandos.
5. **Pedir ao Claude para fazer o resto.** No Terminal:
   ```bash
   mkdir -p ~/dev && cd ~/dev && claude
   ```
   E escreva para ele:
   > Clone https://github.com/bielfernandes-projects/audio-bridge.git em ~/dev/audio-bridge, leia o README.md inteiro e execute a seção "Parte 2: o Claude faz". Me avise do que eu precisar fazer manualmente.

> Não clone o projeto dentro de Mesa, Documentos ou de qualquer pasta do iCloud Drive. O build gera GBs de arquivos e o iCloud tentaria sincronizar tudo. Use `~/dev`.

### Parte 2: o Claude faz

Instruções para o Claude Code na máquina do usuário piloto. Execute em ordem, confira o resultado de cada passo e pare para pedir ajuda ao humano quando um passo exigir senha, clique ou login.

1. **Conferir os pré-requisitos da Parte 1.** Pare e peça ao humano para completar a Parte 1 se algum destes falhar:
   ```bash
   xcode-select -p     # esperado: /Library/Developer/CommandLineTools ou /Applications/Xcode.app/...
   brew --version      # esperado: Homebrew 4.x ou superior
   sw_vers; uname -m   # anote a versão do macOS e a arquitetura (arm64 = Apple Silicon, x86_64 = Intel)
   ```
   O macOS mínimo do projeto é o 12 (Monterey). Se for mais antigo, avise o humano e pare.
2. **Ferramentas de build:**
   ```bash
   brew install cmake node gh
   cmake --version     # esperado: 3.22 ou superior
   node --version      # esperado: v20 ou superior (o CI usa v24)
   ```
   Se o Homebrew falhar por causa da versão do macOS, instale manualmente:
   - o CMake pelo `.dmg` universal de cmake.org/download, depois `sudo "/Applications/CMake.app/Contents/bin/cmake-gui" --install`;
   - o Node LTS pelo `.pkg` de nodejs.org.

   Os dois pedem ação do humano.
3. **Login no GitHub.** Peça ao humano para rodar no Terminal (é interativo e abre o navegador):
   ```bash
   gh auth login       # GitHub.com → HTTPS → Login with a web browser
   ```
   Depois confira com `gh auth status`. Confira também que ele tem acesso de escrita:
   ```bash
   gh api repos/bielfernandes-projects/audio-bridge --jq .permissions.push   # esperado: true
   ```
   Se der `false`, o convite de colaborador ainda não foi aceito ou enviado. Avise o humano.
4. **Identidade do git:**
   ```bash
   git config --global user.name   # se vazio, pergunte o nome ao humano e configure
   git config --global user.email  # se vazio, use o e-mail da conta GitHub do humano
   ```
5. **Plugin superpowers** (skills de planejamento, TDD e debugging usados neste projeto):
   ```bash
   claude plugin marketplace add obra/superpowers-marketplace
   claude plugin install superpowers@superpowers-marketplace
   claude plugin list    # esperado: superpowers aparece habilitado
   ```
   Se o comando `claude plugin` não existir nesta versão, peça ao humano para digitar dentro do Claude Code:
   `/plugin marketplace add obra/superpowers-marketplace` e depois `/plugin install superpowers@superpowers-marketplace`.
   Os skills só carregam numa sessão nova. Avise o humano para sair (`/exit`) e abrir `claude` de novo dentro de `~/dev/audio-bridge`.
6. **Primeiro build** (só se `CMakeLists.txt` já existir no repo; senão, pule para o passo 7):
   ```bash
   cd ~/dev/audio-bridge
   (cd ui && npm ci && npm run build)
   cmake -B build -DCMAKE_BUILD_TYPE=Release
   cmake --build build --parallel
   ctest --test-dir build --output-on-failure
   open build/AudioBridge_artefacts/Release/AudioBridge.app
   ```
   O primeiro configure baixa o JUCE e leva alguns minutos. Quando o app abrir e pedir acesso ao microfone, o humano deve permitir.
7. **Relatório final ao humano.** Diga:
   - o que foi instalado (com versões);
   - modelo e macOS do Mac;
   - se o build e os testes passaram;
   - o que ficou pendente.

   Lembre o humano de mandar ao Gabriel o modelo do Mac e a versão do macOS. Esse é um dado pendente na spec.

### Dia a dia no Mac

```bash
cd ~/dev/audio-bridge && git checkout main && git pull
git checkout -b fix/<assunto>          # ou feat/fase<N>-<assunto>
# ... trabalho com o Claude ...
(cd ui && npm run build) && cmake --build build --parallel && ctest --test-dir build --output-on-failure
git push -u origin HEAD
gh pr create --fill                    # depois edite a descrição com a evidência do teste
```

---

## Compilar no Windows

Pré-requisitos: CMake 3.22+, Visual Studio Build Tools (carga "Desenvolvimento para desktop com C++"), Node 20+. Detalhes na Tarefa 0 do plano da Fase 0.

```powershell
cd ui; npm ci; npm run build; cd ..
cmake -B build
cmake --build build --config Release
ctest --test-dir build -C Release
.\build\AudioBridge_artefacts\Release\AudioBridge.exe
```

## Testar um build baixado (sem compilar)

Quando houver uma versão na página **Releases** do GitHub:

1. Baixe `AudioBridge-macos-universal.tar.gz` e dê dois cliques para extrair.
2. Arraste `AudioBridge.app` para Aplicativos e tente abrir. O macOS vai bloquear, porque o app ainda não tem assinatura paga da Apple.
3. Vá em **Ajustes do Sistema > Privacidade e Segurança**, role até "Segurança" e clique em **Abrir Mesmo Assim**. O botão vale por cerca de 1 hora depois da tentativa. Digite a senha.
4. Permita o acesso ao **microfone**. É assim que o app lê as entradas da placa.

### Checklist da Fase 0

- [ ] O app abre e mostra "AudioBridge v…".
- [ ] A tabela "Dispositivos" lista as placas plugadas com o número certo de entradas e saídas.
- [ ] Desplugar ou plugar uma placa atualiza a tabela sozinha.
- [ ] Teste de passagem: entrada da placa → fone, com buffer 128. Dá para ouvir 5 minutos sem estalos?
- [ ] A latência mostrada parece com o que se ouve?
- [ ] Entrada de uma placa e saída de outra (ex.: UMC404 → 2i2) funciona?
- [ ] Clique em "Abrir log" e mande o `AudioBridge.log` junto com prints de cada item.
