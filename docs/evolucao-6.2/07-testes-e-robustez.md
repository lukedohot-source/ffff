# 7 · Testes e robustez

O projeto tem suíte séria. O risco aqui não é falta de teste — é que os testes
**fixam números** que qualquer acréscimo quebra, e **não cobrem visibilidade**,
que é justamente onde o 6.1.2 regrediu.

---

## 7.1 Os números fixos que vão quebrar (e como tratar cada um)

```csharp
// tests/ProductUiTests.cs:8
if(pages.Count!=27||nav.Count!=9||homeChecks.Count!=131||App.Items.Count!=673)
 throw new Exception("v6 preserved inventory mismatch");
// tests/ProductUiTests.cs:11
if(storageCategories.Items.Count!=8||memoryInterval.Minimum<1)
 throw new Exception("v6 module controls missing");
// tests/EngineTests.cs:38
Assert(App.Items.Count(i=>i.DefaultSelected)==6,"six defaults; power and mouse are opt-in");
// tests/V6Tests.cs
Assert(CommandTools.Catalog().Count==8&&…,"v6 all official diagnostics have timeout and technical source");
```

A intenção é boa: impedir que alguém apague metade do catálogo sem perceber.
Mas igualdade exata transforma **qualquer** acréscimo em falha. Troque por
**piso + invariante**:

```csharp
 // tests/ProductUiTests.cs — 6.2
 if(pages.Count<27)throw new Exception("v6.2 páginas removidas: "+pages.Count);
 if(nav.Count<9)throw new Exception("v6.2 navegação encolheu: "+nav.Count);
 if(App.Items.Count<673)throw new Exception("v6.2 catálogo encolheu: "+App.Items.Count);
 // homeChecks tem que cobrir TODO item nativo — isto é invariante, não número mágico
 if(homeChecks.Count!=CatalogIds.NativeItems().Count+ReviewedDiagnostics.Ids.Length)
  throw new Exception("v6.2 a página Otimizações não cobre o catálogo nativo: "+homeChecks.Count);
 if(storageCategories.Items.Count!=StorageManager.Categories().Count)
  throw new Exception("v6.2 categorias de limpeza fora de sincronia");
```

```csharp
 // tests/EngineTests.cs — o que importa é que power e mouse continuem opt-in
 Assert(!Builtins.Create().Single(i=>i.ActionCode=="power").DefaultSelected&&
        !Builtins.Create().Single(i=>i.ActionCode=="mouse").DefaultSelected,
  "6.2 power and mouse remain opt-in");
 Assert(Builtins.Create().Count(i=>i.DefaultSelected)>=6,"6.2 default profile not silently emptied");
```

```csharp
 // tests/V6Tests.cs
 Assert(CommandTools.Catalog().Count>=8&&
        CommandTools.Catalog().All(c=>c.TimeoutSeconds>0&&c.Source.StartsWith("https://")),
  "6.2 every official diagnostic has timeout and technical source");
```

---

## 7.2 Testes novos de catálogo — `tests/V62Tests.cs`

Arquivo novo, chamado a partir de `EngineTests.Main` (e acrescentado à linha de
compilação dos testes, no `VALIDAR-5.4.ps1`).

