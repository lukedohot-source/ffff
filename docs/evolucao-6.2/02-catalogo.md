# 2 · Catálogo mais completo e mais honesto

Tudo aqui é colável. C# 5, sem interpolação, sem `?.`, sem `=>` de membro.

## 2.1 Primeiro: um campo novo que muda a qualidade de todo o resto — `SourceKind`

Hoje `Source` é uma URL e ponto. Mas há três naturezas muito diferentes de
ajuste misturadas no mesmo catálogo:

| Natureza | Exemplo atual | O que o usuário precisa saber |
|---|---|---|
| Política documentada | `DODownloadMode` | Microsoft garante o comportamento; pode exigir edição Pro+ |
| API documentada | `SPI_SETDRAGFULLWINDOWS` | contrato estável do Win32 |
| Preferência observada | `opt-disableshowmoreoptions` (CLSID `{86ca1aa0-…}`) | funciona hoje, pode sumir numa build |

Sem essa distinção, um CLSID não documentado e uma Policy CSP parecem
igualmente sólidos. Com ela, vira um badge (`04-ui-ux.md §4.3`).

### `source/LukeOptimizer.cs` — campo novo em `Item`

```csharp
public class Item {
 // … campos existentes …
 public string SourceKind {get;set;}   // "Documentado" | "API oficial" | "Observado"
}
```

### `source/PerformanceLibrary.cs` — campo novo em `PerformanceEntry`

```csharp
public class PerformanceEntry {
 // … campos existentes …
 public string SourceKind {get;set;}
}
```

Ambos são `string` e podem vir `null` — JSON antigo continua desserializando
sem alteração (`JavaScriptSerializer` ignora campo ausente). Onde vier vazio,
trate como `"Documentado"`:

```csharp
// source/OptionInfo.cs
internal static string Evidence(Item item){
 string kind=item==null?null:item.SourceKind;
 return String.IsNullOrEmpty(kind)?"Documentado":kind;
}
```

### `source/ImportedOptions.cs` — repassar o campo

```csharp
public class ImportedOption {
 public string Id,Title,Category,Effect,Tradeoff,Source,SourceKind;
 public int Minimum,Maximum;public RegOp[] Operations;
}
```

e dentro de `Items()` / `LibraryEntries()`, acrescente
`SourceKind=String.IsNullOrEmpty(r.SourceKind)?"Documentado":r.SourceKind,`.

**Dívida a pagar junto:** marcar `SourceKind:"Observado"` em
`optimizer-options.json` para pelo menos
`opt-disableshowmoreoptions`, `opt-enablelegacyvolumeslider`,
`opt-removecasttodevice`, `opt-disablestickers` e `pack-8098f03b75de`.

---

## 2.2 Windows leve / Personalização — 3 itens por API oficial

Estes são os melhores acréscimos do plano: usam a mesma
`SystemParametersInfoW` que o app já usa, portanto ganham backup/undo
**de graça**, ficam em `HKEY_CURRENT_USER` de forma implícita, não são
política — e por isso passam em `ExtremeProfile.IsReviewed` e podem entrar no
perfil automático.

### `source/Profile.cs` — 3 entradas em `NativeSettings.Flags`

```csharp
 static readonly Dictionary<string,uint[]> Flags=new Dictionary<string,uint[]> {
  {"client",new uint[]{0x1042,0x1043}}, {"cursorshadow",new uint[]{0x101A,0x101B}},
  {"dragoutline",new uint[]{0x0026,0x0025}}, {"menuanim",new uint[]{0x1002,0x1003}},
  {"combo",new uint[]{0x1004,0x1005}}, {"list",new uint[]{0x1006,0x1007}},
  {"tooltip",new uint[]{0x1016,0x1017}}, {"selection",new uint[]{0x1014,0x1015}},
  // 6.2 — SPI_GET/SET documentados em winuser.h; mesmo contrato BOOL dos acima.
  {"gradient",new uint[]{0x1008,0x1009}},     // SPI_GET/SETGRADIENTCAPTIONS
  {"dropshadow",new uint[]{0x1024,0x1025}},   // SPI_GET/SETDROPSHADOW
  {"uieffects",new uint[]{0x103E,0x103F}}     // SPI_GET/SETUIEFFECTS
 };
```

### `source/Profile.cs` — uma linha em `NativeSettings.Codes`

