# 6 · Módulos novos e ampliados

---

## 6.1 Startup Manager que realmente encontra os apps

### O que existe hoje

`App.ScanStartup()` (`Management.cs:73`) lê **um** local:
`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`. E `App.StartupAllowed`
(`:78`) só marca como alterável o que estiver em `ResourceMonitor.Closable` ou
for `OneDrive`, `steam`, `EpicGamesLauncher`.

Consequência: Discord, Teams, Spotify, GOG, Battle.net, Razer Synapse — nada
disso aparece como opcional. É exatamente a lacuna do pedido.

### `source/Management.cs` — `StartupEntry` com origem

```csharp
public class StartupEntry {
 public string Name {get;set;} public string Command {get;set;} public string Kind {get;set;}
 public string AppName {get;set;} public string Reason {get;set;} public bool Allowed {get;set;}
 // 6.2
 public string Origin {get;set;}      // rótulo legível da origem
 public string Hive {get;set;}        // "" quando a origem é uma pasta (somente leitura)
 public string Key {get;set;}
 public string Impact {get;set;}      // "alto" | "médio" | "baixo" | "desconhecido"
 public override string ToString(){return Name;}
}
```

### Leitura ampla

```csharp
 // Quatro chaves Run + duas pastas Inicializar. Somente as chaves Run são alteráveis
 // pela engine; as pastas são listadas para consulta e abrem no Explorer.
 static readonly string[][] StartupKeys={
  new[]{"HKEY_CURRENT_USER",@"Software\Microsoft\Windows\CurrentVersion\Run","Este usuário"},
  new[]{"HKEY_CURRENT_USER",@"Software\Microsoft\Windows\CurrentVersion\RunOnce","Este usuário (uma vez)"},
  new[]{"HKEY_LOCAL_MACHINE",@"Software\Microsoft\Windows\CurrentVersion\Run","Todos os usuários"},
  new[]{"HKEY_LOCAL_MACHINE",@"Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Run","Todos os usuários (32 bits)"}
 };

 internal static List<StartupEntry> ScanStartupAll(){
  var rows=new List<StartupEntry>();
  if(!IsWindows)return rows;
  foreach(var location in StartupKeys){
   try{
    using(var hive=Hive(location[0]))using(var key=hive.OpenSubKey(location[1],false)){
     if(key==null)continue;
     foreach(string name in key.GetValueNames()){
      var kind=key.GetValueKind(name);
      string command=Convert.ToString(key.GetValue(name,"",Microsoft.Win32.RegistryValueOptions.DoNotExpandEnvironmentNames));
      bool textual=kind==Microsoft.Win32.RegistryValueKind.String||kind==Microsoft.Win32.RegistryValueKind.ExpandString;
      string exe=StartupExecutable(command);
      bool protectedEntry=exe.Length==0||ResourceMonitor.Within(exe,Environment.GetFolderPath(Environment.SpecialFolder.Windows));
      rows.Add(new StartupEntry{
       Name=name,Command=command,Kind=kind.ToString(),Origin=location[2],
       Hive=location[0],Key=location[1],
       AppName=System.IO.Path.GetFileNameWithoutExtension(exe),
       Impact=StartupImpact(exe),
       Allowed=textual&&!protectedEntry,
       Reason=!textual?"Tipo de valor não textual; o aplicativo não altera esta entrada."
        :protectedEntry?"Componente do Windows ou comando não classificado. Use a Inicialização do Windows para revisar."
        :"Opcional: não abrirá no próximo login. Backup exato do valor; sincronização e avisos do app podem parar até você abri-lo."
      });
     }
    }
   }catch(Exception e){
    rows.Add(new StartupEntry{Name=location[2],Command="",Kind="",Origin=location[2],Hive="",Key="",
     Allowed=false,Impact="desconhecido",Reason="Leitura indisponível: "+e.Message});
   }
  }
  foreach(var folder in new[]{
   new[]{Environment.GetFolderPath(Environment.SpecialFolder.Startup),"Pasta Inicializar (usuário)"},
   new[]{Environment.GetFolderPath(Environment.SpecialFolder.CommonStartup),"Pasta Inicializar (todos)"}}){
   try{
    if(String.IsNullOrEmpty(folder[0])||!System.IO.Directory.Exists(folder[0]))continue;
    foreach(string path in System.IO.Directory.GetFiles(folder[0])){
     string name=System.IO.Path.GetFileName(path);
     if(String.Equals(name,"desktop.ini",StringComparison.OrdinalIgnoreCase))continue;
     rows.Add(new StartupEntry{Name=name,Command=path,Kind="Arquivo",Origin=folder[1],Hive="",Key="",
      AppName=System.IO.Path.GetFileNameWithoutExtension(name),Impact="desconhecido",Allowed=false,
      Reason="Atalho em pasta. Esta versão não move arquivos: abra a pasta e remova o atalho manualmente, ou use a Inicialização do Windows."});
    }
   }catch(Exception e){
    rows.Add(new StartupEntry{Name=folder[1],Command="",Kind="",Origin=folder[1],Hive="",Key="",
     Allowed=false,Impact="desconhecido",Reason="Leitura indisponível: "+e.Message});
   }
  }
  return rows.OrderByDescending(x=>x.Allowed).ThenBy(x=>x.Origin).ThenBy(x=>x.Name).ToList();
 }
```