```csharp
using System;
using System.IO;
using System.Linq;
using System.Collections.Generic;

partial class EngineTests {
 static void RunV62CatalogTests(){
  Reset();
  App.Items=CatalogIds.NativeItems();

  // 1. Integridade herdada: IDs únicos e toda ação nativa implementada.
  CatalogIds.Validate();
  Assert(true,"6.2 CatalogIds.Validate passa com o catálogo ampliado");

  // 2. Toda opção nativa tem texto honesto e fonte utilizável.
  foreach(var item in CatalogIds.NativeItems()){
   var review=item.Review==null?new string[0]:item.Review;
   Assert(review.Length>=1&&!String.IsNullOrWhiteSpace(review[0]),
    "6.2 efeito preenchido: "+item.Id);
   Assert(!String.IsNullOrWhiteSpace(item.Url)&&item.Url.StartsWith("https://",StringComparison.Ordinal),
    "6.2 fonte https presente: "+item.Id);
  }

  // 3. Nenhum texto promete FPS. Esta é a regra anti-snake-oil, verificável.
  var forbidden=new[]{"aumenta o fps","mais fps","ganho de fps garantido","turbo","acelera o jogo","boost de desempenho"};
  foreach(var item in CatalogIds.NativeItems()){
   string text=(item.Name+" "+String.Join(" ",item.Review??new string[0])).ToLowerInvariant();
   foreach(string phrase in forbidden)
    Assert(text.IndexOf(phrase,StringComparison.Ordinal)<0,
     "6.2 sem promessa de FPS em "+item.Id+" ("+phrase+")");
  }

  // 4. SourceKind válido em toda parte.
  var kinds=new HashSet<string>(new[]{"Documentado","API oficial","Observado"});
  foreach(var item in CatalogIds.NativeItems())
   Assert(String.IsNullOrEmpty(item.SourceKind)||kinds.Contains(item.SourceKind),
    "6.2 SourceKind reconhecido: "+item.Id);

  // 5. Os itens SPI novos leem e escrevem pelo caminho nativo, sem tocar no Registro.
  foreach(string code in new[]{"gradient","dropshadow","uieffects"}){
   Assert(NativeSettings.Codes(code).Length==1&&NativeSettings.Codes(code)[0]==code,
    "6.2 código SPI mapeado: "+code);
   native[code]="1";
   var item=ReviewedFeatures.Items().Single(i=>i.ActionCode==code);
   Assert(item.Operations.Length==0,"6.2 item SPI não grava Registro: "+code);
   Assert(ExtremeProfile.IsReviewed(item),"6.2 item SPI é elegível a perfil: "+code);
  }
  Reset();
  App.Items=CatalogIds.NativeItems();
  native["gradient"]="1";native["dropshadow"]="1";
  App.ApplyMany(new[]{"native-gradient","native-dropshadow"});
  Assert(native["gradient"]=="0"&&native["dropshadow"]=="0"&&registry.Count==0,
   "6.2 SPI novos aplicados sem gravar Registro");
  App.Undo(Last().Id);
  Assert(native["gradient"]=="1"&&native["dropshadow"]=="1","6.2 SPI novos restaurados exatamente");

  // 6. Delivery Optimization: os três modos se excluem e a engine recusa a mistura.
  var doIds=new[]{"v5-native-delivery-no-peers","v5-native-delivery-lan-only","v5-native-delivery-simple"};
  App.Items=CatalogIds.NativeItems();
  ExpectFailure(new Action(delegate(){App.UniqueOperations(doIds.Select(id=>App.ById(id)).ToList());}),
   "6.2 modos de Delivery Optimization conflitantes rejeitados antes do backup");
  foreach(string id in doIds){
   var item=App.ById(id);
   Assert(item.Conflicts.Length==2,"6.2 conflito declarado em "+id);
   Assert(!ExtremeProfile.IsReviewed(item),"6.2 política de DO fora do perfil automático: "+id);
  }

  // 7. Toda política fica fora do automático — a regra que protege o usuário.
  foreach(var item in CatalogIds.NativeItems()){
   var ops=item.Operations==null?new RegOp[0]:item.Operations;
   bool policy=ops.Any(App.IsPolicy),machineWide=ops.Any(o=>o.Hive!="HKEY_CURRENT_USER");
   if(policy||machineWide)
    Assert(!ExtremeProfile.IsReviewed(item),"6.2 política/HKLM nunca é automática: "+item.Id);
  }

  // 8. Campos manuais: faixa e conjunto fechado respeitados.
  ExpectFailure(new Action(delegate(){ExtraSettings.Manual("responsiveness=101");}),"6.2 SystemResponsiveness acima da faixa recusado");
  ExpectFailure(new Action(delegate(){ExtraSettings.Manual("do-background=0");}),"6.2 banda de DO igual a zero recusada");
  ExpectFailure(new Action(delegate(){ExtraSettings.Manual("storagesense-cadence=15");}),"6.2 cadência fora do conjunto aceito recusada");
  ExpectFailure(new Action(delegate(){ExtraSettings.Manual("inexistente=1");}),"6.2 campo manual desconhecido recusado");
  Assert(ExtraSettings.Manual("storagesense-cadence=7").Count==1,"6.2 cadência válida aceita");
 }
}
```