```csharp
 internal static string[] Codes(string action) {
  if(action=="dragoutline"||action=="cursorshadow")return new[]{action};
  if(action=="gradient"||action=="dropshadow"||action=="uieffects")return new[]{action};   // 6.2
  if(action==ExtraSettings.Compression)return new[]{action};
  // … resto inalterado …
 }
```

Por que nada mais muda: `Read` cai em `Flags[code][0]`, `Write` cai no `else`
final que já trata SET booleano (`Spi(Flags[code][1],0,BoolParameter(value),3)`),
e `Desired` devolve `"0"` por padrão. Exatamente o caminho de `cursorshadow`.

### `source/ReviewedFeatures.cs` — 3 itens novos

```csharp
 internal static List<Item> Items(){return new[]{
  Make("dragoutline","Arrastar janelas pelo contorno","Reduz o redesenho de conteúdo durante o arraste usando SPI_SETDRAGFULLWINDOWS. Pode ajudar em sessões remotas ou interfaces pesadas. O conteúdo reaparece ao soltar a janela; não mede FPS."),
  Make("cursorshadow","Desativar sombra do ponteiro","Desativa a sombra via SPI_SETCURSORSHADOW. É uma preferência visual: não muda polling, Raw Input ou a latência física. Pode reduzir a visibilidade do ponteiro."),
  // 6.2
  Make("gradient","Barra de título sem degradê","Desativa o degradê das barras de título via SPI_SETGRADIENTCAPTIONS. Menos pintura por janela em telas grandes. É aparência: não mede FPS de jogo e não afeta o compositor DWM."),
  Make("dropshadow","Desativar sombra das janelas","Desativa a sombra projetada das janelas via SPI_SETDROPSHADOW. Reduz uma camada de composição por janela. Bordas ficam menos separadas do fundo; apps com moldura própria podem ignorar a preferência."),
  Make("uieffects","Desligar efeitos de interface (amplo)","Desliga de uma vez o conjunto de efeitos de interface via SPI_SETUIEFFECTS: animações, fades, sombras e realce. É a chave-mestra dos itens individuais acima. Alguns efeitos ligados a acessibilidade também param; prefira os itens específicos se quiser controle fino. Não mede FPS.")
 }.ToList();}
```

`Make` já define `Category="-5 - Windows"`, `ReviewLevel="Revisado"`,
`Url=SpiSource` e `Operations=new RegOp[0]` — então `IsReviewed` passa e o
item fica elegível para perfil automático. Se quiser que `uieffects`
**não** entre em perfil automático (recomendado — é amplo demais), basta não
listá-lo em nenhum `ScenarioProfile` (`03-perfis-por-cenario.md`).

### Registrar os títulos bonitos na página “Otimizações”

```csharp
// source/UnifiedInterface.cs, dentro de PrimaryOptions(), no bloco "Windows"
new HomeOption("native-gradient","Barra de título sem degradê","Windows"),
new HomeOption("native-dropshadow","Desativar sombra das janelas","Windows"),
new HomeOption("native-uieffects","Desligar efeitos de interface (amplo)","Windows"),
```

E em `ReviewedFeatures.Library()` nada muda: ele já projeta `Items()` inteiro.

---

## 2.3 Rede e downloads — Delivery Optimization completo

Tudo abaixo vai em `source/V5Options.cs`, dentro de
`static readonly Rule[] Rules`. Todas as chaves ficam em
`SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization` — portanto são
**política**, `ReviewLevel` vira `"Manual"` automaticamente e **ficam fora do
botão automático**, como o pedido exige.

Primeiro, uma constante para evitar repetição (junto às outras do topo da classe):

```csharp
 const string DoKey=@"SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization";
 const string DoSource="https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference";
```

