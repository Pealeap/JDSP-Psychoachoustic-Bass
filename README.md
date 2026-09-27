# Psychoacoustic Bass (Missing Fundamental)

🇬🇧 [English](#english) | 🇧🇷 [Português](#português)

---

<a id="english"></a>
## English

EEL2 script for the [JDSP4Linux](https://github.com/Audio4Linux/JDSP4Linux) and [RootlessJamesDSP](https://github.com/timschneeb/RootlessJamesDSP) LiveProg module that simulates the presence of bass frequencies below what your speakers can physically reproduce, using the psychoacoustic **missing fundamental** phenomenon.

### What is the "missing fundamental"

When a low sound is played, it isn't a single pure frequency — it's a **fundamental** (the lowest note, which gives the sensation of "weight") accompanied by a series of **harmonics** (integer multiples of the fundamental, usually much higher in pitch).

The human ear has an interesting trick: if you remove the fundamental from a sound and keep only the harmonics, the brain still "hears" the fundamental — it reconstructs it from the spacing between the remaining harmonics. This is the same reason small speakers (phones, laptops, bluetooth speakers) can still give a sense of bass even without physically reproducing low frequencies: the driver isn't emitting the fundamental (it physically can't), but it is emitting its harmonics, and the brain fills in the gap.

This script exploits that effect on purpose:

1. Isolates the bass band of the signal (low-pass).
2. Generates artificial harmonics from that band through non-linear distortion (saturation).
3. Mixes even and odd harmonics in adjustable proportions (this changes the "timbre" of the perceived bass).
4. Optionally removes the original fundamental (high-pass), leaving only the synthesized harmonics — useful on systems that can't reproduce real bass anyway, avoiding wasted headroom on a frequency the driver won't play.

### How the script works

#### Sliders

| Slider | Range | What it does |
|---|---|---|
| **Bass band cutoff (Hz)** | 40–250 Hz | Sets where the bass band being processed starts — anything below this is treated as "fundamental". |
| **Harmonic drive** | 0.2–10 | Amount of saturation applied to the bass band to generate harmonics. More drive = stronger, more present harmonics. |
| **Even/odd balance** | 0–1 | Balance between even harmonics (fuller/smoother sound, tube-like) and odd harmonics (more present/aggressive, distortion-like). |
| **Harmonics mix** | 0–2 | How much of the generated harmonics gets summed back into the final signal. |
| **Remove original bass band** | Off / On | When on, applies a high-pass filter to cut the original fundamental, letting only the synthesized harmonics do the work. |
| **Fundamental cut slope** | 12 dB/oct / 24 dB/oct | Steepness of the cut filter above. 24 dB/oct cuts more aggressively (two cascaded high-pass filters) than 12 dB/oct (a single filter). |

#### Signal flow, step by step (`@sample`)

1. **Bass band extraction** — a low-pass filter (`LP_process`) isolates everything below `bassFreq`.
2. **Harmonic generation** — the isolated signal goes through simple saturation (`x / (1 + |x|)`), then is squared (even harmonics) and cubed (odd harmonics), mixed according to `evenMix`.
3. **DC removal** — since squaring shifts the signal's average, a simple DC-blocking filter (leaky integrator) prevents this from adding offset to the audio.
4. **Optional fundamental cut** — if `cutFundamental` is on, the original signal passes through one or two cascaded high-pass filters (`HP_process`), depending on the `cutOrder` slider.
5. **Final mix** — the "dry" signal (with or without the fundamental, depending on the setting) is summed with the synthesized harmonics, weighted by `harmonicsMix`.

The filters (`LP_set`/`HP_set`/`LP_process`/`HP_process`) are standard RBJ (Robert Bristow-Johnson) biquads, implemented manually — each filter instance stores its 9 coefficients/states (`b0, b1, b2, a1, a2, x1, x2, y1, y2`) in a manually addressed memory block, instead of using dot-namespace objects (`this.field`). That's intentional — see the fix section below.

### RootlessJamesDSP fix: why the cut slider wasn't showing up

This script ran fine on JDSP4Linux, but the cut sliders (`Remove original bass band` / `Fundamental cut slope`) simply wouldn't appear in the RootlessJamesDSP UI on Android — no error, the control would just be missing.

**Root cause:** the two apps read sliders in fundamentally different ways.

- **JDSP4Linux** uses the full EEL2 engine (the same base as REAPER/WDL-EEL). When the VM compiles the script, it automatically binds the *default* value declared in the slider header (`name:default<min,max,step>`) as the variable's initial value — the slider declaration itself is the source of truth.

- **RootlessJamesDSP** doesn't run the full VM just to draw the UI. It uses a separate parser, written in Kotlin ([`EelParser.kt`](https://github.com/timschneeb/RootlessJamesDSP/blob/master/app/src/main/java/me/timschneeberger/rootlessjamesdsp/liveprog/EelParser.kt)), that reads the file as plain text via regex. This parser **doesn't understand** that `cutOrder:1<0,1,1{...}>` already defines a default value — it separately requires a literal line like `cutOrder = 1;` somewhere in the code to extract "what's the current value" via regex (`key\s*=\s*(-?\d+\.?\d*)\s*;`). If that line doesn't exist, the parsing function (`findVariable`) returns null, and the whole slider is silently dropped from the property list — no visible log, no crash, it just doesn't show up.

While optimizing the script, a redundant `cutOrder = 1;` line (the value already matched the slider's declared default) had been removed from the `@init` block for looking unnecessary. Result: the slider disappeared specifically on RootlessJamesDSP, because it depends on that line existing — even though, from a "pure" EEL2 logic standpoint, it's redundant.

**Fix:** always keep an explicit assignment (`cutOrder = 1;`) in `@init` for every list-type slider (`{...}`), even when the value already matches the header's declared default. This assignment is also what RootlessJamesDSP automatically rewrites (via regex) whenever you move the slider in the UI, so it works both as the initial value and as the persistence mechanism.

#### Other compatibility fixes

Besides the slider bug, two other engine compatibility differences were addressed:

- **Dot namespace (`object.function()`, `this.field`)**: valid syntax in standard EEL2, but not supported by RootlessJamesDSP's embedded interpreter. Rewritten to pass the memory address (`base`) as an explicit function parameter instead of using object instances.
- **`memory(index) = value` function**: also not supported the way standard EEL2 accepts it. Replaced with bracket indexing syntax (`base[N] = value`), which is the native memory access form supported by virtually any EEL2 implementation.

### Compatibility

Tested and working on:
- ✅ JDSP4Linux
- ✅ RootlessJamesDSP (Android)

---

<a id="português"></a>
## Português

Script EEL2 para o LiveProg do [JDSP4Linux](https://github.com/Audio4Linux/JDSP4Linux) e [RootlessJamesDSP](https://github.com/timschneeb/RootlessJamesDSP) que simula a presença de graves abaixo do que os alto-falantes conseguem reproduzir fisicamente, usando o fenômeno psicoacústico do **fundamental ausente**.

### O que é o "fundamental ausente"

Quando um som grave é tocado, ele não é uma única frequência pura — é uma **fundamental** (a nota mais grave, que dá a sensação de "peso") acompanhada de uma série de **harmônicos** (múltiplos inteiros da fundamental, geralmente bem mais agudos).

O ouvido humano tem um truque interessante: se você remove a fundamental de um som e mantém só os harmônicos, o cérebro ainda "escuta" a fundamental — ele a reconstrói a partir do espaçamento entre os harmônicos que sobraram. É o mesmo motivo pelo qual alto-falantes pequenos (celular, notebook, caixinhas bluetooth) conseguem dar uma sensação de grave mesmo sem fisicamente reproduzir frequências baixas: o driver não emite a fundamental (ele não consegue), mas emite os harmônicos dela, e o cérebro preenche a lacuna.

Esse script explora esse efeito de propósito:

1. Isola a banda de graves do sinal original (passa-baixa).
2. Gera harmônicos artificiais dessa banda através de distorção não-linear (saturação).
3. Mistura harmônicos pares e ímpares em proporções ajustáveis (isso muda o "timbre" da sensação de grave).
4. Opcionalmente remove a fundamental original (passa-alta), deixando só os harmônicos artificiais — útil em sistemas que não reproduzem grave real de qualquer forma, evitando desperdiçar headroom com uma frequência que o driver não vai tocar.

### Como o script funciona

#### Sliders

| Slider | Faixa | O que faz |
|---|---|---|
| **Bass band cutoff (Hz)** | 40–250 Hz | Define onde fica a banda de graves que será processada — abaixo dessa frequência é considerado "fundamental". |
| **Harmonic drive** | 0.2–10 | Quantidade de saturação aplicada à banda de graves para gerar harmônicos. Mais drive = harmônicos mais fortes e presentes. |
| **Even/odd balance** | 0–1 | Equilíbrio entre harmônicos pares (som mais "cheio"/suave, tipo válvula) e ímpares (som mais "presente"/agressivo, tipo distorção). |
| **Harmonics mix** | 0–2 | Quanto dos harmônicos gerados é somado de volta ao sinal final. |
| **Remove original bass band** | Off / On | Se ligado, aplica um filtro passa-alta pra cortar a fundamental original, deixando apenas os harmônicos sintetizados fazerem o trabalho. |
| **Fundamental cut slope** | 12 dB/oct / 24 dB/oct | Inclinação do filtro de corte acima. 24 dB/oct corta de forma mais agressiva (dois filtros passa-alta em cascata) que 12 dB/oct (um único filtro). |

#### Sinal, passo a passo (`@sample`)

1. **Extração da banda de graves** — um filtro passa-baixa (`LP_process`) isola tudo abaixo de `bassFreq`.
2. **Geração de harmônicos** — o sinal isolado passa por uma saturação simples (`x / (1 + |x|)`), e depois é elevado ao quadrado (harmônicos pares) e ao cubo (harmônicos ímpares), misturados conforme `evenMix`.
3. **Remoção de DC** — como elevar ao quadrado desloca a média do sinal, um filtro de remoção de DC (integrador simples) evita que isso jogue offset no áudio.
4. **Corte opcional da fundamental** — se `cutFundamental` estiver ligado, o sinal original passa por um ou dois filtros passa-alta em cascata (`HP_process`), dependendo do slider `cutOrder`.
5. **Mixagem final** — o sinal "seco" (com ou sem a fundamental, dependendo da configuração) é somado aos harmônicos sintetizados, ponderado por `harmonicsMix`.

Os filtros (`LP_set`/`HP_set`/`LP_process`/`HP_process`) são biquads RBJ (Robert Bristow-Johnson) padrão, implementados manualmente — cada instância de filtro guarda seus 9 coeficientes/estados (`b0, b1, b2, a1, a2, x1, x2, y1, y2`) num bloco de memória endereçado manualmente, em vez de usar objetos com namespace (`this.algo`). Isso é proposital — veja a seção do fix abaixo.

### Fix para o RootlessJamesDSP: por que o corte não aparecia

Esse script rodava perfeitamente no JDSP4Linux, mas os sliders de corte (`Remove original bass band` / `Fundamental cut slope`) simplesmente não apareciam na UI do RootlessJamesDSP no Android — sem erro nenhum, o controle só sumia.

**Causa raiz:** os dois apps leem sliders de formas fundamentalmente diferentes.

- O **JDSP4Linux** usa o motor EEL2 completo (mesma base do REAPER/WDL-EEL). Quando a VM compila o script, ela já associa automaticamente o valor *default* declarado no cabeçalho do slider (`nome:default<min,max,step>`) como valor inicial da variável — a declaração do slider já é, por si só, a fonte da verdade.

- O **RootlessJamesDSP** não roda a VM completa só para desenhar a interface. Ele usa um parser separado, escrito em Kotlin ([`EelParser.kt`](https://github.com/timschneeb/RootlessJamesDSP/blob/master/app/src/main/java/me/timschneeberger/rootlessjamesdsp/liveprog/EelParser.kt)), que lê o arquivo como texto puro via regex. Esse parser **não entende** que `cutOrder:1<0,1,1{...}>` já define um valor default — ele exige, separadamente, uma linha literal do tipo `cutOrder = 1;` em algum lugar do código para conseguir extrair "qual é o valor atual" via regex (`chave\s*=\s*(-?\d+\.?\d*)\s*;`). Se essa linha não existir, a função de parsing (`findVariable`) retorna nulo, e o slider inteiro é descartado silenciosamente da lista de propriedades — sem log visível na UI, sem crash, ele só não aparece.

No processo de otimizar o script, uma linha `cutOrder = 1;` redundante (o valor já batia com o default do slider) tinha sido removida do bloco `@init` por parecer desnecessária. Resultado: o slider sumia especificamente no RootlessJamesDSP, porque ele depende dessa linha para existir — mesmo que ela seja, do ponto de vista da lógica EEL2 "pura", redundante.

**Correção:** manter sempre uma atribuição explícita (`cutOrder = 1;`) no `@init` para todo slider tipo lista (`{...}`), mesmo quando o valor já é igual ao default declarado no cabeçalho. Essa atribuição também é o que o RootlessJamesDSP reescreve automaticamente (via regex) sempre que você mexe no slider pela UI, então ela funciona tanto como valor inicial quanto como mecanismo de persistência.

#### Outras incompatibilidades corrigidas

Além do bug do slider, duas outras diferenças de compatibilidade entre os dois motores foram ajustadas:

- **Namespace com ponto (`objeto.função()`, `this.campo`)**: sintaxe válida no EEL2 padrão, mas não suportada pelo interpretador embarcado do RootlessJamesDSP. Reescrito para passar o endereço de memória (`base`) como parâmetro explícito das funções, em vez de usar instâncias de objeto.
- **Função `memory(index) = valor`**: também não suportada da forma como o EEL2 padrão aceita. Substituída pela sintaxe de indexação com colchetes (`base[N] = valor`), que é a forma nativa de acesso à memória global aceita por praticamente qualquer implementação de EEL2.

### Compatibilidade

Testado e funcionando em:
- ✅ JDSP4Linux
- ✅ RootlessJamesDSP (Android)


AI-assisted by:

DeepSeek AI - DeepSeek V4.1 Flash

Anthropic - Claude Sonnet 5 (for revision)

Python is easier than EEL2-