---

## 7.3 Testes de perfis por cenário

```csharp
 static void RunV62ScenarioTests(){
  Reset();
  App.Items=CatalogIds.NativeItems();

  // 1. Nenhum cenário vazio, nenhum ID fantasma, nada não revisado no automático.
  Scenarios.Validate();
  Assert(true,"6.2 Scenarios.Validate aprova todos os cenários");

  // 2. A regressão de 6.1.2: grupo declarado e sem item nenhum.
  foreach(var group in ExtremeProfile.Groups())
   Assert(group.Items.Length>0,"6.2 grupo do perfil não pode ser vazio: "+group.Id);

  // 3. Uma fonte de verdade: o que OptionInfo chama de automático é o que o cenário aplica.
  var fromScenarios=new HashSet<string>(Scenarios.All().SelectMany(s=>s.Items).Select(CatalogIds.Canonical));
  foreach(var item in CatalogIds.NativeItems()){
   var info=OptionInfo.For(item);
   Assert(info.Automatic==(fromScenarios.Contains(item.Id)&&ExtremeProfile.IsReviewed(item)),
    "6.2 ficha e cenário concordam sobre "+item.Id);
  }

  // 4. Notebook nunca recebe plano de energia — é a promessa explícita do cenário.
  var laptop=Scenarios.Find(Scenarios.Laptop);
  Assert(!laptop.Items.Contains("native-power")&&!laptop.Optional.Contains("native-power"),
   "6.2 cenário Notebook não inclui plano de energia");

  // 5. Passo guiado nunca vira otimização aplicada.
  foreach(var scenario in Scenarios.All())
   foreach(string guided in scenario.Guided)
    Assert(!scenario.Items.Contains(guided)&&!scenario.Optional.Contains(guided),
     "6.2 passo guiado fora das faixas aplicáveis: "+guided);

  // 6. Build monta o plano e ignora com motivo, sem lançar.
  var plan=Scenarios.Build(Scenarios.Desktop,App.TestMachine);
  Assert(plan.Blocked==null&&plan.Apply.Count>0,"6.2 cenário Desktop produz plano aplicável");
  Assert(plan.Apply.All(ExtremeProfile.IsReviewed),"6.2 tudo na faixa automática passa em IsReviewed");
  Assert(plan.Skip.All(s=>!String.IsNullOrWhiteSpace(s.Message)),"6.2 todo item ignorado tem motivo");
  Assert(plan.Confirm.All(i=>!ExtremeProfile.IsReviewed(i)||
   (i.Operations??new RegOp[0]).Any(o=>o.Hive!="HKEY_CURRENT_USER")),
   "6.2 faixa de confirmação só tem política ou alteração para todos");

  // 7. O payload do cenário sobrevive ao Resolve da engine.
  string payload=String.Join(",",plan.Apply.Select(i=>i.Id));
  Assert(ExtremeProfile.Resolve(payload).Count==plan.Apply.Count,
   "6.2 payload do cenário é aceito por ExtremeProfile.Resolve");

  // 8. Um cenário com item não revisado tem que EXPLODIR, não passar batido.
  ExpectFailure(new Action(delegate(){
   ExtremeProfile.Resolve(String.Join(",",new[]{"native-window","v5-native-widgets-off"}));
  }),"6.2 política infiltrada no payload automático é rejeitada");

  // 9. O registro do cenário separa aplicado, ignorado e sugerido.
  Reset();App.Items=CatalogIds.NativeItems();
  plan=Scenarios.Build(Scenarios.Clean,App.TestMachine);
  var record=App.RecordScenarioOutcome(plan,new string[0]);
  Assert(record.Status=="cenário somente orientado","6.2 cenário de limpeza não reporta aplicação");
  Assert(record.Steps.Count(s=>s.Status=="sugerido")==plan.Guide.Count,"6.2 sugeridos registrados como sugeridos");
  Assert(record.Steps.All(s=>s.Status!="aplicado"),"6.2 nenhum passo guiado vira aplicado");
 }
```

