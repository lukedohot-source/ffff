# 4 · UI/UX mais clara, bonita e informativa

## 4.1 Correção nº 1 · reexibir o que foi escondido

Esta é a mudança de maior retorno do plano inteiro e cabe em seis linhas.

### `source/UnifiedInterface.cs`

```csharp
// linha 54 — contador e "Simular seleção"
var third=new FlowLayoutPanel{Dock=DockStyle.Fill,WrapContents=false,AutoScroll=true,Margin=Padding.Empty};
// linha 55 — remova Visible=false do homeAdvanced (e passe a LÊ-LO, ver §4.2)
homeAdvanced=new ReadableCheckBox{Text="Mostrar também as opções avançadas",AutoSize=false,Width=280,Height=36,
 Margin=new Padding(6,0,8,0),ForeColor=Theme.Muted,BackColor=Theme.Bg,
 AccessibleName="Mostrar também as opções avançadas",Checked=false};
// linha 58 — busca + filtro de risco
var filters=new FlowLayoutPanel{Dock=DockStyle.Fill,WrapContents=false,AutoScroll=true};
// linha 59 — A GRADE
homeFlow=new FlowLayoutPanel{Dock=DockStyle.Fill,AutoScroll=true,WrapContents=true,Padding=new Padding(0,0,8,12)};
// linha 84 — atalhos por categoria
foreach(var link in CategoryTools(category)){string target=link.Value;var button=Theme.Button(link.Key);
 button.Height=34;button.Dock=DockStyle.Top;button.Margin=new Padding(0,5,0,0); /* sem Visible=false */
```

**Ajuste de altura obrigatório junto.** A linha 0 da página tem 170 px
(`Rows(170,-1)`), mas `top=Rows(86,46,46,40)` soma **218 px**. Com as quatro
faixas visíveis o conteúdo é cortado:

```csharp
 void BuildUnified(){
  var page=Page("Otimizações");var root=Rows(222,-1);page.Controls.Add(root);   // era Rows(170,-1)
  var top=Rows(86,46,46,44);root.Controls.Add(top,0,0);                          // 86+46+46+44 = 222
```

### `source/ProductInterface.cs`

```csharp
// linha 93 — grade de categorias de limpeza
var categoryGrid=new TableLayoutPanel{Dock=DockStyle.Fill,ColumnCount=2,RowCount=categoryRows,
 AutoScroll=true,Margin=Padding.Empty};
// linha 97 — seletor de idade, com padrão honesto de 7 dias
cleanupAge=new NumericUpDown{Minimum=0,Maximum=3650,Value=productPrefs.CleanupAgeDays,Width=70};
bar.Controls.Add(new Label{Text="dias de idade mínima (0 = sem filtro)",AutoSize=true,
 ForeColor=Theme.Muted,Padding=new Padding(4,10,0,0)});
// linha 56 — lista de processos do Memory Manager
memoryList=new CheckableList{CheckBoxes=true,ShowItemToolTips=true,Name="memory-processes"};
// linha 58 — barra do modo automático
var automatic=ProductBar();root.Controls.Add(automatic,0,3);
```

`productPrefs.CleanupAgeDays` já existe e já vem `7` por padrão
(`ProductInterface.cs:14`) — só nunca foi ligado ao controle.
`CleanAllTemporariesOneClick` deve parar de forçar `cleanupAge.Value=0` e passar
a respeitar a preferência.

**O teste tem que acompanhar** (`tests/ProductUiTests.cs`), senão ele passa com
a UI escondida de novo. Ver `07-testes-e-robustez.md §7.3`.

---

## 4.2 Navegação em seções, e o fim da lista de 27 botões

### `source/Interface.cs` — sidebar com seções

Substitua o bloco da linha 86:

```csharp
  // 6.2 — seção, rótulo, página
  var navigation=new[]{
   new[]{"","Painel","Painel"},
   new[]{"OTIMIZAR","Jogos e FPS","Otimizações"},
   new[]{"OTIMIZAR","Apps e processos","Apps e RAM"},
   new[]{"OTIMIZAR","Windows leve","Central de desempenho"},
   new[]{"OTIMIZAR","Rede e downloads","Rede e latência"},
   new[]{"OTIMIZAR","Privacidade","Avançado / Alto Impacto"},
   new[]{"MANTER","Limpezas","Temporários"},
   new[]{"MANTER","Diagnósticos oficiais","Diagnósticos oficiais"},
   new[]{"MANTER","Inicialização e serviços","Manutenção"},
   new[]{"MANTER","Memória","Memory Manager"},
   new[]{"CONSULTAR","Catálogo completo","Catálogo"},
   new[]{"CONSULTAR","Sistema & Info","Hardware e rede"},
   new[]{"CONSULTAR","Histórico","Histórico"},
   new[]{"CONSULTAR","Resultados","Resultados"},
   new[]{"CONSULTAR","Todas as ferramentas","Ferramentas"}
  };
  string section=null;
  for(int n=0;n<navigation.Length;n++){
   if(navigation[n][0]!=section&&navigation[n][0].Length>0){
    section=navigation[n][0];
    var header=Theme.Label(section,8,Theme.Muted,true);
    header.Dock=DockStyle.None;header.Width=176;header.Height=26;header.Margin=new Padding(2,10,0,2);
    links.Controls.Add(header);
   }
   string key=navigation[n][2];
   var b=Theme.Button(navigation[n][1]);
   b.Width=176;b.Height=36;b.Margin=new Padding(0,0,0,4);
   b.Glyph=NavIcon(key);            // ver a mudança em ActionButton logo abaixo
   b.Click+=(s,e)=>ShowPage(key);
   nav[key]=b;links.Controls.Add(b);
  }
```

