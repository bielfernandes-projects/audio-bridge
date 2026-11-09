# AudioBridge — Fase 0 (Fundação) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ter um app JUCE + UI web que compila no CI para macOS (universal) e Windows. Ele lista os dispositivos de áudio reais e faz uma passagem entrada → fone com buffer configurável e latência exibida, para o usuário piloto dar o go/no-go da stack no Mac dele.

**Architecture:**
- **Motor e app:** um app JUCE 9 em C++ tem uma janela com `WebBrowserComponent` (WKWebView no Mac, WebView2 no Windows). Essa janela serve uma UI React compilada num único `index.html` embutido como BinaryData.
- **Ponte UI ↔ motor:** a UI chama o C++ por "native functions" do JUCE, e o C++ empurra eventos (`meter`, `devicesChanged`) para a UI.
- **Áudio:** um `AudioDeviceManager` abre a entrada e a saída escolhidas. O callback copia um canal de entrada para 1–2 canais de saída, sem alocação nem lock.

**Tech Stack:** JUCE 9.0.5 (CMake + FetchContent), C++17, React 19 + TypeScript + Vite 7 + vite-plugin-singlefile, GitHub Actions (`macos-15`, `windows-latest`).

**Spec:** `docs/superpowers/specs/2026-11-09-audiobridge-design.md`

## Global Constraints

- macOS mínimo **12.0**, binário universal **arm64;x86_64**.
- JUCE **9.0.5** (tag exata). Licença do projeto: **AGPLv3** (o JUCE é AGPLv3/comercial; o repo é público). Vender com código fechado pode exigir licença comercial do JUCE.
- WebView2 SDK **1.0.3485.44** (Windows), baixado automaticamente pelo CMake.
- Callback de áudio: **nenhuma alocação, lock, log ou chamada bloqueante**. Comunicação só por `std::atomic`.
- **Nada de dados simulados**: se o backend nativo não existir (UI aberta num browser comum), a UI mostra erro, nunca dados falsos.
- Textos da UI em **pt-BR**. Strings C++ com acento só via `juce::String::fromUTF8(...)`. MSVC compila com `/utf-8`.
- A UI precisa rodar no WebKit do macOS 12 (Safari 15): `build.target = ["es2020", "safari15"]`.
- Nome do usuário piloto **não** aparece em arquivos do repo (o repo é público): usar "usuário piloto".
- Commits terminam com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## File Structure

| Arquivo | Responsabilidade |
|---|---|
| `CMakeLists.txt` | Busca o JUCE e o WebView2, define o app, embute a UI e define os testes. |
| `LICENSE` | Texto da AGPLv3. |
| `README.md` | Build local, instruções do usuário piloto e checklist da Fase 0. |
| `src/AudioMath.h` | Funções puras sem JUCE: latência de ida e volta, índice compactado de canal. |
| `src/Main.cpp` | App JUCE, janela, `MainComponent` (WebView + ponte + timer de meter + logger). |
| `src/Devices.h/.cpp` | `listDevices()`: todos os devices de todos os drivers, com dados reais. |
| `src/Engine.h/.cpp` | Passthrough: abre o device, callback de áudio, pico, xruns, CPU, latência. |
| `tests/AudioMathTests.cpp` | Testes de `AudioMath.h` (executável simples + `ctest`). |
| `ui/` | `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/{main.tsx,App.tsx,juce.ts,types.ts,Devices.tsx,Passthrough.tsx,styles.css}`. |
| `.github/workflows/build.yml` | CI Mac universal + Windows, artifacts e release em tag `v*`. |

## Contrato da ponte (usado nas Tasks 2–4)

```ts
// native functions (UI → C++)
ping(): string                                      // versão do app
listDevices(): DeviceInfo[]
startPassthrough(req: PassthroughRequest): PassthroughStatus
stopPassthrough(): null
revealLog(): string                                 // caminho do log, abre no Finder/Explorer

// eventos (C++ → UI)
"devicesChanged" -> null                            // lista de devices mudou
"meter"          -> Meter                           // ~30 Hz

type DeviceInfo = { type: string; name: string; inputs: string[]; outputs: string[];
                    sampleRates: number[]; bufferSizes: number[]; defaultBufferSize: number };
type PassthroughRequest = { type: string; inputDevice: string; outputDevice: string;
                            sampleRate: number; bufferSize: number;
                            inputChannel: number; outputLeft: number; outputRight: number }; // outputRight -1 = mono
type PassthroughStatus = { ok: boolean; error: string; sampleRate: number; bufferSize: number;
                           inputLatency: number; outputLatency: number; roundTripMs: number };
type Meter = { peak: number; xruns: number; cpu: number; running: boolean };
```

---

### Task 0: Pré-requisitos na máquina Windows (feito pelo Gabriel)

**Files:** nenhum.

- [ ] **Step 1: Instalar CMake e Visual Studio Build Tools (C++)** num PowerShell como usuário normal (o instalador do VS pede UAC):

```powershell
winget install --id Kitware.CMake -e
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --override "--wait --passive --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

- [ ] **Step 2: Abrir um terminal novo e conferir**

Run: `cmake --version`
Expected: `cmake version 3.2x` ou superior (mínimo 3.22).

Run: `& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -products * -property displayName`
Expected: uma linha com `Visual Studio Build Tools 2022`.

- [ ] **Step 3 (recomendado): mover o repo para fora do OneDrive.** O build do JUCE gera GBs em `build/` e `ui/node_modules`, e o OneDrive tentaria sincronizar tudo:

```powershell
git clone https://github.com/bielfernandes-projects/audio-bridge.git C:\dev\audio-bridge
```

Os vídeos de referência ficam onde estão, porque o `.gitignore` já ignora `*.mp4`.

---

### Task 1: Esqueleto CMake + JUCE + testes puros + licença

**Files:**
- Create: `CMakeLists.txt`, `src/AudioMath.h`, `src/Main.cpp`, `tests/AudioMathTests.cpp`, `LICENSE`
- Modify: `.gitignore` (adicionar `build*/`)

**Interfaces:**
- Produces: `struct LatencyInfo { int inputSamples; int outputSamples; double sampleRate; }`, `double roundTripMs(const LatencyInfo&)`, `int compactIndex(const std::vector<int>& activeChannels, int channel)`. O target CMake se chama `AudioBridge`.

- [ ] **Step 1: Escrever o teste que falha** — `tests/AudioMathTests.cpp`:

```cpp
#include "AudioMath.h"
#include <cmath>
#include <cstdio>

static int failures = 0;
#define CHECK(cond) do { if (!(cond)) { std::printf("FALHOU: %s (linha %d)\n", #cond, __LINE__); ++failures; } } while (0)

