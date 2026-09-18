# FASE 1 — Auditoria completa do LukeOptimizer 6.1.2

Todos os números deste documento foram **medidos**, não estimados. Foram obtidos
compilando o projeto e executando uma sonda contra o catálogo que o programa
realmente monta em memória (`App.Items` depois de todas as fontes), não lendo
JSON solto. Onde algo não pôde ser verificado, está dito.

---

## 1. Mapa arquitetural

O projeto é **.NET Framework 4.8 / WinForms / x64**, compilado por `csc` com
`compilacao.rsp` e **`/langversion:5`** — nada de C# 6+ (`$""`, `?.`, `nameof`,
`out var`). 5.091 linhas de código-fonte em 39 arquivos, mas em estilo muito
comprimido: várias linhas passam de 600 caracteres.

### Camadas, de baixo para cima

| Camada | Arquivos | Papel |
|---|---|---|
| **Persistência atômica** | `AtomicStore.cs` | Gravação por `MoveFileEx` com lock de arquivo próprio; leitura coordenada entre a UI e o processo elevado |
| **Motor transacional** | `Profile.cs`, `IsolatedTransactions.cs`, `ChangeAudit.cs`, `BackupBundle.cs`, `SelectionRestore.cs` | `SavedValue` (valor, tipo **e ausência** anteriores) → `Record` → `ApplyTransaction` / `ApplyIsolated`; restauração que **recusa sobrescrever** se outro programa mudou o valor depois |
| **Execução externa** | `ExecutionControl.cs`, `ExecutionHub.cs`, `CommandTools.cs` (`ManagedCommand`) | Processo suspenso + job object *kill-on-close* + `ResumeThread`; sem shell, sem deadlock de pipe, com timeout e cancelamento |
| **Catálogo** | `catalog.json`, `V5Options.cs`, `ImportedOptions.cs`, `V5InputItems.cs`, `ExtraSettings.cs`, `ReviewedFeatures.cs`, `ImportedScripts.cs`, `CatalogIds.cs` | Fontes independentes unificadas em `App.Items` |
| **Metadados** | `OptionInfo.cs`, `PerformanceLibrary.cs`, `ExtremeProfile.cs` | Classificação, risco, reinício, efeito, custo, fonte |
| **Subsistemas** | `Resources.cs`, `Management.cs`, `Maintenance.cs`, `StorageManager.cs`, `NetworkTools.cs`, `NetworkWorkbench.cs`, `InputTuning.cs`, `Gaming.cs`, `AppRemoval.cs`, `HardwareLab.cs`, `SystemTelemetry.cs`, `FrameAnalysis.cs` | Processos, serviços, limpeza, rede, input, jogos, apps, hardware |
| **Interface** | `Interface.cs`, `UnifiedInterface.cs`, `ProductInterface.cs`, `PerformanceInterface.cs`, `FunctionalInterface.cs`, `EasyInterface.cs`, `EmbeddedInterface.cs` | 27 páginas, 9 na barra lateral |
| **Protocolo** | `LiquidProtocol.cs`, `V5Bridge.cs` | Interface de linha/embutida para um cliente externo |

### Fluxo de execução de uma otimização

```
Usuário marca opção
  → HomePayload / ResolveHomeConflicts  (resolve conflitos entre opções)
  → InlineReview                        (revisão obrigatória, na janela)
  → Elevated → novo processo com "runas" (UAC)
  → ApplyTransaction:
       1. ScanHardware
       2. Compatibility(item, machine)   → incompatível vira "ignorado", não erro
       3. BackupBundle.Capture           → grava ANTES de qualquer escrita
       4. Capture(op) por operação       → valor, tipo e ausência anteriores
       5. Expected != atual → aborta     → detecta corrida
       6. Write + readback               → escrita silenciosa é detectada
       7. Falha → rollback do que já foi escrito
  → Record em %LOCALAPPDATA%\LukeOptimizer\historico\<id>.json
```

Isto é sólido. A parte transacional é o melhor do projeto e **não deve ser
reescrita**.

---

## 2. O que existe, em números medidos