> **Por que as pastas não são alteráveis.** A engine transacional guarda
> `SavedValue` de Registro e `NativeSaved` de API — não há caminho testado para
> desfazer movimentação de arquivo. Mover um atalho sem undo confiável violaria
> a regra nº 6 do projeto. Listar e abrir a pasta é o comportamento honesto
> enquanto não existir um `FileSaved` com o mesmo rigor.

### Estimativa de impacto sem inventar número

O Gerenciador de Tarefas mostra “Impacto de inicialização” a partir de dados
que o Windows não expõe publicamente. Em vez de fingir o mesmo, classifique
pelo que dá para medir: tamanho do executável e assinatura do fornecedor.
E diga que é uma estimativa.

```csharp
 internal static string StartupImpact(string exe){
  try{
   if(exe.Length==0||!System.IO.File.Exists(exe))return "desconhecido";
   long bytes=new System.IO.FileInfo(exe).Length;
   if(bytes>=60L*1024*1024)return "alto";
   if(bytes>=8L*1024*1024)return "médio";
   return "baixo";
  }catch{return "desconhecido";}
 }
```

Texto que **precisa** acompanhar a coluna na UI:

> Estimativa pelo tamanho do executável, não pelo tempo real de carga.
> O Windows não publica a medida do Gerenciador de Tarefas. Para o número
> oficial, abra a guia Inicializar do Gerenciador de Tarefas.

### Desativar em qualquer chave Run

```csharp
 internal static void DisableStartup(string requestId){
  var req=ReadRequest(requestId);
  if(req.Startup==null||req.Startup.Count==0||req.Startup.Count>50)throw new Exception("Seleção de inicialização inválida.");
  var allowedKeys=new HashSet<string>(StartupKeys.Select(k=>k[0]+"|"+k[1]),StringComparer.OrdinalIgnoreCase);
  var ops=new List<RegOp>();
  foreach(var e in req.Startup){
   if(String.IsNullOrWhiteSpace(e.Name)||e.Name.Length>16383)throw new Exception("Entrada de inicialização não autorizada.");
   if(String.IsNullOrEmpty(e.Hive)||!allowedKeys.Contains(e.Hive+"|"+e.Key))
    throw new Exception("Origem de inicialização não autorizada: "+e.Origin);
   string exe=StartupExecutable(e.Command);
   if(exe.Length==0||ResourceMonitor.Within(exe,Environment.GetFolderPath(Environment.SpecialFolder.Windows)))
    throw new Exception("Entrada protegida ou não classificada: "+e.Name);
   var op=new RegOp{Hive=e.Hive,Key=e.Key,Name=e.Name,Kind=e.Kind,Data=e.Command,Delete=true};
   var current=Capture(op).Before;
   if(current.Delete||current.Kind!=e.Kind||current.Data!=e.Command)
    throw new Exception("A inicialização mudou desde a análise: "+e.Name+". Analise novamente.");
   ops.Add(op);
  }
  var item=new Item{Id="startup-v4",Name="Desativar inicialização selecionada",
   ReviewTitle="Inicialização de apps selecionados",ActionCode="startup",Native=false,
   Operations=ops.ToArray(),Text="",Path="Inicialização do usuário e do computador",ReviewLevel="Revisado"};
  ApplyTransaction(new[]{item},"Integrado","Desativar inicialização de "+ops.Count+" app(s)");
 }
```

