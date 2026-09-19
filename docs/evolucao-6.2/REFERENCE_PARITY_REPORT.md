# RELATÓRIO DE PARIDADE — LukeOptimizer × quatro projetos de referência

Gerado por auditoria automática do código-fonte dos quatro projetos entregues, cruzado
com o inventário real do LukeOptimizer. **Este documento é técnico e não aparece como
página do programa.**

## 1. Como a comparação foi feita

A chave de comparação **não é o nome da função**. Cada projeto batiza a mesma coisa de um
jeito — "HAGS", "Hardware Accelerated GPU Scheduling", "GPU Scheduling" — e comparar nomes
produziria duplicação exatamente como você descreveu. A chave é **o que o Windows realmente
guarda**: o par (chave de Registro, nome do valor).

| Projeto | O que foi lido |
|---|---|
| Winhance | `src/Winhance.Core/Features/**/*Optimizations.cs` e `*Customizations.cs` — os modelos declarativos `SettingDefinition`/`RegistrySetting` |
| Win11Debloat | os 93 arquivos `.reg` de `Regfiles/`, mais `Config/Apps.json` e `Config/Features.json` |
| Sophia Script | `src/Sophia_Script_for_Windows_11/Module/Sophia.psm1` (a variante que interessa a este produto) |
| optimizerNXT | os 28 arquivos de `yaml/` |

## 2. LICENÇAS — o achado que muda o que pode ser feito

Você pediu (itens 177–179) que a licença de cada projeto fosse verificada antes de qualquer
cópia. Verifiquei, e **duas delas proíbem o reaproveitamento de código neste produto**:

| Projeto | Licença | Consequência | O que foi feito |
|---|---|---|---|
| **Winhance** | PolyForm Shield 1.0.0 | Fonte-disponível, NÃO open source. Proíbe uso concorrente e é **incompatível com a GPLv3** sob a qual este pacote já é distribuído. | Reimplementação independente. Nenhuma linha de código aproveitada. |
| **Win11Debloat** | MIT | Permite reuso com aviso de copyright; compatível com GPLv3. | Comportamento conferido; implementação própria + aviso preservado em `licenses/`. |
| **Sophia Script** | MIT | Permite reuso com aviso de copyright; compatível com GPLv3. | Comportamento conferido; implementação própria + aviso preservado em `licenses/`. |
| **optimizerNXT** | GPL-3.0 | Mesma licença que este pacote já declara em `licenses/INTEGRACAO.txt`, portanto compatível. | Reimplementação própria mesmo assim, para não ampliar a superfície copyleft. Aviso preservado. |

**O ponto de partida importa:** este pacote **já se distribui sob GPLv3**. Está escrito
em `licenses/INTEGRACAO.txt`, por causa das opções adaptadas do Optimizer (hellzerg,
GPLv3) numa fase anterior. Toda a análise abaixo parte disso.

**Winhance — PolyForm Shield 1.0.0.** É licença *fonte-disponível*, não open source. Ela
concede uso "exceto para competir com o licenciante", e um otimizador de Windows
concorre diretamente com o Winhance. Pior: uma licença com restrição de uso é
**incompatível com a GPLv3** — a GPL não admite restrições adicionais. Então não é
questão de preferência: código do Winhance **não pode** entrar neste pacote. Ele foi
usado apenas como **especificação de comportamento**.

**optimizerNXT — GPL-3.0.** Mesma licença que este pacote já declara, portanto
compatível: código dele *poderia* ser incorporado preservando os avisos. Ainda assim
reimplementei, por dois motivos — não ampliar a superfície copyleft do produto e não
herdar decisões que eu discordo (o projeto tem funções para desligar Defender,
SmartScreen e Restauração do Sistema; nenhuma delas entrou aqui).

Nos dois casos, o que foi aproveitado é **fato técnico** — qual chave do Registro o Windows
lê para um determinado comportamento. Isso não é expressão criativa protegida; é a API do
sistema. Cada ajuste implementado aponta para a documentação da Microsoft, não para o
projeto de referência.

## 3. Números

| | |
|---|---:|
| Ajustes de configuração extraídos das quatro referências | **1655** |
| Pares (chave, valor) **distintos** | **935** |
| Funções identificadas nas quatro referências | **588** |
| LukeOptimizer hoje: itens no catálogo | 673 |
| LukeOptimizer hoje: funções executáveis | 127 |
| LukeOptimizer hoje: pares (chave, valor) distintos | 201 |

### A duplicação que você previu é real

| O mesmo ajuste aparece em… | Quantos ajustes |
|---|---:|
| 4 projetos | 20 |
| 3 projetos | 46 |
| 2 projetos | 109 |
| 1 projeto | 760 |

**175 ajustes estão em mais de um projeto.** Vinte deles estão nos quatro ao mesmo tempo —
entre eles `GameDVR_Enabled`, `AllowTelemetry`, `AdvertisingInfo\Enabled`, `HideFileExt`,
`TaskbarAl` e `ShowTaskViewButton`. Cada um virou **uma única feature** no LukeOptimizer,
com **um único id**, e há teste que falha se aparecer um segundo id escrevendo o mesmo par.

### Situação por projeto (nível de função)

| Projeto | Funções | Já existiam | Parciais | Ausentes |
|---|---:|---:|---:|---:|
| Sophia Script | 104 | 14 | 52 | 38 |
| Win11Debloat | 91 | 18 | 45 | 28 |
| Winhance | 366 | 63 | 123 | 180 |
| optimizerNXT | 27 | 6 | 14 | 7 |
| **Total** | **588** | **101** | **234** | **253** |

> **Leia "PARCIAL" com cuidado.** Significa que o LukeOptimizer já mexe na mesma chave do
> Registro, mas não em todos os valores que aquela função da referência escreve. Não
> significa meia funcionalidade entregue.

> **E leia "ausente" com cuidado também.** O extrator conta *valores de Registro tocados*,
> e no Sophia Script ele captura também os caminhos que a função apenas **lê** para
> descobrir o estado atual. Nem todo valor ausente é uma função que falta: parte é leitura
> de estado e parte é metadado (as entradas `NameSpace\...` do Win11Debloat, por exemplo,
> são a definição do *checkbox* do Explorer, não um ajuste).

## 4. O que foi implementado nesta fase

39 configurações novas, em 4 painéis compactos, com **51 entradas de catálogo** (uma
por opção de caixa de escolha). Todas passam pelo motor transacional que já existia:
backup antes de escrever, verificação depois, registro no Histórico e Desfazer.

| Painel | Grupos | Configurações |
|---|---|---:|
| Personalizar Windows | Barra de tarefas · Menu Iniciar · Explorador de Arquivos · Notificações · Mouse e teclado | 26 |
| Privacidade | Windows | 5 |
| Jogos | Principal · Experimental | 5 |
| Windows Update | Como atualizar | 2 |

**Critério de entrada, e ele foi restritivo:** só entrou ajuste confirmado em **mais de
uma referência independente**. Quando quatro projetos que não se falam escrevem o mesmo
valor da mesma chave, o fato está estabelecido. Ajuste que aparecia numa referência só
e que eu não consegui confirmar na documentação da Microsoft **ficou de fora** — está
como ausente na tabela abaixo, e não como função entregue.

### Duplicados encontrados DENTRO do próprio LukeOptimizer