`ActionButton.OnPaint` é totalmente customizado (`Interface.cs:47`) e **ignora**
a propriedade `Image` herdada de `Button`. Por isso o ícone entra como campo
próprio e é desenhado à mão:

```csharp
// source/Interface.cs — ActionButton
class ActionButton : Button {
 internal bool Primary,Active;bool hover;
 internal Image Glyph;                                   // 6.2
 // … construtor e OnMouseEnter/Leave inalterados …
 protected override void OnPaint(PaintEventArgs e){
  var g=e.Graphics;g.SmoothingMode=SmoothingMode.AntiAlias;g.Clear(Parent==null?Theme.Bg:Parent.BackColor);
  Color c=!Enabled?Theme.Line:Primary?Theme.Accent:Active?Color.FromArgb(62,55,27):hover?Color.FromArgb(39,49,62):Theme.Card;
  using(var path=Theme.Round(new Rectangle(0,0,Math.Max(1,Width-1),Math.Max(1,Height-1)),8))
  using(var b=new SolidBrush(c))g.FillPath(b,path);
  var text=ClientRectangle;var flags=TextFormatFlags.HorizontalCenter|TextFormatFlags.VerticalCenter|TextFormatFlags.EndEllipsis;
  if(Glyph!=null){
   g.DrawImage(Glyph,new Rectangle(12,(Height-Glyph.Height)/2,Glyph.Width,Glyph.Height));
   text=new Rectangle(38,0,Math.Max(1,Width-46),Height);
   flags=TextFormatFlags.Left|TextFormatFlags.VerticalCenter|TextFormatFlags.EndEllipsis;
  }
  TextRenderer.DrawText(g,Text,Font,text,!Enabled?Theme.Muted:Primary?Theme.Bg:Active?Theme.Accent:Theme.Text,flags);
  if(Focused)ControlPaint.DrawFocusRectangle(g,new Rectangle(4,4,Width-8,Height-8),Theme.Text,c);
 }
}
```

O `Glyph` só é desenhado quando existe, então todos os outros botões do app
continuam centralizados exatamente como hoje.

`ShowPage` já pinta a aba ativa via `ActionButton.Active` + `Invalidate()`
(`Interface.cs:103`), e `ActionButton.OnPaint` já usa `Active` para o realce —
não precisa de nada novo para “destacar a aba ativa”.

### Ícones por categoria sem dependência externa

O projeto **não** tem biblioteca de ícones no lado WinForms (as fontes Font
Awesome estão só na casca nativa ImGui, em `native/assets/`). Desenhe-os:

```csharp
// source/Interface.cs
 // O cache é limpo por Theme.Apply: a cor do traço vem de Theme.Muted.
 internal static readonly Dictionary<string,Image> navIcons=new Dictionary<string,Image>();
 static Image NavIcon(string page){
  Image cached;if(navIcons.TryGetValue(page,out cached))return cached;
  var bitmap=new Bitmap(18,18);
  using(var g=Graphics.FromImage(bitmap)){
   g.SmoothingMode=SmoothingMode.AntiAlias;g.Clear(Color.Transparent);
   Color color=Theme.Muted;
   using(var pen=new Pen(color,1.7f))using(var brush=new SolidBrush(color)){
    if(page=="Otimizações"){g.DrawRectangle(pen,3,5,12,8);g.DrawLine(pen,6,13,6,16);g.DrawLine(pen,12,13,12,16);}      // CPU
    else if(page=="Apps e RAM"){g.DrawRectangle(pen,2,4,14,10);g.DrawLine(pen,2,7,16,7);}                              // janelas
    else if(page=="Central de desempenho"){g.DrawLine(pen,2,14,6,8);g.DrawLine(pen,6,8,10,11);g.DrawLine(pen,10,11,16,3);} // gráfico
    else if(page=="Rede e latência"){g.DrawArc(pen,2,6,14,14,200,140);g.DrawArc(pen,5,9,8,8,200,140);g.FillEllipse(brush,8,14,3,3);} // wifi
    else if(page=="Avançado / Alto Impacto"){g.DrawRectangle(pen,4,8,10,8);g.DrawArc(pen,6,3,6,8,180,180);}            // cadeado
    else if(page=="Temporários"){g.DrawLine(pen,3,5,15,5);g.DrawRectangle(pen,5,5,8,11);}                              // lixeira
    else if(page=="Diagnósticos oficiais"){g.DrawEllipse(pen,3,3,10,10);g.DrawLine(pen,11,11,16,16);}                  // lupa
    else if(page=="Manutenção"){g.DrawEllipse(pen,5,5,8,8);g.DrawLine(pen,9,1,9,4);g.DrawLine(pen,9,14,9,17);}         // engrenagem
    else if(page=="Memory Manager"){g.DrawRectangle(pen,3,6,12,7);g.DrawLine(pen,6,3,6,6);g.DrawLine(pen,12,3,12,6);}  // RAM
    else if(page=="Hardware e rede"){g.DrawRectangle(pen,2,3,14,10);g.DrawLine(pen,6,16,12,16);}                       // monitor
    else if(page=="Histórico"){g.DrawEllipse(pen,2,2,14,14);g.DrawLine(pen,9,5,9,9);g.DrawLine(pen,9,9,13,11);}        // relógio
    else if(page=="Catálogo"){g.DrawRectangle(pen,3,3,12,13);g.DrawLine(pen,6,7,12,7);g.DrawLine(pen,6,11,12,11);}     // lista
    else {g.DrawRectangle(pen,3,3,5,5);g.DrawRectangle(pen,10,3,5,5);g.DrawRectangle(pen,3,10,5,5);g.DrawRectangle(pen,10,10,5,5);} // grade
   }
  }
  navIcons[page]=bitmap;return bitmap;
 }
```