int main()
{
    // O JUCE já soma buffer + safety offset + latência do device em cada direção.
    CHECK (std::abs (roundTripMs ({ 240, 240, 48000.0 }) - 10.0) < 1e-9);
    CHECK (roundTripMs ({ 100, 100, 0.0 }) == 0.0);

    // O JUCE entrega só os canais ativos, compactados em ordem crescente.
    CHECK (compactIndex ({ 3, 5 }, 3) == 0);
    CHECK (compactIndex ({ 3, 5 }, 5) == 1);
    CHECK (compactIndex ({ 5, 3 }, 5) == 1);   // a ordem escolhida pelo usuário não importa
    CHECK (compactIndex ({ 3, 5 }, 4) == -1);  // canal não ativo
    CHECK (compactIndex ({ 2, 2 }, 2) == 0);   // L e R no mesmo canal

    if (failures == 0)
        std::puts ("AudioMathTests: OK");
    return failures == 0 ? 0 : 1;
}
```

- [ ] **Step 2: Escrever `CMakeLists.txt`**

```cmake
cmake_minimum_required(VERSION 3.22)

set(CMAKE_OSX_DEPLOYMENT_TARGET "12.0" CACHE STRING "macOS minimo")
set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "Binario universal")

project(AudioBridge VERSION 0.0.1 LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# --- Testes puros (sem JUCE) ---
enable_testing()
add_executable(AudioMathTests tests/AudioMathTests.cpp)
target_include_directories(AudioMathTests PRIVATE src)
add_test(NAME AudioMathTests COMMAND AudioMathTests)

# --- WebView2 (Windows): baixa o pacote NuGet uma vez para dentro do build ---
if(WIN32)
    set(WEBVIEW2_VERSION 1.0.3485.44)
    set(WEBVIEW2_DIR "${CMAKE_BINARY_DIR}/webview2/Microsoft.Web.WebView2.${WEBVIEW2_VERSION}")
    if(NOT EXISTS "${WEBVIEW2_DIR}/build/native/include/WebView2.h")
        file(DOWNLOAD "https://www.nuget.org/api/v2/package/Microsoft.Web.WebView2/${WEBVIEW2_VERSION}"
             "${CMAKE_BINARY_DIR}/webview2.zip" STATUS dl_status)
        list(GET dl_status 0 dl_code)
        if(NOT dl_code EQUAL 0)
            message(FATAL_ERROR "Falha ao baixar WebView2: ${dl_status}")
        endif()
        file(ARCHIVE_EXTRACT INPUT "${CMAKE_BINARY_DIR}/webview2.zip" DESTINATION "${WEBVIEW2_DIR}")
    endif()
    set(JUCE_WEBVIEW2_PACKAGE_LOCATION "${CMAKE_BINARY_DIR}/webview2" CACHE PATH "" FORCE)
endif()

# --- JUCE ---
include(FetchContent)
FetchContent_Declare(JUCE
    GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
    GIT_TAG 9.0.5
    GIT_SHALLOW TRUE)
FetchContent_MakeAvailable(JUCE)

juce_add_gui_app(AudioBridge
    PRODUCT_NAME "AudioBridge"
    COMPANY_NAME "AudioBridge"
    BUNDLE_ID "com.audiobridge.app"
    VERSION ${PROJECT_VERSION}
    MICROPHONE_PERMISSION_ENABLED TRUE
    MICROPHONE_PERMISSION_TEXT "O AudioBridge usa as entradas de áudio para a monitoração dos músicos."
    NEEDS_WEBVIEW2 TRUE)

target_sources(AudioBridge PRIVATE src/Main.cpp)

target_compile_definitions(AudioBridge PRIVATE
    JUCE_WEB_BROWSER=1
    JUCE_USE_CURL=0
    JUCE_USE_WIN_WEBVIEW2_WITH_STATIC_LINKING=1
    JUCE_APPLICATION_NAME_STRING="$<TARGET_PROPERTY:AudioBridge,JUCE_PRODUCT_NAME>"
    JUCE_APPLICATION_VERSION_STRING="$<TARGET_PROPERTY:AudioBridge,JUCE_VERSION>")

if(MSVC)
    target_compile_options(AudioBridge PRIVATE /utf-8)
endif()

target_link_libraries(AudioBridge
    PRIVATE juce::juce_audio_utils juce::juce_gui_extra
    PUBLIC juce::juce_recommended_config_flags juce::juce_recommended_warning_flags)
```

- [ ] **Step 3: Rodar o teste e ver falhar**

Run: `cmake -B build && cmake --build build --config Release --target AudioMathTests`
Expected: FAIL de compilação, `AudioMath.h: No such file or directory`. O configure também baixa o JUCE e o WebView2 (alguns minutos na primeira vez).

- [ ] **Step 4: Implementar `src/AudioMath.h`**

```cpp
#pragma once
#include <vector>

// Latências como o JUCE reporta: cada direção já inclui buffer + safety offset + latência do device.
struct LatencyInfo
{
    int inputSamples = 0;
    int outputSamples = 0;
    double sampleRate = 0;
};

inline double roundTripMs (const LatencyInfo& l)
{
    if (l.sampleRate <= 0)
        return 0;
    return (l.inputSamples + l.outputSamples) * 1000.0 / l.sampleRate;
}

// O JUCE entrega ao callback só os canais ativos, compactados em ordem crescente.
// Retorna a posição de `channel` nesse array compactado, ou -1 se ele não estiver ativo.
inline int compactIndex (const std::vector<int>& activeChannels, int channel)
{
    bool found = false;
    int below = 0;
    for (int c : activeChannels)
    {
        if (c == channel)
            found = true;
        else if (c < channel)
            ++below;
    }
    return found ? below : -1;
}
```

- [ ] **Step 5: Implementar `src/Main.cpp` mínimo** (janela vazia; a Task 2 troca o conteúdo):

```cpp
#include <juce_gui_extra/juce_gui_extra.h>

class MainWindow final : public juce::DocumentWindow
{
public:
    explicit MainWindow (const juce::String& name)
        : DocumentWindow (name, juce::Colour (0xff15171a), DocumentWindow::allButtons)
    {
        setUsingNativeTitleBar (true);
        auto* label = new juce::Label ({}, "AudioBridge");
        label->setJustificationType (juce::Justification::centred);
        label->setSize (1100, 720);
        setContentOwned (label, true);
        setResizable (true, true);
        centreWithSize (getWidth(), getHeight());
        setVisible (true);
    }

    void closeButtonPressed() override { juce::JUCEApplication::getInstance()->systemRequestedQuit(); }
};

class AudioBridgeApp final : public juce::JUCEApplication
{
public:
    const juce::String getApplicationName() override    { return JUCE_APPLICATION_NAME_STRING; }
    const juce::String getApplicationVersion() override { return JUCE_APPLICATION_VERSION_STRING; }
    void initialise (const juce::String&) override      { window = std::make_unique<MainWindow> (getApplicationName()); }
    void shutdown() override                            { window = nullptr; }

private:
    std::unique_ptr<MainWindow> window;
};

START_JUCE_APPLICATION (AudioBridgeApp)
```

- [ ] **Step 6: Rodar os testes e o build completo**

Run: `cmake --build build --config Release && ctest --test-dir build -C Release --output-on-failure`
Expected: `AudioMathTests: OK`, `100% tests passed`. O app existe em `build/AudioBridge_artefacts/Release/AudioBridge.exe`, e ao abrir aparece uma janela escura escrita "AudioBridge".

- [ ] **Step 7: Licença e .gitignore**

```bash
curl -sL https://www.gnu.org/licenses/agpl-3.0.txt -o LICENSE
printf 'build*/\n' >> .gitignore
```

- [ ] **Step 8: Commit**

```bash
git add CMakeLists.txt src/AudioMath.h src/Main.cpp tests/AudioMathTests.cpp LICENSE .gitignore
git commit -m "feat: esqueleto JUCE 9 + testes de AudioMath

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: UI web embutida + ponte `ping`

**Files:**
- Create: `ui/package.json`, `ui/vite.config.ts`, `ui/tsconfig.json`, `ui/index.html`, `ui/src/main.tsx`, `ui/src/App.tsx`, `ui/src/juce.ts`, `ui/src/styles.css`
- Modify: `CMakeLists.txt` (binary data da UI), `src/Main.cpp` (MainComponent com WebView)

**Interfaces:**
- Consumes: target `AudioBridge` (Task 1).
- Produces: `call<T>(name, ...params): Promise<T>`, `on(event, fn)`, `backendAvailable(): boolean` em `ui/src/juce.ts`. A classe `MainComponent` em `src/Main.cpp` tem o método `makeOptions()`, onde as próximas tasks registram native functions.

- [ ] **Step 1: Criar `ui/package.json`**

```json
{
  "name": "audiobridge-ui",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^5.0.0",
    "typescript": "^5.9.0",
    "vite": "^7.0.0",
    "vite-plugin-singlefile": "^2.3.4"
  }
}
```

- [ ] **Step 2: Criar `ui/vite.config.ts`, `ui/tsconfig.json`, `ui/index.html`, `ui/src/main.tsx`**

```ts
// ui/vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { viteSingleFile } from "vite-plugin-singlefile";

// Um único index.html (JS/CSS inline) para embutir no app. Alvo: WebKit do macOS 12.
export default defineConfig({
  plugins: [react(), viteSingleFile()],
  build: { target: ["es2020", "safari15"] },
});
```

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

```html
<!doctype html>
<html lang="pt-BR">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>AudioBridge</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

```tsx
// ui/src/main.tsx
import { createRoot } from "react-dom/client";
import { App } from "./App";
import "./styles.css";

createRoot(document.getElementById("root")!).render(<App />);
```

- [ ] **Step 3: Criar `ui/src/juce.ts`.** É o mesmo protocolo de `@juce-framework/webview` (eventos `__juce__invoke`/`__juce__complete`), sem a dependência:

```ts
type Backend = {
  emitEvent(id: string, payload: unknown): void;
  addEventListener(id: string, fn: (payload: any) => void): unknown;
};

declare global {
  interface Window {
    __JUCE__?: { backend: Backend };
  }
}

let nextId = 0;
let listening = false;
const pending = new Map<number, (value: unknown) => void>();

export function backendAvailable(): boolean {
  return !!window.__JUCE__?.backend;
}

export function call<T>(name: string, ...params: unknown[]): Promise<T> {
  const backend = window.__JUCE__?.backend;
  if (!backend) return Promise.reject(new Error("Backend nativo indisponível: abra pelo app AudioBridge."));
  if (!listening) {
    backend.addEventListener("__juce__complete", ({ promiseId, result }) => {
      pending.get(promiseId)?.(result);
      pending.delete(promiseId);
    });
    listening = true;
  }
  const id = nextId++;
  return new Promise<T>((resolve) => {
    pending.set(id, resolve as (value: unknown) => void);
    backend.emitEvent("__juce__invoke", { name, params, resultId: id });
  });
}

export function on(event: string, fn: (payload: any) => void): void {
  window.__JUCE__?.backend.addEventListener(event, fn);
}
```

- [ ] **Step 4: Criar `ui/src/App.tsx` (versão mínima) e `ui/src/styles.css`**

```tsx
import { useEffect, useState } from "react";
import { backendAvailable, call } from "./juce";

export function App() {
  const [version, setVersion] = useState("");
  const [error, setError] = useState("");

  useEffect(() => {
    call<string>("ping").then(setVersion, (e) => setError(String(e)));
  }, []);

  return (
    <main>
      <header>
        <h1>AudioBridge</h1>
        {version && <span className="muted">v{version}</span>}
      </header>
      {!backendAvailable() && <p className="error">Backend nativo indisponível: abra pelo app AudioBridge.</p>}
      {error && <p className="error">{error}</p>}
    </main>
  );
}
```

```css
:root {
  --bg: #15171a; --panel: #1f2226; --line: #2e3238; --text: #e6e8eb;
  --muted: #8b929b; --accent: #3ddc84; --warn: #ffb020; --clip: #ff4d4f;
  color-scheme: dark;
}
* { box-sizing: border-box; }
body { margin: 0; background: var(--bg); color: var(--text); font: 14px/1.4 -apple-system, "Segoe UI", sans-serif; }
main { padding: 16px 20px; display: grid; gap: 16px; }
header { display: flex; align-items: baseline; gap: 12px; }
h1 { font-size: 20px; margin: 0; }
h2 { font-size: 15px; margin: 0 0 8px; }
section { background: var(--panel); border: 1px solid var(--line); border-radius: 8px; padding: 12px; }
table { width: 100%; border-collapse: collapse; }
th, td { text-align: left; padding: 6px 8px; border-bottom: 1px solid var(--line); }
th { color: var(--muted); font-weight: 500; }
label { display: grid; gap: 4px; color: var(--muted); font-size: 12px; }
select, button { background: var(--bg); color: var(--text); border: 1px solid var(--line); border-radius: 6px; padding: 6px 8px; font: inherit; }
button { cursor: pointer; }
button.primary { background: var(--accent); color: #0b0d0f; border: none; font-weight: 600; }
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr)); gap: 10px; }
.row { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; margin-top: 10px; }
.muted { color: var(--muted); }
.error { color: var(--clip); }
.meter { height: 14px; width: 320px; background: var(--bg); border: 1px solid var(--line); border-radius: 4px; overflow: hidden; }
.meter > div { height: 100%; background: var(--accent); }
.meter.clip > div { background: var(--clip); }
```

- [ ] **Step 5: Instalar e compilar a UI**

Run: `cd ui && npm install && npm run build && ls dist`
Expected: `dist/index.html` existe e é o **único** arquivo em `dist/` (nada de `assets/`). Se o `vite-plugin-singlefile` reclamar da versão do vite, fixar `vite@7` e `@vitejs/plugin-react@5` (já são os ranges acima) e rodar `npm install` de novo.

- [ ] **Step 6: Embutir a UI no app — adicionar ao `CMakeLists.txt`, logo antes de `target_link_libraries(AudioBridge ...)`**

```cmake
if(NOT EXISTS "${CMAKE_SOURCE_DIR}/ui/dist/index.html")
    message(FATAL_ERROR "UI nao compilada. Rode antes: cd ui && npm ci && npm run build")
