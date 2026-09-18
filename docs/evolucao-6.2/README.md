# LukeOptimizer 6.1.2 → 6.2 · Análise e plano de evolução

Documento produzido a partir da leitura do pacote `opt.zip`
(`LukeOptimizer-6.1.2/codigo-fonte/LukeOptimizer/`), não de suposições.
Todos os trechos de código respeitam as restrições reais do projeto.

## Restrições técnicas que valem para TODO este plano

Lidas de `compilacao.rsp` e `source/app.config`:

| Restrição | Valor | Consequência prática |
|---|---|---|
| Compilador | `csc` + `compilacao.rsp` | Arquivo novo **precisa ser adicionado ao `.rsp`**, senão não entra no build |
| `langversion` | **5** | **Sem** `$"interpolação"`, `?.`, `nameof`, `out var`, membros `=>`, tuplas |
| Framework | .NET Framework 4.8 | WinForms clássico, `System.Management`, `System.ServiceProcess` |
| Plataforma | `x64` | — |
| Recursos | `/resource:source\x.json,x.json` | JSON novo precisa de linha própria no `.rsp` |
| DPI | `PerMonitorV2` | Layouts precisam sobreviver a 100 %/125 %/150 % |

Parâmetros com valor padrão (`int width=185`) **são** C# 4 e continuam válidos;
`async`/`await` **são** C# 5 e continuam válidos.

## Invariantes de segurança que nenhuma proposta pode quebrar

1. `CatalogIds.Validate()` — nenhum ID duplicado, toda `PerformanceEntry`
   com `Action=="native"` precisa de um `Item` implementado.
2. `ExtremeProfile.IsReviewed(item)` — só entra em perfil automático o item que é
   `Native` + `ManualAllowed` + sem `Blocked` + **todas** as operações em
   `HKEY_CURRENT_USER` + **nenhuma** política (`App.IsPolicy`) +
   `ActionCode != ExtraSettings.Compression`.
3. `App.IsPolicy(op)` — chave começando com `Software\Policies` ou
   `Software\Microsoft\Windows\CurrentVersion\Policies` (case-insensitive,
   portanto `SOFTWARE\Policies\...` em HKLM também conta).
4. `LiquidProtocol.Resolve` — o cliente escolhe IDs, nunca chaves de Registro
   ou comandos.
5. Toda gravação passa por `App.ApplyTransaction` / `App.ApplyIsolated`, com
   `SavedValue`/`NativeSaved` e restauração verificada.
6. Nada que só abre uma tela do Windows pode ser registrado como “aplicado”.

## Índice

| Arquivo | Conteúdo |
|---|---|
| [`01-estado-atual-e-pontos-fracos.md`](01-estado-atual-e-pontos-fracos.md) | Compreensão do estado atual + 14 pontos fracos com arquivo:linha |
| [`02-catalogo.md`](02-catalogo.md) | Novas `Rule`/`Item`/`PerformanceEntry` por categoria, com texto pronto e fonte |
| [`03-perfis-por-cenario.md`](03-perfis-por-cenario.md) | `ScenarioProfile` — jogo, desktop, notebook, limpeza, privacidade |
| [`04-ui-ux.md`](04-ui-ux.md) | Navegação, badges de risco/tipo, dashboard, fluxo prévia → resumo, toasts |
| [`05-limpezas-e-diagnosticos.md`](05-limpezas-e-diagnosticos.md) | Storage Sense, pacotes DISM/SFC, novos `ReviewedDiagnostics`, novas categorias de limpeza |
| [`06-modulos-novos.md`](06-modulos-novos.md) | Startup Manager, Tarefas agendadas, Lixeira, Perfis customizados, Sistema & Info |
| [`07-testes-e-robustez.md`](07-testes-e-robustez.md) | Novos testes de catálogo, perfis, UI e módulos |

## Resumo executivo em uma página

**O motor está bom. A vitrine está fechada.**

O núcleo transacional (`Profile.cs`, `IsolatedTransactions.cs`, `ChangeAudit.cs`,
`ExecutionHub.cs`, `ManagedCommand`) é sério: snapshot exato, restauração
verificada valor a valor, job objects com kill-on-close, allowlist de comandos,
recusa de arquivo alterado entre prévia e exclusão. A honestidade textual
(`Effect`/`Tradeoff`/`Source`) é acima da média do mercado.

O problema não é falta de conteúdo — são **673 `Item`, 131 opções na tela
“Otimizações”, 27 páginas** — é que a consolidação 6.1.2 escondeu a
interface para fazer o smoke test passar:

- `UnifiedInterface.cs:59` — a grade inteira de opções (`homeFlow`) nasce
  `Visible=false` e **nunca** é reexibida. As 131 caixas existem no *object
  tree* (o teste as conta) mas o usuário não vê nenhuma.
- `UnifiedInterface.cs:58` — busca e filtro de risco: `Visible=false`.
- `UnifiedInterface.cs:54` — contador “N selecionadas” e “Simular”: `Visible=false`.
- `ProductInterface.cs:93` — grade de categorias de limpeza: `Visible=false`.
- `ProductInterface.cs:56` — lista de processos do Memory Manager: `Visible=false`.

Resultado: “Selecionar tudo”, “Aplicar selecionadas” e “Otimização Ultra”
operam sobre caixas invisíveis. Isso é, de longe, o item nº 1 a corrigir —
e é uma correção de **linhas**, não de arquitetura.

Depois disso, na ordem de retorno sobre esforço:

1. **Perfis por cenário** (`03`) — hoje `ExtremeProfile` tem 3 grupos com lista
   de itens **vazia** (`background`, `windows`, `services`), e o botão Ultra da
   UI ignora os grupos e aplica “tudo que passa em `IsReviewed`”. Divergência
   real entre o que a tela promete e o que o app faz.