| Item | Quantidade |
|---|---|
| Entradas em `App.Items` | **673** |
| — de `catalog.json` (arquivos `.reg`/`.bat` catalogados, **não executáveis**) | 532 |
| — `ImportedOptions` (executáveis) | 89 |
| — `V5Options` (executáveis) | 24 |
| — `ImportedScripts` (os 14 BATs, revisão) | 14 |
| — `Builtins` / `V5InputItems` / `ReviewedFeatures` / `ExtraSettings` | 8 / 3 / 2 / 1 |
| **Otimizações realmente executáveis (`Native`)** | **127** |
| IDs duplicados | **0** |
| Operações de Registro nas executáveis | 215 (118 HKLM, 97 HKCU) |
| — dessas, sob `Software\Policies` | **85** |
| Entradas da biblioteca de desempenho | 780 (127 `native`, 546 `catalog`, 70 `page`, 23 `link`, 8 `settings`, 6 `system`) |
| Serviços opcionais / protegidos | 41 / 54 |
| Processos fecháveis (3 níveis) | 130 (47 / 37 / 46) |
| Apps no catálogo de remoção | 141 |
| Diagnósticos oficiais / ações de rede | 8 / 6 |
| Testes de engine | **1.114 verificações** |

### Classificação atual das 127 executáveis

| Eixo | Distribuição |
|---|---|
| Classificação | 57 condicional · 52 manual · 8 recomendada · 5 não recomendada · 5 experimental |
| Risco | 97 baixo · 24 moderado · 6 alto |
| Reinício | 61 nova sessão · 32 reiniciar Windows · 23 Explorer · 11 nenhum |
| Fonte declarada | **127 de 127 têm URL** e **127 de 127 têm efeito e custo escritos** |

Isso é genuinamente melhor que a média do gênero. O problema não é ausência de
metadado — é que ele está espalhado e **sem nível de evidência**.

---

## 3. Bugs encontrados (todos confirmados por execução)

### 3.1 · CRÍTICO — a página Otimizações nascia vazia

`ApplyHomeFilter()` relia `Control.Visible` logo após gravá-lo. O *getter* devolve
a visibilidade **efetiva** (sobe até o formulário). A primeira chamada acontece
dentro de `BuildHomeCards()`, **antes de `Show()`** — toda linha lia `false`, as
9 categorias se escondiam e **nunca mais voltavam**. Confirmado: 9 cartões
escondidos, todos com 200 px (largura padrão de `Panel`, isto é, nunca
dimensionados).

**A suíte compilada a partir do ZIP original falha no mesmo ponto.** Corrigido.

### 3.2 · GRAVE — duas páginas inteiras inalcançáveis

`ShowPage` redirecionava `"Energia avançada"` e `"Ajustes manuais"` para
`"Otimizações"` — mas **as duas existem em `pages`**. Os botões que a tela
Ferramentas cria para cada página, os 6 itens `Action="page"` da biblioteca, a
navegação embutida e o mapa do `V5Bridge` apontavam para telas que nunca
apareciam. Os dois campos decimais anunciados pelo próprio texto do programa
eram inacessíveis. Corrigido.

### 3.3 · GRAVE — "Restaurar seleção" aplicava sem confirmação

Única ação da página sem tela de revisão, indo direto para a elevação sem dizer
qual backup usaria nem o que mudaria. Corrigido.

### 3.4 · GRAVE — três grupos do perfil Ultra prometiam e não faziam nada

| Grupo | Itens | Vinha marcado? | Texto prometia |
|---|---|---|---|
| `background` "Menos apps em segundo plano" | **0** | não | "reduz ajustes locais já testados" |
| `windows` "Menos atividade do Windows" | **0** | **SIM** | "reduz sugestões locais" |
| `services` "Serviços que você pode dispensar" | **0** | não | "impede Fax, mapas offline e DLNA de iniciarem" |

Os três eram exportados pelo protocolo com título, efeito e — no caso de
`windows` — `Selected=1`. Nenhum produzia uma única alteração, porque
`ExtremeIds()` itera `group.Items`, que estava vazio.