endif()
juce_add_binary_data(AudioBridgeUI SOURCES ui/dist/index.html)
target_link_libraries(AudioBridge PRIVATE AudioBridgeUI)
```

- [ ] **Step 7: Substituir `src/Main.cpp` inteiro (MainComponent com WebView)**

```cpp
#include <juce_gui_extra/juce_gui_extra.h>
#include <BinaryData.h>

namespace
{
    using Args = juce::Array<juce::var>;
    using Completion = juce::WebBrowserComponent::NativeFunctionCompletion;

    std::optional<juce::WebBrowserComponent::Resource> serveUi (const juce::String& path)
    {
        if (path != "/" && path != "/index.html")
            return std::nullopt;
        const auto* bytes = reinterpret_cast<const std::byte*> (BinaryData::index_html);
        return juce::WebBrowserComponent::Resource { { bytes, bytes + BinaryData::index_htmlSize }, "text/html" };
    }
}

class MainComponent final : public juce::Component
{
public:
    MainComponent()
    {
        addAndMakeVisible (web);
        web.goToURL (juce::WebBrowserComponent::getResourceProviderRoot());
        setSize (1100, 720);
    }

    void resized() override { web.setBounds (getLocalBounds()); }

private:
    juce::WebBrowserComponent::Options makeOptions()
    {
        return juce::WebBrowserComponent::Options {}
            .withBackend (juce::WebBrowserComponent::Options::Backend::webview2)
            .withWinWebView2Options (juce::WebBrowserComponent::Options::WinWebView2 {}
                .withUserDataFolder (juce::File::getSpecialLocation (juce::File::tempDirectory).getChildFile ("AudioBridgeWebView")))
            .withNativeIntegrationEnabled()
            .withResourceProvider ([] (const auto& path) { return serveUi (path); })
            .withNativeFunction ("ping", [] (const Args&, Completion done) { done (juce::String (JUCE_APPLICATION_VERSION_STRING)); });
    }

