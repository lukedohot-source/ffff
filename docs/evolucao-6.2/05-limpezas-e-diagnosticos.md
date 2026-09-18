# 5 · Limpezas e diagnósticos mais profundos, e ainda seguros

Regra que vale para todo este capítulo: **verificação não é reparo, e abrir uma
tela do Windows não é otimização aplicada.** O `Record.Status` reflete isso —
`"consulta concluída"`, `"leitura indisponível"`, `"sugerido"`, nunca `"aplicado"`.

---

## 5.1 Storage Sense — ler antes de configurar

As quatro políticas documentadas estão em `02-catalogo.md §2.7`. Falta a parte
que o pedido realmente pede: **mostrar o estado e explicar as categorias**.

### `source/ReviewedFeatures.cs` — diagnóstico novo `diag62-storagesense`

```csharp
static class ReviewedDiagnostics {
 internal static readonly string[] Ids={"diag54-tcp","diag54-storage","diag54-power","diag54-stability",
  "diag62-storagesense","diag62-smart","diag62-startup"};   // 6.2

 internal static string Title(string id){
  if(id=="diag54-tcp")return "Diagnosticar retransmissões TCP";
  if(id=="diag54-storage")return "Analisar espaço das unidades";
  if(id=="diag54-power")return "Diagnosticar energia";
  if(id=="diag54-stability")return "Verificar estabilidade recente";
  if(id=="diag62-storagesense")return "Conferir o Sensor de Armazenamento";
  if(id=="diag62-smart")return "Ler o aviso de falha dos discos";
  if(id=="diag62-startup")return "Listar o que abre com o Windows";
  throw new ArgumentException("Diagnóstico desconhecido.");
 }
 internal static string Category(string id){
  if(id=="diag54-tcp")return "Rede";
  if(id=="diag54-storage"||id=="diag62-storagesense"||id=="diag62-smart")return "SSD";
  if(id=="diag54-power")return "Energia";
  if(id=="diag62-startup")return "Inicialização";
  return "Diagnóstico";
 }
```

```csharp
 internal static string Detail(string id){
  // … os quatro casos existentes …
  if(id=="diag62-storagesense")
   return "Lê as políticas do Sensor de Armazenamento em HKLM (AllowStorageSenseGlobal, cadência, temporários, Lixeira) e informa quais estão definidas. Não liga, não desliga e não apaga arquivo nenhum. Se não houver política, o Sensor segue a preferência por usuário da tela de Armazenamento, que este diagnóstico não lê.";
  if(id=="diag62-smart")
   return "Consulta MSStorageDriver_FailurePredictStatus, o aviso de falha iminente que o driver expõe. É um sim/não grosseiro: não substitui a ferramenta do fabricante, não lê contadores SMART individuais, não mede desgaste de SSD e não estima vida restante. PredictFailure verdadeiro é motivo para backup imediato; falso não garante disco saudável.";
  if(id=="diag62-startup")
   return "Lista as entradas de inicialização visíveis ao aplicativo: HKCU\\...\\Run, HKLM\\...\\Run, a variante 32 bits e as pastas Inicializar do usuário e de todos os usuários. Somente leitura. Não cobre tarefas agendadas, serviços nem inicialização atrasada; use a página Inicialização e serviços para alterar.";
  throw new ArgumentException("Diagnóstico desconhecido.");
 }
```