A causa é legítima e vira a explicação: o que caberia ali é **política do
Registro** (recusada por construção em `IsReviewed`), **experimental**
(`pack-e8ea680d9741`) ou **serviço** (que nem é item de catálogo — tem
subsistema próprio). Corrigido de forma estrutural: `Selected` passou a ser
`declarado && Items.Length > 0`, e os três textos agora começam com
"NENHUMA opção entra aqui no botão automático" e dizem para onde ir.

### 3.5 · `MissingMethodException: String.TrimEnd(Char)` (rodada anterior)

As sobrecargas de um caractere de `Trim/TrimStart/TrimEnd` só existem a partir
do .NET Core 2.0. No .NET Framework 4.8 o binário quebrava em tempo de execução.
Corrigido nas 5 ocorrências e travado por teste que **executa** as chamadas.

---

## 4. Recursos incompletos, conceituais e código morto

Usando o critério do item 48 do pedido (*não está implementado se apenas abre
Configurações, só tem botão, ou não confirma a alteração*):

| Situação | Quantidade | Observação |
|---|---|---|
| **IMPLEMENTADO** — altera o Windows, com backup e desfazer verificados | **127** | as opções `Native` |
| **IMPLEMENTADO** — subsistemas com ação real | processos, serviços, limpeza, rede, remoção de apps, perfis por jogo, RAM | |
| **ORIENTAÇÃO** — abre página do Windows ou do navegador | **37** da biblioteca (23 `link`, 8 `settings`, 6 `system`) | O campo `Mode` já diz honestamente "Guia no jogo / driver", "Ajuste no Windows", "Ferramenta do Windows". **Não é desonesto**, mas também não é otimização — e não havia distinção legível por máquina |
| **REVISÃO SOMENTE LEITURA** | 546 arquivos catalogados | `.reg`/`.bat` importados: conteúdo preservado, efeitos explicados, **nada executa** |
| **NÃO IMPLEMENTADO** | tarefas agendadas, perfil de máquina por hardware, navegação por seções, badges de risco | projetados em `06-modulos-novos.md` e `04-ui-ux.md` |
| **CÓDIGO MORTO** | `key=="Avançado / Alto Impacto"?"Avançado":key` em `Interface.cs:113` | `names[]` não contém essa string; o ternário nunca dispara |

### Duplicação real encontrada

- `v5-native-theme-dark` e `v5-native-theme-light` escrevem **o mesmo valor**
  (`AppsUseLightTheme`) com dados opostos. Não é bug: `App.UniqueOperations`
  rejeita a combinação. Mas são duas opções para uma configuração.
- **29 opções elegíveis** (revisadas, HKCU, sem política, reversíveis) **não
  pertencem a grupo nenhum** do perfil automático — inclusive
  `pack-e8ea680d9741`, que é exatamente o tema do grupo `background` vazio.

---

## 5. Riscos de compatibilidade

1. **85 das 215 operações executáveis escrevem sob `Software\Policies`.** Isso é
   muito. `IsReviewed` já as mantém fora do perfil automático (correto), mas
   política pode ser bloqueada por edição do Windows, por domínio ou reaplicada
   pelo Intune sem aviso. O produto avisa nos textos; falta marcar isso como
   **dado**, não como frase.
2. **Nenhuma detecção de hardware além de nome de CPU/GPU e GB de RAM.**
   `Machine` tem 17 campos, todos string/bool. Não havia fabricante, núcleos,
   SMT, tipo de mídia do disco, chassi, Secure Boot ou TPM — então nenhuma
   recomendação podia depender de hardware.
3. **Compatibilidade era binária** (`null` = pode, string = não pode). Não havia
   "não detectado", que é diferente de "não compatível".
4. `SvcHostSplitThresholdInKB` aparece em 6 arquivos catalogados com valores
   diferentes por faixa de RAM. Nenhum é executável — correto.

---

## 6. O que o projeto já acerta, e não deve ser mexido

Vale registrar, porque a tentação de reescrever é grande:

- **Anti-placebo já funciona.** Dos tweaks que o pedido manda olhar com lupa —
  `LargeSystemCache`, `DisablePagingExecutive`, `IoPageLockLimit`,
  `TcpAckFrequency`, `TCPNoDelay`, `SvcHostSplitThresholdInKB`, `HwSchMode`,
  `Win32PrioritySeparation`, `SystemResponsiveness` — **nenhum é executável**.
  Todos existem apenas como arquivo catalogado para leitura. O único executável
  da lista é `NetworkThrottlingIndex`, e ele já estava marcado experimental e
  fora de perfil. **Isto agora é um teste**, não uma coincidência.