    juce::WebBrowserComponent web { makeOptions() };
};

class MainWindow final : public juce::DocumentWindow
{
public:
    explicit MainWindow (const juce::String& name)
        : DocumentWindow (name, juce::Colour (0xff15171a), DocumentWindow::allButtons)
    {
        setUsingNativeTitleBar (true);
        setContentOwned (new MainComponent(), true);
        setResizable (true, true);
        centreWithSize (getWidth(), getHeight());
        setVisible (true);
    }

    void closeButtonPressed() override { juce::JUCEApplication::getInstance()->systemRequestedQuit(); }
};

class AudioBridgeApp final : public juce::JUCEApplication
{
public:
    const juce::String getApplicationName() override    { return JUCE_APPLICATION_NAME_STRING; }
    const juce::String getApplicationVersion() override { return JUCE_APPLICATION_VERSION_STRING; }
    void initialise (const juce::String&) override      { window = std::make_unique<MainWindow> (getApplicationName()); }
    void shutdown() override                            { window = nullptr; }

private:
    std::unique_ptr<MainWindow> window;
};

START_JUCE_APPLICATION (AudioBridgeApp)
```

- [ ] **Step 8: Build e verificação manual**

Run: `cmake -B build && cmake --build build --config Release && ctest --test-dir build -C Release`
Expected: build OK e testes OK. Ao abrir `build/AudioBridge_artefacts/Release/AudioBridge.exe`, a janela mostra "AudioBridge **v0.0.1**" em tema escuro, sem a mensagem de erro. Se aparecer "Backend nativo indisponível", a injeção do `__JUCE__` falhou: conferir `withNativeIntegrationEnabled()` e `JUCE_USE_WIN_WEBVIEW2_WITH_STATIC_LINKING=1`.

- [ ] **Step 9: Commit**

```bash
git add ui CMakeLists.txt src/Main.cpp
git commit -m "feat: UI web embutida (React) + ponte nativa ping

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Tela de dispositivos com dados reais

**Files:**
- Create: `src/Devices.h`, `src/Devices.cpp`, `ui/src/types.ts`, `ui/src/Devices.tsx`
- Modify: `CMakeLists.txt` (`target_sources`), `src/Main.cpp`, `ui/src/App.tsx`

**Interfaces:**
- Consumes: `call`, `on` (Task 2); `MainComponent::makeOptions()` (Task 2).
- Produces: `juce::var listDevices (juce::AudioDeviceManager&)`; o membro `juce::AudioDeviceManager deviceManager` no `MainComponent`; o evento `devicesChanged`. Em `ui/src/types.ts`: `DeviceInfo`, `PassthroughRequest`, `PassthroughStatus`, `Meter`. Em `ui/src/Devices.tsx`: `useDevices()` e `DevicesTable`.

- [ ] **Step 1: `src/Devices.h` e `src/Devices.cpp`**

```cpp
// src/Devices.h
#pragma once
#include <juce_audio_devices/juce_audio_devices.h>

// Todos os dispositivos de todos os drivers disponíveis, com dados reais lidos do driver (sem abrir o device).
juce::var listDevices (juce::AudioDeviceManager& dm);
```

```cpp
// src/Devices.cpp
#include "Devices.h"

static juce::var toVar (const juce::StringArray& strings)
{
    juce::Array<juce::var> v;
    for (const auto& s : strings)
        v.add (s);
    return v;
}

juce::var listDevices (juce::AudioDeviceManager& dm)
{
    juce::Array<juce::var> result;

    for (auto* type : dm.getAvailableDeviceTypes())
    {
        type->scanForDevices();
        const auto ins = type->getDeviceNames (true);
        const auto outs = type->getDeviceNames (false);
        juce::StringArray names (outs);
        names.mergeArray (ins);

        for (const auto& name : names)
        {
            std::unique_ptr<juce::AudioIODevice> dev (type->createDevice (outs.contains (name) ? name : juce::String(),
                                                                         ins.contains (name) ? name : juce::String()));
            if (dev == nullptr)
                continue;

            juce::Array<juce::var> rates, sizes;
            for (auto r : dev->getAvailableSampleRates()) rates.add (r);
            for (auto b : dev->getAvailableBufferSizes()) sizes.add (b);

            auto* o = new juce::DynamicObject();
            o->setProperty ("type", type->getTypeName());
            o->setProperty ("name", name);
            o->setProperty ("inputs", toVar (dev->getInputChannelNames()));
            o->setProperty ("outputs", toVar (dev->getOutputChannelNames()));
            o->setProperty ("sampleRates", rates);
            o->setProperty ("bufferSizes", sizes);
            o->setProperty ("defaultBufferSize", dev->getDefaultBufferSize());
            result.add (juce::var (o));
        }
    }
    return result;
}
```

- [ ] **Step 2: Registrar no CMake** — trocar a linha `target_sources(AudioBridge PRIVATE src/Main.cpp)` por:

```cmake
target_sources(AudioBridge PRIVATE src/Main.cpp src/Devices.cpp)
```

- [ ] **Step 3: Ligar no `MainComponent`** (`src/Main.cpp`):
  - Adicionar `#include "Devices.h"` depois de `#include <BinaryData.h>`.
  - Fazer a classe herdar `private juce::ChangeListener`: `class MainComponent final : public juce::Component, private juce::ChangeListener`.
  - Adicionar o membro `juce::AudioDeviceManager deviceManager;` **antes** de `web`, porque a ordem de inicialização importa.
  - No construtor, antes de `addAndMakeVisible (web);`:

```cpp
        deviceManager.initialise (0, 0, nullptr, false);
        deviceManager.closeAudioDevice();  // a Fase 0 só abre device quando o usuário pede
        deviceManager.addChangeListener (this);
```

  - Destrutor e callback:

```cpp
    ~MainComponent() override { deviceManager.removeChangeListener (this); }
```

```cpp
    void changeListenerCallback (juce::ChangeBroadcaster*) override
    {
        web.emitEventIfBrowserIsVisible ("devicesChanged", juce::var());
    }
```

  - Em `makeOptions()`, acrescentar ao final da cadeia:

```cpp
            .withNativeFunction ("listDevices", [this] (const Args&, Completion done) { done (listDevices (deviceManager)); })
```

- [ ] **Step 4: `ui/src/types.ts`**

```ts
export type DeviceInfo = {
  type: string; name: string; inputs: string[]; outputs: string[];
  sampleRates: number[]; bufferSizes: number[]; defaultBufferSize: number;
};
export type PassthroughRequest = {
  type: string; inputDevice: string; outputDevice: string; sampleRate: number; bufferSize: number;
  inputChannel: number; outputLeft: number; outputRight: number; // outputRight -1 = mono
};
export type PassthroughStatus = {
  ok: boolean; error: string; sampleRate: number; bufferSize: number;
  inputLatency: number; outputLatency: number; roundTripMs: number;
};
export type Meter = { peak: number; xruns: number; cpu: number; running: boolean };
```

- [ ] **Step 5: `ui/src/Devices.tsx`**

```tsx
import { useEffect, useState } from "react";
import { call, on } from "./juce";
import type { DeviceInfo } from "./types";

export function useDevices() {
  const [devices, setDevices] = useState<DeviceInfo[]>([]);
  const [error, setError] = useState("");

  useEffect(() => {
    const load = () => call<DeviceInfo[]>("listDevices").then(setDevices, (e) => setError(String(e)));
    load();
    on("devicesChanged", load);
  }, []);

  return { devices, error };
}

export function DevicesTable({ devices }: { devices: DeviceInfo[] }) {
  if (devices.length === 0) return <p className="muted">Nenhum dispositivo encontrado.</p>;
  return (
    <table>
      <thead>
        <tr><th>Driver</th><th>Dispositivo</th><th>Entradas</th><th>Saídas</th><th>Sample rates (Hz)</th><th>Buffers</th></tr>
      </thead>
      <tbody>
        {devices.map((d) => (
          <tr key={d.type + "|" + d.name}>
            <td>{d.type}</td>
            <td>{d.name}</td>
            <td title={d.inputs.join("\n")}>{d.inputs.length}</td>
            <td title={d.outputs.join("\n")}>{d.outputs.length}</td>
            <td>{d.sampleRates.join(", ")}</td>
            <td>{d.bufferSizes.join(", ")}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

- [ ] **Step 6: `ui/src/App.tsx` com a tela de dispositivos**

```tsx
import { useEffect, useState } from "react";
import { backendAvailable, call } from "./juce";
import { DevicesTable, useDevices } from "./Devices";

export function App() {
  const [version, setVersion] = useState("");
  const { devices, error } = useDevices();

  useEffect(() => {
    call<string>("ping").then(setVersion, () => {});
  }, []);

  if (!backendAvailable())
    return <main><h1>AudioBridge</h1><p className="error">Backend nativo indisponível: abra pelo app AudioBridge.</p></main>;

  return (
    <main>
      <header>
        <h1>AudioBridge</h1>
        <span className="muted">v{version}</span>
      </header>
      {error && <p className="error">{error}</p>}
      <section>
        <h2>Dispositivos</h2>
        <DevicesTable devices={devices} />
      </section>
    </main>
  );
}
```

- [ ] **Step 7: Build e verificação manual**

Run: `cd ui && npm run build && cd .. && cmake --build build --config Release && ctest --test-dir build -C Release`
Expected: no Windows, a tabela lista os devices do driver "Windows Audio" (e variantes) com contagens de entrada/saída coerentes com as placas plugadas (passar o mouse mostra os nomes dos canais). Plugar ou desplugar um device USB atualiza a tabela sozinha.

- [ ] **Step 8: Commit**

```bash
git add src/Devices.h src/Devices.cpp src/Main.cpp CMakeLists.txt ui/src
git commit -m "feat: tela de dispositivos com dados reais do driver

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Passthrough entrada → fone, latência, meter e log

**Files:**
- Create: `src/Engine.h`, `src/Engine.cpp`, `ui/src/Passthrough.tsx`
- Modify: `CMakeLists.txt` (`target_sources`), `src/Main.cpp`, `ui/src/App.tsx`

**Interfaces:**
- Consumes: `LatencyInfo`, `roundTripMs`, `compactIndex` (Task 1). `deviceManager`, `makeOptions()` (Tasks 2–3). `DeviceInfo`, `PassthroughRequest`, `PassthroughStatus`, `Meter` (Task 3).
- Produces: `struct PassthroughRequest { static PassthroughRequest fromVar (const juce::var&); }`, `class Engine { juce::var start (const PassthroughRequest&); void stop(); juce::var meterSnapshot(); }`. Native functions `startPassthrough`, `stopPassthrough`, `revealLog`. Evento `meter`.

- [ ] **Step 1: `src/Engine.h`**

```cpp
#pragma once
#include <juce_audio_devices/juce_audio_devices.h>
#include <atomic>

struct PassthroughRequest
{
    juce::String type, inputDevice, outputDevice;
    double sampleRate = 48000;
    int bufferSize = 128;
    int inputChannel = 0, outputLeft = 0, outputRight = -1;  // outputRight -1 = mono

    static PassthroughRequest fromVar (const juce::var& v);
};

// Fase 0: copia um canal de entrada para 1-2 canais de saída. O callback não aloca nem trava.
class Engine final : private juce::AudioIODeviceCallback
{
public:
    explicit Engine (juce::AudioDeviceManager& deviceManager) : dm (deviceManager) {}
    ~Engine() override { stop(); }

    juce::var start (const PassthroughRequest& req);  // PassthroughStatus
    void stop();
    juce::var meterSnapshot();                        // Meter; zera o pico acumulado

private:
    void audioDeviceIOCallbackWithContext (const float* const* in, int numIn, float* const* out, int numOut,
                                           int numSamples, const juce::AudioIODeviceCallbackContext&) override;
    void audioDeviceAboutToStart (juce::AudioIODevice*) override {}
    void audioDeviceStopped() override {}
    void audioDeviceError (const juce::String& message) override;

    juce::AudioDeviceManager& dm;
    std::atomic<int> outL { 0 }, outR { -1 };  // índices compactados (ver compactIndex)
    std::atomic<float> peak { 0.0f };
    bool running = false;
};
```

- [ ] **Step 2: `src/Engine.cpp`**