Mudanças em relação ao original: a allowlist deixa de ser “nome do executável
está numa lista curta” e passa a ser **“a chave é uma das quatro Run
conhecidas, e o executável não está dentro de `%WINDIR%`”**. Isso libera
Discord/Teams/launchers e continua bloqueando componentes do Windows.

`App.RequiresAdmin` já devolve `true` para `HKEY_LOCAL_MACHINE`
(`LukeOptimizer.cs:260`), então a elevação para entradas de “Todos os usuários”
vem de graça — confira que `ActionNeedsAdmin("--startup",id)` consulta o pedido
gravado e não um valor fixo.

### UI (`source/EasyInterface.cs`, seção “Inicialização de apps”)

Quatro colunas em vez de três, e um agrupamento por origem:

```csharp
  startupList=CheckList("Entrada","Origem","Impacto estimado","Comando (somente leitura)");
  startupList.Columns[3].Width=460;
```

e ao preencher:

```csharp
  foreach(var entry in App.ScanStartupAll()){
   var row=new ListViewItem(entry.Name){Tag=entry,
    ForeColor=entry.Allowed?Theme.Text:Theme.Muted,ToolTipText=entry.Reason};
   row.SubItems.Add(entry.Origin);
   row.SubItems.Add(entry.Impact);
   row.SubItems.Add(entry.Command);
   startupList.Items.Add(row);
  }
```

O `ItemCheck` que já existe (`EasyInterface.cs:165`) continua barrando o que
tem `Allowed=false` — não precisa mudar.

---

## 6.2 Visualizador de tarefas agendadas

### `source/ScheduledTasks.cs` (novo; adicione ao `compilacao.rsp`)

Mesmo padrão de `App.OptionalServices`: allowlist fechada, estado lido antes,
Record com o valor anterior, desfazer que confere.

