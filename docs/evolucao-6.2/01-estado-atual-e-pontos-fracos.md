# 1 · Estado atual do LukeOptimizer 6.1.2 e pontos fracos

## 1.1 Arquitetura como ela realmente é

```
codigo-fonte/LukeOptimizer/
  compilacao.rsp            <- lista de fontes + recursos; nada compila fora daqui
  source/
    LukeOptimizer.cs        Main, roteamento de verbos (--apply, --extreme, --undo…),
                            Item/RegOp/Record/SavedValue, App.IsPolicy, mutex global
    Profile.cs              Builtins.Create() (8 itens nativos), NativeSettings (SPI),
                            ApplyTransaction / ApplyMany / ApplyMaximum, Machine
    IsolatedTransactions.cs ApplyIsolated -> um Record filho por item, falha isolada
    ChangeAudit.cs          SettingEvidence, relatório antes/solicitado/depois
    ExecutionHub.cs         ExecutionReport/ActionOutcome, Simulate, logs TXT+JSON
    ExecutionControl.cs     requests assinados por SID, anti-replay, cancelamento
    CatalogIds.cs           Aliases(), Canonical(), NativeItems(), Validate()
    OptionInfo.cs           Classificação / Risco / Automática / Reinício / Undo
    ExtremeProfile.cs       ExtremeGroup[7], IsReviewed(), Resolve(), Apply(), Skipped()
    V5Options.cs            Rule[24] -> v5-native-*  (navegadores, Widgets, temas, reparos)
    ImportedOptions.cs      optimizer-options.json (89 regras) -> opt-* / pack-*
    ExtraSettings.cs        compressão de memória (MMAgent) + campos manuais numéricos
    ReviewedFeatures.cs     2 itens SPI + ReviewedDiagnostics (4 leituras)
    PerformanceLibrary.cs   performance-library.json (99) + v5-advanced (39) + derivados
    Management.cs           ScanStartup/DisableStartup, OptionalServices(10), ApplyFocus
    StorageManager.cs       CleanupCategory(14), Scan/Execute com DeleteLocked
    Maintenance.cs          TempCleaner (.tmp/.temp/.etl/~*, >7 dias)
    CommandTools.cs         OfficialCommand(8) + ManagedCommand (job object)
    SystemTelemetry.cs      GetPerformanceInfo, WMI GPU, rede, HealthScore
    LiquidProtocol.cs       protocolo local para a casca nativa (ImGui)
    Interface.cs            Theme, MainForm, 9 links de navegação, Elevated()
    UnifiedInterface.cs     página “Otimizações” (131 caixas) — ver 1.3
    ProductInterface.cs     Painel, Memory Manager, Temporários, Avançado, Resultados…
    EasyInterface.cs        Modo fácil, Apps e RAM, Manutenção, Seus 14 BATs
    PerformanceInterface.cs Central de desempenho, Perfis de jogos, Energia, FPS, Hardware
  tests/  EngineTests + V3..V6Tests + UiSmoke + ProductUiTests + FunctionalUiTests
```

Inventário fixado pelos testes (`tests/ProductUiTests.cs:8`):
**27 páginas, 9 links de navegação, 131 caixas em “Otimizações”, 673 `App.Items`.**

### O que está genuinamente bem feito

- **Transação com evidência.** `SavedValue` guarda `Before`, `After`,
  `KeyExisted` e `AbsentKeys`; a restauração confere valor, tipo *e ausência*,
  e recusa quando algo mudou depois (`Management.cs:188`, `ChangeAudit`).
- **Isolamento por item.** `ApplyIsolated` cria um `Record` filho por item —
  uma falha não derruba o lote inteiro.
- **Execução de comando sem furo.** `ManagedCommand` cria o processo suspenso,
  prende num job object `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`, e só então dá
  `ResumeThread`. Sem shell, sem pipe que trava, com timeout e cancelamento.
  O teste `v6 timeout also terminates descendant; no orphan` prova o kill em cascata.
- **Limpeza que recusa arquivo alterado.** `StorageManager.Execute` revalida
  tamanho/datas contra a prévia e aborta o arquivo se mudou; o teste
  `v6 changed file refused after preview` cobre isso.
- **Allowlists reais.** Serviços (`App.OptionalServices`, 10 nomes), comandos
  (`CommandTools.Resolve`), processos fecháveis (`ResourceMonitor.Closable`).
  `Defender`, `Update`, áudio, rede, drivers e anticheat ficam fora por
  construção, não por texto.