O teste de deduplicação varre o catálogo inteiro e falha se dois ids gravarem o mesmo
valor da mesma chave. Na primeira execução ele achou dois — que já existiam antes desta
fase:

| Valor | Ids em conflito | Resolução |
|---|---|---|
| `Mozilla\Firefox · DisableTelemetry` | `v5-native-firefox-telemetry` e `opt-disablefirefoxtelemetry` | Ficou o **v5**, que é a implementação mais completa: só aplica se o Firefox 60+ estiver realmente instalado. O outro perdeu esse valor e ficou com o que é dele (`DisableDefaultBrowserAgent`). |
| `Windows\Explorer · DisableSearchBoxSuggestions` | `opt-disablestartmenuads` (pacote de 22 valores) e `opt-usersearchsuggestions` | Ficou o **dedicado**. O pacote perdeu esse valor — o que também atende ao seu item 44: pacote que muda 22 coisas de uma vez não deve ser dono de uma configuração que tem linha própria. |

### O que foi deliberadamente reclassificado

`HwSchMode`, `Win32PrioritySeparation` e `SystemResponsiveness` estavam numa lista de
"placebos conhecidos" e eram **proibidos** de virar opção executável. Seus itens 20, 21
e 125 pedem os três. Resolvi assim, e está registrado em teste:

- **Agendamento de GPU (`HwSchMode`) não é placebo.** É recurso documentado da Microsoft,
  com interruptor próprio em Configurações do Windows. Chamá-lo de placebo era exagero
  meu. Agora é opção **avançada**, com o texto dizendo que o efeito varia por driver.
- **`Win32PrioritySeparation` e `SystemResponsiveness` são reais, o *ganho* é que não é
  comprovado.** Isso é a definição de **experimental**, não de placebo — e é como eles
  entraram, com 🧪 no nome e "EXPERIMENTAL" escrito no texto que a pessoa lê.
- Os placebos que continuam **proibidos de executar**: `LargeSystemCache`,
  `DisablePagingExecutive`, `IoPageLockLimit`, `TcpAckFrequency`, `TCPNoDelay`,
  `SvcHostSplitThresholdInKB`.

Três testes garantem o que protege de verdade: nenhum deles vem marcado, nenhum entra
em perfil automático, e o experimental diz que é experimental.

### O que NÃO entrou, e por quê

| Pedido | Situação |
|---|---|
| Gerenciador de AppX / remoção de apps (itens 39–42) | O LukeOptimizer **já tem** remoção de apps com 141 definições e varredura de estado. O que falta é o conceito de *aplicativos do fabricante* com detecção de OEM. **MISSING** |
| Central de IA, Recall, Click to Do (itens 49–55, 123) | Copilot já tem dono (`opt-disablecopilotai`). Recall e Click to Do dependem de build e hardware específicos que não consigo detectar sem uma máquina com eles. **MISSING** |
| Energia completa (itens 33, 168) | Existe plano de energia; faltam os controles finos AC/DC, sono, USB, PCI Express. **MISSING** |
| Recursos do Windows / WSL / Sandbox / Hyper-V (itens 76, 77, 101) | Exige DISM e enumeração de features. **MISSING** |
| Telemetria por aplicativo — Chrome, Edge, Firefox, Office, Visual Studio, NVIDIA (itens 140–146) | Firefox e Edge já têm dono. Chrome, Office, Visual Studio e NVIDIA: **MISSING** |
| VBS, Defender, SmartScreen, System Restore (itens 149, 159–161) | Desligar proteção não vira botão aqui. Mostrar **estado** é o caminho, e ainda não existe. **INTENTIONALLY OMITTED** para o desligamento; **MISSING** para a leitura de estado |
| Criador de mídia do Windows (item 38) | Fase posterior, como você mesmo colocou. **INTENTIONALLY OMITTED** |
| Barra de pendências global, Ctrl+K, assistente de primeira execução | A barra de pendências existe **por painel**; global ainda não. **MISSING** |

## 5. Tabela completa — função por função

Colunas: origem · função encontrada · ajustes que ela escreve · quantos o LukeOptimizer já
tinha · status · id da feature correspondente.


### Sophia Script