18×18 sobrevive a 125 % e 150 % de escala sem borrar tanto quanto um PNG,
porque é vetor redesenhado. Se preferir nitidez perfeita em DPI alto, gere o
bitmap na escala corrente usando `CreateGraphics().DpiX/96f`.

### Categorias da página “Otimizações”

`HomeCategories` deve alinhar com a navegação e ganhar “Privacidade”:

```csharp
 static readonly string[] HomeCategories={"FPS","Input Lag","Windows","Processos",
  "Inicialização","Rede","Privacidade","Energia","SSD","Limpeza","Diagnóstico"};
```

E `AdvancedCategory` precisa parar de jogar tudo em `"Windows"`. Use a
categoria declarada do item em vez de adivinhar por substring:

```csharp
 static readonly Dictionary<string,string> CategoryMap=new Dictionary<string,string>(StringComparer.OrdinalIgnoreCase){
  {"Jogos e FPS","FPS"},{"Apps e processos","Processos"},{"Windows leve","Windows"},
  {"Personalização","Windows"},{"Rede e downloads","Rede"},{"Privacidade opcional","Privacidade"},
  {"Limpeza e manutenção","Limpeza"},{"CPU e energia","Energia"},{"Armazenamento","SSD"},
  {"Input lag","Input Lag"},{"Input e monitor","Input Lag"},{"Reparos de acesso","Windows"},
  {"Avançado: perde recursos","Windows"}
 };
 static string AdvancedCategory(Item item){
  string declared=item.Category==null?"":item.Category;
  int dash=declared.IndexOf(" - ",StringComparison.Ordinal);       // tira o "-3 - " de ordenação
  if(dash>=0)declared=declared.Substring(dash+3);
  string mapped;
  if(CategoryMap.TryGetValue(declared.Trim(),out mapped))return mapped;
  string text=(declared+" "+item.ReviewTitle).ToLowerInvariant();  // heurística só como último recurso
  if(text.Contains("energia")||text.Contains("power"))return "Energia";
  if(text.Contains("rede")||text.Contains("dns")||text.Contains("tcp"))return "Rede";
  if(text.Contains("privacidade")||text.Contains("telemetria"))return "Privacidade";
  if(text.Contains("mouse")||text.Contains("tecl")||text.Contains("input"))return "Input Lag";
  if(text.Contains("game")||text.Contains("jog")||text.Contains("fps"))return "FPS";
  if(text.Contains("disco")||text.Contains("ssd")||text.Contains("ntfs"))return "SSD";
  if(text.Contains("serviço")||text.Contains("segundo plano"))return "Processos";
  return "Windows";
 }
```

E finalmente **ler** `homeAdvanced`, que hoje é decorativo:

```csharp
 bool HomeVisible(HomeOption option,HashSet<string> primary){
  if(!homeAdvanced.Checked&&!primary.Contains(option.Id))return false;   // 6.2 — a caixa passa a fazer algo
  string query=homeSearch==null?"":homeSearch.Text.Trim();
  if(query.Length>0&&(option.Title+" "+option.Category+" "+option.Id).IndexOf(query,StringComparison.CurrentCultureIgnoreCase)<0)return false;
  string risk=homeRisk==null?"Todos os riscos":Convert.ToString(homeRisk.SelectedItem);
  if(risk=="Favoritos")return favorites.Contains(option.Id);
  if(risk=="Todos os riscos")return true;
  if(option.Diagnostic)return risk=="Baixo";
  return OptionInfo.For(App.ById(option.Id)).Risk==risk;
 }
```

