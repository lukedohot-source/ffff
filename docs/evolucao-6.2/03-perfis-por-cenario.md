# 3 · Perfis inteligentes por cenário

## 3.1 O problema a resolver antes de acrescentar perfis

Hoje existem **duas** respostas para “esta opção entra no automático?”:

```csharp
// OptionInfo.cs:8 — resposta A: os grupos de ExtremeProfile
var automatic=new HashSet<string>(ExtremeProfile.Groups().Where(g=>g.Selected).SelectMany(g=>g.Items));

// UnifiedInterface.cs:139 — resposta B: tudo que passa em IsReviewed
var allowed=new HashSet<string>(CatalogIds.NativeItems().Where(ExtremeProfile.IsReviewed).Select(i=>i.Id));
```

A ficha da opção usa A (“Condicional / escolha individual”), o botão Ultra usa B
(aplica assim mesmo). E três dos sete grupos de A têm `Items` **vazio**.

Antes de criar cenário nenhum: **uma fonte de verdade**. É o que o
`ScenarioProfile` abaixo faz.

---

## 3.2 Arquivo novo · `source/Scenarios.cs`

Adicione ao `compilacao.rsp` logo após `source\ExtremeProfile.cs`.

```csharp
using System;
using System.Linq;
using System.Collections.Generic;

// Um cenário = conjunto fechado de IDs existentes, dividido em três faixas:
//  Items    -> aplicados pela engine transacional; TODOS devem passar em ExtremeProfile.IsReviewed
//  Optional -> políticas / HKLM / alto risco; exigem confirmação individual, nunca entram no "fazer tudo"
//  Guided   -> diagnósticos e ferramentas; registrados como "sugerido", NUNCA como "aplicado"
// O cliente não pode inventar cenário: a lista é estática e validada no arranque.
class ScenarioProfile {
 internal string Id,Title,Subtitle,Effect,Tradeoff;
 internal string[] Items,Optional,Guided;
 internal Func<Machine,string> Unavailable;   // null ou motivo textual
 internal ScenarioProfile(string id,string title,string subtitle,string effect,string tradeoff){
  Id=id;Title=title;Subtitle=subtitle;Effect=effect;Tradeoff=tradeoff;
  Items=new string[0];Optional=new string[0];Guided=new string[0];
 }
}

class ScenarioPlan {
 internal ScenarioProfile Profile;
 internal List<Item> Apply=new List<Item>();
 internal List<Item> Confirm=new List<Item>();
 internal List<StepResult> Skip=new List<StepResult>();
 internal List<StepResult> Guide=new List<StepResult>();
 internal string Blocked;
}

static class Scenarios {
 internal const string Game="cenario-jogo",Desktop="cenario-desktop",Laptop="cenario-notebook",
  Clean="cenario-limpeza",Privacy="cenario-privacidade";

 internal static ScenarioProfile[] All(){
  var game=new ScenarioProfile(Game,"Sessão de jogo rápida",
   "Antes de entrar na partida · reversível no Histórico",
   "Liga a preferência do Modo Jogo, desliga a gravação em segundo plano da Game Bar e, só em desktop ligado à tomada, seleciona o plano Alto desempenho.",
   "A Game Bar deixa de gravar clipes automáticos. O plano de energia aumenta consumo e calor; em notebook e em máquina fora da tomada essa etapa é ignorada com motivo registrado. Nada aqui foi medido em FPS pelo aplicativo — compare você mesmo na página FPS e stutter.");
  game.Items=new[]{"native-gamemode","native-capture","native-power","native-gradient"};
  game.Guided=new[]{"guide-overlays","diag54-power"};

  var desktop=new ScenarioProfile(Desktop,"Desktop gamer silencioso",
   "Menos coisa rodando por trás · sem tocar em política protegida",
   "Reduz animações, transparência, sombras e espera de submenu, liga o Modo Jogo e desliga a gravação em segundo plano. Tudo por preferência do usuário atual, sem política de computador.",
   "A interface fica visualmente mais simples e a sensação de resposta melhora; isso não é FPS de jogo. Nada aqui desativa serviço, atualização ou proteção. Itens de navegador, Widgets e Delivery Optimization ficam na faixa de confirmação individual porque são políticas para todos os usuários.");
  desktop.Items=new[]{"native-gamemode","native-capture","native-window","native-effects","native-menu",
   "native-transparency","native-dragoutline","native-gradient","native-dropshadow",
   "pack-d1b3e7ae9f70","pack-3d76b7e8529c","v5-native-taskview-hide","v5-native-gamebar-tips-off"};
  desktop.Optional=new[]{"v5-native-edge-background","v5-native-edge-startupboost","v5-native-edge-sleeping-tabs",
   "v5-native-chrome-background","v5-native-delivery-lan-only","v5-native-widgets-off"};
  desktop.Guided=new[]{"guide-startup","guide-overlays"};

  var laptop=new ScenarioProfile(Laptop,"Notebook na bateria",
   "Autonomia primeiro · nenhum plano de energia agressivo",
   "Reduz o trabalho visual e o que roda em segundo plano sem mexer no plano de energia. Mantém brilho, suspensão e gerenciamento térmico como o fabricante configurou.",
   "Deliberadamente NÃO inclui o plano Alto desempenho: em notebook ele reduz autonomia e aumenta ruído do ventilador sem ganho confiável. Se quiser mesmo, aplique a opção nativa de energia à parte, ligado à tomada. A interface fica mais simples; isso não altera FPS.");
  laptop.Items=new[]{"native-window","native-effects","native-menu","native-transparency",
   "native-dropshadow","native-gradient","pack-d1b3e7ae9f70","pack-3d76b7e8529c",
   "v5-native-tailored-experiences-off","v5-native-start-recent-off"};
  laptop.Optional=new[]{"v5-native-edge-sleeping-tabs","v5-native-delivery-background-cap"};
  laptop.Guided=new[]{"guide-startup","diag54-power","diag54-stability"};

  var privacy=new ScenarioProfile(Privacy,"Privacidade moderada",
   "Menos dado opcional saindo · sem quebrar recurso do Windows",
   "Desliga personalização baseada em diagnóstico, coleta de escrita e digitação e o rastreio de itens recentes — tudo no perfil do usuário atual.",
   "Escolha de privacidade, NÃO de desempenho: nada aqui muda FPS. Correção automática de texto e sugestões de digitação ficam piores. Telemetria obrigatória de segurança e atualização continua, por design do Windows. Itens que exigem política de computador ficam na faixa de confirmação individual.");
  privacy.Items=new[]{"v5-native-tailored-experiences-off","v5-native-start-recent-off",
   "pack-0b9181ab23ef","pack-1b4dfbdca78f","pack-c8023d85a069","pack-b8cef6963679"};
  privacy.Optional=new[]{"opt-splitdisabledbygrouppolicy","opt-splituploaduseractivities",
   "opt-splitenableactivityfeed","opt-splitallowlinguisticdatacollection",
   "v5-native-telemetry-required","v5-native-firefox-telemetry","v5-native-office-telemetry"};
  privacy.Guided=new string[0];

  var clean=new ScenarioProfile(Clean,"Limpezas e diagnósticos oficiais",
   "Só leitura e ferramentas do Windows · nenhum tweak",
   "Executa os diagnósticos somente-leitura do aplicativo e as verificações oficiais DISM e SFC, e abre a prévia de limpeza de temporários. Nenhum valor do Registro é alterado por este cenário.",
   "DISM ScanHealth e SFC podem levar vários minutos e usar CPU e disco — não rode durante uma partida. São VERIFICAÇÕES: nenhuma delas repara sozinha. A limpeza de temporários é permanente e sem desfazer, por isso só a prévia é aberta; a exclusão continua exigindo confirmação sua.");
  clean.Items=new string[0];
  clean.Optional=new[]{"v5-native-storagesense-on","v5-native-storagesense-temp","v5-native-storagesense-monthly"};
  clean.Guided=new[]{"diag54-storage","diag54-tcp","diag54-power","diag54-stability",
   "cmd-dism-check","cmd-dism-scan","cmd-sfc-verify","guide-cleanup","guide-storagesense"};

  return new[]{game,desktop,laptop,privacy,clean};
 }

 internal static ScenarioProfile Find(string id){
  var profile=All().SingleOrDefault(s=>s.Id==id);
  if(profile==null)throw new Exception("Cenário desconhecido: "+id);
  return profile;
 }

 // Toda a decisão fica aqui. A UI só desenha o resultado.
 internal static ScenarioPlan Build(string id,Machine machine){
  var profile=Find(id);var plan=new ScenarioPlan{Profile=profile};
  if(profile.Unavailable!=null)plan.Blocked=profile.Unavailable(machine);
  if(plan.Blocked!=null)return plan;
  foreach(string itemId in profile.Items){
   var item=App.Items.FirstOrDefault(i=>i.Id==CatalogIds.Canonical(itemId));
   if(item==null){plan.Skip.Add(Step(itemId,itemId,"ignorado","Opção não existe mais neste catálogo."));continue;}
   if(!ExtremeProfile.IsReviewed(item)){plan.Skip.Add(Step(item.Id,item.ReviewTitle,"ignorado","Deixou de ser elegível ao perfil automático; aplique individualmente."));continue;}
   var state=App.Observe(item,machine);
   if(state.State=="Não compatível"||state.State=="Falha na leitura"){plan.Skip.Add(Step(item.Id,item.ReviewTitle,"ignorado",state.Detail));continue;}
   plan.Apply.Add(item);
  }
  foreach(string itemId in profile.Optional){
   var item=App.Items.FirstOrDefault(i=>i.Id==CatalogIds.Canonical(itemId));
   if(item==null)continue;
   var state=App.Observe(item,machine);
   if(state.State=="Não compatível"){plan.Skip.Add(Step(item.Id,item.ReviewTitle,"ignorado",state.Detail));continue;}
   plan.Confirm.Add(item);
  }
  foreach(string guideId in profile.Guided)
   plan.Guide.Add(Step(guideId,GuideTitle(guideId),"sugerido",GuideDetail(guideId)));
  return plan;
 }

 static StepResult Step(string id,string name,string status,string message){
  return new StepResult{Id=id,Name=name,Status=status,Message=message};
 }

 internal static string GuideTitle(string id){
  if(ReviewedDiagnostics.Valid(id))return ReviewedDiagnostics.Title(id);
  if(id.StartsWith("cmd-",StringComparison.Ordinal))return CommandTools.Resolve(id.Substring(4)).Name;
  if(id=="guide-startup")return "Revisar apps que abrem com o Windows";
  if(id=="guide-overlays")return "Revisar sobreposições de jogo";
  if(id=="guide-cleanup")return "Abrir a prévia de limpeza de temporários";
  if(id=="guide-storagesense")return "Abrir o Sensor de Armazenamento do Windows";
  return id;
 }
 internal static string GuideDetail(string id){
  if(ReviewedDiagnostics.Valid(id))return ReviewedDiagnostics.Detail(id);
  if(id.StartsWith("cmd-",StringComparison.Ordinal))return CommandTools.Resolve(id.Substring(4)).Description;
  if(id=="guide-startup")return "Abre o Startup Manager do aplicativo. Cada entrada desativada tem backup exato do valor e desfazer no Histórico. O passo em si não altera nada.";
  if(id=="guide-overlays")return "Não existe política oficial de Registro para sobreposição de Steam, Discord, NVIDIA ou AMD. Este passo apenas abre as telas certas; o aplicativo não marca nada como aplicado.";
  if(id=="guide-cleanup")return "Abre a página Temporários com a prévia por categoria. A exclusão é permanente e continua exigindo confirmação sua.";
  if(id=="guide-storagesense")return "Abre Configurações > Sistema > Armazenamento. O Windows decide o quê e quando limpar; o aplicativo não apaga nada por você neste passo.";
  return "";
 }
 internal static string Rows(){
  var body="";
  foreach(var s in All())
   body+=LiquidProtocol.Row("scenario",s.Id,s.Title,s.Subtitle,s.Effect,s.Tradeoff,
    String.Join(",",s.Items),String.Join(",",s.Optional),String.Join(",",s.Guided));
  return body;
 }
 // Chamado por CatalogIds.Validate(): nenhum cenário pode referenciar ID inexistente
 // nem colocar item não revisado na faixa automática.
 internal static void Validate(){
  var known=new HashSet<string>(CatalogIds.NativeItems().Select(i=>i.Id));
  var reviewed=new HashSet<string>(CatalogIds.NativeItems().Where(ExtremeProfile.IsReviewed).Select(i=>i.Id));
  if(All().Select(s=>s.Id).Distinct().Count()!=All().Length)throw new Exception("Cenários com ID duplicado.");
  foreach(var s in All()){
   foreach(string id in s.Items){
    string canonical=CatalogIds.Canonical(id);
    if(!known.Contains(canonical))throw new Exception("Cenário "+s.Id+" cita opção inexistente: "+id);
    if(!reviewed.Contains(canonical))throw new Exception("Cenário "+s.Id+" coloca opção não revisada na faixa automática: "+id);
   }
   foreach(string id in s.Optional)
    if(!known.Contains(CatalogIds.Canonical(id)))throw new Exception("Cenário "+s.Id+" cita opção opcional inexistente: "+id);
   foreach(string id in s.Guided)
    if(!ReviewedDiagnostics.Valid(id)&&!id.StartsWith("guide-",StringComparison.Ordinal)&&!id.StartsWith("cmd-",StringComparison.Ordinal))
     throw new Exception("Cenário "+s.Id+" cita passo guiado desconhecido: "+id);
   if(s.Items.Length==0&&s.Optional.Length==0&&s.Guided.Length==0)
    throw new Exception("Cenário vazio: "+s.Id);   // <- impede a regressão dos grupos vazios de 6.1.2
  }
 }
}
```