```cpp
#include "Engine.h"
#include "AudioMath.h"

PassthroughRequest PassthroughRequest::fromVar (const juce::var& v)
{
    PassthroughRequest r;
    r.type = v["type"].toString();
    r.inputDevice = v["inputDevice"].toString();
    r.outputDevice = v["outputDevice"].toString();
    r.sampleRate = (double) v["sampleRate"];
    r.bufferSize = (int) v["bufferSize"];
    r.inputChannel = (int) v["inputChannel"];
    r.outputLeft = (int) v["outputLeft"];
    r.outputRight = (int) v["outputRight"];
    return r;
}

juce::var Engine::start (const PassthroughRequest& req)
{
    stop();

    dm.setCurrentAudioDeviceType (req.type, true);
    auto setup = dm.getAudioDeviceSetup();
    setup.inputDeviceName = req.inputDevice;
    setup.outputDeviceName = req.outputDevice;
    setup.sampleRate = req.sampleRate;
    setup.bufferSize = req.bufferSize;
    setup.useDefaultInputChannels = false;
    setup.useDefaultOutputChannels = false;
    setup.inputChannels.clear();
    setup.inputChannels.setBit (req.inputChannel);
    setup.outputChannels.clear();
    setup.outputChannels.setBit (req.outputLeft);
    if (req.outputRight >= 0)
        setup.outputChannels.setBit (req.outputRight);

    auto* status = new juce::DynamicObject();
    juce::var result (status);

    const auto error = dm.setAudioDeviceSetup (setup, true);
    auto* dev = dm.getCurrentAudioDevice();
    if (error.isNotEmpty() || dev == nullptr)
    {
        const auto message = error.isNotEmpty() ? error : juce::String ("Nenhum dispositivo aberto");
        juce::Logger::writeToLog ("Passthrough falhou: " + message);
        status->setProperty ("ok", false);
        status->setProperty ("error", message);
        return result;
    }

    std::vector<int> active { req.outputLeft };
    if (req.outputRight >= 0)
        active.push_back (req.outputRight);
    outL = compactIndex (active, req.outputLeft);
    outR = req.outputRight >= 0 ? compactIndex (active, req.outputRight) : -1;
    peak = 0.0f;
    dm.addAudioCallback (this);
    running = true;

    const LatencyInfo lat { dev->getInputLatencyInSamples(), dev->getOutputLatencyInSamples(), dev->getCurrentSampleRate() };
    status->setProperty ("ok", true);
    status->setProperty ("error", "");
    status->setProperty ("sampleRate", dev->getCurrentSampleRate());
    status->setProperty ("bufferSize", dev->getCurrentBufferSizeSamples());
    status->setProperty ("inputLatency", lat.inputSamples);
    status->setProperty ("outputLatency", lat.outputSamples);
    status->setProperty ("roundTripMs", roundTripMs (lat));

    juce::Logger::writeToLog ("Passthrough: " + req.type + " | in=" + req.inputDevice + " ch" + juce::String (req.inputChannel)
                              + " | out=" + req.outputDevice + " ch" + juce::String (req.outputLeft) + "/" + juce::String (req.outputRight)
                              + " | " + juce::String (dev->getCurrentSampleRate()) + " Hz, buffer " + juce::String (dev->getCurrentBufferSizeSamples())
                              + " | latencia " + juce::String (roundTripMs (lat), 2) + " ms");
    return result;
}

void Engine::stop()
{
    if (running)
    {
        dm.removeAudioCallback (this);
        running = false;
        juce::Logger::writeToLog ("Passthrough parado");
    }
    dm.closeAudioDevice();
}

juce::var Engine::meterSnapshot()
{
    auto* o = new juce::DynamicObject();
    o->setProperty ("peak", peak.exchange (0.0f));
    o->setProperty ("xruns", dm.getXRunCount());
    o->setProperty ("cpu", dm.getCpuUsage());
    o->setProperty ("running", running);
    return juce::var (o);
}

void Engine::audioDeviceError (const juce::String& message)
{
    // Pode chegar fora da thread de mensagens: o log vai para lá.
    juce::MessageManager::callAsync ([message] { juce::Logger::writeToLog ("Erro do device: " + message); });
}

void Engine::audioDeviceIOCallbackWithContext (const float* const* in, int numIn, float* const* out, int numOut,
                                               int numSamples, const juce::AudioIODeviceCallbackContext&)
{
    for (int c = 0; c < numOut; ++c)
        if (out[c] != nullptr)
            juce::FloatVectorOperations::clear (out[c], numSamples);

    if (numIn < 1 || in[0] == nullptr)
        return;

    const auto range = juce::FloatVectorOperations::findMinAndMax (in[0], numSamples);
    const float p = std::max (std::abs (range.getStart()), std::abs (range.getEnd()));
    float prev = peak.load (std::memory_order_relaxed);
    while (p > prev && ! peak.compare_exchange_weak (prev, p, std::memory_order_relaxed)) {}

    const int l = outL.load (std::memory_order_relaxed);
    const int r = outR.load (std::memory_order_relaxed);
    if (l >= 0 && l < numOut && out[l] != nullptr) juce::FloatVectorOperations::copy (out[l], in[0], numSamples);
    if (r >= 0 && r < numOut && out[r] != nullptr) juce::FloatVectorOperations::copy (out[r], in[0], numSamples);
}
```

- [ ] **Step 3: Registrar no CMake**

```cmake
target_sources(AudioBridge PRIVATE src/Main.cpp src/Devices.cpp src/Engine.cpp)
```

- [ ] **Step 4: Ligar no `MainComponent`** (`src/Main.cpp`):
  - Adicionar `#include "Engine.h"`.
  - Herdar também `private juce::Timer`.
  - Membros, nesta ordem (o logger primeiro, `web` por último):

```cpp
    std::unique_ptr<juce::FileLogger> logger { juce::FileLogger::createDefaultAppLogger ("AudioBridge", "AudioBridge.log", "AudioBridge log") };
    juce::AudioDeviceManager deviceManager;
    Engine engine { deviceManager };
    juce::WebBrowserComponent web { makeOptions() };
```

  - Início do construtor: `juce::Logger::setCurrentLogger (logger.get()); juce::Logger::writeToLog ("AudioBridge " JUCE_APPLICATION_VERSION_STRING " iniciado");`.
  - Final do construtor: `startTimerHz (30);`.
  - Destrutor completo:

```cpp
    ~MainComponent() override
    {
        stopTimer();
        deviceManager.removeChangeListener (this);
        engine.stop();
        juce::Logger::setCurrentLogger (nullptr);
    }
```

  - Timer:

```cpp
    void timerCallback() override { web.emitEventIfBrowserIsVisible ("meter", engine.meterSnapshot()); }
```

  - Em `makeOptions()`, acrescentar:

```cpp
            .withNativeFunction ("startPassthrough", [this] (const Args& a, Completion done) { done (engine.start (PassthroughRequest::fromVar (a[0]))); })
            .withNativeFunction ("stopPassthrough", [this] (const Args&, Completion done) { engine.stop(); done (juce::var()); })
            .withNativeFunction ("revealLog", [this] (const Args&, Completion done)
            {
                logger->getLogFile().revealToUser();
                done (logger->getLogFile().getFullPathName());
            })
```