Com `homeAdvanced` desmarcado, a página abre com ~18 opções curadas em vez de
131 — que é o verdadeiro motivo pelo qual alguém escondeu a grade.

---

## 4.3 Badges de risco e de tipo

### Controle novo · `source/Badges.cs` (adicione ao `compilacao.rsp`)

```csharp
using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Collections.Generic;
using System.Windows.Forms;

class Badge {
 internal string Text;internal Color Ink,Edge;
 internal Badge(string text,Color ink,Color edge){Text=text;Ink=ink;Edge=edge;}
}

// Faixa de pílulas desenhadas; sem imagem, sem dependência, escala com DPI.
class BadgeStrip : Control {
 internal List<Badge> Badges=new List<Badge>();
 public BadgeStrip(){SetStyle(ControlStyles.UserPaint|ControlStyles.AllPaintingInWmPaint|ControlStyles.OptimizedDoubleBuffer|ControlStyles.SupportsTransparentBackColor,true);
  Height=20;Font=Theme.Font(7.5f,true);BackColor=Color.Transparent;}
 protected override void OnPaint(PaintEventArgs e){
  var g=e.Graphics;g.SmoothingMode=SmoothingMode.AntiAlias;
  g.Clear(Parent==null?Theme.Card:Parent.BackColor);
  int x=0;
  foreach(var badge in Badges){
   var size=TextRenderer.MeasureText(g,badge.Text,Font);
   int width=size.Width+12;
   if(x+width>Width)break;
   var box=new Rectangle(x,1,width,Height-3);
   using(var path=Theme.Round(box,(Height-3)/2))
   using(var fill=new SolidBrush(Color.FromArgb(38,badge.Edge)))
   using(var pen=new Pen(badge.Edge)){g.FillPath(fill,path);g.DrawPath(pen,path);}
   TextRenderer.DrawText(g,badge.Text,Font,box,badge.Ink,
    TextFormatFlags.HorizontalCenter|TextFormatFlags.VerticalCenter|TextFormatFlags.SingleLine);
   x+=width+5;
  }
 }
 internal string Description(){
  var parts=new List<string>();
  foreach(var badge in Badges)parts.Add(badge.Text);
  return String.Join(" · ",parts);
 }
}

static class BadgeFactory {
 // Tipo: de onde vem a mudança. Responde "isto mexe no quê?".
 internal static Badge Kind(Item item){
  if(item==null)return new Badge("Diagnóstico",Theme.Blue,Theme.Blue);
  var ops=item.Operations==null?new RegOp[0]:item.Operations;
  if(ops.Length==0)return new Badge("API nativa",Theme.Blue,Theme.Blue);
  foreach(var op in ops)if(App.IsPolicy(op))return new Badge("Política",Theme.Amber,Theme.Amber);
  foreach(var op in ops)if(op.Hive!="HKEY_CURRENT_USER")return new Badge("Registro (PC)",Theme.Amber,Theme.Amber);
  return new Badge("Registro (usuário)",Theme.Text,Theme.Line);
 }
 // Risco: já calculado por OptionInfo; só ganha cor.
 internal static Badge Risk(OptionInfo info){
  if(info.Risk=="Alto")return new Badge("Perde recursos",Theme.Red,Theme.Red);
  if(info.Risk=="Moderado")return new Badge("Avançado",Theme.Amber,Theme.Amber);
  return new Badge("Seguro",Theme.Accent,Theme.Accent);
 }
 // Evidência: Policy CSP documentada vs. preferência observada.
 internal static Badge Evidence(OptionInfo info){
  if(info.Evidence=="Observado")return new Badge("Observado",Theme.Amber,Theme.Amber);
  if(info.Evidence=="API oficial")return new Badge("API oficial",Theme.Blue,Theme.Blue);
  return new Badge("Documentado",Theme.Muted,Theme.Line);
 }
 internal static Badge Admin(OptionInfo info){return new Badge("Administrador",Theme.Blue,Theme.Blue);}
 internal static Badge Scenario(string title){return new Badge(title,Theme.Accent,Theme.Accent);}

 internal static BadgeStrip For(string optionId,bool diagnostic){
  var strip=new BadgeStrip{Dock=DockStyle.Fill,Margin=new Padding(0,0,4,0)};
  if(diagnostic){
   strip.Badges.Add(new Badge("Só diagnóstico",Theme.Blue,Theme.Blue));
   strip.Badges.Add(new Badge("Somente leitura",Theme.Accent,Theme.Accent));
  }else{
   var item=App.ById(optionId);var info=OptionInfo.For(item);
   strip.Badges.Add(Risk(info));
   strip.Badges.Add(Kind(item));
   strip.Badges.Add(Evidence(info));
   if(info.Admin)strip.Badges.Add(Admin(info));
   if(info.InScenarios!=null&&info.InScenarios.Length>0)strip.Badges.Add(Scenario("Em perfil"));
  }
  strip.AccessibleName=strip.Description();   // leitor de tela recebe o texto, não só a cor
  return strip;
 }
}
```