- **Texto honesto.** Quase todo `Tradeoff` diz o que se perde, e vários dizem
  explicitamente “não mede FPS”. Isso é raro e deve ser preservado à risca.

## 1.2 Onde a informação está hoje (mapa para quem for editar)

| Quero mudar… | Arquivo | Estrutura |
|---|---|---|
| Item nativo por API (SPI) | `source/Profile.cs` | `Builtins.Create()` + `NativeSettings.Flags` |
| Item SPI “revisado” | `source/ReviewedFeatures.cs` | `Make(code,title,effect)` |
| Regra de Registro curada | `source/V5Options.cs` | `static readonly Rule[] Rules` |
| Regra de Registro importada | `source/optimizer-options.json` | `ImportedOption` |
| Texto longo do catálogo | `source/performance-library.json` | `PerformanceEntry` |
| Perfil automático | `source/ExtremeProfile.cs` | `ExtremeGroup[]` |
| Classificação/risco | `source/OptionInfo.cs` | `OptionInfo.For(item)` |
| Comando oficial | `source/CommandTools.cs` | `CommandTools.Catalog()` |
| Diagnóstico só-leitura | `source/ReviewedFeatures.cs` | `ReviewedDiagnostics` |
| Categoria de limpeza | `source/StorageManager.cs` | `StorageManager.Categories()` |
| Serviço opcional | `source/Management.cs` | `App.OptionalServices` |

## 1.3 Pontos fracos — com localização exata

### F1 (crítico) · A grade de opções está invisível

```
UnifiedInterface.cs:59
homeFlow=new FlowLayoutPanel{...,Visible=false};root.Controls.Add(homeFlow,0,1);
```

`homeFlow` nunca recebe `Visible=true` em lugar nenhum do projeto
(`grep -n "homeFlow" source/*.cs` confirma). As 131 caixas construídas por
`BuildHomeCards()` existem, são contadas pelo teste, respondem a
“Selecionar tudo” — e **não aparecem**. A página “Otimizações” mostra só duas
fileiras de botões de um clique.

Junto com ela ficaram escondidas:

| Linha | Controle | Efeito |
|---|---|---|
| `UnifiedInterface.cs:54` | `third` | contador “N selecionadas” e botão “Simular seleção” somem |
| `UnifiedInterface.cs:55` | `homeAdvanced` | caixa “Opções avançadas” some **e nunca é lida** por `HomeVisible` |
| `UnifiedInterface.cs:58` | `filters` | busca por texto e filtro de risco somem |
| `UnifiedInterface.cs:84` | botões de `CategoryTools` | atalhos por categoria nascem `Visible=false` |
| `ProductInterface.cs:93` | `categoryGrid` | escolha de categoria de limpeza some |
| `ProductInterface.cs:92` | `storageCategories` | **não é defeito**: a `ListView` é o modelo oculto que alimenta `categoryGrid` e deve continuar invisível |
| `ProductInterface.cs:56` | `memoryList` | lista de processos do Memory Manager some |
| `ProductInterface.cs:58` | barra `automatic` | intervalo do modo automático some |
| `ProductInterface.cs:97` | `cleanupAge` | seletor de idade some, fixo em `0` = sem filtro de idade |

**Consequência de segurança em `cleanupAge`:** com `Value=0`, o corte em
`StorageManager.Scan` vira `DateTime.UtcNow`, e a condição
`file.LastWriteTimeUtc>=cutoff` só é verdadeira para arquivos do futuro.
Ou seja: o botão “Limpar todos os temporários” **não tem filtro de idade**.
Ele ainda é contido pela allowlist de extensão (`TempCleaner.AllowedExtension`),
pela prévia e pela confirmação — mas o usuário não sabe disso, e não pode mudar.

Correção em `04-ui-ux.md §4.1` e `05-limpezas-e-diagnosticos.md §5.4`.

### F2 (alto) · `ExtremeProfile` promete grupos que não aplicam nada

```
ExtremeProfile.cs:13  new ExtremeGroup("background","Menos apps em segundo plano",...,false)   // 0 itens
ExtremeProfile.cs:14  new ExtremeGroup("windows","Menos atividade do Windows",...,true)        // 0 itens
ExtremeProfile.cs:16  new ExtremeGroup("services","Serviços que você pode dispensar",...,false)// 0 itens
```

Três dos sete grupos têm `Items` vazio. O grupo `windows` inclusive vem
**marcado** (`Selected=true`) e não faz nada. `ExtremeProfile.Rows()` publica
esses grupos para a casca nativa, que mostra promessas vazias.