```csharp
 static string StorageSenseRead(){
  var report=new StringBuilder();
  var fields=new[]{
   new[]{"AllowStorageSenseGlobal","Sensor de Armazenamento ligado por política"},
   new[]{"ConfigStorageSenseGlobalCadence","Cadência (0 = quando faltar espaço, 1, 7 ou 30 dias)"},
   new[]{"AllowStorageSenseTemporaryFilesCleanup","Limpar temporários de aplicativo"},
   new[]{"ConfigStorageSenseRecycleBinCleanupThreshold","Esvaziar Lixeira após N dias (0 = nunca)"},
   new[]{"ConfigStorageSenseDownloadsCleanupThreshold","Limpar Downloads após N dias (0 = nunca)"}
  };
  foreach(var field in fields){
   var op=new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SOFTWARE\Policies\Microsoft\Windows\StorageSense",
    Name=field[0],Kind="DWord"};
   try{var before=App.Capture(op).Before;
    report.AppendLine(field[1]+": "+(before.Delete?"não definida por política":before.Data));}
   catch(Exception e){report.AppendLine(field[1]+": leitura indisponível — "+e.Message);}
  }
  report.AppendLine();
  report.AppendLine("Nenhuma política definida significa que o Windows usa a preferência da tela Configurações > Sistema > Armazenamento, que este diagnóstico não lê.");
  return report.ToString();
 }

 static string SmartRead(){
  var report=new StringBuilder();int rows=0;
  try{
   using(var search=new System.Management.ManagementObjectSearcher(@"root\wmi",
    "SELECT InstanceName,PredictFailure,Reason FROM MSStorageDriver_FailurePredictStatus")){
    search.Options.Timeout=TimeSpan.FromSeconds(6);
    using(var results=search.Get())foreach(System.Management.ManagementObject row in results)using(row){
     rows++;
     bool predict=Convert.ToBoolean(row["PredictFailure"]);
     report.AppendLine(Convert.ToString(row["InstanceName"])+": "+
      (predict?"AVISO DE FALHA — faça backup agora e confira com a ferramenta do fabricante"
              :"sem aviso de falha")+
      " (código "+Convert.ToString(row["Reason"])+")");
    }
   }
  }catch(Exception e){return "Leitura indisponível: "+e.Message+
   "\r\nMuitos NVMe e controladoras RAID não expõem esta classe WMI; isso não indica problema.";}
  if(rows==0)return "Nenhuma unidade reportou o status pelo driver. Comum em NVMe e RAID; use a ferramenta do fabricante.";
  return report.ToString();
 }

 static string StartupRead(){
  var report=new StringBuilder();
  foreach(var entry in App.ScanStartupAll()){   // ver 06-modulos-novos.md §6.1
   report.AppendLine(entry.Origin+" · "+entry.Name+
    (entry.Allowed?"  [opcional]":"  [protegido/não classificado]"));
   report.AppendLine("    "+entry.Command);
  }
  if(report.Length==0)return "Nenhuma entrada de inicialização encontrada nos locais lidos.";
  report.AppendLine();
  report.AppendLine("Somente leitura. Para desativar com backup e desfazer, use Inicialização e serviços.");
  return report.ToString();
 }
```

E no `Read(string id)`, antes do bloco de estabilidade:

```csharp
  if(id=="diag62-storagesense")return StorageSenseRead();
  if(id=="diag62-smart")return SmartRead();
  if(id=="diag62-startup")return StartupRead();
```

`ReviewedFeatures.Library()` já projeta `ReviewedDiagnostics.Ids` inteiro em
`PerformanceEntry`, então os três novos aparecem no catálogo sem mais nada.
`UnifiedInterface.PrimaryOptions()` também os incorpora automaticamente
(`options.AddRange(ReviewedDiagnostics.Ids.Select(...))`).

### A ferramenta guiada de Storage Sense

`Scenarios.GuideDetail("guide-storagesense")` (`03 §3.2`) já entrega o texto.
Na UI, um botão que abre e explica — e **não** registra nada como aplicado:

```csharp
// source/ProductInterface.cs, na página Temporários
 ProductButton(bar,"cleanup-storagesense","Sensor de Armazenamento do Windows",(s,e)=>{
  OpenLocal("ms-settings:storagesense");
  Notify("Tela do Windows aberta",
   "O Sensor de Armazenamento decide o quê e quando limpar. O Luke Optimizer não apagou nada neste passo.");
 },false,250);
```

### O que cada categoria de “Arquivos temporários” do Windows significa

Texto pronto para a coluna “Impacto” ou para o painel lateral — vale colocar
literalmente na UI, porque é aqui que o usuário se perde:

| Categoria do Windows | O que é | Risco de perder algo |
|---|---|---|
| Arquivos temporários | `%TEMP%` de apps que não limparam sozinhos | baixo; app aberto pode recriar |
| Miniaturas | cache visual do Explorer | nenhum; recriado sob demanda, mais lento na primeira visita |
| Arquivos de otimização de entrega | pedaços de atualização já baixados | nenhum; pode rebaixar apenas se a atualização for repetida |
| Limpeza do Windows Update | versões antigas de componentes já instalados | **impede desinstalar atualizações antigas** |
| Instalações anteriores do Windows | `Windows.old` | **impede voltar à versão anterior**; some sozinho em 10 dias |
| Lixeira | arquivos que você apagou | **exclusão definitiva** |
| Downloads | sua pasta Downloads | **exclusão definitiva de arquivos seus** — nunca marque por padrão |
| Relatórios de erro | despejos de falha | perde material de diagnóstico de travamento |
| Cache de sombreador DirectX | shaders compilados | stutter e recompilação no primeiro jogo depois |

---

## 5.2 Pacotes de diagnóstico guiados (DISM + SFC)

### `source/CommandTools.cs` — comandos novos

```csharp
public class OfficialCommand {
 public string Id,Name,Category,Executable,Arguments,Description,Source;
 public int TimeoutSeconds;public bool Administrator;
 public string OutputFile;      // 6.2: nome do arquivo gerado dentro de ExecutionHub.LogsRoot
}
```

```csharp
  new OfficialCommand{Id="dism-componentstore",Name="DISM · analisar repositório de componentes",Category="Sistema",
   Executable="dism.exe",Arguments="/Online /Cleanup-Image /AnalyzeComponentStore",TimeoutSeconds=1200,Administrator=true,
   Description="Informa o tamanho do WinSxS e se a limpeza é recomendada. Somente análise: não executa StartComponentCleanup nem ResetBase, portanto não impede desinstalar atualizações.",
   Source="https://learn.microsoft.com/en-us/windows/win32/wua_sdk/using-the-component-store"},

  new OfficialCommand{Id="powercfg-battery",Name="Energia · relatório de bateria",Category="Energia",
   Executable="powercfg.exe",Arguments="/batteryreport",TimeoutSeconds=60,OutputFile="relatorio-bateria.html",
   Description="Gera o relatório HTML de capacidade projetada e ciclos da bateria. Só faz sentido em notebook. Somente leitura; não altera plano nem limites de carga.",
   Source="https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options"},

  new OfficialCommand{Id="powercfg-sleepstudy",Name="Energia · relatório de suspensão moderna",Category="Energia",
   Executable="powercfg.exe",Arguments="/sleepstudy",TimeoutSeconds=120,Administrator=true,OutputFile="relatorio-suspensao.html",
   Description="Gera o relatório de Modern Standby: quem acordou o PC e quanto consumiu dormindo. Requer que o PC suporte suspensão moderna; em S3 clássico o comando informa indisponível.",
   Source="https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/powercfg-command-line-options"},
```

Com `OutputFile`, `Run` passa a montar o argumento:

```csharp
 internal static ManagedCommandResult Run(string id,CancellationToken token){
  var command=Resolve(id);
  string arguments=command.Arguments;
  if(!String.IsNullOrEmpty(command.OutputFile)){
   if(command.OutputFile.IndexOfAny(System.IO.Path.GetInvalidFileNameChars())>=0)
    throw new ArgumentException("Nome de relatório inválido.");
   System.IO.Directory.CreateDirectory(ExecutionHub.LogsRoot);
   string target=System.IO.Path.GetFullPath(System.IO.Path.Combine(ExecutionHub.LogsRoot,command.OutputFile));
   if(!target.StartsWith(ExecutionHub.LogsRoot+System.IO.Path.DirectorySeparatorChar,StringComparison.OrdinalIgnoreCase))
    throw new ArgumentException("Relatório fora da pasta de logs.");
   arguments=arguments+" /output "+App.Quote(target);   // App.Quote recusa aspas e quebras de linha
  }
  return ManagedCommand.Run(App.SystemExe(command.Executable),arguments,command.TimeoutSeconds,token);
 }
```

`App.Quote` (`LukeOptimizer.cs:97`) lança se o caminho contiver `"`, `\r` ou
`\n`, então a injeção por nome de usuário esquisito fica barrada. O executável
continua fixo e resolvido por `App.SystemExe`; nada disso abre shell.

### `source/DiagnosticPackages.cs` (novo; adicione ao `compilacao.rsp`)