```csharp
using System;
using System.Linq;
using System.Text;
using System.Collections.Generic;
using System.Threading;

public class TaskState {public string Path {get;set;} public string Status {get;set;} public string Detail {get;set;}}
public class TaskSaved {public string Path {get;set;} public string Before {get;set;}}

// E em source/LukeOptimizer.cs, na classe Record, ao lado de ServiceValues:
//   public List<TaskSaved> TaskValues {get;set;}
// Sem isso o desfazer pelo Histórico não encontra o estado anterior — foi por
// esse campo que os serviços opcionais conseguiram ter undo confiável.

static class ScheduledTasks {
 // Nada de Defender, Windows Update, Ponto de Restauração, Chkdsk, Defrag ou anticheat aqui.
 // Cada entrada é uma tarefa de coleta/telemetria ou de conveniência, documentada.
 internal static readonly Dictionary<string,string> Optional=new Dictionary<string,string>(StringComparer.OrdinalIgnoreCase){
  {@"\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser",
   "Programa de Aperfeiçoamento da Experiência: coleta dados de compatibilidade de aplicativos. Desativar reduz coleta e uso de disco periódico; pode afetar avisos de compatibilidade em atualizações de versão do Windows."},
  {@"\Microsoft\Windows\Application Experience\ProgramDataUpdater",
   "Atualiza o banco de compatibilidade usado pelo Appraiser. Mesma natureza; sem efeito em FPS."},
  {@"\Microsoft\Windows\Customer Experience Improvement Program\Consolidator",
   "Envia os dados do Programa de Aperfeiçoamento da Experiência do Cliente. Desativar reduz envio; não desativa telemetria obrigatória de segurança."},
  {@"\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip",
   "Coleta dados de uso de USB para o mesmo programa. Desativar não afeta o funcionamento de dispositivos USB."},
  {@"\Microsoft\Windows\Feedback\Siuf\DmClient",
   "Agenda solicitações de feedback do Windows. Desativar remove pedidos de avaliação; não afeta atualizações."},
  {@"\Microsoft\Windows\Feedback\Siuf\DmClientOnScenarioDownload",
   "Variante da tarefa de feedback acionada por cenário. Mesma natureza."},
  {@"\Microsoft\Windows\Maps\MapsToastTask",
   "Notificações do aplicativo Mapas. Desativar silencia avisos de mapas offline."},
  {@"\Microsoft\Windows\Maps\MapsUpdateTask",
   "Baixa atualizações de mapas offline em segundo plano. Desativar economiza rede e disco; mapas offline deixam de atualizar."}
 };

 internal static string Validate(string path){
  if(!Optional.ContainsKey(path))throw new ArgumentException("Tarefa fora da lista opcional. Nenhuma alteração.");
  if(path.IndexOfAny(new[]{'"','\r','\n','&','|','<','>','^'})>=0)throw new ArgumentException("Caminho de tarefa inválido.");
  return Optional.Keys.Single(k=>String.Equals(k,path,StringComparison.OrdinalIgnoreCase));  // devolve a grafia canônica
 }

 // schtasks.exe é resolvido por App.SystemExe e executado por ManagedCommand:
 // sem shell, dentro de job object, com timeout. O único trecho variável é o
 // caminho da tarefa, que passou pela allowlist acima e por App.Quote.
 internal static TaskState Read(string path){
  string canonical=Validate(path);
  var result=ManagedCommand.Run(App.SystemExe("schtasks.exe"),
   "/query /tn "+App.Quote(canonical)+" /fo LIST",30,CancellationToken.None);
  if(result.ExitCode!=0)
   return new TaskState{Path=canonical,Status="ausente",
    Detail="Tarefa não encontrada nesta edição/versão do Windows. Nenhuma entrada será criada."};
  string status="desconhecido";
  foreach(string line in result.Output.Split(new[]{'\r','\n'},StringSplitOptions.RemoveEmptyEntries)){
   int colon=line.IndexOf(':');
   if(colon<0)continue;
   string label=line.Substring(0,colon).Trim();
   if(label=="Status"||label=="Estado"){status=line.Substring(colon+1).Trim();break;}
  }
  return new TaskState{Path=canonical,Status=status,Detail=Optional[canonical]};
 }

 internal static void ChangeTask(string canonical,bool enable){
  var result=ManagedCommand.Run(App.SystemExe("schtasks.exe"),
   "/change /tn "+App.Quote(canonical)+(enable?" /enable":" /disable"),60,CancellationToken.None);
  if(result.ExitCode!=0)
   throw new Exception("O Windows recusou a alteração da tarefa "+canonical+": "+result.Output);
 }

 internal static string Report(){
  var report=new StringBuilder();
  foreach(string path in Optional.Keys){
   try{var state=Read(path);report.AppendLine(path+" — "+state.Status);}
   catch(Exception e){report.AppendLine(path+" — leitura indisponível: "+e.Message);}
  }
  report.AppendLine();
  report.AppendLine("Somente leitura. Desativar uma tarefa de coleta reduz envio de dados; NÃO produz FPS.");
  return report.ToString();
 }
}

static partial class App {
 internal static void DisableOptionalTasks(string[] paths){
  if(paths.Length==0||paths.Length>ScheduledTasks.Optional.Count||
     paths.Distinct(StringComparer.OrdinalIgnoreCase).Count()!=paths.Length)
   throw new Exception("Seleção de tarefas inválida.");
  var canonical=paths.Select(ScheduledTasks.Validate).ToArray();   // allowlist inteira antes de qualquer mudança

  var record=NewRecord("Desativar tarefas agendadas opcionais","tasks-v1","Tarefas");
  record.Steps=new List<StepResult>();
  var saved=new List<TaskSaved>();
  Save(record);
  try{
   foreach(string path in canonical){
    var state=ScheduledTasks.Read(path);
    if(state.Status=="ausente"){
     record.Steps.Add(new StepResult{Id=path,Name=path,Status="ignorado",Message=state.Detail});continue;}
    if(state.Status.IndexOf("Desabilit",StringComparison.OrdinalIgnoreCase)>=0||
       state.Status.IndexOf("Disabled",StringComparison.OrdinalIgnoreCase)>=0){
     record.Steps.Add(new StepResult{Id=path,Name=path,Status="já configurado",Message="Já estava desativada."});continue;}
    saved.Add(new TaskSaved{Path=path,Before=state.Status});
    record.Steps.Add(new StepResult{Id=path,Name=path,Status="preparando",Message=state.Detail});
   }
   Save(record);
   if(saved.Count==0){
    record.Status="sem alterações";
    record.Message="As tarefas selecionadas já estavam desativadas ou não existem nesta versão do Windows.";
    Save(record);return;
   }
   record.Status="aplicando";record.ChangesStarted=true;
   record.TaskValues=saved;                     // estado anterior viaja no Record, como ServiceValues
   Save(record);
   foreach(var task in saved){
    WriteProgress(40,"Desativando tarefa agendada",task.Path);
    ScheduledTasks.ChangeTask(task.Path,false);
    var after=ScheduledTasks.Read(task.Path);
    if(after.Status.IndexOf("Desabilit",StringComparison.OrdinalIgnoreCase)<0&&
       after.Status.IndexOf("Disabled",StringComparison.OrdinalIgnoreCase)<0)
     throw new Exception("Estado da tarefa não conferido: "+task.Path);
    record.Steps.Single(s=>s.Id==task.Path).Status="conferido";
    Save(record);
   }
   record.Status="aplicado";
   record.Message=saved.Count+" tarefa(s) desativada(s) e conferida(s). Elas deixam de executar até você desfazer. "+
    "Nenhum ganho de FPS foi medido; o efeito é menos coleta e menos trabalho periódico em segundo plano.";
   Save(record);
  }catch(Exception e){
   if(record.ChangesStarted){
    try{RestoreOptionalTasks(record);record.Status="falhou e foi desfeito";}
    catch(Exception undo){record.Status="falha na restauração";e=new Exception(e.Message+" | "+undo.Message);}
   }else record.Status="falhou antes de alterar";
   record.Message=e.Message;Save(record);throw;
  }
 }

 // Assinatura igual à de RestoreOptionalServices: o Histórico chama com o Record.
 internal static void RestoreOptionalTasks(Record record){
  var saved=record.TaskValues??new List<TaskSaved>();
  foreach(var task in saved)ScheduledTasks.Validate(task.Path);      // allowlist de novo, sempre
  record.Status="restaurando";Save(record);
  foreach(var task in saved){
   ScheduledTasks.ChangeTask(task.Path,true);
   var after=ScheduledTasks.Read(task.Path);
   if(after.Status=="ausente")throw new Exception("Tarefa desapareceu durante a restauração: "+task.Path);
  }
  record.Status="desfeito";
  record.Message="Tarefas reativadas e conferidas. Agendamentos e gatilhos originais não foram alterados.";
  Save(record);
 }
}
```

