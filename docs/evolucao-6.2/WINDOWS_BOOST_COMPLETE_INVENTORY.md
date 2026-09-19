# INVENTÁRIO COMPLETO — Windows Boost e Windows Boost Essential

Auditoria do **conteúdo** de cada arquivo. O pedido foi explícito em não confiar no
nome, e a auditoria confirmou por quê: vários nomes descrevem algo diferente do que o
arquivo faz.

| Pacote | Arquivo | Comportamento real | Já no LukeOptimizer | Status |
|---|---|---|---|---|
| Essential | `Fazer backup do Windows.bat` | 1 valores de Registro | — | NEW |
| Essential | `Ajustes de energia.reg` | 12 valores de Registro | — | NEW |
| Essential | `Aumentar Prioridade da GPU (DirectX e Renderização).` | 1 valores de Registro — Apesar do nome, grava `NtfsDisableLastAccessUpdate` — não tem relação com GPU. É o MESMO valor do arquivo "Aumentar velocidade ao abrir pastas". | opt-disablentfstimestamp | DUPLICATE |
| Essential | `Aumentar velocidade ao abrir pastas e arquivos.reg` | 1 valores de Registro — Grava `NtfsDisableLastAccessUpdate`, idêntico ao arquivo de "prioridade da GPU". Uma feature só: já existe como `opt-disablentfstimestamp`. | opt-disablentfstimestamp | DUPLICATE |
| Essential | `Desabilitar Prefetch e Superfetch.reg` | 2 valores de Registro | pack-a137e56bbd0e, pack-f06301afb713 | EXISTING |
| Essential | `Desabilitar SmartSceen e Downloads Blocks.reg` | 1 valores de Registro | opt-disablesmartscreen | EXISTING |
| Essential | `Desativar Animações no Sistema.bat` | 1 valores de Registro | — | NEW |
| Essential | `Desativar Bing Search.bat` | 2 valores de Registro | opt-disablecortana | MERGED |
| Essential | `Desativar Cortana.bat` | 1 valores de Registro | opt-disablecortana | EXISTING |
| Essential | `Desativar Game DVR.reg` | 9 valores de Registro | native-capture | MERGED |
| Essential | `Desativar Hibernação.bat` | 1 comandos de energia | — | NOT APPLICABLE |
| Essential | `Desativar Serviço de Relógio do Windows.bat` | 2 operações de serviço | — | NEW |
| Essential | `Desativar Serviços Xbox.reg` | 4 valores de Registro | opt-disablexboxlive | EXISTING |
| Essential | `Desativar Sugestões de Pesquisa.bat` | 1 valores de Registro | — | NEW |
| Essential | `Desativar Transparência do Windows.bat` | 1 valores de Registro | native-transparency | EXISTING |
| Essential | `Desativar VBS (Isolamento de núcleo).bat` | 2 valores de Registro · 1 alterações de inicialização | opt-disablevirtualizationbasedsecurity, opt-userhvci | EXISTING |
| Essential | `Desativar Windows Telemetria (Coleta de dados).bat` | 1 valores de Registro | — | NEW |
| Essential | `Desativar atualizações automaticas.bat` | 1 valores de Registro · 6 operações de serviço | opt-disableautomaticupdates | EXISTING |
| Essential | `Desativar efeitos visuais.bat` | 3 valores de Registro | — | NEW |
| Essential | `Desativar histórico de atividade.bat` | 1 valores de Registro | — | NEW |
| Essential | `Desativar indexação.bat` | 2 operações de serviço | — | NEW |
| Essential | `Desativar o Hyper-V (não usar se você usa maquina vi` | 1 alterações de inicialização | — | NOT APPLICABLE |
| Essential | `Desativar seviços.bat` | 36 operações de serviço | — | NEW |
| Essential | `Habilitar a otimização do sistema de arquivos.reg` | 1 valores de Registro — Apesar do nome genérico, altera `EnableOplocks` do LanmanServer — cache de arquivos compartilhados em rede, não "otimização do sistema de arquivos". | — | NEW |
| Essential | `Aumentar Prioridade de jogos no Sistema.bat` | 246 valores de Registro · 246 prioridades por executável | — | NEW |
| Essential | `Forçar o windows a priorizar tarefas de jogos.reg` | 4 valores de Registro | — | NEW |
| Essential | `Otimizar Foreground.reg` | 4 valores de Registro | — | NEW |
| Essential | `Otimizar BATTLEFIELD (TODOS).bat` | 41 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 18 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar CALL OF DUTY BLACK OPS (TODOS).bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia — Grava prioridade para **FortniteClient-Win64-Shipping.exe** — executável de outro jogo. Erro de cópia; não reproduzido. | native-capture, opt-disableautomaticupdates | BROKEN |
| Essential | `Otimizar CS2.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar FIVE M.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar Fortnite.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 13 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar GTA V.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia — Também grava prioridade para **FortniteClient-Win64-Shipping.exe** em vez de GTA5.exe. Não reproduzido. | native-capture, opt-disableautomaticupdates | BROKEN |
| Essential | `Otimizar Minecraft.bat` | 32 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 9 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar RED DEAD REDEMPTION 2.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 13 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia — Limpa `FortniteGame\Saved\WebCache` — cache de outro jogo. Não reproduzido. | native-capture, opt-disableautomaticupdates | BROKEN |
| Essential | `Otimizar ROBLOX.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar Valorant.bat` | 29 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 6 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Otimizar WARZONE.bat` | 26 valores de Registro · 44 operações de serviço · 15 pacotes de aplicativo · 3 prioridades por executável · 12 exclusões de arquivo · 4 tarefas agendadas · 1 comandos de energia | native-capture, opt-disableautomaticupdates | MERGED |
| Essential | `Arrumar Windows.bat` | 2 comandos de reparo | — | NOT APPLICABLE |
| Essential | `Limpeza Completa PC.bat` | 2 operações de serviço · 4 exclusões de arquivo · 1 comandos de rede | — | NEW |
| Essential | `Ativar Pacote Office (Desative antes o Anti-Vírus).b` | 1 downloads — Mesmo payload de ativação. NÃO integrado. | — | INTENTIONALLY OMITTED |
| Essential | `Ativar Windows (Desative antes o Anti-Vírus).bat` | 1 downloads — Baixa e executa payload remoto de ativação (`get.activated.win`). Não é otimização. NÃO integrado, não executado, sem botão. | — | INTENTIONALLY OMITTED |
| Essential | `Baixar Driver AMD.bat` | 1 downloads | — | NOT APPLICABLE |
| Essential | `Desativar AMD Crash Defender (serviços.bat` | 2 operações de serviço | — | NEW |
| Essential | `Desativar AMD Overlay e Telemetria.reg` | 2 valores de Registro | — | NEW |
| Essential | `Desativar Hardware Accelerated GPU Scheduling.reg` | 1 valores de Registro | set-gaming-hags | EXISTING |
| Essential | `Desativar MPO.reg` | 1 valores de Registro | — | NEW |
| Essential | `Desativar ULPS (Stutter e quedas de clock).reg` | 2 valores de Registro | — | NEW |
| Essential | `Forçar Shader Cache sempre ativo (AMD).reg` | 1 valores de Registro | — | NEW |
| Essential | `Baixar Driver INTEL.bat` | 1 downloads | — | NOT APPLICABLE |
| Essential | `Desativar MPO (Multiplane Overlay).bat` | 1 valores de Registro | — | NEW |
| Essential | `Intel Priority Optimization.bat` | 1 valores de Registro | set-gaming-foreground-priority | EXISTING |
| Essential | `Intel Timer Optimization (bcedit).bat` | 3 alterações de inicialização | — | NOT APPLICABLE |
| Essential | `Baixar Driver NVIDIA.bat` | 1 downloads | — | NOT APPLICABLE |
| Essential | `Desativar HGS.reg` | 1 valores de Registro | set-gaming-hags | EXISTING |
| Essential | `Desativar MPO.reg` | 1 valores de Registro | — | NEW |
| Essential | `Desativar NVIDIA ShadowPlay.reg` | 1 valores de Registro | — | NEW |
| Essential | `Desativar Telemetria NVIDIA.bat` | 4 operações de serviço | — | NEW |
| Essential | `Limpar Shader Cache NVIDIA.bat` | 2 exclusões de arquivo | — | NEW |
| Essential | `Desativar Copilot.bat` | 3 valores de Registro | opt-disablecopilotai | MERGED |
| Essential | `Desativar Cortana.bat` | 1 valores de Registro · 1 pacotes de aplicativo | opt-disablecortana | EXISTING |
| Essential | `Linkedin.bat` | 3 pacotes de aplicativo | — | NEW |
| Essential | `Loja do Windows.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `REMOVA TUDO DE UMA VEZ SÓ.bat` | 5 valores de Registro · 19 pacotes de aplicativo | opt-disablecopilotai, opt-disablecortana | MERGED |
| Essential | `REVERTA OS DEBLOATERS.bat` | 5 valores de Registro · 19 pacotes de aplicativo | opt-disablecopilotai, opt-disablecortana | MERGED |
| Essential | `Vincular celular.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Whatsapp.bat` | 3 pacotes de aplicativo | — | NEW |
| Essential | `Windows 3Dbuilder.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Alarms.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Calculadora.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Calendario.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Camera.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Getstarted.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Groove Música.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Messaging.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Music.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows News.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Officehub.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows OneDrive.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows People.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Photos.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows Xbox.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `Windows maps.bat` | 1 pacotes de aplicativo | — | NEW |
| Essential | `iGust Debloater.bat` | 15 valores de Registro · 55 pacotes de aplicativo · 1 downloads | opt-disablecopilotai, opt-disablecortana | MERGED |
| Windows Boost | `debloater.bat` | 6 valores de Registro · 55 pacotes de aplicativo | opt-disablecopilotai, opt-disablecortana | MERGED |
| Windows Boost | `iGust Windows Boost.bat` | 252 valores de Registro · 37 operações de serviço · 24 pacotes de aplicativo · 141 prioridades por executável · 4 exclusões de arquivo · 4 comandos de energia · 2 alterações de inicialização · 4 comandos de rede · 2 comandos de reparo · 2 comandos de memória | native-capture, native-gamemode | MERGED |