```csharp
using System;
using System.Linq;
using System.Text;
using System.Collections.Generic;
using System.Threading;

class DiagnosticPackage {
 internal string Id,Title,Purpose,Warning;internal string[] Commands,Readings;
 internal DiagnosticPackage(string id,string title,string purpose,string warning,string[] commands,string[] readings){
  Id=id;Title=title;Purpose=purpose;Warning=warning;Commands=commands;Readings=readings;}
}

static class DiagnosticPackages {
 internal static DiagnosticPackage[] All(){return new[]{
  new DiagnosticPackage("pack-reparos","Reparos de sistema · verificação",
   "Confere se a imagem do Windows e os arquivos protegidos estão íntegros.",
   "SÃO VERIFICAÇÕES, NÃO REPAROS. Nenhuma delas corrige nada sozinha. DISM ScanHealth e SFC podem levar de 5 a 30 minutos e usar CPU e disco de forma intensa — não rode durante uma partida ou uma renderização. Se algo for reportado como corrompido, o passo seguinte (DISM /RestoreHealth) é decisão sua e não é executado por este aplicativo.",
   new[]{"dism-check","dism-scan","sfc-verify"},new string[0]),

  new DiagnosticPackage("pack-espaco","Espaço e armazenamento",
   "Mostra quanto espaço há, quanto o repositório de componentes ocupa e se algum disco avisa falha.",
   "Somente leitura. A análise do repositório de componentes não remove nada. O aviso de falha do driver é grosseiro: use a ferramenta do fabricante para conferir desgaste de SSD.",
   new[]{"dism-componentstore"},new[]{"diag54-storage","diag62-smart","diag62-storagesense"}),

  new DiagnosticPackage("pack-energia","Energia e suspensão",
   "Identifica quem impede o PC de dormir e como a bateria está se comportando.",
   "Somente leitura. Os relatórios HTML podem conter nomes de aplicativos e dispositivos; revise antes de compartilhar. O relatório de suspensão moderna só existe em PCs compatíveis.",
   new[]{"energy-requests","powercfg-battery","powercfg-sleepstudy"},new[]{"diag54-power"}),

  new DiagnosticPackage("pack-rede","Rede e conectividade",
   "Consulta adaptadores, DNS, estado do TCP e uma amostra de retransmissões.",
   "Somente leitura; nenhuma configuração de rede é alterada. O cache DNS contém nomes que você visitou — revise o relatório antes de compartilhar.",
   new[]{"ip-config","tcp-state","dns-cache"},new[]{"diag54-tcp"}),

  new DiagnosticPackage("pack-estabilidade","Estabilidade e dispositivos",
   "Lê erros críticos recentes e dispositivos com problema no Plug and Play.",
   "Somente leitura. Um evento no log NÃO prova a causa de um travamento, e um dispositivo com código de problema pode estar apenas desconectado. Nenhum driver é instalado ou removido.",
   new[]{"driver-errors"},new[]{"diag54-stability"})
 };}

 internal static DiagnosticPackage Resolve(string id){
  var found=All().SingleOrDefault(p=>p.Id==id);
  if(found==null)throw new ArgumentException("Pacote de diagnóstico desconhecido.");
  return found;
 }
}

static partial class App {
 // Um Record só, com um StepResult por etapa. O log completo de cada comando
 // continua em ExecutionHub.LogsRoot, escrito por ManagedCommand.
 internal static Record RunDiagnosticPackage(string id,CancellationToken token){
  var package=DiagnosticPackages.Resolve(id);
  var record=NewRecord(package.Title,package.Id,"Diagnóstico");
  record.Steps=new List<StepResult>();
  record.Status="executando";Save(record);
  var summary=new StringBuilder();
  summary.AppendLine(package.Purpose).AppendLine().AppendLine(package.Warning).AppendLine();

  foreach(string readingId in package.Readings){
   if(token.IsCancellationRequested)break;
   var step=new StepResult{Id=readingId,Name=ReviewedDiagnostics.Title(readingId),Status="consulta concluída"};
   try{step.Message=ReviewedDiagnostics.Read(readingId);}
   catch(Exception e){step.Status="leitura indisponível";step.Message=ErrorHelp.For(e).FriendlyError+" "+e.Message;}
   record.Steps.Add(step);Save(record);
  }
  foreach(string commandId in package.Commands){
   if(token.IsCancellationRequested)break;
   var command=CommandTools.Resolve(commandId);
   var step=new StepResult{Id=commandId,Name=command.Name,Status="executando"};
   record.Steps.Add(step);Save(record);
   try{
    var result=CommandTools.Run(commandId,token);
    step.Status=result.Cancelled?"cancelado":result.TimedOut?"tempo esgotado"
     :result.ExitCode!=0?"leitura indisponível":"consulta concluída";
    step.Message=command.Description+
     "\r\nCódigo de saída: "+result.ExitCode+
     "\r\n"+result.Output+
     "\r\nLog completo: "+result.LogPath+
     "\r\nFonte: "+command.Source;
   }catch(Exception e){
    step.Status="leitura indisponível";
    step.Message=ErrorHelp.For(e).FriendlyError+" "+e.Message;
   }
   Save(record);
  }

  int ok=record.Steps.Count(s=>s.Status=="consulta concluída");
  record.Status=token.IsCancellationRequested?"cancelado"
   :ok==record.Steps.Count?"consulta concluída":"consulta parcial";
  summary.AppendLine("Etapas concluídas: "+ok+" de "+record.Steps.Count);
  summary.AppendLine("Nenhuma configuração foi alterada por este pacote.");
  summary.AppendLine("Relatórios e logs: "+ExecutionHub.LogsRoot);
  record.Message=summary.ToString();
  Save(record);return record;
 }
}
```