- **Segurança:** 54 serviços protegidos que nunca podem ser tocados; processos
  de shell, sessão, anticheat e antivírus fora de qualquer nível; identidade de
  processo reconferida por PID + caminho + horário de início antes de encerrar.
- **Limpeza:** prévia, revalidação antes de excluir, recusa em raiz de volume,
  recusa em junction/symlink, recusa de arquivo alterado depois da prévia.
- **Undo:** guarda valor, tipo **e ausência**; recusa sobrescrever mudança
  posterior de terceiros.

---

## 7. As 10 melhorias de arquitetura de maior impacto

| # | Melhoria | Estado |
|---|---|---|
| 1 | Contrato único por otimização (metadado em um lugar só) | **FEITO** — `OptimizationDescriptor` / `OptimizationRegistry` |
| 2 | Nível de evidência e fonte por otimização | **FEITO** — `Evidence` |
| 3 | Confiança separada do risco | **FEITO** |
| 4 | Detecção estruturada de hardware | **FEITO** — `MachineProfile` |
| 5 | Motor de compatibilidade com 4 estados | **FEITO** — `CompatibilityEngine` |
| 6 | Feature flags / estágio de maturidade | **FEITO** — `Stage` |
| 7 | Acúmulo de reinício (pedir uma vez) | **FEITO** — `RestartPlan` |
| 8 | Invariantes de cadastro conferidos por teste | **FEITO** — `OptimizationRegistry.Validate()` |
| 9 | Perfis compostos dinamicamente por hardware | pendente — FASE 4 |
| 10 | Navegação por seções, busca global e badges | pendente — FASE 3 |

---

## 8. Dívida técnica que permanece

1. **Estilo comprimido.** Linhas de 600+ caracteres com múltiplas instruções
   dificultam revisão e escondem bugs — o defeito 3.1 morava numa dessas.
   Não é para reescrever tudo; é para não escrever mais assim.
2. **`OptionInfo` mistura regra e dado.** Listas `high` e `experimental` estão
   escritas à mão dentro de `Compute`. Deveriam ser cadastro.
3. **Duas fontes de "o que é automático":** os grupos de `ExtremeProfile` e
   `OptionInfo.Automatic`. Hoje coincidem — por teste, não por construção.
4. **546 arquivos catalogados** dominam qualquer contagem e confundem "quantidade
   de recursos" com "quantidade de otimizações".
5. **A suíte de interface estava vermelha.** Como `COMPILAR.cmd` só publica o
   binário se tudo passar, o executável distribuído ficava parado na versão
   anterior. Isso explica o aplicativo continuar no tema antigo.

---

## 9. Roadmap priorizado

| Fase | Conteúdo | Estado |
|---|---|---|
| **1 — Auditoria** | este documento | **concluída** |
| **2 — Fundação** | contrato, evidência, compatibilidade, perfil de máquina, estágio, reinício, invariantes | **concluída** — ver `FASE-2-FUNDACAO.md` |
| **3 — UX** | dashboard, busca global, filtros, prévia universal, modo básico/avançado | próxima |
| **4 — Recomendações** | perfis compostos por hardware (Seguro, Equilibrado, Gaming, Competitivo, Low-end, Notebook, Workstation, Streaming) | depende da 3 |
| **5 — Módulos** | tarefas agendadas, personalização (Taskbar, Start, Explorer, aparência), GPU, áudio, monitores | depende da 2 (o contrato é o que permite adicionar sem tocar em dezenas de arquivos) |
| **6 — Perfis por jogo** | Game Session completa: preparar → capturar → aplicar → monitorar → restaurar | parcial (já existe `GameProfiles`) |
| **7 — Benchmark A/B** | baseline, comparação, manter ou desfazer | parcial (`ResourceSample`, `FrameAnalysis`) |
| **8 — Qualidade** | documentação para o usuário, instalador, release | pendente |