| Função encontrada | Ajustes | Já existiam | Status | Feature LukeOptimizer |
|---|---:|---:|---|---|
| ActiveHours | 4 | 1 | PARTIAL | opt-disableautomaticupdates |
| AdminApprovalMode | 11 | 0 | PARTIAL | — |
| AdvertisingID | 5 | 1 | PARTIAL | opt-splitdisabledbygrouppolicy |
| AeroShaking | 4 | 2 | PARTIAL | opt-splitdisallowshaking |
| AppColorMode | 2 | 2 | IMPLEMENTED | v5-native-theme-dark, v5-native-theme-light |
| AppsSilentInstalling | 3 | 2 | PARTIAL | opt-disablestartmenuads |
| AppsSmartScreen | 2 | 2 | IMPLEMENTED | opt-disablesmartscreen |
| Autoplay | 4 | 0 | PARTIAL | — |
| BSoDStopError | 2 | 0 | MISSING | — |
| BingSearch | 4 | 2 | PARTIAL | opt-disablestartmenuads, opt-usersearchsuggestions |
| ButtonInstallClicked | 1 | 0 | MISSING | — |
| CABInstallContext | 1 | 0 | MISSING | — |
| CapsLock | 3 | 0 | MISSING | — |
| CheckBoxes | 2 | 0 | PARTIAL | — |
| CleanupTask | 13 | 1 | PARTIAL | pack-8f317bf49ced |
| ClockInNotificationCenter | 2 | 0 | PARTIAL | — |
| ControlPanelView | 9 | 0 | PARTIAL | — |
| CreateRestorePoint | 3 | 0 | MISSING | — |
| DNSoverHTTPS | 12 | 0 | MISSING | — |
| DefaultTerminalApp | 7 | 0 | MISSING | — |
| DeliveryOptimization | 1 | 1 | IMPLEMENTED | v5-native-delivery-no-peers |
| DiagnosticDataLevel | 13 | 0 | PARTIAL | — |
| ErrorReporting | 3 | 0 | MISSING | — |
| EventViewerCustomView | 12 | 0 | MISSING | — |
| Export-Associations | 7 | 0 | MISSING | — |
| F1HelpPage | 4 | 0 | MISSING | — |
| FeedbackFrequency | 6 | 1 | PARTIAL | opt-splitdonotshowfeedbacknotifications |
| FileExplorerCompactMode | 2 | 2 | IMPLEMENTED | v5-native-explorer-compact |
| FileExtensions | 2 | 2 | IMPLEMENTED | v5-native-file-extensions |
| FileTransferDialog | 4 | 0 | MISSING | — |
| FirstLogonAnimation | 3 | 0 | PARTIAL | — |
| FolderGroupBy | 11 | 0 | MISSING | — |
| GPUScheduling | 2 | 0 | MISSING | — |
| HiddenItems | 2 | 2 | IMPLEMENTED | opt-splithidden |
| InputMethod | 1 | 0 | MISSING | — |
| Install-Cursors | 63 | 0 | MISSING | — |
| Install-DotNetRuntimes | 1 | 0 | MISSING | — |
| Install-VCRedist | 1 | 0 | MISSING | — |
| JPEGWallpapersQuality | 2 | 0 | MISSING | — |
| LanguageListAccess | 2 | 0 | MISSING | — |
| LocalSecurityAuthority | 6 | 0 | PARTIAL | — |
| MergeConflicts | 2 | 0 | PARTIAL | — |
| MostUsedStartApps | 6 | 0 | PARTIAL | — |
| NavigationPaneExpand | 2 | 0 | PARTIAL | — |
| NetworkAdaptersSavePower | 5 | 0 | PARTIAL | — |
| OneDrive | 9 | 0 | MISSING | — |
| OneDriveFileExplorerAd | 2 | 0 | PARTIAL | — |
| OpenFileExplorerTo | 2 | 2 | IMPLEMENTED | opt-disablequickaccesshistory |
| OpenWindowsTerminalAdminContext | 2 | 0 | MISSING | — |
| PowerPlan | 1 | 0 | MISSING | — |
| PreventEdgeShortcutCreation | 7 | 0 | MISSING | — |
| PrtScnSnippingTool | 2 | 0 | MISSING | — |
| QuickAccessFrequentFolders | 2 | 2 | IMPLEMENTED | opt-disablequickaccesshistory |
| QuickAccessRecentFiles | 4 | 2 | PARTIAL | opt-disablequickaccesshistory |
| RecentlyAddedStartApps | 4 | 0 | PARTIAL | — |
| RecommendedTroubleshooting | 12 | 0 | PARTIAL | — |
| RecycleBinDeleteConfirmation | 5 | 0 | PARTIAL | — |
| RegistryBackup | 3 | 0 | MISSING | — |
| RestartDeviceAfterUpdate | 3 | 0 | PARTIAL | — |
| RestartNotification | 3 | 0 | PARTIAL | — |
| RestorePreviousFolders | 2 | 0 | PARTIAL | — |
| SaveRestartableApps | 2 | 0 | MISSING | — |
| SaveZoneInformation | 5 | 3 | PARTIAL | opt-disablesmartscreen |
| ScanRegistryPolicies | 4 | 0 | MISSING | — |
| SearchHighlights | 5 | 2 | PARTIAL | opt-disablecortana, opt-disablestartmenuads |
| SecondsInSystemClock | 2 | 0 | PARTIAL | — |
| Set-Association | 12 | 0 | MISSING | — |
| Set-UserShellFolderLocation | 3 | 0 | MISSING | — |
| SettingsSuggestedContent | 6 | 6 | IMPLEMENTED | opt-disablestartmenuads |
| ShortcutsSuffix | 5 | 0 | PARTIAL | — |
| SigninInfo | 5 | 0 | PARTIAL | — |
| SnapAssist | 3 | 0 | PARTIAL | — |
| SoftwareDistributionTask | 8 | 1 | PARTIAL | pack-8f317bf49ced |
| StartAccountNotifications | 2 | 0 | PARTIAL | — |
| StartAppsView | 5 | 0 | PARTIAL | — |
| StartRecommendationsTips | 2 | 0 | PARTIAL | — |
| StartRecommendedSection | 20 | 0 | PARTIAL | — |
| StickyShift | 2 | 0 | MISSING | — |
| StorageSense | 9 | 0 | MISSING | — |
| TailoredExperiences | 3 | 0 | PARTIAL | — |
| TaskViewButton | 4 | 0 | PARTIAL | — |
| TaskbarAlignment | 2 | 2 | IMPLEMENTED | opt-aligntaskbartoleft |
| TaskbarCombine | 5 | 0 | PARTIAL | — |
| TaskbarEndTask | 4 | 0 | MISSING | — |
| TaskbarSearch | 6 | 4 | PARTIAL | opt-hidetaskbarsearch |
| TaskbarWidgets | 4 | 1 | PARTIAL | v5-native-widgets-off |
| TempTask | 8 | 1 | PARTIAL | pack-8f317bf49ced |
| ThisPC | 4 | 0 | MISSING | — |
| ThumbnailCacheRemoval | 4 | 0 | MISSING | — |
| UnpinAllStartTiles | 7 | 0 | PARTIAL | — |
| UpdateMicrosoftProducts | 3 | 0 | PARTIAL | — |
| UseStoreOpenWith | 5 | 0 | PARTIAL | — |
| WhatsNewInWindows | 4 | 2 | PARTIAL | opt-disablestartmenuads |
| Win32LongPathsSupport | 2 | 2 | IMPLEMENTED | opt-enablelongpaths |
| WinPrtScrFolder | 5 | 0 | MISSING | — |
| WindowsAI | 5 | 0 | PARTIAL | — |
| WindowsColorMode | 2 | 2 | IMPLEMENTED | v5-native-theme-dark, v5-native-theme-light |
| WindowsLatestUpdate | 3 | 0 | PARTIAL | — |
| WindowsManageDefaultPrinter | 2 | 0 | MISSING | — |
| WindowsSandbox | 1 | 0 | MISSING | — |
| WindowsTips | 3 | 2 | PARTIAL | opt-disablestartmenuads |
| WindowsWelcomeExperience | 2 | 2 | IMPLEMENTED | opt-disablestartmenuads |
| XboxGameBar | 4 | 4 | IMPLEMENTED | native-capture |
| XboxGameTips | 2 | 0 | PARTIAL | — |

### Win11Debloat