---

## 7.4 Testes de UI — o buraco que deixou a grade invisível passar

O teste atual conta `homeChecks.Count` no *object tree*. Um controle
`Visible=false` continua no *object tree*. Por isso a regressão passou.

```csharp
// tests/ProductUiTests.cs — 6.2
 // Visibilidade EFETIVA: sobe a cadeia de pais. Um pai invisível esconde o filho.
 static bool EffectivelyVisible(Control control){
  for(Control current=control;current!=null;current=current.Parent)
   if(!current.Visible)return false;
  return true;
 }

 internal void TestVisibility(){
  ShowPage("Otimizações");PerformLayout();Application.DoEvents();
  if(!EffectivelyVisible(homeFlow))throw new Exception("6.2 a grade de opções está invisível (regressão de 6.1.2)");
  if(!EffectivelyVisible(homeSearch))throw new Exception("6.2 a busca está invisível");
  if(!EffectivelyVisible(homeRisk))throw new Exception("6.2 o filtro de risco está invisível");
  if(!EffectivelyVisible(homeCount))throw new Exception("6.2 o contador de seleção está invisível");

  int shown=0;
  foreach(var pair in homeChecks)if(EffectivelyVisible(pair.Value))shown++;
  if(shown<10)throw new Exception("6.2 só "+shown+" opção(ões) visível(is) na página Otimizações");

  // Com "avançadas" desmarcado, a página abre curada; marcando, mostra tudo.
  homeAdvanced.Checked=false;ApplyHomeFilter();Application.DoEvents();
  int curated=homeChecks.Count(p=>EffectivelyVisible(p.Value));
  homeAdvanced.Checked=true;ApplyHomeFilter();Application.DoEvents();
  int all=homeChecks.Count(p=>EffectivelyVisible(p.Value));
  if(all<=curated)throw new Exception("6.2 a caixa de opções avançadas não muda a lista");

  ShowPage("Temporários");PerformLayout();Application.DoEvents();
  var grid=Controls.Find("cleanup-categories",true);
  foreach(Control control in grid)
   if(control.Parent!=null&&!EffectivelyVisible(control.Parent))
    throw new Exception("6.2 as categorias de limpeza estão invisíveis");
  if(!EffectivelyVisible(cleanupAge))throw new Exception("6.2 o seletor de idade mínima está invisível");
  if(cleanupAge.Value<1)throw new Exception("6.2 a limpeza de um clique ficou sem filtro de idade");

  ShowPage("Memory Manager");PerformLayout();Application.DoEvents();
  if(!EffectivelyVisible(memoryList))throw new Exception("6.2 a lista de processos está invisível");
 }
```

E a cobertura de badges e resoluções:

```csharp
 internal void TestBadges(){
  ShowPage("Otimizações");homeAdvanced.Checked=true;ApplyHomeFilter();Application.DoEvents();
  foreach(var card in homeCards)
   foreach(Control child in card.Controls[0].Controls){
    if(child.Name==null||!child.Name.StartsWith("home-row-",StringComparison.Ordinal))continue;
    bool found=false;
    foreach(Control inner in child.Controls)if(inner is BadgeStrip){
     found=true;
     var badges=((BadgeStrip)inner).Badges;
     if(badges.Count<2)throw new Exception("6.2 badge insuficiente em "+child.Name);
     if(String.IsNullOrEmpty(inner.AccessibleName))throw new Exception("6.2 badge sem AccessibleName em "+child.Name);
    }
    if(!found)throw new Exception("6.2 linha sem faixa de badges: "+child.Name);
   }
 }

 internal void TestResolutions(string folder){
  string[] targets={"Painel","Otimizações","Temporários","Diagnósticos oficiais",
   "Manutenção","Resultados","Hardware e rede","Histórico"};
  foreach(var size in new[]{new Size(1280,720),new Size(1366,768),new Size(1920,1080),
   new Size(2560,1440),new Size(3840,2160)}){
   Size=size;
   foreach(string target in targets){
    ShowPage(target);PerformLayout();Application.DoEvents();
    var page=pages[target];
    if(page.ClientSize.Width<600||page.ClientSize.Height<380)
     throw new Exception("6.2 página colapsada em "+size.Width+"x"+size.Height+": "+target);
    foreach(Control control in page.Controls)
     if(control.Width<=0||control.Height<=0)
      throw new Exception("6.2 controle sem área em "+target);
    // Nenhum texto pode transbordar a faixa de ações.
    foreach(Control control in page.Controls)
     if(control is FlowLayoutPanel&&control.Height>0&&((FlowLayoutPanel)control).PreferredSize.Height>control.Height*2)
      throw new Exception("6.2 barra de ações transbordou em "+target+" @ "+size.Width);
   }
  }
  // 125 % e 150 % simulados; o DPI real de monitor continua exigindo validação manual em VM.
  foreach(float scale in new[]{1.25f,1.5f}){
   Size=new Size(1920,1080);Scale(new SizeF(scale,scale));PerformLayout();
   ShowPage("Otimizações");Application.DoEvents();
   if(homeFlow.ClientSize.Height<200)throw new Exception("6.2 grade colapsou em escala "+scale);
   using(var bitmap=new Bitmap(Width,Height)){
    DrawToBitmap(bitmap,new Rectangle(Point.Empty,Size));
    bitmap.Save(Path.Combine(folder,"v62-scale-"+((int)(scale*100))+".png"));
   }
  }
 }
```

O teste de resolução passa a incluir **1280×720 e 1366×768** — resoluções de
notebook que a suíte atual não cobre (ela começa em 1080×720 e pula para
1920×1080). `MinimumSize` da janela é `1080×720` (`Interface.cs:79`), então
1366×768 é o caso realista mais apertado.

---

## 7.5 Testes dos módulos novos

