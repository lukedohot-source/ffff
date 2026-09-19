# RELATÓRIO DE PARIDADE — Windows Boost e Windows Boost Essential

| | |
|---|---:|
| Arquivos analisados (com comportamento) | **86** |
| Comportamentos extraídos | **2518** |
| Gravações de Registro | 608 |
| Pares (chave, valor) **distintos** | **183** |
| …presentes nos **dois** pacotes (duplicação interna) | **73** |
| …que **já existiam** no LukeOptimizer | **35** |
| …**ausentes** | **148** |
| Serviços distintos tocados | 31 (15 já no catálogo) |
| Pacotes de aplicativo distintos | 31 |
| Executáveis com prioridade por IFEO | **87** (em 444 gravações) |
| Tarefas agendadas distintas | 4 (em 44 gravações) |

## A duplicação é massiva

| Comportamento | Vezes repetido |
|---|---:|
| Tarefa `Consolidator` | 11 |
| Tarefa `ProgramDataUpdater` | 11 |
| Tarefa `Proxy` | 11 |
| Tarefa `Microsoft-Windows-DiskDiagnosticDataCollector` | 11 |
| Exclusão `/s /f /q "%temp%\*.*" 2>nul` | 13 |
| Exclusão `/s /f /q "%windows%\temp\*.*" 2>nul` | 11 |
| Exclusão `/s /f /q "%windows%\Prefetch\*.exe" 2>nul` | 11 |
| Exclusão `/s /f /q "%windows%\Prefetch\*.dll" 2>nul` | 11 |

**73 dos 183 ajustes de Registro aparecem nos dois pacotes.** Cada um virou **uma**
feature no LukeOptimizer, com um id só. O teste de deduplicação derruba o build se
aparecer um segundo id gravando o mesmo valor.