### Roteamento e UI

```csharp
// source/LukeOptimizer.cs, junto dos outros verbos em Main
  else if(args[0]=="--diagnostic-package")RunDiagnosticPackage(args[1],CancellationToken.None);
```

`App.ActionNeedsAdmin` precisa conhecer o verbo novo — DISM e SFC exigem
elevação, então devolva `true` quando **qualquer** comando do pacote tiver
`Administrator`:

```csharp
  if(action=="--diagnostic-package"){
   var package=DiagnosticPackages.Resolve(id);
   foreach(string commandId in package.Commands)
    if(CommandTools.Resolve(commandId).Administrator)return true;
   return false;
  }
```

```csharp
// source/ProductInterface.cs, página "Diagnósticos oficiais"
 foreach(var package in DiagnosticPackages.All()){
  var current=package;
  ProductButton(bar,"package-"+current.Id,current.Title,async(s,e)=>{
   if(!await Confirm(current.Purpose+"\r\n\r\n"+current.Warning+
    "\r\n\r\nEtapas: "+(current.Readings.Length+current.Commands.Length)+
    "\r\nNenhuma configuração será alterada. Executar agora?",current.Title))return;
   await Elevated("--diagnostic-package",current.Id);
  },false,250);
 }
```

---

## 5.3 Categorias de limpeza novas

Todas seguem o padrão existente de `CleanupCategory` e herdam automaticamente a
proteção de `TempCleaner.SafePath` (recusa junção/symlink e caminho fora da
raiz) e a revalidação de tamanho/data antes de excluir.

```csharp
// source/StorageManager.cs, dentro de Categories()
   new CleanupCategory{Id="nvidia-shaders",Name="Cache de shaders NVIDIA",
    Root=Path.Combine(local,@"NVIDIA\DXCache"),Pattern="*",
    Warning="Mesma natureza do cache DirectX: pode causar stutter e recompilação no primeiro jogo depois. Escolha individual; não é ganho garantido."},
   new CleanupCategory{Id="nvidia-glcache",Name="Cache OpenGL/Vulkan NVIDIA",
    Root=Path.Combine(local,@"NVIDIA\GLCache"),Pattern="*",
    Warning="Recompilação no primeiro uso de jogos OpenGL/Vulkan. Feche os jogos antes."},
   new CleanupCategory{Id="amd-dxcache",Name="Cache de shaders AMD",
    Root=Path.Combine(local,@"AMD\DxCache"),Pattern="*",
    Warning="Recompilação no primeiro jogo depois. Feche os jogos e o AMD Software antes."},
   new CleanupCategory{Id="brave-cache",Name="Cache Brave · perfil Default",
    Root=Path.Combine(local,@"BraveSoftware\Brave-Browser\User Data\Default\Cache"),Pattern="*",
    Warning="Feche o navegador. Cookies, senhas e histórico ficam fora do escopo."},
   new CleanupCategory{Id="firefox-cache",Name="Cache Firefox · perfil padrão",
    Root=FirefoxCacheRoot(),Pattern="*",
    Warning="Feche o Firefox. Somente o cache do perfil padrão; marcadores, senhas e sessões não fazem parte."},
   new CleanupCategory{Id="edge-code-cache",Name="Cache de código Edge",
    Root=Path.Combine(local,@"Microsoft\Edge\User Data\Default\Code Cache"),Pattern="*",
    Warning="JavaScript compilado; é recriado. Primeira abertura de sites pesados fica um pouco mais lenta."},
   new CleanupCategory{Id="installer-logs",Name="Logs de instalação do usuário",
    Root=Path.Combine(local,"Temp"),Pattern="*.log",
    Warning="Somente arquivos .log dentro do Temp do usuário. Se você está investigando uma instalação que falhou, exporte antes."},
```