A última verificação (`Cenário vazio`) é a que impede que os grupos
`background`, `windows` e `services` do 6.1.2 voltem a acontecer.

---

## 3.3 `source/ExtremeProfile.cs` — uma fonte de verdade

Substitua `Groups()` por uma projeção do cenário “Desktop gamer silencioso”,
e acrescente o registro de orientação.

```csharp
 // 6.2: os grupos deixam de ser uma lista paralela e passam a ser a vitrine
 // do cenário Desktop. Quem quiser outro conjunto usa Scenarios.Build.
 internal static ExtremeGroup[] Groups(){
  var desktop=Scenarios.Find(Scenarios.Desktop);
  var visual=new[]{"native-window","native-effects","native-menu","native-transparency",
   "native-dragoutline","native-gradient","native-dropshadow","pack-d1b3e7ae9f70","pack-3d76b7e8529c"};
  var games=new[]{"native-gamemode","native-capture"};
  return new[]{
   new ExtremeGroup("games","Priorizar o jogo",
    "Ativa o Modo Jogo e desliga a gravação de clipes em segundo plano.",
    "A Game Bar deixa de gravar clipes em segundo plano. Confira o resultado no seu jogo.",
    true,games.Where(id=>desktop.Items.Contains(id)).ToArray()),
   new ExtremeGroup("visual","Interface mais leve",
    "Reduz animações, transparência, sombras, degradê da barra de título e espera dos menus.",
    "O Windows fica visualmente mais simples. Isso melhora a sensação de resposta da interface; não mede FPS do jogo.",
    true,visual.Where(id=>desktop.Items.Contains(id)).ToArray()),
   new ExtremeGroup("apps","Menos ruído dos apps",
    "Oculta a Visão de Tarefas e o painel de dicas da Game Bar para o usuário atual.",
    "Preferências visuais reversíveis. Políticas de navegador, Widgets e Delivery Optimization NÃO entram aqui: elas valem para todos os usuários e ficam na confirmação individual.",
    true,new[]{"v5-native-taskview-hide","v5-native-gamebar-tips-off"}.Where(id=>desktop.Items.Contains(id)).ToArray()),
   new ExtremeGroup("power","Energia para desempenho",
    "Seleciona Alto desempenho em desktops compatíveis ligados à tomada.",
    "Opcional: aumenta consumo e calor. A vantagem depende do hardware e do jogo. Em notebook a etapa é ignorada com motivo registrado. O plano anterior fica salvo para desfazer.",
    false,"native-power")
  };
 }
```