```csharp
  new Rule{Code="delivery-lan-only",Title="Atualizações: compartilhar só na rede local",Category="Rede e downloads",Minimum=19041,Edition=1,
   Effect="Define DODownloadMode=1: o PC troca partes de atualizações apenas com máquinas atrás do mesmo NAT, e continua baixando o restante por HTTP.",
   Tradeoff="Útil em casa ou escritório com vários PCs no mesmo roteador: reduz o tráfego externo total. Não acelera download individual e não muda a fila do Windows Update. Política para todos os usuários; pode ser sobreposta por política de trabalho/escola.",
   Source=DoSource+"#download-mode",SourceKind="Documentado",Operations=new[]{Device(DoKey,"DODownloadMode","1")}},

  new Rule{Code="delivery-simple",Title="Atualizações: modo simples, sem par e sem cache",Category="Rede e downloads",Minimum=19041,Edition=1,
   Effect="Define DODownloadMode=99: baixa direto da origem, sem troca entre PCs e sem usar Microsoft Connected Cache.",
   Tradeoff="O modo mais previsível para quem tem franquia de dados por PC. Cada máquina baixa tudo por conta própria, então a soma do tráfego da casa aumenta. Conflita com os outros modos de Delivery Optimization; escolha um.",
   Source=DoSource+"#download-mode",SourceKind="Documentado",Operations=new[]{Device(DoKey,"DODownloadMode","99")}},

  new Rule{Code="delivery-background-cap",Title="Atualizações: limitar banda de segundo plano a 20 %",Category="Rede e downloads",Minimum=19041,Edition=1,
   Effect="Define DOPercentageMaxBackgroundBandwidth=20: downloads em segundo plano do Delivery Optimization usam no máximo 20 % da banda medida.",
   Tradeoff="Ajuda quando uma atualização começa no meio da partida. Em troca, atualizações demoram mais e podem ficar pendentes por dias. Não limita jogos, navegador, Steam ou Xbox — só o Delivery Optimization. Para escolher outro percentual use Ajustes manuais.",
   Source=DoSource+"#percentagemaxbackgrounddownloadbandwidth",SourceKind="Documentado",Operations=new[]{Device(DoKey,"DOPercentageMaxBackgroundBandwidth","20")}},

  new Rule{Code="delivery-foreground-cap",Title="Atualizações: limitar banda de primeiro plano a 60 %",Category="Rede e downloads",Minimum=19041,Edition=1,
   Effect="Define DOPercentageMaxForegroundBandwidth=60: downloads que o usuário está esperando usam no máximo 60 % da banda medida.",
   Tradeoff="Mantém folga para jogo e chamada de voz durante uma instalação que você mesmo pediu. Se você quer a atualização o mais rápido possível, não use. Aplica-se apenas ao Delivery Optimization.",
   Source=DoSource+"#percentagemaxforegrounddownloadbandwidth",SourceKind="Documentado",Operations=new[]{Device(DoKey,"DOPercentageMaxForegroundBandwidth","60")}},
```

**Conflito a declarar.** `delivery-no-peers` (já existe, `DODownloadMode=0`),
`delivery-lan-only` (1) e `delivery-simple` (99) escrevem o mesmo valor.
`App.UniqueOperations` já rejeita a seleção contraditória em tempo de
`Resolve`, mas o usuário merece ver isso antes. Declare no `Items()`:

```csharp
// source/V5Options.cs, dentro do Select de Items(), substituindo a linha de Conflicts
Conflicts=r.Code=="theme-dark"?new[]{Prefix+"theme-light"}
  :r.Code=="theme-light"?new[]{Prefix+"theme-dark"}
  :r.Code=="delivery-no-peers"?new[]{Prefix+"delivery-lan-only",Prefix+"delivery-simple"}
  :r.Code=="delivery-lan-only"?new[]{Prefix+"delivery-no-peers",Prefix+"delivery-simple"}
  :r.Code=="delivery-simple"?new[]{Prefix+"delivery-no-peers",Prefix+"delivery-lan-only"}
  :new string[0],
```

### Apps em segundo plano (o pedido “apps que consomem rede”)

```csharp
 const string PrivacyKey=@"SOFTWARE\Policies\Microsoft\Windows\AppPrivacy";
```

```csharp
  new Rule{Code="background-apps-deny",Title="Impedir apps da Loja de rodar em segundo plano",Category="Rede e downloads",Minimum=14393,Edition=1,
   Effect="Define LetAppsRunInBackground=2 (negar): apps empacotados da Microsoft Store não recebem execução em segundo plano.",
   Tradeoff="RISCO REAL DE PERDER FUNÇÃO. Param notificações push, sincronização de Email/Calendário, alarmes, convites do Xbox, atualização de blocos e apps de mensagem da Loja. Não afeta programas Win32 (Discord, Steam, Chrome), que é justamente onde costuma estar o consumo. Fora de qualquer perfil automático; aplique só se aceitar perder esses recursos.",
   Source="https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-privacy#letappsruninbackground",SourceKind="Documentado",Operations=new[]{Device(PrivacyKey,"LetAppsRunInBackground","2")}},
```