### F3 (alto) · O botão “Ultra” da UI ignora os grupos

```
UnifiedInterface.cs:139  // HomePayload(extreme:true)
var allowed=new HashSet<string>(CatalogIds.NativeItems().Where(ExtremeProfile.IsReviewed).Select(i=>i.Id));
```

`RunUltraProfile` → `SetHomeSelection(true)` → `HomePayload(true)` aplica
**todo item que passa em `IsReviewed`**, não o conteúdo dos `ExtremeGroup`.
Já `OptionInfo.For(...).Automatic` calcula “está no perfil?” a partir dos
grupos. Então a mesma opção pode aparecer como *“Condicional / escolha
individual”* na ficha e mesmo assim ser aplicada pelo Ultra.
Duas fontes de verdade para a mesma pergunta.

### F4 (alto) · Custo de CPU por tecla digitada na busca

`BuildHomeCards()` é reconstruído inteiro a cada `TextChanged` e, por opção:

- `HomeVisible` → `App.Items.Single(i=>i.Id==option.Id)` — varredura linear em **673** itens;
- `OptionInfo.For(item)` → aloca `ExtremeProfile.Groups()` (7 grupos, 7 arrays),
  monta 2 `HashSet`, roda `LiquidProtocol.NeedsAdmin`, e faz
  `ImportedOptions.Rules().FirstOrDefault` em **89** regras.

131 × (673 + 89 + 7 grupos) por tecla, mais o `Dispose()` e a recriação de
131 `TableLayoutPanel` + 131 `CheckBox`. É a razão pela qual a busca precisou
ser escondida para o smoke test passar. Correção: índice + cache
(`04-ui-ux.md §4.6`).

### F5 (médio) · Navegação esconde 18 das 27 páginas

```
Interface.cs:86
string[] names={"Painel","Otimizações","Temporários","Memory Manager","Ferramentas",
                "Catálogo","Diagnóstico","Histórico","Resultados"};
```

Existem 27 páginas. As outras 18 — “Central de desempenho”, “Perfis de jogos”,
“Energia avançada”, “Rede e latência”, “Hardware e rede”, “Manutenção”,
“Remover apps”, “Diagnósticos oficiais”, “Avançado / Alto Impacto”,
“Configurações”, “Modo fácil”, “Visão geral”… — só existem atrás do botão
“Ferramentas”, que as lista como uma parede indiferenciada de botões
(`UnifiedInterface.cs:175`).

Pior: `BuildDashboard()` cria a página **“Visão geral”**, mas a navegação
aponta para **“Painel”** (`ProductInterface.cs:37`). A tela de abertura
histórica do produto virou órfã.

### F6 (médio) · Categorias do catálogo com prefixo numérico vazando

`ImportedOptions.Items()` gera `Category="-4 - "+r.Category`,
`V5Options` gera `"-3 - "`, `ReviewedFeatures` `"-5 - "`,
`Builtins` `"-1 - Ajustes revisados"`. O prefixo existe só para ordenar e
aparece na UI. E a taxonomia tem três vocabulários concorrentes:

- `optimizer-options.json`: 5 categorias (`Avançado: perde recursos` 35,
  `Personalização` 24, `Privacidade opcional` 24, `Windows leve` 4, `Rede e downloads` 2);
- `performance-library.json`: 17 categorias (`GPU e drivers`, `Windows e apps`,
  `Jogos e FPS`, `CPU e energia`, `RAM e estabilidade`, `Input e monitor`…);
- `UnifiedInterface.HomeCategories`: 9 (`FPS`, `Input Lag`, `Windows`,
  `Processos`, `Inicialização`, `Rede`, `Energia`, `SSD`, `Diagnóstico`).

O mapeamento entre elas é o heurístico de string
`AdvancedCategory` (`UnifiedInterface.cs:66`), que classifica por
`Contains("energia")`, `Contains("jog")` etc. Qualquer opção que não bata cai
em `"Windows"` — e é por isso que a categoria “Windows” fica lotada.

### F7 (médio) · `Categoria "-4 - Avançado: perde recursos"` para 35 itens inofensivos

`ImportedOptions` marca **toda** regra importada com `Category="-4 - "+r.Category`,
e o `optimizer-options.json` classifica 35 regras como
“Avançado: perde recursos”. Entre elas há coisas como
`opt-enableperiodicregistrybackup` (que *habilita* um backup) e vários ajustes
de aparência. O rótulo assusta sem motivo, e o rótulo real do risco já existe
em `OptionInfo.Risk`.