**Acessibilidade:** cor nunca é o único canal — cada pílula carrega o texto, e
`AccessibleName` repete a faixa inteira. Quem usa leitor de tela ou não
distingue vermelho/âmbar recebe a mesma informação.

### Ligar no cartão de opção · `source/UnifiedInterface.cs`

Dentro de `BuildHomeCards()`, a linha `row` hoje tem `ColumnCount=1`.
Passe a duas linhas:

```csharp
    var row=new TableLayoutPanel{Height=productPrefs==null||productPrefs.Compact?46:56,
     Dock=DockStyle.Top,ColumnCount=1,RowCount=2,Margin=Padding.Empty,Name="home-row-"+option.Id,
     Visible=HomeVisible(option,primary)&&!collapsedHome.Contains(category)};
    row.RowStyles.Add(new RowStyle(SizeType.Absolute,productPrefs==null||productPrefs.Compact?26:34));
    row.RowStyles.Add(new RowStyle(SizeType.Absolute,20));
    // … o CheckBox continua em (0,0), sem o sufixo " · Manual" no texto …
    row.Controls.Add(BadgeFactory.For(option.Id,option.Diagnostic),0,1);
```

E o cálculo de altura do cartão acompanha:

```csharp
   int rowHeight=productPrefs==null||productPrefs.Compact?46:56;
   int visibleRows=collapsedHome.Contains(category)?0:options.Count(o=>HomeVisible(o,primary));
   card.Height=52+rowHeight*visibleRows+39*(collapsedHome.Contains(category)||!String.IsNullOrWhiteSpace(homeSearch.Text)?0:CategoryTools(category).Count);
```

Com o badge, some o sufixo `" · Manual"` que hoje é concatenado no texto da
caixa (`UnifiedInterface.cs:78`) — a informação passa a ser visual e legível.

---

## 4.4 Página “Painel” como visão geral de verdade

`SystemTelemetry` já coleta quase tudo; o Painel só não mostra. Complete
`BuildLiveDashboard` (`ProductInterface.cs:37`) com os campos que já existem em
`TelemetrySnapshot` e três que faltam.

```csharp
// source/SystemTelemetry.cs — três leituras novas, todas só-leitura
 internal static string ActivePowerPlan(){
  try{
   using(var search=new ManagementObjectSearcher(@"root\cimv2\power",
    "SELECT ElementName FROM Win32_PowerPlan WHERE IsActive = True")){
    search.Options.Timeout=TimeSpan.FromSeconds(3);
    using(var rows=search.Get())foreach(ManagementObject row in rows)using(row)
     return Convert.ToString(row["ElementName"]);
   }
  }catch(Exception e){return "Plano não disponível: "+e.Message;}
  return "Plano não identificado";
 }
 internal static string GameModeState(){
  try{
   using(var hive=Microsoft.Win32.RegistryKey.OpenBaseKey(Microsoft.Win32.RegistryHive.CurrentUser,Microsoft.Win32.RegistryView.Default))
   using(var key=hive.OpenSubKey(@"Software\Microsoft\GameBar",false)){
    if(key==null)return "Modo Jogo: preferência não gravada (padrão do Windows)";
    object value=key.GetValue("AutoGameModeEnabled");
    if(value==null)return "Modo Jogo: preferência não gravada (padrão do Windows)";
    return Convert.ToInt64(value)!=0?"Modo Jogo: ligado":"Modo Jogo: desligado";
   }
  }catch(Exception e){return "Modo Jogo: leitura indisponível ("+e.Message+")";}
 }
 internal static string BackgroundSummary(){
  try{
   var startup=App.ScanStartup();int total=startup.Count,optional=0;
   foreach(var entry in startup)if(entry.Allowed)optional++;
   return total+" app(s) no Run do usuário · "+optional+" classificado(s) como opcional";
  }catch(Exception e){return "Inicialização: leitura indisponível ("+e.Message+")";}
 }
```

Layout proposto do Painel (cinco cartões `Surface`, reusando o `Metric` que já
existe em `Interface.cs:119`):

| Cartão | Fonte já existente | Fonte nova |
|---|---|---|
| CPU % · RAM usada/total · rede ↓↑ | `TelemetrySnapshot.CpuPercent`, `.Memory`, `.NetworkReceive/SendBytesPerSecond` | — |
| GPU e driver | `SystemTelemetry.Gpu()` | — |
| Disco: livre/total por unidade | `ReviewedDiagnostics.Read("diag54-storage")` | — |
| Plano de energia · Modo Jogo | `TelemetrySnapshot.PowerPlan` | `ActivePowerPlan()`, `GameModeState()` |
| Apps em segundo plano · reinício pendente | `.RestartPending`, `.StartupCount` | `BackgroundSummary()` |

**Não invente barra de “saúde”.** `SystemTelemetry.Score` já existe, já tem uma
fórmula documentada e já se recusa a dar nota sem medição
(`v6 no health score without measurements`). Mostre o número **com** a
`HealthExplanation` ao lado, nunca sozinho.