---

## 2.4 Apps e processos — navegadores

```csharp
  new Rule{Code="edge-sleeping-tabs",Title="Edge: hibernar abas inativas",Category="Apps e processos",Product="edge",ProductMinimum=88,
   Effect="Ativa SleepingTabsEnabled: abas paradas liberam CPU e memória, mantendo o título e o conteúdo restaurável ao clicar.",
   Tradeoff="Ganho real de RAM em quem deixa 20+ abas abertas. Abas que tocam áudio, usam a câmera ou têm sessão ativa não hibernam. Alguns sites recarregam ao voltar e perdem formulário não salvo. Confira em edge://policy; vale para todos os usuários até desfazer.",
   Source="https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/sleepingtabsenabled",SourceKind="Documentado",Operations=new[]{Device(@"SOFTWARE\Policies\Microsoft\Edge","SleepingTabsEnabled","1")}},

  new Rule{Code="edge-sleeping-timeout",Title="Edge: hibernar abas após 5 minutos",Category="Apps e processos",Product="edge",ProductMinimum=88,
   Effect="Define SleepingTabsTimeout=300 segundos, o tempo de inatividade antes de a aba hibernar.",
   Tradeoff="Só tem efeito com a hibernação de abas ligada. Tempo curto libera memória mais cedo e recarrega mais; tempo longo faz o contrário. A política aceita apenas os intervalos listados na página oficial.",
   Source="https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/sleepingtabstimeout",SourceKind="Documentado",Operations=new[]{Device(@"SOFTWARE\Policies\Microsoft\Edge","SleepingTabsTimeout","300")}},
```

> **Antes de fixar `EfficiencyMode`.** A política existe (Edge 116+) e é o
> caminho certo para “modo eficiência”, mas o mapa de valores
> (`0..4`) precisa ser conferido na página oficial
> `https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/efficiencymode`
> **antes** de gravar um `Data` fixo. Enquanto não conferir, deixe fora do
> catálogo — é exatamente o tipo de palpite que este projeto não aceita.

### O que fazer com Discord, Teams, Steam, Epic e overlays

Não existe política de Registro oficial para “não iniciar com o Windows” nem
para “desligar o overlay” nesses produtos. Inventar uma chave seria snake oil.
As duas saídas corretas já estão na arquitetura:

1. **Inicialização** → Startup Manager ampliado (`06-modulos-novos.md §6.1`),
   que faz backup exato do valor `Run` e desfaz.
2. **Overlays** → passo guiado, registrado como `sugerido` e **nunca** como
   `aplicado` (`04-ui-ux.md §4.7`), com deep link para a tela certa:

| Overlay | Onde desligar | Deep link / caminho |
|---|---|---|
| Xbox Game Bar | Configurações do Windows | `ms-settings:gaming-gamebar` |
| Gravação em segundo plano | já é item nativo | `native-capture` |
| Steam | Steam → Configurações → No Jogo | `steam://open/settings` |
| Discord | Configurações → Sobreposição de jogo | interno do app |
| NVIDIA | NVIDIA App → Sobreposição no jogo | interno do app |
| AMD | AMD Software → Preferências | interno do app |

---

## 2.5 Privacidade moderada — 3 políticas documentadas