| Função encontrada | Ajustes | Já existiam | Status | Feature LukeOptimizer |
|---|---:|---:|---|---|
| Add_All_Folders_Under_This_PC | 13 | 0 | MISSING | — |
| Align_Taskbar_Left | 1 | 1 | IMPLEMENTED | opt-aligntaskbartoleft |
| Combine_MMTaskbar_Always | 1 | 0 | PARTIAL | — |
| Combine_MMTaskbar_Never | 1 | 0 | PARTIAL | — |
| Combine_MMTaskbar_When_Full | 1 | 0 | PARTIAL | — |
| Combine_Taskbar_Always | 1 | 0 | PARTIAL | — |
| Combine_Taskbar_Never | 1 | 0 | PARTIAL | — |
| Combine_Taskbar_When_Full | 1 | 0 | PARTIAL | — |
| Disable_AI_Recall | 4 | 2 | PARTIAL | opt-disablecopilotai |
| Disable_AI_Service_Auto_Start | 1 | 0 | MISSING | — |
| Disable_Animations | 1 | 0 | PARTIAL | — |
| Disable_Bing_Cortana_In_Search | 3 | 2 | PARTIAL | opt-disablecortana, opt-disablestartmenuads |
| Disable_Bitlocker_Auto_Encryption | 1 | 0 | MISSING | — |
| Disable_Brave_Bloat | 6 | 0 | MISSING | — |
| Disable_Chat_Taskbar | 2 | 2 | IMPLEMENTED | opt-disablechat |
| Disable_Click_to_Do | 2 | 0 | PARTIAL | — |
| Disable_Copilot | 3 | 3 | IMPLEMENTED | opt-disablecopilotai |
| Disable_DVR | 3 | 2 | PARTIAL | native-capture |
| Disable_Delivery_Optimization | 1 | 0 | MISSING | — |
| Disable_Desktop_Spotlight | 2 | 0 | PARTIAL | — |
| Disable_Device_Auto_App_Download | 1 | 0 | MISSING | — |
| Disable_Edge_AI_Features | 8 | 2 | PARTIAL | opt-disablecopilotai, opt-disableedgediscoverbar |
| Disable_Edge_Ads_And_Suggestions | 13 | 3 | PARTIAL | opt-disablecopilotai, opt-disableedgetelemetry |
| Disable_Enhance_Pointer_Precision | 3 | 0 | MISSING | — |
| Disable_Fast_Startup | 1 | 1 | IMPLEMENTED | pack-ad17e5deb1d8 |
| Disable_Find_My_Device | 1 | 0 | MISSING | — |
| Disable_Game_Bar_Integration | 9 | 0 | PARTIAL | — |
| Disable_Give_access_to_context_menu | 6 | 0 | MISSING | — |
| Disable_Include_in_library_from_context_menu | 2 | 0 | MISSING | — |
| Disable_Location_Services | 1 | 0 | MISSING | — |
| Disable_Lockscreen_Tips | 2 | 1 | PARTIAL | opt-disablestartmenuads |
| Disable_Modern_Standby_Networking | 2 | 0 | MISSING | — |
| Disable_Notepad_AI_Features | 1 | 0 | MISSING | — |
| Disable_Notifications | 1 | 0 | MISSING | — |
| Disable_Paint_AI_Features | 5 | 0 | MISSING | — |
| Disable_Phone_Link_In_Start | 1 | 0 | MISSING | — |
| Disable_Search_Highlights | 1 | 0 | PARTIAL | — |
| Disable_Search_History | 1 | 1 | IMPLEMENTED | opt-disablecortana |
| Disable_Settings_365_Ads | 1 | 0 | PARTIAL | — |
| Disable_Settings_Home | 1 | 0 | PARTIAL | — |
| Disable_Share_Drag_Tray | 1 | 0 | MISSING | — |
| Disable_Share_from_context_menu | 1 | 0 | MISSING | — |
| Disable_Show_More_Options_Context_Menu | 1 | 0 | PARTIAL | — |
| Disable_Snap_Assist | 1 | 0 | PARTIAL | — |
| Disable_Snap_Layouts | 2 | 1 | PARTIAL | opt-disablesnapassist |
| Disable_Start_All_Apps | 1 | 0 | PARTIAL | — |
| Disable_Start_Recommended | 1 | 0 | PARTIAL | — |
| Disable_Sticky_Keys_Shortcut | 1 | 0 | MISSING | — |
| Disable_Storage_Sense | 1 | 0 | MISSING | — |
| Disable_Telemetry | 15 | 4 | PARTIAL | opt-disableedgetelemetry, pack-0b9181ab23ef |
| Disable_Transparency | 1 | 1 | IMPLEMENTED | native-transparency |
| Disable_Update_ASAP | 1 | 0 | MISSING | — |
| Disable_Window_Snapping | 1 | 0 | PARTIAL | — |
| Disable_Windows_Suggestions | 18 | 12 | PARTIAL | opt-disablestartmenuads |
| Enable_Dark_Mode | 2 | 2 | IMPLEMENTED | v5-native-theme-dark, v5-native-theme-light |
| Enable_Desktop_Spotlight | 2 | 0 | PARTIAL | — |
| Enable_End_Task | 1 | 0 | MISSING | — |
| Enable_Last_Active_Click | 1 | 0 | PARTIAL | — |
| Hide_3D_Objects_Folder | 2 | 0 | MISSING | — |
| Hide_Desktop_Spotlight_Icon | 2 | 0 | PARTIAL | — |
| Hide_Drive_Letters | 1 | 0 | PARTIAL | — |
| Hide_Gallery_from_Explorer | 10 | 0 | MISSING | — |
| Hide_Home_from_Explorer | 11 | 0 | MISSING | — |
| Hide_Music_Folder | 2 | 0 | MISSING | — |
| Hide_Onedrive_Folder | 10 | 0 | MISSING | — |
| Hide_Search_Taskbar | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| Hide_Tabs_In_Alt_Tab | 1 | 0 | PARTIAL | — |
| Hide_Taskview_Taskbar | 1 | 0 | PARTIAL | — |
| Hide_duplicate_removable_drives_from_navigation_pane_of_File_Exp | 1 | 0 | MISSING | — |
| Launch_File_Explorer_To_Downloads | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| Launch_File_Explorer_To_Home | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| Launch_File_Explorer_To_OneDrive | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| Launch_File_Explorer_To_This_PC | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| MMTaskbarMode_Active | 1 | 0 | PARTIAL | — |
| MMTaskbarMode_All | 1 | 0 | PARTIAL | — |
| MMTaskbarMode_Main_Active | 1 | 0 | PARTIAL | — |
| Prevent_Auto_Reboot | 1 | 1 | IMPLEMENTED | opt-disableautomaticupdates |
| Show_20_Tabs_In_Alt_Tab | 1 | 0 | PARTIAL | — |
| Show_3_Tabs_In_Alt_Tab | 1 | 0 | PARTIAL | — |
| Show_5_Tabs_In_Alt_Tab | 1 | 0 | PARTIAL | — |
| Show_Drive_Letters_First | 1 | 0 | PARTIAL | — |
| Show_Drive_Letters_Last | 1 | 0 | PARTIAL | — |
| Show_Extensions_For_Known_File_Types | 1 | 1 | IMPLEMENTED | v5-native-file-extensions |
| Show_Hidden_Folders | 1 | 1 | IMPLEMENTED | opt-splithidden |
| Show_Network_Drive_Letters_First | 1 | 0 | PARTIAL | — |
| Show_Search_Box | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| Show_Search_Icon | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| Show_Search_Icon_And_Label | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| Start_AllApps_Category | 2 | 0 | PARTIAL | — |
| Start_AllApps_Grid | 2 | 0 | PARTIAL | — |
| Start_AllApps_List | 2 | 0 | PARTIAL | — |

### Winhance