Note que `services` sumiu: desativar serviço nunca deveria estar num botão
“fazer tudo”. Ele continua disponível, com confirmação individual, na página
Manutenção — que é onde já está implementado (`EasyInterface.cs:176`).

### Registro de orientação — o log que o pedido cobra

```csharp
// source/ExtremeProfile.cs (ou Scenarios.cs), dentro de static partial class App
static partial class App {
 internal static Record RecordScenarioOutcome(ScenarioPlan plan,string[] appliedIds){
  var record=NewRecord("Cenário: "+plan.Profile.Title,plan.Profile.Id,"Cenário");
  record.Steps=new List<StepResult>();
  foreach(var item in plan.Apply)
   record.Steps.Add(new StepResult{Id=item.Id,Name=item.ReviewTitle,
    Status=appliedIds.Contains(item.Id)?"aplicado":"não executado",
    Message=appliedIds.Contains(item.Id)?"Valor anterior salvo; desfazer disponível no Histórico.":"Não chegou a ser gravado nesta execução."});
  foreach(var item in plan.Confirm)
   record.Steps.Add(new StepResult{Id=item.Id,Name=item.ReviewTitle,Status="confirmação individual",
    Message="Política ou alteração para todos os usuários. Fica fora do automático por decisão de projeto."});
  foreach(var step in plan.Skip)record.Steps.Add(step);
  foreach(var step in plan.Guide)record.Steps.Add(step);
  record.Status=appliedIds.Length>0?"cenário aplicado":"cenário somente orientado";
  record.Message=plan.Profile.Effect+"\r\n\r\n"+plan.Profile.Tradeoff+
   "\r\n\r\nAplicados: "+appliedIds.Length+
   " · Ignorados: "+plan.Skip.Count+
   " · Para confirmar individualmente: "+plan.Confirm.Count+
   " · Sugeridos: "+plan.Guide.Count+
   "\r\nPassos sugeridos NÃO foram executados e não constam como otimização aplicada.";
  Save(record);return record;
 }
}
```