```csharp
  new Rule{Code="telemetry-required",Title="Diagnóstico do Windows: somente dados obrigatórios",Category="Privacidade opcional",Minimum=14393,Edition=1,
   Effect="Define AllowTelemetry=1 (Obrigatório/Básico): mantém o mínimo necessário para segurança e atualização, sem dados opcionais.",
   Tradeoff="Escolha de privacidade, não de desempenho — não muda FPS. O valor 0 (Segurança) só é respeitado em Enterprise, Education e IoT Enterprise; em Home e Pro o Windows trata 0 como 1. Reduz o material disponível à Microsoft para diagnosticar falhas suas. Confira em Configurações > Privacidade > Diagnóstico e comentários.",
   Source="https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-system#allowtelemetry",SourceKind="Documentado",Operations=new[]{Device(@"SOFTWARE\Policies\Microsoft\Windows\DataCollection","AllowTelemetry","1")}},

  new Rule{Code="location-apps-deny",Title="Impedir que apps acessem a localização",Category="Privacidade opcional",Minimum=14393,Edition=1,
   Effect="Define LetAppsAccessLocation=2 (negar) para apps empacotados da Microsoft Store.",
   Tradeoff="Mapas, Clima, Buscar meu dispositivo e fuso horário automático param de funcionar. Não afeta programas Win32 nem o serviço de localização do próprio Windows. Política para todos os usuários; use a página Privacidade do Windows se quiser exceções por app.",
   Source="https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-privacy#letappsaccesslocation",SourceKind="Documentado",Operations=new[]{Device(PrivacyKey,"LetAppsAccessLocation","2")}},

  new Rule{Code="tailored-experiences-off",Title="Não personalizar dicas com dados de diagnóstico",Category="Privacidade opcional",Minimum=17134,
   Effect="Define TailoredExperiencesWithDiagnosticDataEnabled=0 para o usuário atual.",
   Tradeoff="Dicas, anúncios e recomendações deixam de usar seus dados de diagnóstico para personalização — elas continuam aparecendo, só que genéricas. Preferência do usuário, não política; reversível pela própria tela de Privacidade.",
   Source="https://learn.microsoft.com/en-us/windows/privacy/manage-windows-11-endpoints",SourceKind="Observado",Operations=new[]{User(@"Software\Microsoft\Windows\CurrentVersion\Privacy","TailoredExperiencesWithDiagnosticDataEnabled","0")}},
```

Nota: `tailored-experiences-off` usa `User(...)` em chave **não** política, logo
`IsReviewed` aceita — é um candidato legítimo ao cenário “Privacidade moderada”.

O resto do que o pedido cita (Advertising ID, Activity history, Inking & typing)
**já existe** no catálogo: `opt-splitdisabledbygrouppolicy`,
`opt-splituploaduseractivities`, `opt-splitenableactivityfeed`,
`opt-splitallowlinguisticdatacollection`, `pack-0b9181ab23ef`,
`pack-1b4dfbdca78f`. Não duplique: agrupe num cenário (`03`).

---

## 2.6 Barra de tarefas e Explorer — preferências do usuário

Eligíveis a perfil automático (HKCU, não política). Marque-as como
`SourceKind="Observado"`: são as preferências que a própria barra de tarefas
grava, não uma política com contrato.

```csharp
  new Rule{Code="taskbar-widgets-user",Title="Windows 11: ocultar botão de Widgets (por usuário)",Category="Windows leve",Minimum=22000,
   Effect="Define TaskbarDa=0, a mesma preferência que o menu de contexto da barra grava ao desmarcar Widgets.",
   Tradeoff="Alternativa sem administrador à política AllowNewsAndInterests: vale só para este usuário e não exige edição Pro. É aparência e consumo de rede em segundo plano, não FPS. A Microsoft não documenta este valor como política; uma atualização pode mudá-lo.",
   Source="https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-newsandinterests#allownewsandinterests",SourceKind="Observado",Operations=new[]{User(ExplorerKey,"TaskbarDa","0")}},

  new Rule{Code="taskview-hide",Title="Ocultar botão de Visão de Tarefas",Category="Personalização",
   Effect="Define ShowTaskViewButton=0 na barra de tarefas do usuário atual.",
   Tradeoff="Puramente visual. O atalho Win+Tab e as áreas de trabalho virtuais continuam funcionando. Pode exigir reabrir o Explorer.",
   Source="https://support.microsoft.com/en-us/windows/customize-the-taskbar-in-windows-92196ea1-9ed7-4e7b-a1a7-dbd2b0e8c0d1",SourceKind="Observado",Operations=new[]{User(ExplorerKey,"ShowTaskViewButton","0")}},

  new Rule{Code="start-recent-off",Title="Iniciar: não listar itens recém-abertos",Category="Privacidade opcional",
   Effect="Define Start_TrackDocs=0: o menu Iniciar e as Listas de Atalhos param de mostrar arquivos abertos recentemente.",
   Tradeoff="Menos leitura de disco ao abrir o Iniciar e menos exposição de nomes de arquivo em tela compartilhada. Você perde o acesso rápido ao que abriu por último. Equivale à chave de Personalização > Iniciar do Windows.",
   Source="https://support.microsoft.com/en-us/windows/customize-the-start-menu-c67ca4d0-7ef7-4a26-a0d9-05a9f2d9d5f9",SourceKind="Observado",Operations=new[]{User(ExplorerKey,"Start_TrackDocs","0")}},

  new Rule{Code="gamebar-tips-off",Title="Game Bar: não mostrar o painel de dicas ao abrir jogos",Category="Apps e processos",Minimum=17134,
   Effect="Define ShowStartupPanel=0 em Software\\Microsoft\\GameBar para o usuário atual.",
   Tradeoff="Remove o aviso “Pressione Win+G” que aparece sobre o jogo. Não desativa a Game Bar, a captura nem o Modo Jogo — para gravação use a opção nativa Desativar captura de jogos. É conforto visual, não FPS.",
   Source="https://support.microsoft.com/en-us/windows/game-bar-in-windows-8f2a4f4f-5f4a-4c1e-9a5f-1c7f6bdb1c5a",SourceKind="Observado",Operations=new[]{User(@"Software\Microsoft\GameBar","ShowStartupPanel","0")}},
```