```csharp
 static void RunV62ModuleTests(){
  Reset();

  // --- Startup Manager -------------------------------------------------
  // A allowlist agora é por CHAVE, não por nome de executável.
  var foreign=new StartupEntry{Name="X",Command="\"C:\\Apps\\x.exe\"",Kind="String",
   Hive="HKEY_CURRENT_USER",Key=@"Software\Microsoft\Windows\CurrentVersion\Explorer\Run"};
  ExpectFailure(new Action(delegate(){
   App.DisableStartup(App.SaveRequest(new OperationRequest{Startup=new List<StartupEntry>{foreign}}));
  }),"6.2 chave de inicialização fora da allowlist é recusada");

  var windowsEntry=new StartupEntry{Name="Y",
   Command="\""+Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Windows),"system32\\x.exe")+"\"",
   Kind="String",Hive="HKEY_CURRENT_USER",Key=@"Software\Microsoft\Windows\CurrentVersion\Run"};
  ExpectFailure(new Action(delegate(){
   App.DisableStartup(App.SaveRequest(new OperationRequest{Startup=new List<StartupEntry>{windowsEntry}}));
  }),"6.2 executável dentro de %WINDIR% continua protegido");

  // Uma entrada legítima: aplica, confere backup, desfaz e volta ao valor exato.
  string runKey=@"Software\Microsoft\Windows\CurrentVersion\Run";
  var op=new RegOp{Hive="HKEY_CURRENT_USER",Key=runKey,Name="Discord",Kind="String",Data="\"C:\\Apps\\Discord.exe\" --start"};
  registry[Key(op)]=Copy(op);
  var entry=new StartupEntry{Name="Discord",Command=op.Data,Kind="String",Hive=op.Hive,Key=runKey};
  App.DisableStartup(App.SaveRequest(new OperationRequest{Startup=new List<StartupEntry>{entry}}));
  Assert(!registry.ContainsKey(Key(op)),"6.2 entrada de inicialização removida");
  var startupRecord=Last();
  Assert(startupRecord.Values.Count==1&&startupRecord.Values[0].Before.Data==op.Data,
   "6.2 valor original guardado por inteiro");
  App.Undo(startupRecord.Id);
  Assert(registry.ContainsKey(Key(op))&&registry[Key(op)].Data==op.Data&&registry[Key(op)].Kind=="String",
   "6.2 desfazer restaura valor e tipo da inicialização");

  // Mudou entre análise e aplicação -> recusa.
  registry[Key(op)]=new RegOp{Hive=op.Hive,Key=op.Key,Name=op.Name,Kind="String",Data="outro"};
  ExpectFailure(new Action(delegate(){
   App.DisableStartup(App.SaveRequest(new OperationRequest{Startup=new List<StartupEntry>{entry}}));
  }),"6.2 inicialização alterada depois da análise é recusada");

  // Estimativa de impacto nunca inventa valor para caminho inexistente.
  Assert(App.StartupImpact("C:\\nao\\existe.exe")=="desconhecido","6.2 impacto desconhecido quando não dá para medir");

  // --- Tarefas agendadas -----------------------------------------------
  ExpectFailure(new Action(delegate(){ScheduledTasks.Validate(@"\Microsoft\Windows\Windows Defender\Scan");}),
   "6.2 tarefa do Defender fora da allowlist");
  ExpectFailure(new Action(delegate(){ScheduledTasks.Validate(@"\Microsoft\Windows\UpdateOrchestrator\Schedule Scan");}),
   "6.2 tarefa do Windows Update fora da allowlist");
  ExpectFailure(new Action(delegate(){ScheduledTasks.Validate("\\Qualquer\" & calc.exe");}),
   "6.2 caminho de tarefa com metacaractere recusado");
  Assert(ScheduledTasks.Optional.Keys.All(k=>k.StartsWith(@"\Microsoft\",StringComparison.Ordinal)),
   "6.2 allowlist de tarefas restrita ao namespace da Microsoft");
  ExpectFailure(new Action(delegate(){App.DisableOptionalTasks(new string[0]);}),"6.2 seleção de tarefas vazia recusada");
  ExpectFailure(new Action(delegate(){
   string dup=ScheduledTasks.Optional.Keys.First();App.DisableOptionalTasks(new[]{dup,dup});
  }),"6.2 tarefa repetida na seleção recusada");

  // --- Perfis customizados ---------------------------------------------
  var prefs=new ProductPreferences();
  prefs.Profiles["Meu jogo"]=new[]{"native-window","native-capture"};
  prefs.Save();
  Assert(ProductPreferences.Load().Profiles["Meu jogo"].Length==2,"6.2 perfil customizado persiste");
  prefs.Profiles["Grande"]=Enumerable.Range(0,201).Select(n=>"x"+n).ToArray();
  ExpectFailure(new Action(delegate(){prefs.Save();}),"6.2 perfil acima de 200 IDs recusado");
  prefs.Profiles.Remove("Grande");
  prefs.Profiles[new String('n',61)]=new[]{"native-window"};
  ExpectFailure(new Action(delegate(){prefs.Save();}),"6.2 nome de perfil acima de 60 caracteres recusado");

  // --- Limpeza: relatório por categoria e espaço realmente liberado ------
  var preview=new CleanupPreview{AgeDays=7,Time=DateTime.UtcNow.ToString("o")};
  preview.Files.Add(new CleanupFile{CategoryId="user-temp",Path="a.tmp",Bytes=2048});
  preview.Files.Add(new CleanupFile{CategoryId="user-temp",Path="b.tmp",Bytes=1024});
  preview.Files.Add(new CleanupFile{CategoryId="thumbnails",Path="c.db",Bytes=4096});
  string report=StorageManager.PerCategory(preview);
  Assert(report.Contains("Total estimado")&&report.Contains("tamanho lógico"),
   "6.2 prévia informa total e avisa que é tamanho lógico");

  // --- Pacotes de diagnóstico ------------------------------------------
  foreach(var package in DiagnosticPackages.All()){
   Assert(!String.IsNullOrWhiteSpace(package.Warning),"6.2 pacote com aviso: "+package.Id);
   foreach(string commandId in package.Commands)
    Assert(CommandTools.Catalog().Any(c=>c.Id==commandId),"6.2 pacote cita comando existente: "+commandId);
   foreach(string readingId in package.Readings)
    Assert(ReviewedDiagnostics.Valid(readingId),"6.2 pacote cita diagnóstico existente: "+readingId);
  }
  ExpectFailure(new Action(delegate(){DiagnosticPackages.Resolve("pack-inventado");}),
   "6.2 pacote fora do catálogo recusado");

  // Relatório com OutputFile não pode escapar da pasta de logs.
  foreach(var command in CommandTools.Catalog())
   if(!String.IsNullOrEmpty(command.OutputFile))
    Assert(command.OutputFile.IndexOfAny(Path.GetInvalidFileNameChars())<0&&
           command.OutputFile.IndexOf("..",StringComparison.Ordinal)<0,
     "6.2 nome de relatório seguro: "+command.Id);
 }
```