| Função encontrada | Ajustes | Já existiam | Status | Feature LukeOptimizer |
|---|---:|---:|---|---|
| StartMenuCustomizations | 2 | 0 | PARTIAL | — |
| TaskbarCustomizations | 1 | 0 | MISSING | — |
| accessibility-filterkeys-hotkey | 1 | 0 | MISSING | — |
| accessibility-highcontrast-hotkey | 1 | 0 | MISSING | — |
| accessibility-mousekeys-hotkey | 1 | 0 | MISSING | — |
| accessibility-stickykeys-hotkey | 1 | 0 | MISSING | — |
| accessibility-togglekeys-hotkey | 1 | 0 | MISSING | — |
| combo-box-animation | 1 | 0 | PARTIAL | — |
| devices-default-printer-management | 1 | 0 | MISSING | — |
| devices-dynamic-lighting-ambient | 1 | 0 | MISSING | — |
| devices-dynamic-lighting-foreground-app | 1 | 0 | MISSING | — |
| drag-full-windows | 1 | 0 | PARTIAL | — |
| drop-shadows | 1 | 1 | IMPLEMENTED | pack-6dc6c301e7b8 |
| enable-peek | 1 | 1 | IMPLEMENTED | pack-d1b3e7ae9f70 |
| explorer-autoplay | 2 | 0 | PARTIAL | — |
| explorer-context-menu-chkdsk | 1 | 0 | MISSING | — |
| explorer-context-menu-compress-to | 1 | 0 | MISSING | — |
| explorer-context-menu-dism | 5 | 0 | MISSING | — |
| explorer-context-menu-ps1-edit-run | 1 | 0 | MISSING | — |
| explorer-context-menu-sfc | 1 | 0 | MISSING | — |
| explorer-context-menu-toggle-extensions | 1 | 0 | MISSING | — |
| explorer-context-menu-windows-terminal | 1 | 0 | PARTIAL | — |
| explorer-customization-3d-objects | 2 | 0 | MISSING | — |
| explorer-customization-browse-folders | 1 | 0 | MISSING | — |
| explorer-customization-checkbox-select | 1 | 0 | PARTIAL | — |
| explorer-customization-click-items | 2 | 0 | PARTIAL | — |
| explorer-customization-compressed-color | 1 | 0 | PARTIAL | — |
| explorer-customization-context-menu | 1 | 1 | IMPLEMENTED | opt-disableshowmoreoptions |
| explorer-customization-currency-decimal | 1 | 0 | MISSING | — |
| explorer-customization-desktop-icon-control-panel | 2 | 0 | MISSING | — |
| explorer-customization-desktop-icon-network | 2 | 0 | MISSING | — |
| explorer-customization-desktop-icon-recycle-bin | 2 | 0 | MISSING | — |
| explorer-customization-desktop-icon-this-pc | 2 | 0 | MISSING | — |
| explorer-customization-desktop-icon-users-files | 2 | 0 | MISSING | — |
| explorer-customization-disable-sync-provider-notifications | 1 | 0 | PARTIAL | — |
| explorer-customization-duplicate-removable-drives | 2 | 0 | MISSING | — |
| explorer-customization-first-day-of-week | 1 | 0 | MISSING | — |
| explorer-customization-folder-tips | 1 | 0 | PARTIAL | — |
| explorer-customization-full-path | 1 | 0 | MISSING | — |
| explorer-customization-gallery | 3 | 0 | MISSING | — |
| explorer-customization-hide-empty-drives | 1 | 0 | PARTIAL | — |
| explorer-customization-hide-merge-conflicts | 1 | 0 | PARTIAL | — |
| explorer-customization-hide-protected-files | 1 | 0 | PARTIAL | — |
| explorer-customization-home-folder | 3 | 0 | MISSING | — |
| explorer-customization-icon-cache-size | 1 | 0 | PARTIAL | — |
| explorer-customization-icon-thumbnails | 1 | 0 | PARTIAL | — |
| explorer-customization-item-space | 1 | 1 | IMPLEMENTED | v5-native-explorer-compact |
| explorer-customization-launch-to | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| explorer-customization-legacy-notepad | 2 | 0 | MISSING | — |
| explorer-customization-list-separator | 1 | 0 | MISSING | — |
| explorer-customization-measurement-system | 1 | 0 | MISSING | — |
| explorer-customization-nav-expand-current | 1 | 0 | PARTIAL | — |
| explorer-customization-nav-saf-desktop | 2 | 0 | MISSING | — |
| explorer-customization-nav-saf-documents | 2 | 0 | MISSING | — |
| explorer-customization-nav-saf-downloads | 2 | 0 | MISSING | — |
| explorer-customization-nav-saf-music | 2 | 0 | MISSING | — |
| explorer-customization-nav-saf-pictures | 2 | 0 | MISSING | — |
| explorer-customization-nav-saf-videos | 2 | 0 | MISSING | — |
| explorer-customization-nav-show-all-folders | 1 | 0 | PARTIAL | — |
| explorer-customization-nav-show-libraries | 3 | 0 | MISSING | — |
| explorer-customization-netplwiz-auto-login | 1 | 0 | MISSING | — |
| explorer-customization-number-decimal | 1 | 0 | MISSING | — |
| explorer-customization-persist-browsers | 1 | 0 | PARTIAL | — |
| explorer-customization-popup-descriptions | 1 | 0 | PARTIAL | — |
| explorer-customization-preview-handlers | 1 | 0 | PARTIAL | — |
| explorer-customization-separate-process | 1 | 0 | PARTIAL | — |
| explorer-customization-sharing-wizard | 1 | 0 | PARTIAL | — |
| explorer-customization-short-date | 1 | 0 | MISSING | — |
| explorer-customization-shortcut-arrow | 1 | 0 | MISSING | — |
| explorer-customization-shortcut-suffix | 1 | 0 | PARTIAL | — |
| explorer-customization-show-availability-status | 1 | 0 | PARTIAL | — |
| explorer-customization-show-drive-letters | 1 | 0 | PARTIAL | — |
| explorer-customization-show-file-ext | 1 | 1 | IMPLEMENTED | v5-native-file-extensions |
| explorer-customization-show-frequent-folders | 1 | 1 | IMPLEMENTED | opt-disablequickaccesshistory |
| explorer-customization-show-hidden-files | 1 | 1 | IMPLEMENTED | opt-splithidden |
| explorer-customization-show-lnk-extension | 1 | 0 | MISSING | — |
| explorer-customization-show-menus | 1 | 0 | PARTIAL | — |
| explorer-customization-show-office-files | 1 | 0 | PARTIAL | — |
| explorer-customization-show-recent-files | 2 | 1 | PARTIAL | opt-disablequickaccesshistory |
| explorer-customization-show-thumbnails | 1 | 0 | PARTIAL | — |
| explorer-customization-status-bar | 1 | 0 | PARTIAL | — |
| explorer-customization-thispc-folder-desktop | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-desktop-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-documents | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-documents-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-downloads | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-downloads-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-music | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-music-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-pictures | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-pictures-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-videos | 2 | 0 | MISSING | — |
| explorer-customization-thispc-folder-videos-win10 | 2 | 0 | MISSING | — |
| explorer-customization-thumbnail-cache-cleanup | 2 | 0 | MISSING | — |
| explorer-customization-typing-behavior | 1 | 0 | PARTIAL | — |
| explorer-enable-photo-viewer | 1 | 0 | MISSING | — |
| explorer-long-file-paths | 1 | 1 | IMPLEMENTED | opt-enablelongpaths |
| explorer-take-ownership | 1 | 0 | MISSING | — |
| fade-menu-items | 1 | 0 | PARTIAL | — |
| fade-tooltip | 1 | 0 | PARTIAL | — |
| font-smoothing | 1 | 0 | PARTIAL | — |
| gaming-ai-fabric-service | 1 | 0 | MISSING | — |
| gaming-auto-color-management | 1 | 0 | MISSING | — |
| gaming-background-apps | 2 | 0 | PARTIAL | — |
| gaming-biometric-service | 1 | 1 | IMPLEMENTED | pack-3c2c1523c0fa |
| gaming-compatibility-assistant-service | 1 | 1 | IMPLEMENTED | opt-disablecompatibilityassistant |
| gaming-connected-devices-platform-service | 2 | 0 | MISSING | — |
| gaming-cpu-priority | 1 | 0 | MISSING | — |
| gaming-directx-auto-hdr | 1 | 0 | MISSING | — |
| gaming-directx-flip-model | 1 | 0 | MISSING | — |
| gaming-directx-vrr-optimizations | 1 | 0 | MISSING | — |
| gaming-disable-all-overlays | 1 | 0 | MISSING | — |
| gaming-disable-mpo | 1 | 0 | PARTIAL | — |
| gaming-disable-mpo-min-fps | 1 | 0 | PARTIAL | — |
| gaming-error-reporting-service | 1 | 1 | IMPLEMENTED | opt-disableerrorreporting |
| gaming-explorer-alt-tab-filter | 1 | 0 | PARTIAL | — |
| gaming-fax-service | 1 | 1 | IMPLEMENTED | opt-disablefaxservice |
| gaming-fullscreen-optimizations | 1 | 0 | PARTIAL | — |
| gaming-game-bar-controller | 1 | 0 | PARTIAL | — |
| gaming-game-bar-tips | 1 | 0 | PARTIAL | — |
| gaming-game-mode | 1 | 1 | IMPLEMENTED | native-gamemode |
| gaming-geolocation-service | 1 | 0 | MISSING | — |
| gaming-gpu-priority | 1 | 0 | MISSING | — |
| gaming-gpu-scheduling | 1 | 0 | MISSING | — |
| gaming-insider-service | 1 | 1 | IMPLEMENTED | opt-disableinsiderservice |
| gaming-maps-broker-service | 1 | 1 | IMPLEMENTED | pack-ecf7f01e174b |
| gaming-memory-integrity | 3 | 1 | PARTIAL | opt-userhvci |
| gaming-mixed-reality-service | 1 | 0 | MISSING | — |
| gaming-mobile-hotspot-service | 1 | 0 | MISSING | — |
| gaming-nagle-algorithm | 2 | 0 | MISSING | — |
| gaming-narrator-hotkey | 1 | 0 | MISSING | — |
| gaming-network-throttling | 1 | 1 | IMPLEMENTED | opt-disablenetworkthrottling |
| gaming-nvidia-sharpening | 1 | 0 | MISSING | — |
| gaming-parental-controls-service | 1 | 0 | MISSING | — |
| gaming-payments-nfc-service | 1 | 0 | MISSING | — |
| gaming-performance-autostart-delay | 1 | 0 | MISSING | — |
| gaming-performance-background-services | 1 | 0 | MISSING | — |
| gaming-performance-desktop-composition | 1 | 0 | PARTIAL | — |
| gaming-performance-explorer-menu-show-delay | 1 | 0 | PARTIAL | — |
| gaming-performance-explorer-mouse-precision | 1 | 0 | MISSING | — |
| gaming-performance-explorer-search | 1 | 0 | MISSING | — |
| gaming-performance-mouse-hover-time | 1 | 0 | MISSING | — |
| gaming-performance-prefetch | 1 | 1 | IMPLEMENTED | pack-f06301afb713 |
| gaming-performance-search-webview2 | 5 | 0 | MISSING | — |
| gaming-performance-svchost-split-threshold | 1 | 0 | MISSING | — |
| gaming-performance-wallpaper-compression | 1 | 0 | PARTIAL | — |
| gaming-phone-service | 1 | 0 | MISSING | — |
| gaming-print-spooler-service | 1 | 1 | IMPLEMENTED | opt-disableprintservice |
| gaming-remote-access-auto | 1 | 0 | MISSING | — |
| gaming-remote-access-manager | 1 | 0 | MISSING | — |
| gaming-remote-desktop-configuration | 1 | 0 | MISSING | — |
| gaming-remote-desktop-port-redirector | 1 | 0 | MISSING | — |
| gaming-remote-desktop-services | 1 | 0 | MISSING | — |
| gaming-retail-demo-service | 1 | 0 | MISSING | — |
| gaming-sensor-data-service | 1 | 0 | MISSING | — |
| gaming-sensor-monitoring-service | 1 | 1 | IMPLEMENTED | opt-disablesensorservices |
| gaming-smart-card-services | 3 | 0 | MISSING | — |
| gaming-sms-router-service | 1 | 0 | MISSING | — |
| gaming-spot-verifier-service | 1 | 0 | MISSING | — |
| gaming-storage-sense | 2 | 0 | MISSING | — |
| gaming-sysmain-service | 1 | 1 | IMPLEMENTED | opt-disablesuperfetch |
| gaming-telemetry-service | 1 | 1 | IMPLEMENTED | pack-42de4cc8102e |
| gaming-telephony-service | 1 | 0 | MISSING | — |
| gaming-touch-keyboard-service | 2 | 0 | MISSING | — |
| gaming-virtualization-based-security | 3 | 1 | PARTIAL | opt-disablevirtualizationbasedsecurity |
| gaming-wallet-service | 1 | 0 | MISSING | — |
| gaming-win32-priority | 1 | 0 | MISSING | — |
| gaming-windows-search-service | 1 | 1 | IMPLEMENTED | opt-disablesearch |
| gaming-wmp-network-service | 1 | 1 | IMPLEMENTED | opt-disablemediaplayersharing |
| gaming-xbox-auth-manager | 1 | 1 | IMPLEMENTED | opt-disablexboxlive |
| gaming-xbox-game-dvr | 3 | 2 | PARTIAL | native-capture |
| gaming-xbox-game-save | 1 | 1 | IMPLEMENTED | opt-disablexboxlive |
| gaming-xbox-networking | 1 | 1 | IMPLEMENTED | opt-disablexboxlive |
| lid-close-action | 1 | 0 | MISSING | — |
| menu-animation | 1 | 0 | PARTIAL | — |
| mouse-shadow | 1 | 0 | PARTIAL | — |
| notifications-app-location-request | 1 | 0 | MISSING | — |
| notifications-capability-access | 1 | 0 | MISSING | — |
| notifications-clock-change | 1 | 0 | PARTIAL | — |
| notifications-critical-toast-above-lock | 1 | 0 | MISSING | — |
| notifications-security-maintenance | 1 | 0 | MISSING | — |
| notifications-show-bell-icon | 1 | 0 | PARTIAL | — |
| notifications-sound | 1 | 0 | MISSING | — |
| notifications-startup-app | 1 | 0 | MISSING | — |
| notifications-system-pane-suggestions | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| notifications-system-setting-engagement | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| notifications-tips-suggestions | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| notifications-toast-above-lock | 2 | 0 | MISSING | — |
| notifications-welcome-experience | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| notifications-windows-security | 5 | 0 | MISSING | — |
| power-button-action | 1 | 0 | MISSING | — |
| power-fast-startup | 1 | 0 | MISSING | — |
| power-hibernation-enable | 1 | 0 | PARTIAL | — |
| power-throttling | 1 | 1 | IMPLEMENTED | pack-b735bf30fb1a |
| privacy-account-info-access | 1 | 0 | MISSING | — |
| privacy-activity-history | 2 | 0 | PARTIAL | — |
| privacy-ads-promotional-master | 1 | 0 | MISSING | — |
| privacy-advertising-id | 4 | 2 | PARTIAL | opt-splitdisabledbygrouppolicy |
| privacy-allow-cortana | 2 | 2 | IMPLEMENTED | opt-disablecortana |
| privacy-app-diagnostic-access | 1 | 0 | MISSING | — |
| privacy-app-launch-tracking | 1 | 0 | PARTIAL | — |
| privacy-block-recall-enablement | 1 | 0 | PARTIAL | — |
| privacy-camera-access | 1 | 0 | MISSING | — |
| privacy-content-delivery-allowed | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-copilot-unavailable | 1 | 0 | MISSING | — |
| privacy-deny-copilot-microphone | 2 | 0 | MISSING | — |
| privacy-deny-generative-ai-access | 2 | 0 | PARTIAL | — |
| privacy-deny-system-ai-models | 3 | 0 | PARTIAL | — |
| privacy-diagnostics | 9 | 0 | PARTIAL | — |
| privacy-disable-agent-connectors | 1 | 0 | PARTIAL | — |
| privacy-disable-agent-workspaces | 1 | 0 | PARTIAL | — |
| privacy-disable-ai-data-analysis | 1 | 1 | IMPLEMENTED | opt-disablecopilotai |
| privacy-disable-bing-chat | 1 | 0 | MISSING | — |
| privacy-disable-click-to-do | 1 | 0 | PARTIAL | — |
| privacy-disable-consumer-ai-content | 1 | 0 | PARTIAL | — |
| privacy-disable-copilot-hardware-key | 1 | 0 | MISSING | — |
| privacy-disable-copilot-nudges | 1 | 0 | PARTIAL | — |
| privacy-disable-copilot-runtime | 1 | 0 | PARTIAL | — |
| privacy-disable-input-insights | 1 | 1 | IMPLEMENTED | opt-disablespellingandtypingfeatures |
| privacy-disable-paint-ai-cocreator | 1 | 0 | MISSING | — |
| privacy-disable-paint-ai-image-creator | 1 | 0 | MISSING | — |
| privacy-disable-paint-generative-erase | 1 | 0 | MISSING | — |
| privacy-disable-paint-generative-fill | 1 | 0 | MISSING | — |
| privacy-disable-paint-remove-background | 1 | 0 | MISSING | — |
| privacy-disable-recall-snapshots | 1 | 0 | PARTIAL | — |
| privacy-disable-remote-agent-connectors | 1 | 0 | PARTIAL | — |
| privacy-disable-settings-agent | 1 | 0 | PARTIAL | — |
| privacy-edge-ai-history-search | 1 | 0 | PARTIAL | — |
| privacy-edge-ai-themes | 1 | 0 | PARTIAL | — |
| privacy-edge-builtin-ai-apis | 1 | 0 | PARTIAL | — |
| privacy-edge-copilot-cdp-page-context | 1 | 0 | PARTIAL | — |
| privacy-edge-copilot-page-context | 1 | 0 | PARTIAL | — |
| privacy-edge-copilot-sidebar | 1 | 1 | IMPLEMENTED | opt-disableedgediscoverbar |
| privacy-edge-devtools-ai | 1 | 0 | PARTIAL | — |
| privacy-edge-entra-copilot | 1 | 0 | PARTIAL | — |
| privacy-edge-inline-compose | 1 | 1 | IMPLEMENTED | opt-disablecopilotai |
| privacy-edge-local-ai-model | 1 | 0 | PARTIAL | — |
| privacy-edge-m365-copilot-icon | 1 | 0 | PARTIAL | — |
| privacy-edge-share-history-copilot | 1 | 0 | PARTIAL | — |
| privacy-excel-copilot | 1 | 0 | MISSING | — |
| privacy-feature-management | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-feedback-frequency | 3 | 2 | PARTIAL | opt-splitdonotshowfeedbacknotifications |
| privacy-improve-inking-typing | 2 | 0 | MISSING | — |
| privacy-inking-typing-dictionary | 4 | 2 | PARTIAL | pack-1b4dfbdca78f, pack-c8023d85a069 |
| privacy-language-list | 1 | 1 | IMPLEMENTED | pack-b8cef6963679 |
| privacy-location-services | 3 | 0 | MISSING | — |
| privacy-lock-screen | 1 | 0 | MISSING | — |
| privacy-lock-screen-overlay | 2 | 1 | PARTIAL | opt-disablestartmenuads |
| privacy-microphone-access | 1 | 0 | MISSING | — |
| privacy-narrator-online-services | 1 | 0 | MISSING | — |
| privacy-narrator-scripting | 1 | 0 | MISSING | — |
| privacy-oem-preinstalled-apps | 1 | 0 | PARTIAL | — |
| privacy-office-ai-training | 1 | 0 | MISSING | — |
| privacy-office-connected-services | 2 | 0 | MISSING | — |
| privacy-office-content-safety-ai | 1 | 0 | MISSING | — |
| privacy-onedrive-auto-backup | 2 | 0 | MISSING | — |
| privacy-onenote-copilot | 3 | 0 | MISSING | — |
| privacy-preinstalled-apps | 1 | 0 | PARTIAL | — |
| privacy-preinstalled-apps-ever | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-rotating-lock-screen | 1 | 0 | PARTIAL | — |
| privacy-search-aad-cloud | 1 | 0 | PARTIAL | — |
| privacy-search-highlights | 1 | 0 | PARTIAL | — |
| privacy-search-history | 1 | 1 | IMPLEMENTED | opt-disablecortana |
| privacy-search-msa-cloud | 1 | 0 | PARTIAL | — |
| privacy-settings-content | 3 | 3 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-settings-notifications | 1 | 0 | MISSING | — |
| privacy-silent-installed-apps | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-soft-landing | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-speech-recognition | 3 | 2 | PARTIAL | opt-splitallowinputpersonalization |
| privacy-subscribed-content | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| privacy-tailored-experiences | 3 | 0 | PARTIAL | — |
| privacy-timeline-suggestions | 1 | 0 | PARTIAL | — |
| privacy-turn-off-copilot | 2 | 2 | IMPLEMENTED | opt-disablecopilotai |
| privacy-word-copilot | 1 | 0 | MISSING | — |
| processor-core-parking-max-cores | 1 | 0 | MISSING | — |
| processor-core-parking-min-cores | 1 | 0 | MISSING | — |
| processor-energy-performance-preference | 1 | 0 | MISSING | — |
| processor-performance-boost-mode | 1 | 0 | MISSING | — |
| processor-performance-decrease-policy | 1 | 0 | MISSING | — |
| processor-performance-decrease-threshold | 1 | 0 | MISSING | — |
| processor-performance-increase-policy | 1 | 0 | MISSING | — |
| processor-performance-increase-threshold | 1 | 0 | MISSING | — |
| security-automatic-maintenance | 1 | 0 | MISSING | — |
| security-bitlocker-auto-encryption | 1 | 0 | MISSING | — |
| security-developer-mode | 1 | 0 | MISSING | — |
| security-error-reporting | 2 | 0 | MISSING | — |
| security-powershell-execution-policy | 2 | 0 | MISSING | — |
| security-remote-assistance | 1 | 0 | MISSING | — |
| security-smart-app-control | 1 | 0 | MISSING | — |
| security-uac-level | 2 | 0 | PARTIAL | — |
| security-wifi-sense | 2 | 0 | MISSING | — |
| security-workplace-join-messages | 2 | 0 | MISSING | — |
| show-thumbnails | 1 | 0 | PARTIAL | — |
| sleep-button-action | 1 | 0 | MISSING | — |
| smooth-scroll-listboxes | 1 | 0 | PARTIAL | — |
| sound-accessibility-activation | 1 | 0 | MISSING | — |
| sound-accessibility-warnings | 1 | 0 | MISSING | — |
| sound-communication-ducking | 1 | 1 | IMPLEMENTED | pack-3b6ba7cc6df5 |
| sound-narrator-audio-ducking | 1 | 0 | MISSING | — |
| sound-startup | 2 | 1 | PARTIAL | pack-8098f03b75de |
| sound-voice-activation | 1 | 0 | MISSING | — |
| sound-voice-activation-last-used | 1 | 0 | MISSING | — |
| start-all-apps-view | 1 | 0 | MISSING | — |
| start-disable-bing-search-results | 3 | 3 | IMPLEMENTED | opt-disablecortana, opt-disablestartmenuads |
| start-menu-layout | 1 | 0 | PARTIAL | — |
| start-menu-recommendations | 1 | 0 | PARTIAL | — |
| start-power-hibernate-option | 1 | 0 | MISSING | — |
| start-power-lock-option | 1 | 0 | MISSING | — |
| start-power-sleep-option | 1 | 0 | MISSING | — |
| start-recommended-section | 4 | 0 | PARTIAL | — |
| start-show-account-notifications | 1 | 0 | PARTIAL | — |
| start-show-frequent-list | 1 | 0 | MISSING | — |
| start-show-recently-added-apps | 1 | 0 | MISSING | — |
| start-show-recommended-files | 1 | 0 | PARTIAL | — |
| start-show-suggestions | 1 | 1 | IMPLEMENTED | opt-disablestartmenuads |
| start-track-progs | 1 | 0 | PARTIAL | — |
| system-cooling-policy | 1 | 0 | MISSING | — |
| taskbar-alignment | 1 | 1 | IMPLEMENTED | opt-aligntaskbartoleft |
| taskbar-animations | 1 | 1 | IMPLEMENTED | pack-3d76b7e8529c |
| taskbar-auto-hide | 1 | 0 | MISSING | — |
| taskbar-badges | 1 | 0 | PARTIAL | — |
| taskbar-button-size | 1 | 0 | PARTIAL | — |
| taskbar-combine-buttons | 1 | 0 | PARTIAL | — |
| taskbar-combine-buttons-other | 1 | 0 | PARTIAL | — |
| taskbar-copilot | 1 | 1 | IMPLEMENTED | opt-disablecopilotai |
| taskbar-copilot-companion | 1 | 0 | PARTIAL | — |
| taskbar-copilot-pwa-pin | 1 | 0 | PARTIAL | — |
| taskbar-end-task | 1 | 0 | MISSING | — |
| taskbar-extended-hover-time | 1 | 0 | PARTIAL | — |
| taskbar-flashing | 1 | 0 | PARTIAL | — |
| taskbar-meet-now | 3 | 2 | PARTIAL | opt-disablechat |
| taskbar-multi-display | 1 | 0 | PARTIAL | — |
| taskbar-multi-display-apps | 1 | 0 | PARTIAL | — |
| taskbar-news-and-interests | 2 | 2 | IMPLEMENTED | v5-native-news-off |
| taskbar-recall-pin | 1 | 0 | PARTIAL | — |
| taskbar-search-box-10 | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| taskbar-search-box-11 | 1 | 1 | IMPLEMENTED | opt-hidetaskbarsearch |
| taskbar-share-window | 1 | 0 | PARTIAL | — |
| taskbar-show-desktop | 1 | 0 | PARTIAL | — |
| taskbar-small | 1 | 0 | PARTIAL | — |
| taskbar-system-tray-icons | 1 | 1 | IMPLEMENTED | opt-showalltrayicons |
| taskbar-task-view | 1 | 0 | PARTIAL | — |
| taskbar-thumbnails | 1 | 0 | PARTIAL | — |
| taskbar-transparent | 1 | 0 | PARTIAL | — |
| taskbar-widgets | 2 | 2 | IMPLEMENTED | v5-native-widgets-off |
| theme-transparency | 1 | 1 | IMPLEMENTED | native-transparency |
| translucent-selection | 1 | 0 | PARTIAL | — |
| ui-effects | 1 | 0 | PARTIAL | — |
| updates-delivery-optimization | 2 | 2 | IMPLEMENTED | v5-native-delivery-no-peers |
| updates-driver-coinstallers | 1 | 0 | MISSING | — |
| updates-driver-controls | 2 | 2 | IMPLEMENTED | opt-excludedrivers |
| updates-latest-updates | 1 | 0 | MISSING | — |
| updates-metered-connection | 1 | 0 | MISSING | — |
| updates-notification-level | 2 | 0 | PARTIAL | — |
| updates-other-products | 1 | 0 | MISSING | — |
| updates-policy-mode | 28 | 4 | PARTIAL | opt-disableautomaticupdates |
| updates-restart-asap | 1 | 0 | MISSING | — |
| updates-restart-notification | 1 | 0 | MISSING | — |
| updates-restart-options | 2 | 2 | IMPLEMENTED | opt-disableautomaticupdates |
| updates-store-auto-download | 2 | 2 | IMPLEMENTED | opt-disablestoreupdates |
| usb-hub-selective-suspend-timeout | 1 | 0 | MISSING | — |
| usb3-link-power-management | 1 | 0 | MISSING | — |
| visual-effects-mode | 1 | 0 | MISSING | — |
| window-animation | 1 | 0 | MISSING | — |
| window-shadows | 1 | 0 | PARTIAL | — |
| windows-pushnotifications | 1 | 0 | MISSING | — |