**Não coloque a página “Visão geral” órfã na navegação.** Ou mescle o conteúdo
dela no “Painel” e apague `BuildDashboard()`, ou renomeie o link. Ter as duas é
o que causa a confusão de hoje (**F5**).

---

## 4.5 Feedback não bloqueante · toasts

### `source/Toast.cs` (adicione ao `compilacao.rsp`)

```csharp
using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Windows.Forms;

// Aviso discreto no canto inferior direito da própria janela.
// Não é uma janela nova: é um Panel filho, então herda DPI, tema e Z-order.
class Toast : Panel {
 System.Windows.Forms.Timer life;Color accent;string caption,body;
 internal Toast(string title,string message,Color color){
  caption=title;body=message;accent=color;
  DoubleBuffered=true;Size=new Size(360,74);BackColor=Theme.Card;Cursor=Cursors.Hand;
  AccessibleName=title+". "+message;AccessibleRole=AccessibleRole.Alert;
  Click+=(s,e)=>Dismiss();
 }
 protected override void OnPaint(PaintEventArgs e){
  var g=e.Graphics;g.SmoothingMode=SmoothingMode.AntiAlias;
  g.Clear(Parent==null?Theme.Bg:Parent.BackColor);
  using(var path=Theme.Round(new Rectangle(0,0,Width-1,Height-1),10))
  using(var fill=new SolidBrush(Theme.Card))
  using(var pen=new Pen(accent,1.5f)){g.FillPath(fill,path);g.DrawPath(pen,path);}
  using(var bar=new SolidBrush(accent))g.FillRectangle(bar,new Rectangle(1,10,4,Height-20));
  TextRenderer.DrawText(g,caption,Theme.Font(10,true),new Rectangle(16,10,Width-28,20),accent,
   TextFormatFlags.Left|TextFormatFlags.VerticalCenter|TextFormatFlags.EndEllipsis);
  TextRenderer.DrawText(g,body,Theme.Font(9),new Rectangle(16,30,Width-28,Height-38),Theme.Muted,
   TextFormatFlags.Left|TextFormatFlags.WordBreak|TextFormatFlags.EndEllipsis);
 }
 internal void Start(int milliseconds){
  life=new System.Windows.Forms.Timer{Interval=milliseconds};
  life.Tick+=(s,e)=>Dismiss();life.Start();
 }
 void Dismiss(){
  if(life!=null){life.Stop();life.Dispose();life=null;}
  if(Parent!=null)Parent.Controls.Remove(this);
  Dispose();
 }
}

public partial class MainForm {
 readonly System.Collections.Generic.List<Toast> toasts=new System.Collections.Generic.List<Toast>();
 internal void Notify(string title,string message,bool problem=false){
  if(IsDisposed||!IsHandleCreated)return;
  footer.Text=title+" · "+message;                    // o rodapé continua sendo o registro textual
  if(App.IsUiTest())return;                           // teste de UI não deve depender de temporizador
  var toast=new Toast(title,message,problem?Theme.Red:Theme.Accent);
  toasts.RemoveAll(t=>t.IsDisposed);
  toast.Location=new Point(ClientSize.Width-toast.Width-24,
   ClientSize.Height-toast.Height-24-(toast.Height+10)*Math.Min(toasts.Count,3));
  toast.Anchor=AnchorStyles.Bottom|AnchorStyles.Right;
  Controls.Add(toast);toast.BringToFront();toasts.Add(toast);
  toast.Start(problem?8000:4500);
 }
}
```

### Onde trocar `MessageBox` por toast

| Local hoje | Vira |
|---|---|
| `Interface.cs:211` sucesso/erro de elevação | `Notify("Operação concluída", "…")` |
| `Interface.cs:219/220` falha ao abrir app externo | `Notify("Não foi possível abrir", e.Message, true)` |
| `EasyInterface.cs:186` “abra um jogo e atualize a lista” | `Notify("Nenhum app elegível", "…")` |
| `Interface.cs:99` “aguarde a operação terminar” | **continua `MessageBox`** — é bloqueio de fechamento, tem que bloquear |
| `Confirm()` de exclusão permanente | **continua modal** — decisão destrutiva pede modal |

E `Confirm()` deveria usar o overlay `InlineReview` sempre, não só no modo
embutido (**F13**):

```csharp
 Task<bool> Confirm(string text,string title="Confirmar seleção"){return InlineReview(title,text,true);}
```

O overlay já desabilita os controles de trás, já responde a `Escape`
(`Interface.cs:98`) e já é testado pelo smoke test.

---

## 4.6 Desempenho da UI — a correção que permite reexibir a busca

Sem isto, reabrir busca e filtros (`§4.1`) devolve a lentidão que motivou
escondê-los (**F4**).

### `source/LukeOptimizer.cs` — índice por ID

```csharp
 static Dictionary<string,Item> index;
 internal static Item ById(string id){
  if(index==null){
   index=new Dictionary<string,Item>(StringComparer.Ordinal);
   foreach(var item in Items)index[item.Id]=item;   // último vence, igual ao SingleOrDefault de hoje
  }
  Item found;return index.TryGetValue(id,out found)?found:null;
 }
 internal static void ResetIndex(){index=null;}      // chame após Items.AddRange no arranque
```