`Kind="Cenário"` aparece na coluna “Tipo” do Histórico sem nenhuma outra
mudança (`Interface.cs:151` já monta a coluna a partir de `Record.Kind`).

---

## 3.4 `source/CatalogIds.cs` — validar cenário junto com o catálogo

```csharp
 internal static void Validate(){
  var items=NativeItems();var all=PerformanceLibrary.Load();
  if(items.Select(i=>i.Id).Distinct().Count()!=items.Count||all.Select(i=>i.Id).Distinct().Count()!=all.Count)
   throw new Exception("Catálogo contém IDs duplicados.");
  foreach(var entry in all.Where(e=>e.Action=="native"))
   if(!items.Any(i=>i.Id==Canonical(entry.Target)))throw new Exception("Ação da interface sem implementação: "+entry.Id);
  Scenarios.Validate();   // 6.2
 }
```

---

## 3.5 `source/OptionInfo.cs` — “em quais cenários esta opção entra?”

```csharp
class OptionInfo {
 public string Id,Classification,Risk,AutomaticReason,Restart,Undo,Compatibility,Evidence;
 public bool Automatic,Admin;
 public string[] InScenarios;      // 6.2

 static Dictionary<string,string[]> scenarioIndex;
 static Dictionary<string,string[]> ScenarioIndex(){
  if(scenarioIndex!=null)return scenarioIndex;
  var map=new Dictionary<string,List<string>>(StringComparer.Ordinal);
  foreach(var scenario in Scenarios.All())
   foreach(string id in scenario.Items){
    string key=CatalogIds.Canonical(id);
    if(!map.ContainsKey(key))map[key]=new List<string>();
    map[key].Add(scenario.Title);
   }
  var result=new Dictionary<string,string[]>(StringComparer.Ordinal);
  foreach(var pair in map)result[pair.Key]=pair.Value.ToArray();
  scenarioIndex=result;return scenarioIndex;
 }
```