`firefox-cache` precisa do perfil, que tem sufixo aleatório:

```csharp
 // Sem curinga na Root: resolvemos o perfil padrão uma vez e devolvemos caminho concreto.
 // Se houver ambiguidade, devolvemos vazio e a categoria é ignorada por "caminho ausente".
 static string FirefoxCacheRoot(){
  try{
   string profiles=Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
    @"Mozilla\Firefox\Profiles");
   if(!Directory.Exists(profiles))return "";
   var candidates=new DirectoryInfo(profiles).GetDirectories("*.default*");
   if(candidates.Length!=1)return "";
   return Path.Combine(candidates[0].FullName,"cache2");
  }catch{return "";}
 }
```

`Scan` já trata `Root` inexistente com `"caminho ausente; ignorado."`, mas
proteja contra `Root` vazio, que hoje viraria uma raiz perigosa:

```csharp
  foreach(var category in selection){
   if(String.IsNullOrEmpty(category.Root)||!Directory.Exists(category.Root)){
    preview.Notes.Add(category.Name+": caminho ausente; ignorado.");continue;}
```

### Estimativa e log por categoria

```csharp
// source/StorageManager.cs
 internal static string PerCategory(CleanupPreview preview){
  if(preview==null||preview.Files.Count==0)return "Nenhum arquivo elegível na prévia.";
  var names=new Dictionary<string,string>(StringComparer.Ordinal);
  foreach(var category in Categories())names[category.Id]=category.Name;
  var report=new System.Text.StringBuilder();
  foreach(var group in preview.Files.GroupBy(f=>f.CategoryId).OrderByDescending(g=>g.Sum(f=>f.Bytes))){
   string name;if(!names.TryGetValue(group.Key,out name))name=group.Key;
   report.AppendLine(name+": "+group.Count()+" arquivo(s) · "+SystemTelemetry.Format(group.Sum(f=>f.Bytes)));
  }
  report.AppendLine("Total estimado: "+SystemTelemetry.Format(preview.Files.Sum(f=>f.Bytes)));
  report.AppendLine("É o tamanho lógico. O espaço realmente liberado pode diferir por compressão, blocos e cópias do sistema de arquivos.");
  return report.ToString();
 }

 // Depois de Execute, o que foi REALMENTE apagado — separado da estimativa.
 internal static string Freed(ExecutionReport report,CleanupPreview preview){
  long bytes=0;int count=0;
  foreach(var action in report.Actions){
   if(action.State!="aplicada")continue;
   var file=preview.Files.FirstOrDefault(f=>f.Path==action.Id);
   if(file!=null){bytes+=file.Bytes;count++;}
  }
  return count+" arquivo(s) removido(s) permanentemente · "+SystemTelemetry.Format(bytes)+" lógicos. "+
   (report.Actions.Count-count)+" ignorado(s) por estarem em uso, terem mudado ou serem protegidos.";
 }
```

Ligue na UI e no `Record.Message`:

```csharp
// source/ProductInterface.cs, ao fim de ScanStorage()
 storageInfo.Text=StorageManager.PerCategory(cleanupPreview)+"\r\n"+
  (cleanupPreview.Truncated?"Análise limitada por tempo, quantidade ou profundidade.\r\n":"")+
  String.Join("; ",cleanupPreview.Notes);
// e ao fim de RunStorage(false)
 Notify("Limpeza concluída",StorageManager.Freed(lastProductReport,selected));
```

