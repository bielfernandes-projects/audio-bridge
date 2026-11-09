# AudioBridge — Design v1 (aprovado em 2026-11-09)

## Context
O usuário piloto quer uma "mesa de som do ao vivo" em software, separada da DAW, que:
1. junte interfaces de áudio quaisquer (UMC404, 2i2, mesas como Ui24/X32...) como uma só (agregado);
2. monte um mix de fone por músico, como o Mixing Station (sends on fader), misturando entradas ao vivo e o
   playback da DAW (click/bases);
3. rode plugins VST3/AU de terceiros por canal (EQ, comp, Waves CLA-2A).

Enquanto isso, a DAW (Reaper/Logic) grava as mesmas entradas em paralelo, sem bagunçar a sessão de mix nem
gastar RAM com sends.
Dor demonstrada em vídeo (09/11): fazer os fones dentro do Reaper com buses e grupos é confuso e não escala
(40 tracks × 10 músicos).
Uso pessoal primeiro, produto depois. Mac primeiro, Windows depois, mesmo código.
Gabriel (Windows, sem Mac) desenvolve com o Claude por vibe coding. O usuário piloto testa no Mac dele e decide o negócio.
Este plano substitui a spec original do ChatGPT.

## Decisões fechadas (grill)
| Tema | Decisão |
|---|---|
| Stack | **JUCE 8 (C++) + UI web** (WebView do JUCE 8, React+TS). O JUCE traz I/O de áudio (CoreAudio/WASAPI/ASIO) e hospedagem VST3/AU com a janela do plugin, no Mac e no Windows. Rust+Tauri foi descartado porque hospedar plugins em Rust é imaturo. Go foi descartado porque o GC causa estalos. |
| Licença JUCE | AGPLv3 enquanto o repo for público e pessoal. Ao virar produto, avaliar a licença comercial (Starter/Indie). Os termos atuais precisam ser verificados nessa hora. |
| macOS mínimo | **12 Monterey**, binário universal (Intel + Apple Silicon). |
| Windows | Mesmo código para motor, mixer, plugins e UI. Multi-device e retorno da DAW (driver ASIO virtual) numa fase futura. |
| Retorno da DAW (v1) | **BlackHole 16ch** (grátis, notarizado, instalado pelo usuário piloto) entra no agregado. A DAW toca nos canais do BlackHole e o AudioBridge mixa esses canais. O macOS corrige o drift. |
| Driver próprio | Só na fase de produto, junto com a conta Apple Developer (US$ 99/ano). |
| Custo | Zero na fase pessoal. Repo **público** no GitHub, com CI grátis (`macos-15` fixo + `windows-latest`). |
| Latência | Monitoração por software. Buffer configurável e latência exibida, incluindo a latência de cada plugin no canal. |
| Mixes | **Só fones na v1**, sem L/R/PA. Cada mix é independente (não existe fader "principal" do canal afetando os músicos). Mix mono ou estéreo, atribuído a saídas físicas. |
| Por fonte em cada mix | Nível, pan, mute, solo, link estéreo de fontes e master por mix. Meters com clip. |
| Vistas | **Mix N** (um mix, todas as fontes) e **Canal → Sends** (uma fonte, todos os mixes), como no Mixing Station. |
| Plugins | Cadeia de inserts VST3/AU **por canal de entrada**, antes dos sends, com bypass e janela do plugin. |
| Cenas | Salvar e trocar a configuração inteira: agregado, canais, plugins e estado deles, mixes. |
| Fora da v1 | Grupos/DCA, mix L/R/PA, controle remoto (a UI web já facilita), driver próprio, Windows multi-device, plugins em mixes. |

## Como fica a máquina de trabalho
- **Gabriel (Windows):** compila e roda local (CMake + VS Build Tools + Node) com as placas dele via WASAPI/ASIO. Itera mixer, UI, plugins e cenas.
- **CI (GitHub Actions):** a cada push gera o `.app` universal (tar.gz, para não perder o bit de execução) e o `.exe`.
- **Usuário piloto:** baixa o artifact e abre via Ajustes > Privacidade e Segurança > "Abrir Mesmo Assim". Testa com o checklist da fase e manda o log exportado pelo app.
- **Pré-requisitos no Windows:** Visual Studio Build Tools (C++), CMake. Node 24, git e gh já existem.