> `ChangeTask` é `internal` de propósito: só `App.DisableOptionalTasks` e
> `App.RestoreOptionalTasks` devem chamá-la, sempre depois da allowlist.
> Toda entrada pública passa por `ScheduledTasks.Validate`.

### Roteamento

```csharp
// source/LukeOptimizer.cs
  else if(args[0]=="--tasks")DisableOptionalTasks(args[1].Split(new[]{'|'}));   // '|' porque o caminho contém '\'
```

Use `|` como separador: caminhos de tarefa contêm `\` e podem conter `,`.
E acrescente `--tasks` em `ActionNeedsAdmin` devolvendo `true` — `schtasks
/change` em `\Microsoft\Windows\...` exige elevação.

### UI

Uma sexta entrada no seletor de “Manutenção” (`EasyInterface.cs:159`):

```csharp
  selector.Items.AddRange(new object[]{"Inicialização de apps","Serviços opcionais",
   "Tarefas agendadas","Limpeza de temporários","Foco: jogo / CAD / render","Windows, rede e discos"});
```

com cartões idênticos aos de serviço (caixa + descrição + estado lido), e o
mesmo aviso de topo:

> Desativar uma tarefa de coleta reduz envio de dados e trabalho periódico.
> Não produz FPS. Tarefas de Defender, Windows Update, Ponto de Restauração,
> Chkdsk e Desfragmentação **não** estão nesta lista e não são tocadas.

---

## 6.3 Perfis customizados — promover o que já existe

`ProductPreferences.Profiles` (`ProductInterface.cs:13`) já é um
`Dictionary<string,string[]>` persistido, validado (≤ 50 perfis, ≤ 200 IDs cada)
e coberto por teste (`v6 profile and preferences round trip`). Os botões
`profile-save` / `profile-load` existem na página “Configurações”. Falta
interface e robustez.

```csharp
// source/ProductInterface.cs
 ComboBox profileList;TextBox profileName;

 void BuildCustomProfiles(TableLayoutPanel root,int row){
  var bar=ProductBar();root.Controls.Add(bar,0,row);
  profileName=new TextBox{Width=220,Font=Theme.Font(10),BackColor=Theme.Card,ForeColor=Theme.Text,
   AccessibleName="Nome do perfil"};
  bar.Controls.Add(profileName);
  profileList=Combo();profileList.Width=240;profileList.Dock=DockStyle.None;
  bar.Controls.Add(profileList);
  RefreshProfileList();

  ProductButton(bar,"profile-save","Salvar seleção atual",(s,e)=>{
   string name=profileName.Text.Trim();
   if(name.Length==0||name.Length>60){Notify("Nome inválido","Use de 1 a 60 caracteres.",true);return;}
   var ids=HomePayload();
   if(ids.Length==0){Notify("Nada selecionado","Marque ao menos uma opção na página Otimizações.",true);return;}
   productPrefs.Profiles[name]=ids;
   try{productPrefs.Save();RefreshProfileList();Notify("Perfil salvo",name+" · "+ids.Length+" opção(ões)");}
   catch(Exception error){Notify("Não foi possível salvar",error.Message,true);}
  },true,195);

  ProductButton(bar,"profile-load","Carregar perfil",(s,e)=>{
   string name=Convert.ToString(profileList.SelectedItem);
   string[] ids;
   if(name==null||!productPrefs.Profiles.TryGetValue(name,out ids))return;
   var known=new HashSet<string>(homeChecks.Keys,StringComparer.Ordinal);
   var missing=ids.Where(id=>!known.Contains(CatalogIds.Canonical(id))).ToArray();
   homeSync=true;
   foreach(var pair in homeChecks)pair.Value.Checked=ids.Contains(pair.Key)||ids.Contains(CatalogIds.Canonical(pair.Key));
   homeSync=false;UpdateHome();ShowPage("Otimizações");
   Notify("Perfil carregado",
    missing.Length==0?name+" · "+ids.Length+" opção(ões) marcadas"
     :name+" · "+missing.Length+" opção(ões) não existem mais neste catálogo e foram ignoradas",
    missing.Length>0);
  },false,175);

  ProductButton(bar,"profile-delete","Excluir perfil",async(s,e)=>{
   string name=Convert.ToString(profileList.SelectedItem);
   if(name==null)return;
   if(!await Confirm("Excluir o perfil \""+name+"\"?\r\n\r\nIsto apaga só a lista de opções salva. Nada aplicado no Windows é revertido.","Excluir perfil"))return;
   productPrefs.Profiles.Remove(name);
   try{productPrefs.Save();RefreshProfileList();Notify("Perfil excluído",name);}
   catch(Exception error){Notify("Não foi possível salvar",error.Message,true);}
  },false,155);

  ProductButton(bar,"profile-export","Exportar JSON",(s,e)=>{
   using(var dialog=new SaveFileDialog{Title="Exportar perfis",Filter="Perfis do Luke Optimizer|*.json",
    FileName="LukeOptimizer-perfis-"+DateTime.Now.ToString("yyyyMMdd")+".json"}){
    if(dialog.ShowDialog(this)!=DialogResult.OK)return;
    try{File.WriteAllText(dialog.FileName,App.Json.Serialize(productPrefs.Profiles),new UTF8Encoding(false));
     Notify("Perfis exportados",dialog.FileName);}
    catch(Exception error){Notify("Falha ao exportar",error.Message,true);}
   }
  },false,160);

  ProductButton(bar,"profile-import","Importar JSON",(s,e)=>{
   using(var dialog=new OpenFileDialog{Title="Importar perfis",Filter="Perfis do Luke Optimizer|*.json"}){
    if(dialog.ShowDialog(this)!=DialogResult.OK)return;
    try{
     var info=new FileInfo(dialog.FileName);
     if(info.Length>262144)throw new Exception("Arquivo muito grande para um conjunto de perfis.");
     var imported=App.Json.Deserialize<Dictionary<string,string[]>>(File.ReadAllText(dialog.FileName));
     if(imported==null)throw new Exception("Conteúdo inválido.");
     // Um perfil importado só pode citar IDs que existem — nunca chave de Registro nem comando.
     var known=new HashSet<string>(CatalogIds.NativeItems().Select(i=>i.Id),StringComparer.Ordinal);
     int accepted=0,rejected=0;
     foreach(var entry in imported){
      var valid=entry.Value==null?new string[0]
       :entry.Value.Select(CatalogIds.Canonical).Where(known.Contains).Distinct().ToArray();
      rejected+=(entry.Value==null?0:entry.Value.Length)-valid.Length;
      if(valid.Length==0)continue;
      productPrefs.Profiles[entry.Key]=valid;accepted++;
     }
     productPrefs.Save();RefreshProfileList();
     Notify("Perfis importados",accepted+" perfil(is) · "+rejected+" ID(s) desconhecido(s) descartado(s)",rejected>0);
    }catch(Exception error){Notify("Falha ao importar",error.Message,true);}
   }
  },false,160);
 }

 void RefreshProfileList(){
  profileList.Items.Clear();
  foreach(var name in productPrefs.Profiles.Keys.OrderBy(n=>n,StringComparer.CurrentCultureIgnoreCase))
   profileList.Items.Add(name);
  if(profileList.Items.Count>0)profileList.SelectedIndex=0;
 }