**Importante:** a limpeza permanente não tem desfazer, e isso não muda.
Continue exigindo prévia + `Confirm` modal (`§4.5`) e mantenha o texto
`"Não vão para a Lixeira e não possuem rollback"` que já existe.

---

## 5.4 Idade mínima: o problema silencioso de hoje

Com `cleanupAge.Value=0` (`ProductInterface.cs:97`), o corte em
`StorageManager.Scan` fica em `DateTime.UtcNow`, e
`file.LastWriteTimeUtc >= cutoff` só é verdadeiro para arquivos com data futura.
Resultado: **o botão “Limpar todos os temporários” não filtra por idade**.

Ele ainda é seguro pela allowlist de extensão, pela prévia e pela confirmação —
mas é uma decisão que o usuário nunca tomou e não pode ver. Correção:

```csharp
// source/ProductInterface.cs
 async Task CleanAllTemporariesOneClick(){
  if(productRunning||busy||!ready)return;
  foreach(ListViewItem row in storageCategories.Items)row.Checked=true;
  cleanupAge.Value=productPrefs.CleanupAgeDays;          // 7 por padrão, visível, editável
  footer.Text="Analisando temporários com "+productPrefs.CleanupAgeDays+" dia(s) de idade mínima…";
  await ScanStorage();
  if(cleanupPreview!=null&&cleanupPreview.Files.Count>0)await RunStorage(false);
  else Notify("Nada a limpar","Nenhum temporário elegível com a idade mínima atual.");
 }
```

E salvar quando o usuário mudar:

```csharp
 cleanupAge.ValueChanged+=(s,e)=>{
  productPrefs.CleanupAgeDays=Math.Max(1,(int)cleanupAge.Value);
  try{productPrefs.Save();}catch{}
 };
```

`ProductPreferences.Validate()` exige `CleanupAgeDays>=1`, daí o `Math.Max`.
Se quiser permitir `0` (“sem filtro”) na interface, relaxe a validação **e**
deixe o rótulo dizer explicitamente `0 = sem filtro de idade`.

---

## 5.5 Lixeira como módulo próprio

Categoria de limpeza não serve: os itens não são arquivos comuns e a exclusão
usa API própria. Duas chamadas documentadas do shell resolvem.

```csharp
// source/StorageManager.cs
 // SHQUERYRBINFO: DWORD cbSize; __int64 i64Size; __int64 i64NumItems.
 // Empacotamento natural (24 bytes em x64) — o projeto compila só com /platform:x64.
 [StructLayout(LayoutKind.Sequential)] struct RecycleInfo {
  public int Size;public long Bytes;public long Items;
 }
 [DllImport("shell32.dll",CharSet=CharSet.Unicode)] static extern int SHQueryRecycleBin(string root,ref RecycleInfo info);
 [DllImport("shell32.dll",CharSet=CharSet.Unicode)] static extern int SHEmptyRecycleBin(IntPtr owner,string root,uint flags);

 internal static string RecycleBinSummary(){
  if(!App.IsWindows)return "Disponível no Windows.";
  var info=new RecycleInfo();info.Size=Marshal.SizeOf(typeof(RecycleInfo));
  int result=SHQueryRecycleBin(null,ref info);
  if(result!=0)return "Consulta indisponível (HRESULT "+result+").";
  return info.Items+" item(ns) · "+SystemTelemetry.Format(info.Bytes)+
   "\r\nEsvaziar é DEFINITIVO: estes arquivos deixam de ser recuperáveis pelo Windows.";
 }

 internal static Record EmptyRecycleBin(){
  if(!App.IsWindows)throw new Exception("Requer Windows.");
  var record=App.NewRecord("Esvaziar a Lixeira","recycle-v1","Limpeza");
  record.Steps=new List<StepResult>();
  string before=RecycleBinSummary();
  record.Status="executando";App.Save(record);
  // 0x1 sem confirmação do shell (o app já confirmou), 0x2 sem barra de progresso, 0x4 sem som
  int hresult=SHEmptyRecycleBin(IntPtr.Zero,null,0x1|0x2|0x4);
  record.Steps.Add(new StepResult{Id="recycle",Name="Lixeira",
   Status=hresult==0?"removido":"ignorado",
   Message=hresult==0?"Antes: "+before:"O Windows recusou a operação (HRESULT "+hresult+")."});
  record.Status=hresult==0?"concluído":"falhou";
  record.Message=(hresult==0
   ?"Lixeira esvaziada. Exclusão definitiva, SEM DESFAZER.\r\n\r\nEstado anterior:\r\n"+before
   :"Nada foi removido. HRESULT "+hresult+". A Lixeira pode estar em uso ou protegida por política.");
  App.Save(record);return record;
 }
```