e dentro de `For(Item item)`:

```csharp
  string[] scenarios;
  if(!ScenarioIndex().TryGetValue(item.Id,out scenarios))scenarios=new string[0];
  bool auto=scenarios.Length>0&&ExtremeProfile.IsReviewed(item);
  // … monte o OptionInfo com InScenarios=scenarios, Automatic=auto, Evidence=Evidence(item) …
  AutomaticReason=policy
   ?"Política protegida: somente escolha manual, fora dos cenários automáticos. Permissões verificadas antes das alterações."
   :auto?"Incluída em: "+String.Join(", ",scenarios)+". Depende da compatibilidade e da leitura deste PC."
   :risk?"Reduz proteção ou afeta funções do sistema. Exige escolha individual."
   :test?"Depende de teste A/B. Pode piorar o desempenho."
   :"Preferência pessoal, recurso opcional ou reparo. Escolha individual necessária.",
```

Com isso a ficha da opção e o botão do cenário param de discordar (**F3**).

---

## 3.6 `source/UnifiedInterface.cs` — aplicar um cenário

```csharp
 async Task RunScenario(string scenarioId){
  if(busy||scanning||resourceScanning||easyRunning||!machine.Windows)return;
  var plan=Scenarios.Build(scenarioId,machine);
  if(plan.Blocked!=null){footer.Text=plan.Blocked;return;}
  if(!await InlineReview(plan.Profile.Title,ScenarioPreview(plan),true))return;

  var appliedIds=new List<string>();
  if(plan.Apply.Count>0){
   string payload=String.Join(",",plan.Apply.Select(i=>i.Id));
   if(await Elevated("--extreme",payload))appliedIds.AddRange(plan.Apply.Select(i=>i.Id));
  }
  foreach(var item in plan.Confirm.ToArray()){
   var info=OptionInfo.For(item);
   string text=item.ReviewTitle+"\r\n\r\n"+String.Join("\r\n",item.Review??new string[0])+
    "\r\n\r\nClassificação: "+info.Classification+" · Risco: "+info.Risk+" · Evidência: "+info.Evidence+
    "\r\nVale para todos os usuários deste PC."+
    "\r\n\r\nAplicar esta opção agora?";
   if(!await InlineReview("Confirmação individual",text,true))continue;
   if(await Elevated("--apply",item.Id))appliedIds.Add(item.Id);
  }
  await Task.Run(new Action(delegate(){App.RecordScenarioOutcome(plan,appliedIds.ToArray());}));
  RefreshHistory();ShowPage("Resultados");
  ShowScenarioSummary(plan,appliedIds.ToArray());
 }

 string ScenarioPreview(ScenarioPlan plan){
  var text=new System.Text.StringBuilder();
  text.AppendLine(plan.Profile.Effect).AppendLine();
  if(plan.Apply.Count>0){
   text.AppendLine("SERÁ APLICADO AGORA ("+plan.Apply.Count+")");
   foreach(var group in plan.Apply.GroupBy(new Func<Item,string>(delegate(Item i){return AdvancedCategory(i);}))){
    text.AppendLine("  "+group.Key);
    foreach(var item in group)text.AppendLine("    • "+item.ReviewTitle);
   }
   text.AppendLine();
  }
  if(plan.Confirm.Count>0){
   text.AppendLine("PEDE CONFIRMAÇÃO INDIVIDUAL ("+plan.Confirm.Count+") — política ou alteração para todos os usuários");
   foreach(var item in plan.Confirm)text.AppendLine("  • "+item.ReviewTitle);
   text.AppendLine();
  }
  if(plan.Guide.Count>0){
   text.AppendLine("SUGERIDO, NÃO EXECUTADO ("+plan.Guide.Count+")");
   foreach(var step in plan.Guide)text.AppendLine("  • "+step.Name);
   text.AppendLine();
  }
  if(plan.Skip.Count>0){
   text.AppendLine("IGNORADO NESTE PC ("+plan.Skip.Count+")");
   foreach(var step in plan.Skip)text.AppendLine("  • "+step.Name+" — "+step.Message);
   text.AppendLine();
  }
  text.AppendLine("EFEITOS E LIMITES").AppendLine(plan.Profile.Tradeoff);
  text.AppendLine().AppendLine("Cada item tem backup próprio e desfazer individual no Histórico.");
  return text.ToString();
 }
```