- [ ] **Step 5: `ui/src/Passthrough.tsx`**

```tsx
import { useEffect, useState } from "react";
import { call, on } from "./juce";
import type { DeviceInfo, Meter, PassthroughRequest, PassthroughStatus } from "./types";

function Select({ label, value, options, onChange }: {
  label: string; value: string | number; options: { value: string | number; label: string }[]; onChange: (v: string) => void;
}) {
  return (
    <label>
      {label}
      <select value={value} onChange={(e) => onChange(e.target.value)}>
        {options.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
      </select>
    </label>
  );
}

export function Passthrough({ devices }: { devices: DeviceInfo[] }) {
  const [type, setType] = useState("");
  const [inName, setInName] = useState("");
  const [outName, setOutName] = useState("");
  const [inCh, setInCh] = useState(0);
  const [outL, setOutL] = useState(0);
  const [outR, setOutR] = useState(1);
  const [rate, setRate] = useState(48000);
  const [buffer, setBuffer] = useState(128);
  const [status, setStatus] = useState<PassthroughStatus | null>(null);
  const [meter, setMeter] = useState<Meter | null>(null);

  useEffect(() => { on("meter", (m: Meter) => setMeter(m)); }, []);

  const types = [...new Set(devices.map((d) => d.type))];
  const t = type || types[0] || "";
  const ofType = devices.filter((d) => d.type === t);
  const inputs = ofType.filter((d) => d.inputs.length > 0);
  const outputs = ofType.filter((d) => d.outputs.length > 0);
  const inDev = inputs.find((d) => d.name === inName) ?? inputs[0];
  const outDev = outputs.find((d) => d.name === outName) ?? outputs[0];

  const start = async () => {
    if (!inDev || !outDev) return;
    const req: PassthroughRequest = {
      type: t, inputDevice: inDev.name, outputDevice: outDev.name, sampleRate: rate, bufferSize: buffer,
      inputChannel: inCh, outputLeft: outL, outputRight: outR,
    };
    setStatus(await call<PassthroughStatus>("startPassthrough", req));
  };
  const stop = async () => { await call("stopPassthrough"); setStatus(null); };

  const peakDb = meter && meter.peak > 0 ? 20 * Math.log10(meter.peak) : -Infinity;
  const width = Math.max(0, Math.min(1, (peakDb + 60) / 60)) * 100;
  const clip = (meter?.peak ?? 0) >= 1;

  return (
    <div>
      <div className="grid">
        <Select label="Driver" value={t} options={types.map((x) => ({ value: x, label: x }))} onChange={setType} />
        <Select label="Dispositivo de entrada" value={inDev?.name ?? ""} options={inputs.map((d) => ({ value: d.name, label: d.name }))} onChange={setInName} />
        <Select label="Canal de entrada" value={inCh} options={(inDev?.inputs ?? []).map((n, i) => ({ value: i, label: `${i + 1}: ${n}` }))} onChange={(v) => setInCh(Number(v))} />
        <Select label="Dispositivo de saída (fone)" value={outDev?.name ?? ""} options={outputs.map((d) => ({ value: d.name, label: d.name }))} onChange={setOutName} />
        <Select label="Saída L" value={outL} options={(outDev?.outputs ?? []).map((n, i) => ({ value: i, label: `${i + 1}: ${n}` }))} onChange={(v) => setOutL(Number(v))} />
        <Select label="Saída R" value={outR} options={[{ value: -1, label: "— (mono)" }, ...(outDev?.outputs ?? []).map((n, i) => ({ value: i, label: `${i + 1}: ${n}` }))]} onChange={(v) => setOutR(Number(v))} />
        <Select label="Sample rate" value={rate} options={(inDev?.sampleRates ?? []).map((r) => ({ value: r, label: `${r} Hz` }))} onChange={(v) => setRate(Number(v))} />
        <Select label="Buffer (amostras)" value={buffer} options={(outDev?.bufferSizes ?? []).map((b) => ({ value: b, label: String(b) }))} onChange={(v) => setBuffer(Number(v))} />
      </div>

      <div className="row">
        <button className="primary" onClick={start} disabled={!inDev || !outDev}>Iniciar</button>
        <button onClick={stop}>Parar</button>
        <div className={clip ? "meter clip" : "meter"}><div style={{ width: `${width}%` }} /></div>
        <span className="muted">{Number.isFinite(peakDb) ? `${peakDb.toFixed(1)} dBFS` : "sem sinal"}{clip && " — CLIP"}</span>
      </div>

      {status && !status.ok && <p className="error">Erro: {status.error}</p>}
      {status?.ok && (
        <p>
          Rodando a {status.sampleRate} Hz, buffer {status.bufferSize}. Latência de ida e volta reportada pelo driver:{" "}
          <b>{status.roundTripMs.toFixed(1)} ms</b>{" "}
          <span className="muted">(entrada {status.inputLatency} + saída {status.outputLatency} amostras)</span>
        </p>
      )}
      {meter?.running && <p className="muted">Xruns: {meter.xruns} · CPU do áudio: {(meter.cpu * 100).toFixed(1)}%</p>}
    </div>
  );
}
```

- [ ] **Step 6: `ui/src/App.tsx` final da Fase 0**

```tsx
import { useEffect, useState } from "react";
import { backendAvailable, call } from "./juce";
import { DevicesTable, useDevices } from "./Devices";
import { Passthrough } from "./Passthrough";

export function App() {
  const [version, setVersion] = useState("");
  const { devices, error } = useDevices();

  useEffect(() => {
    call<string>("ping").then(setVersion, () => {});
  }, []);

  if (!backendAvailable())
    return <main><h1>AudioBridge</h1><p className="error">Backend nativo indisponível: abra pelo app AudioBridge.</p></main>;

  return (
    <main>
      <header>
        <h1>AudioBridge</h1>
        <span className="muted">v{version}</span>
        <button onClick={() => call("revealLog")}>Abrir log</button>
      </header>
      {error && <p className="error">{error}</p>}
      <section>
        <h2>Teste de passagem (entrada → fone)</h2>
        <Passthrough devices={devices} />
      </section>
      <section>
        <h2>Dispositivos</h2>
        <DevicesTable devices={devices} />
      </section>
    </main>
  );
}
```

- [ ] **Step 7: Build, testes e verificação manual no Windows**

Run: `cd ui && npm run build && cd .. && cmake --build build --config Release && ctest --test-dir build -C Release --output-on-failure`
Expected: `AudioMathTests: OK`. No app:
1. Escolher o microfone como entrada e o fone/caixa como saída.
2. Clicar em **Iniciar**: ouve-se o microfone, o meter se mexe e a latência aparece em ms.
3. Trocar o buffer para 512 e reiniciar: a latência mostrada aumenta.
4. Clicar em **Parar**: o som para.
5. Clicar em **Abrir log**: abre a pasta do `AudioBridge.log`, que tem as linhas "Passthrough: ...".