Na UI, sempre com o tamanho **antes** de perguntar:

```csharp
 ProductButton(actions,"cleanup-empty-recycle","Esvaziar a Lixeira",async(s,e)=>{
  string summary=StorageManager.RecycleBinSummary();
  if(!await Confirm(summary+"\r\n\r\nEsvaziar agora? Esta ação não pode ser desfeita.","Esvaziar a Lixeira"))return;
  var record=await Task.Run(new Func<Record>(StorageManager.EmptyRecycleBin));
  RefreshHistory();Notify("Lixeira",record.Status=="concluído"?"Esvaziada.":"Nada foi removido.",record.Status!="concluído");
 },false,185);
```

---

## 5.6 Cache do Delivery Optimization — pelo caminho documentado

Apagar `C:\Windows\SoftwareDistribution\DeliveryOptimization` na mão é ruim.
Existe cmdlet oficial, e o projeto já tem `AppRemoval.PowerShell`:

```csharp
// source/StorageManager.cs
 internal static Record ClearDeliveryOptimizationCache(){
  var record=App.NewRecord("Limpar cache do Delivery Optimization","do-cache-v1","Limpeza");
  record.Steps=new List<StepResult>();record.Status="executando";App.Save(record);
  var result=AppRemoval.PowerShell(
   "$b=(Get-DeliveryOptimizationStatus -ErrorAction SilentlyContinue|Measure-Object -Property FileSize -Sum).Sum;"+
   "Delete-DeliveryOptimizationCache -Force -ErrorAction Stop;"+
   "[Console]::Write('Cache antes: '+[string]$b+' bytes')");
  record.Steps.Add(new StepResult{Id="do-cache",Name="Cache do Delivery Optimization",
   Status=result.ExitCode==0?"removido":"ignorado",Message=result.Output});
  record.Status=result.ExitCode==0?"concluído":"falhou";
  record.Message=(result.ExitCode==0
   ?"Cache de atualização removido pelo cmdlet oficial. Sem desfazer: o conteúdo será baixado de novo se necessário. Não remove atualizações já instaladas."
   :"Nada foi removido. "+result.Output+
    "\r\nO cmdlet exige Windows 10 1709+ e administrador, e falha se o serviço DoSvc estiver parado.");
  App.Save(record);return record;
 }
```

Fonte: `https://learn.microsoft.com/en-us/powershell/module/deliveryoptimization/delete-deliveryoptimizationcache`.

Não confundir com a **categoria** do Sensor de Armazenamento: esta é a ação
explícita do usuário, registrada como `"concluído"`, não como “otimização”.

---

## 5.7 Resumo das mudanças por arquivo

| Arquivo | Mudança |
|---|---|
| `ReviewedFeatures.cs` | 3 diagnósticos novos (`diag62-storagesense`, `diag62-smart`, `diag62-startup`) |
| `CommandTools.cs` | campo `OutputFile`; 3 comandos novos; validação do caminho de relatório |
| `DiagnosticPackages.cs` | **novo** — 5 pacotes guiados + `App.RunDiagnosticPackage` |
| `LukeOptimizer.cs` | verbo `--diagnostic-package` e `ActionNeedsAdmin` correspondente |
| `StorageManager.cs` | 7 categorias novas; `FirefoxCacheRoot`; guarda de `Root` vazio; `PerCategory`; `Freed`; Lixeira; cache do DO |
| `ProductInterface.cs` | ligar `cleanupAge`; resumo por categoria; botões de Lixeira, DO e pacotes |
| `V5Options.cs` | 4 políticas do Sensor de Armazenamento (`02-catalogo.md §2.7`) |
| `compilacao.rsp` | `source\DiagnosticPackages.cs` |