---

## 7.6 O que continua exigindo teste manual no Windows

O relatório de consolidação 6.1.2 já diz isso e está certo. Vale manter a lista
explícita, porque ela é curta e é onde mora o risco real:

| Item | Por quê |
|---|---|
| Gravação real de HKLM e política | O ambiente de teste usa `App.TestWrite`; permissão, WOW64 e política de domínio só aparecem no Windows real |
| Elevação UAC | `Elevated` depende de `runas`; cancelamento e código 1223 precisam de clique humano |
| DPI de monitor | `Scale()` simula; `PerMonitorV2` com dois monitores de DPI diferente exige hardware |
| `schtasks /change` | Só confirma no Windows; em Home algumas tarefas não existem |
| `SHEmptyRecycleBin` | Precisa de Lixeira com conteúdo real |
| Plano de energia | `PowerTuning` usa `powercfg`; a verificação de desktop/tomada depende do firmware |
| DISM / SFC | Minutos de execução real; o teste de timeout usa `ping` como substituto |
| Compressão de memória (MMAgent) | Exige administrador e reinício para conferir |

Antes de distribuir, rode em Windows 10 **e** 11, em máquina virtual
descartável, com conta administrador e com conta padrão — a diferença entre as
duas é justamente onde a faixa “confirmação individual” do cenário precisa se
comportar bem.

---

## 7.7 Plugar os testes novos

```csharp
// tests/EngineTests.cs, ao fim de Main, antes do return
  RunV62CatalogTests();
  RunV62ScenarioTests();
  RunV62ModuleTests();
```

```csharp
// tests/UiSmoke.cs, dentro do using(var form=new MainForm())
  form.TestRender(folder);form.TestUnified(folder);form.TestFunctional(folder);form.TestProduct(folder);
  form.TestVisibility();form.TestBadges();form.TestResolutions(folder);
```

E acrescente `tests\V62Tests.cs` à lista de fontes do script de validação
(`VALIDAR-5.4.ps1`), junto de `tests\V6Tests.cs`.

---

## 7.8 Um teste que vale mais que os outros

Se só um puder entrar, que seja este — é o que teria impedido o 6.1.2:

```csharp
 if(!EffectivelyVisible(homeFlow))
  throw new Exception("6.2 a grade de opções está invisível (regressão de 6.1.2)");
```