Troque em massa `App.Items.Single(i=>i.Id==x)` / `.FirstOrDefault(i=>i.Id==x)`
por `App.ById(x)`. Ocorrências em `UnifiedInterface.cs`: linhas 52, 78, 92,
119, 122, 123, 133, 144, 146 e 168.

### `source/OptionInfo.cs` — cache

```csharp
 static readonly Dictionary<string,OptionInfo> cache=new Dictionary<string,OptionInfo>(StringComparer.Ordinal);
 internal static OptionInfo For(Item item){
  if(item==null)return null;
  OptionInfo found;
  if(cache.TryGetValue(item.Id,out found))return found;
  found=Compute(item);cache[item.Id]=found;return found;
 }
 internal static void Invalidate(){cache.Clear();scenarioIndex=null;}
 static OptionInfo Compute(Item item){ /* … corpo atual do For(…) … */ }
```

`Invalidate()` deve ser chamado só quando cenários ou catálogo mudarem
(carregar um perfil salvo, por exemplo). Classificação, risco e evidência não
dependem do estado da máquina — quem depende é `App.Observe`, que continua
sendo lido a cada análise.

### `source/ExtremeProfile.cs` — grupos cacheados

```csharp
 static ExtremeGroup[] groups;
 internal static ExtremeGroup[] Groups(){if(groups==null)groups=Build();return groups;}
 static ExtremeGroup[] Build(){ /* … corpo do §3.3 … */ }
```

### Rebuild da grade com atraso

Reconstruir 131 cartões por tecla é caro mesmo com cache. Debounce:

```csharp
 System.Windows.Forms.Timer homeSearchDebounce;
 // em BuildUnified, no lugar de: homeSearch.TextChanged+=(s,e)=>BuildHomeCards();
 homeSearchDebounce=new System.Windows.Forms.Timer{Interval=220};
 homeSearchDebounce.Tick+=(s,e)=>{homeSearchDebounce.Stop();BuildHomeCards();};
 homeSearch.TextChanged+=(s,e)=>{homeSearchDebounce.Stop();homeSearchDebounce.Start();};
 FormClosed+=(s,e)=>{if(homeSearchDebounce!=null)homeSearchDebounce.Dispose();};
```

Melhor ainda, e sem debounce: filtrar sem reconstruir. `BuildHomeCards` já cria
todas as linhas e usa `Visible` para filtrar — basta separar as duas
responsabilidades:

```csharp
 void ApplyHomeFilter(){
  var primary=new HashSet<string>(PrimaryOptions().Select(o=>o.Id));
  homeFlow.SuspendLayout();
  foreach(var card in homeCards){
   string category=card.Name.Substring("home-category-".Length);
   int visible=0;
   foreach(Control child in card.Controls[0].Controls){
    if(child.Name==null||!child.Name.StartsWith("home-row-",StringComparison.Ordinal))continue;
    string id=child.Name.Substring("home-row-".Length);
    bool show=HomeVisible(homeOptions[id],primary)&&!collapsedHome.Contains(category);
    child.Visible=show;if(show)visible++;
   }
   int rowHeight=productPrefs==null||productPrefs.Compact?46:56;
   card.Height=52+rowHeight*visible;
   card.Visible=visible>0||collapsedHome.Contains(category);
  }
  homeFlow.ResumeLayout();homeLayoutWidth=0;SizeHomeCards();UpdateHome();
 }
```

Ligue `homeSearch.TextChanged`, `homeRisk.SelectedIndexChanged` e
`homeAdvanced.CheckedChanged` a `ApplyHomeFilter()`; deixe `BuildHomeCards()`
apenas para mudança de densidade e para o arranque. Efeito colateral bom:
a seleção deixa de ser salva/restaurada a cada tecla, o que hoje é feito por
um `saved` array (`UnifiedInterface.cs:68`) justamente porque tudo é destruído.

---

## 4.7 Fluxo de aplicação: prévia → execução → resumo

A prévia já está no `§3.6` (`ScenarioPreview`). Falta o **resumo**.

```csharp
// source/UnifiedInterface.cs
 void ShowScenarioSummary(ScenarioPlan plan,string[] appliedIds){
  var text=new System.Text.StringBuilder();
  text.AppendLine(plan.Profile.Title).AppendLine(new String('=',plan.Profile.Title.Length)).AppendLine();
  text.AppendLine("Aplicados ............ "+appliedIds.Length);
  text.AppendLine("Ignorados ............ "+plan.Skip.Count);
  text.AppendLine("Para confirmar ....... "+plan.Confirm.Count);
  text.AppendLine("Sugeridos ............ "+plan.Guide.Count+"  (não executados)");
  text.AppendLine();
  if(plan.Skip.Count>0){
   text.AppendLine("POR QUE ALGO FOI IGNORADO");
   foreach(var step in plan.Skip)text.AppendLine("  • "+step.Name+" — "+step.Message);
   text.AppendLine();
  }
  var record=records.FirstOrDefault();
  if(record!=null){
   text.AppendLine("REGISTRO");
   text.AppendLine("  Histórico: "+record.Id);
   text.AppendLine("  Pasta de backup: "+App.RecordsRoot);
   text.AppendLine("  Logs de execução: "+ExecutionHub.LogsRoot);
   text.AppendLine();
   text.AppendLine("Desfazer restaura os valores exatos anteriores, item a item.");
  }
  productResult.Text=text.ToString();
  Notify("Cenário concluído",appliedIds.Length+" aplicado(s), "+plan.Skip.Count+" ignorado(s)");
 }
```