Escolher um canal de saída inexistente não pode travar o app: ele fica em silêncio ou mostra erro.

- [ ] **Step 8: Commit**

```bash
git add src/Engine.h src/Engine.cpp src/Main.cpp CMakeLists.txt ui/src
git commit -m "feat: passthrough entrada->fone com latência, meter e log

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: CI (macOS universal + Windows), release e README

**Files:**
- Create: `.github/workflows/build.yml`, `README.md`

**Interfaces:**
- Consumes: os comandos de build das Tasks 1–4. O artefato do app fica em `build/AudioBridge_artefacts/Release/`.
- Produces: artifacts `AudioBridge-macos-universal` (`.tar.gz`) e `AudioBridge-windows-x64` (`.zip`). Um release público quando há push de tag `v*`.

- [ ] **Step 1: `.github/workflows/build.yml`**

```yaml
name: build

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
  workflow_dispatch:

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: macos-15
            name: macos-universal
          - os: windows-latest
            name: windows-x64
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
          cache-dependency-path: ui/package-lock.json

      - name: UI
        working-directory: ui
        run: |
          npm ci
          npm run build

      - name: Configurar
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release

      - name: Compilar
        run: cmake --build build --config Release --parallel

      - name: Testes
        run: ctest --test-dir build -C Release --output-on-failure

      - name: Empacotar (macOS)
        if: runner.os == 'macOS'
        run: |
          APP=$(find build -maxdepth 5 -type d -name "AudioBridge.app" | head -1)
          lipo -archs "$APP/Contents/MacOS/AudioBridge"
          codesign --force --deep --sign - "$APP"
          tar -czf AudioBridge-macos-universal.tar.gz -C "$(dirname "$APP")" AudioBridge.app

      - name: Empacotar (Windows)
        if: runner.os == 'Windows'
        shell: pwsh
        run: |
          $exe = Get-ChildItem build -Recurse -Filter AudioBridge.exe | Select-Object -First 1
          Compress-Archive -Path $exe.FullName -DestinationPath AudioBridge-windows-x64.zip

      - uses: actions/upload-artifact@v4
        with:
          name: AudioBridge-${{ matrix.name }}
          path: AudioBridge-*.*

  release:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          merge-multiple: true
      - name: Publicar release
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "$GITHUB_REF_NAME" AudioBridge-* --repo "$GITHUB_REPOSITORY" --prerelease --title "AudioBridge $GITHUB_REF_NAME" --notes "Build de teste. macOS 12+ (Intel e Apple Silicon) e Windows 10+."
```

- [ ] **Step 2: `README.md`**

````markdown
# AudioBridge

Mesa de monitoração ao vivo em software. Junta interfaces de áudio, monta um mix de fone por músico e (em breve) roda plugins VST3/AU por canal. A DAW continua gravando em paralelo.
Design: `docs/superpowers/specs/2026-11-09-audiobridge-design.md`.

Licença: AGPLv3. Usa o JUCE (AGPLv3/comercial); uma versão comercial de código fechado pode exigir licença comercial do JUCE.

## Compilar (Windows)
Pré-requisitos: CMake 3.22+, Visual Studio Build Tools (C++), Node 24.
```powershell
cd ui; npm ci; npm run build; cd ..
cmake -B build
cmake --build build --config Release
ctest --test-dir build -C Release
.\build\AudioBridge_artefacts\Release\AudioBridge.exe
```

## Testar no Mac (usuário piloto)
1. Baixe `AudioBridge-macos-universal.tar.gz` na página **Releases** do GitHub e dê dois cliques para extrair.
2. Arraste `AudioBridge.app` para Aplicativos e tente abrir. O macOS vai bloquear (app sem assinatura paga).
3. Vá em **Ajustes do Sistema > Privacidade e Segurança**, role até "Segurança" e clique em **Abrir Mesmo Assim** (vale por ~1 h após a tentativa). Digite a senha.
4. Quando pedir acesso ao **microfone**, permita: é como o app lê as entradas da placa.

### Checklist da Fase 0
- [ ] O app abre e mostra "AudioBridge v…".
- [ ] A tabela "Dispositivos" lista as placas plugadas com o número certo de entradas e saídas.
- [ ] Desplugar ou plugar uma placa atualiza a tabela sozinha.
- [ ] Teste de passagem: entrada da placa → fone, com buffer 128. Ouve sem estalos por 5 minutos?
- [ ] A latência mostrada parece com o que se ouve?
- [ ] Teste com entrada de uma placa e saída de outra (ex.: UMC404 → 2i2). Funciona?
- [ ] Clique em "Abrir log" e mande o arquivo `AudioBridge.log` + prints de cada item.
````

- [ ] **Step 3: Gerar o lockfile da UI (se ainda não estiver versionado) e commitar**

Run: `git ls-files ui/package-lock.json`
Expected: imprime `ui/package-lock.json`. Se não imprimir, rodar `cd ui && npm install` e fazer `git add ui/package-lock.json`.

```bash
git add .github/workflows/build.yml README.md ui/package-lock.json
git commit -m "ci: build macOS universal + Windows, release em tag

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
```

- [ ] **Step 4: Verificar o CI**

Run: `gh run watch --exit-status $(gh run list --workflow build --limit 1 --json databaseId --jq '.[0].databaseId')`
Expected: os jobs `macos-universal` e `windows-x64` ficam verdes. No log do macOS, o passo "Empacotar" imprime `x86_64 arm64` (a ordem pode variar). Se falhar, ler o log com `gh run view --log-failed`, corrigir e fazer push de novo.

---

### Task 6: Release de teste e go/no-go no Mac do usuário piloto

**Files:**
- Modify: `docs/superpowers/specs/2026-11-09-audiobridge-design.md` (acrescentar "## Resultado da Fase 0")

- [ ] **Step 1: Publicar o release**

```bash
git tag v0.0.1
git push origin v0.0.1
gh run watch --exit-status $(gh run list --workflow build --limit 1 --json databaseId --jq '.[0].databaseId')
gh release view v0.0.1
```

Expected: o release `v0.0.1` (pre-release) tem os dois arquivos. Mandar o link do release + a seção "Testar no Mac" do README ao usuário piloto.

- [ ] **Step 2: Coletar a evidência.** São os itens do checklist, o `AudioBridge.log`, prints e o modelo do Mac + versão do macOS. **Não** declarar nada como funcionando sem essa evidência.

- [ ] **Step 3: Registrar o resultado na spec**

```markdown
## Resultado da Fase 0 (AAAA-MM-DD)
- Mac: <modelo>, macOS <versão>, <Intel|Apple Silicon>
- Checklist: <item → ok/falhou + observação>
- Latência medida/ouvida: <...>
- Decisão: GO / NO-GO (motivo)
```

```bash
git add docs/superpowers/specs/2026-11-09-audiobridge-design.md
git commit -m "docs: resultado da Fase 0

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push
```

- [ ] **Step 4: Com GO, abrir o brainstorming/plano da Fase 1 (Agregado). Com NO-GO, voltar à decisão de stack com a evidência na mão.**