## Arquitetura (poucos arquivos, sem abstração especulativa)
```
audio-bridge/
  CMakeLists.txt            JUCE via CPM/FetchContent, alvo app universal (macOS 12)
  ui/                       React+TS: Devices, Aggregate, Mixer (Mix N / Canal→Sends), Settings
  src/
    Main.cpp                app JUCE + janela com WebView + ponte UI↔motor (native functions/events)
    Devices.cpp             enumeração (AudioDeviceManager + propriedades CoreAudio no Mac)
    AggregateMac.mm         criar/editar/remover agregados (API C do CoreAudio), listeners de conexão
    Engine.cpp              callback: entradas → inserts por canal → mixes → saídas; xruns, CPU
    Mixer.h/.cpp            DSP puro (ganho, pan, mute/solo, link, master, peak), sem I/O, com testes
    Plugins.cpp             scan VST3/AU, carregar/instanciar, abrir editor, salvar estado
    Scene.cpp               modelo de cena (JSON) + salvar/carregar
  tests/MixerTests.cpp
  .github/workflows/build.yml
  docs/superpowers/specs/2026-11-09-audiobridge-design.md
```
Regras do motor:
- Nenhuma alocação ou lock no callback.
- Os parâmetros do mixer vão por snapshot lock-free (estrutura de capacidade fixa).
- Meters voltam por atômicos, e a UI faz polling a ~30 Hz.
- Trocar um plugin acontece fora do callback, com swap atômico da cadeia.

Regras do agregado:
- O app só toca em agregados com UID `com.audiobridge.*`. Os de terceiros aparecem somente leitura.
- O agregado é público (a DAW usa o mesmo).
- O usuário escolhe o clock master, e o drift fica ligado por padrão nos demais subdevices.
- 48 kHz por padrão.

Ao vivo:
- Se um device cair, o motor continua, os canais dele ficam em silêncio e reconectam sozinhos.
- Se um plugin travar ou falhar, ele entra em bypass e aparece um alerta.
- Tudo vai para o log.

## Fases (cada uma termina com build no CI e teste do usuário piloto)
0. **Spike/fundação.**
   - Repo + JUCE + WebView UI + CI (Mac universal + Windows).
   - Tela Devices com dados reais.
   - Passthrough: entrada → saída com buffer configurável e latência exibida.
   - **Go/no-go no Mac do usuário piloto.**
1. **Agregado (Mac).** Criar/editar/remover agregados próprios, subdevices (detectando o BlackHole), clock, drift, sample rate e desconexão.
2. **Motor + mixer.**
   - Fontes: entradas físicas + canais do BlackHole.
   - N mixes mono/estéreo atribuídos a saídas.
   - Toda a lista "Por fonte em cada mix".
   - xruns e CPU.
3. **UI do mixer.**
   - Vistas Mix N e Canal→Sends, faders verticais e meters com clip.
   - Tema escuro de console, nomes de canal e mix editáveis.
   - Indicadores de clock/SR e alertas.
4. **Plugins por canal.**
   - Scan VST3/AU, inserts por canal, bypass, janela do plugin e latência exibida.
   - Teste com plugin gratuito + um Waves do usuário piloto.
5. **Cenas + robustez ao vivo.** Salvar e trocar cenas (com o estado dos plugins), reconexão automática e "exportar log".

Futuro, cada item com spec própria: grupos/DCA, mix L/R, remoto na LAN, driver AudioBridge + assinatura, Windows multi-device + ASIO virtual.

## Primeiros passos após aprovação
1. Gravar este design em `docs/superpowers/specs/2026-11-09-audiobridge-design.md`, rodar `git init` e commitar.
2. Criar o repo público no GitHub via `gh` (com confirmação do Gabriel antes de publicar).
3. Invocar o skill writing-plans para o plano detalhado da **Fase 0**. Nada além da Fase 0 até o go/no-go.

## Verificação
- Testes unitários de `Mixer` e `Scene` (soma, lei de pan, mute/solo, link estéreo, serialização), rodando no Windows local e no CI.
- O CI precisa ficar verde nas duas plataformas, com artifacts gerados.
- Checklist manual do usuário piloto por fase, por exemplo na Fase 0: o app abre, lista as interfaces reais com os canais corretos e a entrada sai no fone sem estalo com buffer de 128.
- Evidência devolvida: print/vídeo + log exportado. Nada é declarado pronto sem o teste do usuário piloto.

## Fatos pendentes (não bloqueiam a Fase 0)
- Modelo do Mac do usuário piloto (Intel/M-series) e versão do macOS. No vídeo aparece "MacBook Pro"; o ano ainda não se sabe.
- Usuário piloto instalar o BlackHole 16ch antes da Fase 1 (ele já tem, aparece no vídeo).
- Quais plugins ele quer usar primeiro (para o teste da Fase 4).