```

Os três pontos que fazem isso ser seguro:

1. **Importação filtra por `CatalogIds.NativeItems()`** — um JSON malicioso não
   consegue introduzir chave de Registro nem comando; no máximo marca opções
   que já existem.
2. **`productPrefs.Save()` valida** tamanho e contagem antes de gravar.
3. **Carregar perfil não aplica nada** — só marca as caixas. A aplicação
   continua passando por prévia, confirmação e engine.

---

## 6.4 Sistema & Info

`HardwareLab.Query` (`HardwareLab.cs:19`) já é um leitor WMI genérico com
timeout. Três leituras completam a tela.

```csharp
// source/HardwareLab.cs
 internal static string WindowsIdentity(){
  var report=new StringBuilder();
  try{
   foreach(var row in Query("SELECT Caption,Version,BuildNumber,OSArchitecture,InstallDate FROM Win32_OperatingSystem")){
    report.AppendLine("Edição: "+Get(row,"Caption"));
    report.AppendLine("Versão: "+Get(row,"Version")+" · build "+Get(row,"BuildNumber")+" · "+Get(row,"OSArchitecture"));
   }
  }catch(Exception e){report.AppendLine("Leitura do sistema indisponível: "+e.Message);}
  try{
   using(var hive=Microsoft.Win32.RegistryKey.OpenBaseKey(Microsoft.Win32.RegistryHive.LocalMachine,Microsoft.Win32.RegistryView.Registry64))
   using(var key=hive.OpenSubKey(@"SOFTWARE\Microsoft\Windows NT\CurrentVersion",false)){
    if(key!=null)report.AppendLine("Lançamento: "+Convert.ToString(key.GetValue("DisplayVersion"))+
     " · UBR "+Convert.ToString(key.GetValue("UBR")));
   }
  }catch(Exception e){report.AppendLine("DisplayVersion indisponível: "+e.Message);}
  return report.ToString();
 }

 // Nunca mostramos a chave completa. PartialProductKey são os 5 últimos caracteres,
 // que é o que o próprio Windows exibe.
 internal static string ActivationState(){
  try{
   var report=new StringBuilder();int rows=0;
   foreach(var row in Query("SELECT Description,LicenseStatus,PartialProductKey,Name FROM SoftwareLicensingProduct WHERE PartialProductKey IS NOT NULL")){
    rows++;
    long status=(long)Number(row,"LicenseStatus");
    report.AppendLine(Get(row,"Name"));
    report.AppendLine("  "+Get(row,"Description"));
    report.AppendLine("  Estado: "+LicenseText(status)+"  ·  chave parcial ..."+Get(row,"PartialProductKey"));
   }
   if(rows==0)return "Nenhum produto licenciado reportado pelo WMI.";
   report.AppendLine("Leitura informativa. O aplicativo não ativa, não altera e não remove licenças.");
   return report.ToString();
  }catch(Exception e){return "Estado de ativação indisponível: "+e.Message;}
 }
 static string LicenseText(long status){
  if(status==0)return "não licenciado";
  if(status==1)return "licenciado";
  if(status==2)return "período de carência inicial";
  if(status==3)return "carência adicional (KMS ou atualização)";
  if(status==4)return "carência por perda de token";
  if(status==5)return "notificação (não conforme)";
  if(status==6)return "carência estendida";
  return "estado "+status+" não reconhecido";
 }

 internal static string PowerPlans(){
  try{
   var report=new StringBuilder();
   foreach(var row in Query("SELECT ElementName,IsActive FROM Win32_PowerPlan",@"\\.\root\cimv2\power"))
    report.AppendLine((Convert.ToString(Get(row,"IsActive"))=="True"?"► ":"  ")+Get(row,"ElementName"));
   return report.Length==0?"Nenhum plano reportado.":report.ToString();
  }catch(Exception e){return "Planos de energia indisponíveis: "+e.Message;}
 }