### F8 (médio) · Chave sem valor no catálogo importado

```
opt-disableshowmoreoptions:
  HKCU\Software\Classes\CLSID\{86ca1aa0-...}\InprocServer32  Name=""  Data=""
```

É o truque do menu de contexto clássico do Windows 11. Funciona, mas o texto
não avisa que ele depende de um CLSID não documentado e que a Microsoft pode
remover o comportamento em qualquer build. Precisa de `SourceKind=Observado`
(ver `02-catalogo.md §2.1`).

### F9 (médio) · Código morto na página de otimizações

- `RunQuickProfile()` (`UnifiedInterface.cs:165`) — completo, nunca ligado a botão.
- `inlineTools` (`:63`), `filterHint` (`:58`), `note` (`:171`) — criados e nunca
  adicionados a um pai; vazam até o GC.
- `SetHomeSelection` filtra por `pair.Value.Text.Contains("Diagnóstico")`
  (`:132`), mas nenhum título de diagnóstico contém essa palavra
  (“Diagnosticar retransmissões TCP”, “Analisar espaço das unidades”…).
  Só funciona porque `App.Items.FirstOrDefault` devolve `null` para diagnósticos.
  Deve usar `homeOptions[pair.Key].Diagnostic`.

### F10 (médio) · O Startup Manager quase não encontra nada

`App.StartupAllowed` (`Management.cs:67`) só aceita entrada cujo executável
esteja em `ResourceMonitor.Closable` ou seja `OneDrive`/`steam`/`EpicGamesLauncher`.
E `ScanStartup` lê **apenas** `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`.
Ficam de fora: `HKLM\...\Run`, `Wow6432Node\...\Run`, as pastas Startup, e o
mecanismo que o Gerenciador de Tarefas realmente usa (`StartupApproved`).
Ou seja: exatamente os Discord/Teams/launchers que o pedido cita.
Proposta em `06-modulos-novos.md §6.1`.

### F11 (baixo) · Não existe visão de tarefas agendadas

`grep -rin "schtasks\|ScheduledTask" source/` → zero ocorrências.

### F12 (baixo) · Storage Sense só existe como link

Duas ocorrências de `ms-settings:storagesense` (`Interface.cs:146`,
`EasyInterface.cs:197`), nenhuma leitura ou configuração.
Proposta documentada em `05-limpezas-e-diagnosticos.md §5.1`.

### F13 (baixo) · Diálogos bloqueantes fora do modo embutido

`Confirm()` (`EasyInterface.cs:125`) usa `InlineReview` (overlay, bom) **só**
quando `EmbeddedInterface.Parent != IntPtr.Zero`. No app normal cai em
`MessageBox.Show` modal. O overlay já está pronto e é melhor; falta ligá-lo.

### F14 (baixo) · Sem toasts, sem feedback não bloqueante

Todo retorno vai para `footer.Text` (uma `Label` de 9 pt no rodapé) ou para um
`MessageBox`. Não há nível intermediário. Proposta em `04-ui-ux.md §4.5`.

## 1.4 Ordem de execução recomendada

| Etapa | Escopo | Arquivos tocados | Risco |
|---|---|---|---|
| 1 | F1 · reexibir grade, busca, filtros, categorias de limpeza | `UnifiedInterface.cs`, `ProductInterface.cs` | baixo |
| 2 | F4 · `App.ById` + cache de `OptionInfo`/`Groups` | `LukeOptimizer.cs`, `OptionInfo.cs`, `ExtremeProfile.cs` | baixo |
| 3 | F2+F3 · `ScenarioProfile` único como fonte de verdade | `ExtremeProfile.cs`, `UnifiedInterface.cs` | médio |
| 4 | Badges + navegação por seções (F5, F6, F7) | `Interface.cs`, `UnifiedInterface.cs`, `OptionInfo.cs` | médio |
| 5 | Startup Manager ampliado (F10) | `Management.cs`, `EasyInterface.cs` | médio |
| 6 | Storage Sense + pacotes DISM/SFC (F12) | `V5Options.cs`, `CommandTools.cs` | baixo |
| 7 | Tarefas agendadas (F11) | novo `ScheduledTasks.cs` | médio |
| 8 | Catálogo novo (20 regras) | `V5Options.cs`, `ReviewedFeatures.cs`, JSON | baixo |

Etapas 1 e 2 sozinhas já transformam a percepção do produto e não mexem em
nenhuma gravação.
