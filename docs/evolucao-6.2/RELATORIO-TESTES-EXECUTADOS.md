# Testes realmente executados — e os bugs que só apareceram ao executá-los

> ## ✔ Confirmado em Windows real
>
> O usuário executou `COMPILAR.cmd` no próprio PC (Windows, PowerShell 5.1) e o
> resultado foi:
>
> ```
> Publicado em native\LukeOptimizer-6.1.2.exe - SHA256 2b8efcd87ec2…
> Manifestos atualizados: 650 arquivos.
> TOTAL: 1415 checks passed, 0 skipped
> UI smoke passed: navigation, filtering, forms, chart and offscreen rendering.
> Validacao concluida.
> ```
>
> Isso confirma, em Windows de verdade e não por inferência:
>
> | Item | Antes | Agora |
> |---|---|---|
> | Publicação em `native\LukeOptimizer-6.1.2.exe` | revisado, **não executado** | **funciona**, com SHA-256 conferido |
> | Os 6 blocos pulados no Linux | pulados por premissa do Windows | **rodaram e passaram** |
> | Compilação com `csc.exe` do .NET Framework | não testada | **compila** |
> | Suíte de interface com WinForms real | rodada sob Xvfb/mono | **passa no WinForms do Windows** |
>
> 1390 verificações aqui (6 puladas) → **1415 no Windows (0 puladas)**: as 25 a
> mais são exatamente os blocos que dependem de API do Windows.
>
> **Continua sem verificação:** elevação por UAC, aplicar uma otimização de
> verdade, desfazer a partir do Histórico, DPI de monitor, e os valores que as
> leituras WMI do `MachineProfile` devolvem num PC real (a lógica é testada; os
> valores retornados, não).


Nas rodadas anteriores eu disse que não conseguia compilar aqui. Isso mudou:
instalei `mono-devel` 6.8.0.105 e `Xvfb` neste contêiner, e **compilei e rodei o
projeto de verdade** — executável, suíte de engine e suíte de interface.

Isso encontrou **três defeitos reais de produto** que nenhuma leitura de código
tinha pegado, todos já presentes no ZIP que você me enviou.

---

## 1. Como foi executado (comandos exatos)

O `compilacao.rsp` usa caminhos com `\` e o compilador do Windows. Para rodar
aqui, converto os separadores; **nada mais é alterado**, inclusive
`/langversion:5`:

```bash
sed 's|\\|/|g' compilacao.rsp | grep -v '^/langversion' > linux.rsp
echo "/langversion:5" >> linux.rsp

# 1. Executável
mcs @linux.rsp -target:winexe -out:LukeOptimizer.exe

# 2. Suíte de engine
mcs @linux.rsp -define:TESTS -target:exe -main:EngineTests -out:tests.exe \
  tests/EngineTests.cs tests/V3Tests.cs tests/V4Tests.cs tests/V5Tests.cs \
  tests/V52Tests.cs tests/V53Tests.cs tests/V54Tests.cs tests/RegressionTests.cs tests/V6Tests.cs
mono tests.exe

# 3. Suíte de interface (WinForms precisa de servidor X)
mcs @linux.rsp -define:TESTS -target:exe -main:UiSmoke -out:ui.exe \
  tests/UiSmoke.cs tests/UnifiedUiTests.cs tests/FunctionalUiTests.cs tests/ProductUiTests.cs