```

Layout da página “Sistema & Info” (renomeie “Hardware e rede” ou crie uma nova
e mova a parte de rede), com o que já existe:

| Bloco | Origem |
|---|---|
| Windows: edição, versão, build, UBR, arquitetura | `WindowsIdentity()` |
| Ativação | `ActivationState()` |
| CPU, RAM, placa-mãe, BIOS | `HardwareLab` (já existe) |
| GPU e versão de driver | `SystemTelemetry.Gpu()` |
| Discos: modelo, tipo, espaço, aviso de falha | `diag54-storage` + `diag62-smart` |
| Planos de energia | `PowerPlans()` |
| Inicialização | `diag62-startup` |
| Reinício pendente | `TelemetrySnapshot.RestartPending` |

Botão “Exportar diagnóstico” já existe (`Interface.cs:146`, `ExportReport()`) —
inclua os blocos novos no HTML gerado, com o aviso de privacidade que já está
lá (“pode conter nomes de apps e caminhos locais”).

---

## 6.5 Resumo por arquivo

| Arquivo | Mudança |
|---|---|
| `Management.cs` | `StartupEntry` com `Origin`/`Hive`/`Key`/`Impact`; `ScanStartupAll`; `StartupImpact`; `DisableStartup` por allowlist de chave |
| `ScheduledTasks.cs` | **novo** — allowlist de 8 tarefas, `Read`/`ChangeTask`/`Report`, `App.DisableOptionalTasks`, `RestoreOptionalTasks` |
| `LukeOptimizer.cs` | campo `Record.TaskValues`; verbo `--tasks`; `ActionNeedsAdmin` para `--tasks` e `--startup` em HKLM; ramo de `Undo` chamando `RestoreOptionalTasks` |
| `ProductInterface.cs` | UI de perfis customizados com exportar/importar validado |
| `HardwareLab.cs` | `WindowsIdentity`, `ActivationState`, `PowerPlans` |
| `EasyInterface.cs` | colunas novas na inicialização; seção “Tarefas agendadas” |
| `compilacao.rsp` | `source\ScheduledTasks.cs` |