---

## 2.7 Limpezas — Storage Sense por Policy CSP (nada de chave observada)

A configuração por usuário do Sensor de Armazenamento vive num `StoragePolicy`
binário **não documentado**. O caminho documentado é o Policy CSP `Storage`,
em `SOFTWARE\Policies\Microsoft\Windows\StorageSense`. Use ele.

```csharp
 const string StorageSenseKey=@"SOFTWARE\Policies\Microsoft\Windows\StorageSense";
 const string StorageSenseSource="https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-storage";
```

```csharp
  new Rule{Code="storagesense-on",Title="Sensor de Armazenamento: ativar",Category="Limpeza e manutenção",Minimum=17763,Edition=1,
   Effect="Define AllowStorageSenseGlobal=1: o Windows passa a liberar espaço sozinho quando o disco fica cheio.",
   Tradeoff="Manutenção automática, não desempenho. O usuário deixa de poder desligar o recurso pela tela de Armazenamento enquanto a política estiver aplicada. Nada é apagado no momento em que você aplica: o Windows decide o quê e quando, segundo as regras abaixo.",
   Source=StorageSenseSource+"#allowstoragesenseglobal",SourceKind="Documentado",Operations=new[]{Device(StorageSenseKey,"AllowStorageSenseGlobal","1")}},

  new Rule{Code="storagesense-monthly",Title="Sensor de Armazenamento: executar todo mês",Category="Limpeza e manutenção",Minimum=17763,Edition=1,
   Effect="Define ConfigStorageSenseGlobalCadence=30 dias.",
   Tradeoff="Cadência conservadora. Valores aceitos: 1 (diário), 7 (semanal), 30 (mensal) e 0 (quando o espaço acabar). Depende de o Sensor estar ativado.",
   Source=StorageSenseSource+"#configstoragesenseglobalcadence",SourceKind="Documentado",Operations=new[]{Device(StorageSenseKey,"ConfigStorageSenseGlobalCadence","30")}},

  new Rule{Code="storagesense-temp",Title="Sensor de Armazenamento: limpar arquivos temporários",Category="Limpeza e manutenção",Minimum=17763,Edition=1,
   Effect="Define AllowStorageSenseTemporaryFilesCleanup=1: temporários de app não usados entram na limpeza automática.",
   Tradeoff="É a categoria mais segura do Sensor. Não toca em Downloads, Documentos nem em arquivos do OneDrive sob demanda. Depende de o Sensor estar ativado.",
   Source=StorageSenseSource+"#allowstoragesensetemporaryfilescleanup",SourceKind="Documentado",Operations=new[]{Device(StorageSenseKey,"AllowStorageSenseTemporaryFilesCleanup","1")}},

  new Rule{Code="storagesense-recycle",Title="Sensor de Armazenamento: esvaziar Lixeira após 30 dias",Category="Limpeza e manutenção",Minimum=17763,Edition=1,
   Effect="Define ConfigStorageSenseRecycleBinCleanupThreshold=30 dias.",
   Tradeoff="EXCLUSÃO DEFINITIVA AUTOMÁTICA. Arquivos na Lixeira há mais de 30 dias deixam de ser recuperáveis. Use 0 para nunca esvaziar. Não afeta arquivos fora da Lixeira.",
   Source=StorageSenseSource+"#configstoragesenserecyclebincleanupthreshold",SourceKind="Documentado",Operations=new[]{Device(StorageSenseKey,"ConfigStorageSenseRecycleBinCleanupThreshold","30")}},
```