E os botões — substituindo o `home-extreme` atual:

```csharp
  foreach(var scenario in Scenarios.All()){
   var current=scenario;   // captura por valor: C# 5 fecha sobre a variável do foreach
   actions.Controls.Add(HomeButton("home-scenario-"+current.Id,current.Title,
    async(s,e)=>await RunScenario(current.Id),
    current.Id==Scenarios.Desktop?200:186,current.Id==Scenarios.Desktop));
  }
```

> **Armadilha de C# 5 aqui.** Em C# 5 a variável de `foreach` **já é** por
> iteração (mudou no C# 5 justamente para isso), então `current` é redundante
> mas inofensivo. Se este código for retroportado para um `for` clássico, a
> cópia local passa a ser obrigatória.

### E `HomePayload(true)` deixa de existir

```csharp
 internal string[] HomePayload(){                       // sem o parâmetro "extreme"
  var selected=homeChecks.Where(x=>x.Value.Checked&&!homeOptions[x.Key].Diagnostic).Select(x=>x.Key);
  return ResolveHomeConflicts(selected);
 }
```

O caminho “aplicar tudo que passa em `IsReviewed`” some. Quem quiser o conjunto
amplo escolhe o cenário Desktop, que diz exatamente o que vai fazer.

---

## 3.7 Tabela de referência dos cenários

| Cenário | Automático | Confirmação individual | Sugerido | Exclui de propósito |
|---|---|---|---|---|
| Sessão de jogo rápida | 4 | 0 | 2 | tudo que exige reiniciar |
| Desktop gamer silencioso | 13 | 6 | 2 | serviços, Widgets por política |
| Notebook na bateria | 10 | 2 | 3 | **`native-power`** |
| Privacidade moderada | 6 | 7 | 0 | telemetria nível 0 (não honrada em Home/Pro) |
| Limpezas e diagnósticos | 0 | 3 | 9 | qualquer tweak |

“Automático” = passa em `IsReviewed`, tem backup, desfaz pelo Histórico.
Nenhum cenário aplica política sem o usuário clicar em cada uma.

## 3.8 Por que o notebook não recebe plano de energia

Vale deixar escrito no produto, porque é a diferença entre um otimizador e um
vendedor de mágica:

> Em notebook, selecionar “Alto desempenho” tipicamente troca autonomia e
> ruído por um ganho que o aplicativo **não mede**. O próprio Windows já
> gerencia frequência por demanda e o firmware do fabricante impõe limites
> térmicos. Por isso o cenário Notebook não inclui o plano — e diz isso na
> tela, em vez de simplesmente omitir.