xvfb-run -a --server-args="-screen 0 1600x1000x24" mono ui.exe saida/
```

Saídas completas em `verificacao/tests-linux-mono.log` e
`verificacao/ui-linux-mono.log`.

## 2. Resultado

| Suíte | Resultado |
|---|---|
| Compilação do executável | **OK**, 2 avisos (campos não usados), 0 erros |
| Engine | **1019 verificações passaram, 6 puladas** |
| Interface | **passou por inteiro**, 1 bloco pulado |

### O que “pulado” significa aqui

Nenhum teste foi afrouxado para passar. Onde a **premissa** do teste é um
comportamento que só existe no Windows, ele é marcado `SKIP` com o motivo
escrito, e o total final lista cada um. O que foi pulado:

| Pulado | Por quê |
|---|---|
| Trava de arquivo impedindo gravação atômica | POSIX não tem trava obrigatória; o `rename` sempre vence |
| Persistência/transação/undo de perfil por jogo | Exige caminho com letra de unidade (`C:\…`) e arquivo real |
| Quantidades de memória somente leitura | `GetPerformanceInfo` está em `psapi.dll` |
| Proteção do próprio processo no working set | `SetProcessWorkingSetSizeEx` é do Windows |
| Detecção de arquivo em uso e exclusão permanente | A exclusão usa `FILE_DISPOSITION_INFO` do Win32 |
| Job object, saída de processo filho, timeout, handles | Job object, `cmd.exe`, `ping.exe`, `ProcessFixture.exe` |
| Renderização do modo embutido | Acoplamento por HWND e `PostMessage` do `user32` |

**Isto não substitui rodar `COMPILAR.cmd` no Windows.** O que está provado aqui é
lógica, contrato e fluxo de interface. Elevação de UAC, DPI real de monitor,
gravação em HKLM sob política de domínio e `schtasks` continuam sem verificação.

---

## 3. Os três bugs encontrados por executar

### 3.1 A página Otimizações nascia vazia (crítico)

Em `source/UnifiedInterface.cs`, `ApplyHomeFilter()` fazia:

```csharp
child.Visible = matches && !collapsed;
if (child.Visible) visibleRows++;      // <- relê o valor
```

O *getter* de `Control.Visible` no WinForms **não devolve o que você acabou de
gravar**: ele devolve a visibilidade **efetiva**, subindo até o formulário. Se
qualquer ancestral estiver escondido, a leitura volta `false`.

Como o cartão da categoria é ancestral da linha, isso virou uma trava:

1. `BuildHomeCards()` termina chamando `ApplyHomeFilter()` — **no construtor,
   antes de `Show()`**, quando nada está efetivamente visível;
2. toda linha lê `false`, `visibleRows` fica 0 nas 9 categorias;
3. `showCard = collapsed || visibleRows > 0` → **todas escondidas**;
4. na próxima passada o cartão já está escondido, então as linhas leem `false`
   de novo. **Nunca mais voltava.**

Confirmado por execução: `hidden=9`, os 9 cartões com 200 px (largura padrão de
`Panel`, isto é, `SizeHomeCards` nunca os tocou). Depois da correção: 7 visíveis
(2 filtradas legitimamente por estarem sem opções no modo básico) e todas com a
largura da coluna.

**O mesmo teste falha na versão que você me enviou** — rodei a suíte de
interface compilada a partir do `opt.zip` original e ela para no mesmo ponto
(`Collapsed category`). O bug é anterior a qualquer alteração minha.

Correção: usar o booleano recém-calculado em vez de reler `Visible`. E
`SizeHomeCards` passou a definir a largura **antes** de pular um cartão
filtrado, para ele reaparecer no tamanho certo.

### 3.2 Duas páginas inteiras eram inalcançáveis

`source/Interface.cs`:

```csharp
void ShowPage(string key){
  if(key=="Avançado / Alto Impacto"||key=="Energia avançada"||key=="Ajustes manuais")
    key="Otimizações";
```

`Avançado / Alto Impacto` de fato não existe mais (foi consolidada). Mas
**`Energia avançada` e `Ajustes manuais` existem em `pages`** — e eram
redirecionadas junto. Consequências reais:

- a tela **Ferramentas** cria um botão para **cada** página, inclusive essas
  duas: dois botões que abriam outra coisa em silêncio;
- os 6 itens de `performance-library.json` com `Action:"page"` e
  `Target:"Energia avançada"` nunca abriam a página — e
  `PerformanceInterface.cs:79` ainda chamava `ReadPowerOptions()` para
  preencher uma tela que ninguém veria;
- `EmbeddedInterface.Pages` oferece `Energia avançada` no modo embutido;
- o mapa do `V5Bridge` aponta `energia` e `manual` para elas;
- os dois campos decimais (`SystemResponsiveness`, `Win32PrioritySeparation`),
  descritos no texto do próprio programa como estando em **Ajustes manuais**,
  eram inacessíveis pela interface;
- a própria suíte tirava “print” de `Energia avançada` — e vinha fotografando a
  página Otimizações com esse nome.

Correção: a regra virou estrutural em vez de lista de nomes —
`if(!pages.ContainsKey(key)) key="Otimizações";`. Só cai no fallback o que não é
página de verdade. Isso também elimina a exceção em `pages[key]` para uma chave
desconhecida.

### 3.3 “Restaurar seleção” aplicava sem confirmação

Em `source/UnifiedInterface.cs` o botão ia direto para a elevação:

```csharp
var rec=records.FirstOrDefault(App.CanUndo); if(rec==null)return;
var ids=HomePayload(false);
if(ids.Length>0) await Elevated("--restore-selection", rec.Id+"|"+String.Join(",",ids));
```

Era a **única** ação da página sem tela de revisão. Aplicar, Ultra, remoção de
apps e ajustes manuais todas revisam antes. Restaurar reescreve o Windows com
valores gravados antes — é alteração real, com administrador — e não dizia
**qual backup** usaria nem **o que** mudaria.

O teste `selective restore reviews captured backup before launch` existia e
descrevia o comportamento certo; executando, ele mostrou `requests=3` e nenhuma
sobreposição: a elevação acontecia sem revisão nenhuma.

Correção: revisão obrigatória, nomeando o backup (nome + data/hora local), a
lista de configurações afetadas e o aviso de que uma alteração posterior feita
por outro programa faz a restauração **recusar sobrescrever** em vez de apagar
em silêncio.

---

## 4. Consequência prática para você

A suíte de interface **já estava vermelha na versão que você tem**. Como o
`COMPILAR.cmd` (`VALIDAR-5.4.ps1`) **só publica o binário se tudo passar**, e
preserva o executável anterior quando um teste falha, é coerente com o que você
descreveu: rodar a compilação e continuar com o aplicativo antigo, no tema
antigo.

Com os três defeitos corrigidos, as três suítes passam. Rode `COMPILAR.cmd` no
Windows e confira `verificacao\atual\tests.log` e `ui.log`.

---

## 5. Ajustes nos testes (e por que não são afrouxamento)

| Teste | Antes | Agora | Motivo |
|---|---|---|---|
| `sync client preserved by RAM closure` | `!Closable.Contains("OneDrive")` | OneDrive é nível `agressivo` e o texto do nível nomeia o app e diz que a sincronização para | Contrato antigo caiu junto com os níveis; o novo é mais forte, porque exige o aviso escrito |
| `protected process cannot be closed: steam` | steam intocável | steam existe, mas só no nível `extremo`, nunca no de um clique | Idem |
| `Navigation/control count mismatch` | `servicePicks.Count!=10` | `servicePicks.Count!=App.OptionalServices.Count` | O número cru envelhece a cada catálogo; o invariante é “um controle por serviço opcional” |
| `protected app cannot be selected` | uma asserção | duas: a seleção que chega à elevação (sempre) e o veto visual (só onde a plataforma honra `ListView.ItemCheck`) | O que protege o app é `removalChecked`, o filtro `CanRemove` e a releitura dentro do processo elevado — isso é conferido sempre |

Nenhum teste foi removido, desativado ou marcado como “ignorar”.