2. **Badges de risco/tipo** (`04`) — a informação já existe em `OptionInfo`,
   só não é desenhada.
3. **Startup Manager de verdade** (`06`) — hoje lê só `HKCU\...\Run` com
   allowlist fechada; Discord/Teams/launchers ficam de fora justamente por isso.
4. **Storage Sense e pacotes de reparo guiados** (`05`) — via Policy CSP
   documentado, não via chaves observadas.
5. **Catálogo** (`02`) — 20 regras novas, todas com fonte oficial; nenhuma
   promete FPS.

## O que este plano recusa de propósito

- Tweaks de TCP/Registro “mágicos” sem referência (`NetworkThrottlingIndex`
  já está no catálogo como **experimental** e deve continuar fora de perfil).
- Desativar Defender, SmartScreen, UAC, Windows Update ou drivers.
- Fingir que “abrir Configurações” é uma otimização aplicada.
- Escrever `HKCU\...\StorageSense\Parameters\StoragePolicy` — não é
  documentado pela Microsoft. Usamos o Policy CSP `Storage`, que é.
- Desativar overlays de Discord/Steam/NVIDIA por Registro: não existe política
  oficial. Viram **passo guiado**, registrado como `sugerido`.

---

## Estado da implementação

| Etapa | Estado | Onde |
|---|---|---|
| 1 · reexibir a interface escondida | **aplicada** | `docs/evolucao-6.2/patches/etapa-1-2.patch` · `opt.zip` |
| 2 · `App.ById` + caches + filtro sem rebuild | **aplicada** | idem |
| 3 · perfis por cenário | proposta | `03-perfis-por-cenario.md` |
| — · redução profunda de processos e serviços | **aplicada** | `patches/etapa-3-processos-servicos-e-tema-azul.patch` · `opt.zip` |
| — · tema azul | **aplicada** | idem |
| — · módulo de rede (diagnósticos + ações) | **aplicada** | `patches/etapa-4-rede.patch` · `opt.zip` |
| 4 · badges, navegação, dashboard, toasts | proposta | `04-ui-ux.md` |
| 5 · limpezas e diagnósticos | proposta | `05-limpezas-e-diagnosticos.md` |
| 6 · módulos novos | proposta | `06-modulos-novos.md` |
| 7 · testes | parcial (visibilidade e contagem de categorias) | `07-testes-e-robustez.md` |

**O `.exe` dentro de `opt.zip` continua sendo o antigo.** As etapas 1 e 2 estão
só no código-fonte; não há `csc.exe` nem mono no ambiente onde o patch foi
produzido. Compile com `codigo-fonte\LukeOptimizer\COMPILAR.cmd` no Windows.
Detalhes e pendências em `ETAPA-1-2-NOTAS.txt`.

Os patches aplicam na raiz `LukeOptimizer-6.1.2` com `patch -p1 -i <arquivo>`, na
ordem `etapa-1-2.patch` e depois `etapa-3-processos-servicos-e-tema-azul.patch`.
Ambos foram verificados: reproduzem a árvore alterada byte a byte.

### Redução profunda (fora da numeração original)

Serviços opcionais: **10 → 41**, em três níveis, com uma lista **protegida de 54**
que nunca pode ser desativada (Defender, Update, RPC/DCOM, PlugPlay, rede, áudio,
VSS, SysMain). `App.ValidateServiceCatalog()` confere antes de cada lote que as
duas listas não se cruzam.

Processos fecháveis: **46 → 130**, em três níveis. Launchers e software de
periférico ficam no nível 3 e **nunca** no botão de um clique, porque fechar um
launcher pode encerrar o jogo aberto por ele. Fechar apps deixou de ser
irreversível: o executável de cada app encerrado fica no registro do Histórico e
o botão **Reabrir apps fechados** inicia todos de novo — sem restaurar abas,
janelas ou trabalho não salvo, e sem entrar na cadeia de Desfazer.

### Tema azul

A paleta inteira mora em `Theme.Apply()` (`source/Interface.cs`);
`Theme.Apply(false)` devolve a pele âmbar original. Seleção, hover, linha
alternada, destaque e caixa de marcar eram `Color.FromArgb` fixos em cinco
arquivos — agora todos têm nome no `Theme`, que é o que torna a troca possível.

Duas correções que a troca revelou: `Theme.Blue` era a linha "antes" do gráfico
de FPS e ficaria a 1,49:1 do acento azul (virou `Theme.Contrast`, magenta, a
110° do acento); e `Line` pintava borda **e** botão desligado, papéis que
puxavam em direções opostas (virou `Line` + `Disabled`, ambos melhores que no
tema original).

Os 17 pares de texto reais passam de 4,5:1 nas duas peles, medidos pelo
`TestPalette` em `tests/ProductUiTests.cs`, que implementa a fórmula do WCAG e
falha a validação se alguém baixar um contraste.

### Módulo de rede

A página de rede só media; não tinha nenhuma ação. `source/NetworkTools.cs`
acrescenta três diagnósticos somente-leitura (servidores DNS, sondagem de MTU
por `Ping` com *don't fragment*, perda e variação) e seis ações com allowlist
fechada (`flushdns`, `registerdns`, ARP, `renew`, `winsock reset`,
`int ip reset`), separadas por consequência: cache, conexão e reparo.

As duas destrutivas gravam a configuração atual no Histórico antes de mudar, e
**não executam** se essa gravação falhar. Nenhuma ação de rede tem desfazer,
por isso `Kind="Rede"` fica fora de `App.CanUndo` — há teste garantindo isso.

O status completo do pedido de 13 itens, com o que foi recusado e por quê, está
em [`ENTREGA-REDE-E-STATUS.md`](ENTREGA-REDE-E-STATUS.md).