### optimizerNXT

| Função encontrada | Ajustes | Já existiam | Status | Feature LukeOptimizer |
|---|---:|---:|---|---|
| add-open-cmd | 6 | 0 | MISSING | — |
| add-take-ownership | 4 | 0 | MISSING | — |
| disable-auto-updates | 17 | 6 | PARTIAL | opt-disableautomaticupdates, opt-disablestartmenuads |
| disable-chrome-telemetry | 6 | 3 | PARTIAL | opt-disablechrometelemetry |
| disable-copilot | 18 | 18 | IMPLEMENTED | opt-disablecopilotai, opt-disablecortana |
| disable-edge-telemetry | 19 | 12 | PARTIAL | opt-disablecopilotai, opt-disableedgediscoverbar |
| disable-firefox-telemetry | 2 | 2 | IMPLEMENTED | opt-disablefirefoxtelemetry, v5-native-firefox-telemetry |
| disable-modern-standby | 1 | 1 | IMPLEMENTED | opt-disablemodernstandby |
| disable-office-telemetry | 32 | 0 | MISSING | — |
| disable-smartscreen | 8 | 7 | PARTIAL | opt-disablesmartscreen |
| disable-sticky-keys | 6 | 0 | MISSING | — |
| disable-superfetch | 3 | 2 | PARTIAL | pack-a137e56bbd0e, pack-f06301afb713 |
| disable-system-restore | 2 | 0 | MISSING | — |
| disable-tablet-features | 9 | 9 | IMPLEMENTED | opt-disablespellingandtypingfeatures, opt-disablewindowsink |
| disable-telemetry-services | 17 | 0 | PARTIAL | — |
| disable-tpm-enforce | 7 | 7 | IMPLEMENTED | opt-disabletpmcheck |
| disable-vbs | 1 | 1 | IMPLEMENTED | opt-disablevirtualizationbasedsecurity |
| disable-visual-studio-telemetry | 12 | 8 | PARTIAL | opt-disablevisualstudiotelemetry |
| disable-windows-defender | 25 | 0 | MISSING | — |
| disable-windows-security | 37 | 0 | MISSING | — |
| enable-game-mode | 11 | 4 | PARTIAL | native-capture, native-gamemode |
| enable-performance-tweaks | 43 | 10 | PARTIAL | native-transparency, opt-disablenetworkthrottling |
| enhance-privacy | 57 | 14 | PARTIAL | opt-splitallowinputpersonalization, opt-splitallowlinguisticdatacollection |
| exclude-drivers | 5 | 1 | PARTIAL | opt-excludedrivers |
| simplify-windows10-11 | 41 | 34 | PARTIAL | opt-disablecloudclipboard, opt-disablemypeople |
| simplify-windows11 | 8 | 7 | PARTIAL | opt-aligntaskbartoleft, opt-disablechat |
| uninstall-onedrive | 2 | 1 | PARTIAL | opt-disableonedrive |