E dois botões no rodapé da página “Resultados” que já existe
(`ProductInterface.cs:118` já tem `results-logs` e `results-txt`):

```csharp
 ProductButton(bar,"results-open-record","Abrir TXT deste registro",(s,e)=>{
  var record=records.FirstOrDefault();
  if(record==null)return;
  string path=System.IO.Path.Combine(App.RecordsRoot,record.Id+".txt");
  if(System.IO.File.Exists(path))OpenLocal(path);else Notify("TXT indisponível","O registro ainda não gerou arquivo de texto.",true);
 },false,210);
```

---

## 4.8 Tema claro

`Theme` é um conjunto de campos `static` mutáveis (`Interface.cs:12`) — ou seja,
o tema claro é viável sem refatorar nada, desde que trocado **antes** de
`new MainForm()`.

```csharp
// source/Interface.cs
static class Theme {
 internal static bool Light;
 internal static Color Bg,Side,Card,Line,Text,Muted,Accent,Amber,Red,Blue;
 static Theme(){Apply(false);}
 internal static void Apply(bool light){
  Light=light;
  if(light){
   Bg=Color.FromArgb(246,247,249);Side=Color.FromArgb(234,236,240);Card=Color.FromArgb(255,255,255);
   Line=Color.FromArgb(203,208,216);Text=Color.FromArgb(24,26,30);Muted=Color.FromArgb(92,99,110);
   Accent=Color.FromArgb(150,105,0);Amber=Color.FromArgb(150,92,0);
   Red=Color.FromArgb(176,32,40);Blue=Color.FromArgb(24,86,158);
  }else{
   Bg=Color.FromArgb(15,16,18);Side=Color.FromArgb(20,21,24);Card=Color.FromArgb(27,29,33);
   Line=Color.FromArgb(65,68,74);Text=Color.FromArgb(245,246,248);Muted=Color.FromArgb(180,185,194);
   Accent=Color.FromArgb(255,208,48);Amber=Color.FromArgb(244,191,112);
   Red=Color.FromArgb(248,139,143);Blue=Color.FromArgb(139,190,241);
  }
 }
}
```

No tema claro o acento vira âmbar escuro e o azul escurece — amarelo `255,208,48`
sobre branco dá contraste ~1.6:1, abaixo de qualquer limiar legível.
Os valores acima ficam ≥ 4.5:1 contra `Card`.

```csharp
// source/Interface.cs — ao fim de Theme.Apply
  foreach(var icon in navIcons.Values)icon.Dispose();
  navIcons.Clear();

// source/ProductInterface.cs — ProductPreferences
 public bool LightTheme {get;set;}
// source/LukeOptimizer.cs — antes de Application.Run(new MainForm())
 Theme.Apply(ProductPreferences.Load().LightTheme);
```

Trocar o tema com a janela aberta exigiria recriar os controles; o caminho
honesto é um botão em “Configurações” que salva a preferência e avisa
“aplicado ao reabrir o aplicativo”. Prometer troca ao vivo aqui seria inventar
trabalho que a arquitetura atual não suporta.

---

## 4.9 Resumo das mudanças de UI por arquivo

| Arquivo | Mudança | Linhas de referência |
|---|---|---|
| `UnifiedInterface.cs` | remover 5 `Visible=false`; `Rows(222,-1)`; badges; `ApplyHomeFilter`; `AdvancedCategory` por mapa; `RunScenario` | 37, 54, 55, 58, 59, 66, 73-87, 139 |
| `ProductInterface.cs` | remover 4 `Visible=false`; ligar `cleanupAge`; Painel completo; botão de TXT do registro | 56, 58, 92, 93, 97, 118 |
| `Interface.cs` | navegação por seções; `NavIcon`; `ActionButton.Glyph`; `Theme.Apply`; `Confirm` via overlay | 12, 47, 86, 103 |
| `Badges.cs` | **novo** | — |
| `Toast.cs` | **novo** | — |
| `LukeOptimizer.cs` | `App.ById`, `App.ResetIndex` | 106-118 |
| `OptionInfo.cs` | cache, `Evidence`, `InScenarios` | todo o arquivo |
| `SystemTelemetry.cs` | `ActivePowerPlan`, `GameModeState`, `BackgroundSummary` | fim do arquivo |
| `compilacao.rsp` | `source\Badges.cs`, `source\Toast.cs`, `source\Scenarios.cs` | — |