Categoria nova `"Limpeza e manutenção"` — acrescente-a às listas de filtro da UI
(`04-ui-ux.md §4.2`).

---

## 2.8 Valores numéricos escolhidos pelo usuário

Percentual de banda, cadência do Sensor e timeout de aba são números, não
chaves booleanas. O projeto já tem o padrão certo em
`ExtraSettings.Target/Manual` (`SystemResponsiveness`, `Win32PrioritySeparation`).
Generalize-o em vez de multiplicar `Rule`s com valor fixo.

```csharp
// source/ExtraSettings.cs — substitui Target(string) e complementa Manual(string)
 sealed class ManualField {
  internal string Id,Label,Source;internal RegOp Op;internal uint Minimum,Maximum;internal uint[] Allowed;
  internal ManualField(string id,string label,RegOp op,uint minimum,uint maximum,string source,uint[] allowed){
   Id=id;Label=label;Op=op;Minimum=minimum;Maximum=maximum;Source=source;Allowed=allowed;}
 }
 static readonly ManualField[] ManualFields={
  new ManualField("responsiveness","SystemResponsiveness (0–100)",
   new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile",Name="SystemResponsiveness",Kind="DWord"},
   0,100,"https://learn.microsoft.com/en-us/windows/win32/procthread/multimedia-class-scheduler-service",null),
  new ManualField("priority","Win32PrioritySeparation (0–63)",
   new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SYSTEM\CurrentControlSet\Control\PriorityControl",Name="Win32PrioritySeparation",Kind="DWord"},
   0,63,"https://learn.microsoft.com/en-us/windows-server/administration/performance-tuning/",null),
  // 6.2
  new ManualField("do-background","Banda de segundo plano do Delivery Optimization (1–100 %)",
   new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization",Name="DOPercentageMaxBackgroundBandwidth",Kind="DWord"},
   1,100,"https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference#percentagemaxbackgrounddownloadbandwidth",null),
  new ManualField("do-foreground","Banda de primeiro plano do Delivery Optimization (1–100 %)",
   new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization",Name="DOPercentageMaxForegroundBandwidth",Kind="DWord"},
   1,100,"https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference#percentagemaxforegrounddownloadbandwidth",null),
  new ManualField("storagesense-cadence","Cadência do Sensor de Armazenamento (0, 1, 7 ou 30 dias)",
   new RegOp{Hive="HKEY_LOCAL_MACHINE",Key=@"SOFTWARE\Policies\Microsoft\Windows\StorageSense",Name="ConfigStorageSenseGlobalCadence",Kind="DWord"},
   0,30,"https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-storage#configstoragesenseglobalcadence",new uint[]{0,1,7,30})
 };
 static ManualField Field(string id){
  foreach(var f in ManualFields)if(f.Id==id)return f;
  throw new Exception("Campo manual desconhecido.");
 }
 internal static RegOp Target(string id){var f=Field(id);
  return new RegOp{Hive=f.Op.Hive,Key=f.Op.Key,Name=f.Op.Name,Kind=f.Op.Kind};}
 internal static string[] FieldIds(){var ids=new string[ManualFields.Length];
  for(int n=0;n<ManualFields.Length;n++)ids[n]=ManualFields[n].Id;return ids;}
 internal static string FieldLabel(string id){return Field(id).Label;}
 internal static string FieldSource(string id){return Field(id).Source;}
 internal static List<Item> Manual(string payload){
  var result=new List<Item>();var chunks=payload.Split(new[]{','});
  if(chunks.Length>ManualFields.Length)throw new Exception("Seleção manual inválida.");
  foreach(var c in chunks){
   var p=c.Split(new[]{'='});uint value;
   if(p.Length!=2||!UInt32.TryParse(p[1],out value))throw new Exception("Informe um valor decimal válido.");
   var field=Field(p[0]);
   if(value<field.Minimum||value>field.Maximum)throw new Exception("Fora do intervalo aceito: "+field.Label);
   if(field.Allowed!=null&&!field.Allowed.Contains(value))throw new Exception("Valor não aceito por esta política: "+field.Label);
   if(result.Any(i=>i.Id=="manual-"+p[0]))throw new Exception("Campo repetido.");
   var o=Target(p[0]);o.Data=value.ToString();
   result.Add(new Item{Id="manual-"+p[0],Name=o.Name,ReviewTitle=o.Name+" = "+value,ActionCode="manualregistry",
    Native=true,Operations=new[]{o},Available=true,Url=field.Source,SourceKind="Documentado"});
  }
  return result;
 }
 internal static string ManualRows(){
  var b="";
  foreach(string id in FieldIds()){var o=Target(id);
   try{var v=App.Capture(o).Before;b+=LiquidProtocol.Row("manual",id,v.Delete?"":v.Data,ChangeAudit.Value(v));}
   catch(Exception e){b+=LiquidProtocol.Row("manual",id,"","Não foi possível ler: "+e.Message);}}
  return b;
 }
```

O `ManualRows()` continua com a mesma forma de linha, então a casca nativa não
quebra; só passam a existir 5 campos em vez de 2. A página “Ajustes manuais”
(`FunctionalInterface.cs:111`) deve iterar `ExtraSettings.FieldIds()` em vez de
ter duas caixas fixas.

---

## 2.9 O que este plano recusa — e por quê

Vale colar no próprio produto, numa tela “O que não fazemos”:

| Tweak popular | Por que fica de fora |
|---|---|
| `GameDVR_FSEBehaviorMode=2` (“desativar otimizações de tela cheia”) | Valor não documentado; o comportamento de Fullscreen Optimizations mudou várias vezes desde o Win10 1703. Sem medição, é palpite. |
| `Win32PrioritySeparation=26/38` fixo | Já existe como **campo manual** com faixa e sem valor recomendado. Recomendar um número seria inventar. |
| `NetworkThrottlingIndex=0xFFFFFFFF` | Já está no catálogo marcado como **experimental** (`OptionInfo` o lista em `experimental`). Continua fora de perfil automático. |
| `TcpAckFrequency`, `TCPNoDelay`, `TcpDelAckTicks` | Chaves por interface, não documentadas para uso geral, sem efeito medível fora de cenários de laboratório. |
| `DisableDynamicTick`, `useplatformclock` (BCD) | Mexe em boot. Risco alto, ganho não comprovado, e a restauração depende do BCD e não da engine do app. |
| `MemoryCompression` desligado | Já existe como item explicitamente rotulado **teste A/B** com aviso de piora possível. Nunca em perfil. |
| Limpar “memória standby” | `ProductInterface.cs:55` já explica que exige mecanismo não documentado e permanece indisponível. Mantenha essa honestidade. |
| Desativar `SysMain`/`Superfetch` | Não está em `App.OptionalServices` de propósito. Em SSD o ganho é nulo e pode piorar tempo de abertura. |
| “Ultimate Performance” como padrão | O plano existe, mas só em algumas edições, e aumenta consumo sem ganho medido em jogo. `native-power` (Alto desempenho, só desktop na tomada) já é o limite razoável. |
| `StoragePolicy` binário do Sensor de Armazenamento | Não documentado. Usamos Policy CSP (`§2.7`). |
| Deletar `C:\Windows\SoftwareDistribution\Download` | Requer parar `wuauserv` e pode forçar rebaixamento de atualizações. O caminho certo é o Sensor de Armazenamento / “Arquivos temporários” do Windows. |

---

## 2.10 Checklist de integração

- [ ] `Rule` novo tem `Code`, `Title`, `Category`, `Effect`, `Tradeoff`, `Source`, `SourceKind`, `Operations`, e `Minimum`/`Edition` quando a política exigir.
- [ ] Item novo por SPI tem entrada em `NativeSettings.Flags` **e** em `NativeSettings.Codes`.
- [ ] Nenhum ID repetido — rode `CatalogIds.Validate()`.
- [ ] Entrada nova em `PerformanceLibrary` só com `Action=="native"` se existir `Item` com esse `Target`.
- [ ] Conflitos declarados em `Item.Conflicts` (Delivery Optimization, tema claro/escuro).
- [ ] `HomeOption` com título curto para todo item novo que deva aparecer na página “Otimizações”.
- [ ] Arquivo `.cs` novo adicionado ao `compilacao.rsp`.
