# IgnisRig

Hub de instalação, otimização e gerenciamento de componentes essenciais para Windows, pensado para o momento em que um PC é formatado ou comprado novo: drivers, DLLs, runtimes, launchers de jogos e utilitários de diagnóstico, tudo configurado a partir de um pequeno questionário inicial.

## 1. Requisitos

- Windows 10/11 (os instaladores `.exe`/`.msi` do catálogo são específicos para Windows).
- Python 3.9 ou superior.
- Biblioteca `customtkinter`.

## 2. Instalação das dependências (para rodar via `python main.py`)

Dê duplo clique em **`Instalador.bat`** — ele solicita administrador, localiza todas as instalações do Python no computador e instala as dependências automaticamente. Alternativa manual:

```bash
pip install -r requirements.txt
```

## 3. Gerando o executável (.exe) e o instalador (Setup.exe)

Esta parte precisa ser feita **no Windows** — o PyInstaller não compila de forma cruzada (não dá pra gerar um `.exe` do Linux/macOS) e o compilador do Inno Setup também é Windows-only. O processo tem duas etapas:

### Etapa 1 — Compilar o `.exe` (`build_exe.bat`)

Na raiz do projeto, dê duplo clique em **`build_exe.bat`**. Ele vai:
1. Instalar/atualizar o PyInstaller e as dependências do projeto.
2. Compilar `main.py` (com todo o pacote `core/` e `ui/`) em `dist\IgnisRig\IgnisRig.exe`, usando o ícone em `installer\icon.ico`.

O script já inclui a flag `--collect-all customtkinter`, que é **essencial** — sem ela, o `.exe` compila mas trava ao abrir (o customtkinter carrega arquivos de tema `.json` em tempo de execução que o PyInstaller não detecta sozinho).

### Etapa 2 — Gerar o instalador (`installer\build_installer.bat`)

Requer o **Inno Setup** (gratuito): baixe em [jrsoftware.org/isdl.php](https://jrsoftware.org/isdl.php) e instale uma vez.

Depois, entre na pasta `installer\` e rode **`build_installer.bat`**. Ele compila `IgnisRig.iss` e gera **`installer\output\IgnisRig_Setup.exe`** — esse é o arquivo final que você distribui.

### O que o `Setup.exe` faz ao ser executado pelo usuário final

- Instala o IgnisRig em **`%APPDATA%\IgnisRig`** (pasta do próprio usuário — não pede privilégio de administrador, já que `%APPDATA%` já é gravável por ele).
- Cria atalho no Menu Iniciar e, se marcado, na Área de Trabalho.
- Registra um desinstalador padrão do Windows (aparece em "Aplicativos instalados").
- **Não inclui** os instaladores de terceiros que você coloca em `assets/` — o próprio IgnisRig recria essa árvore de pastas vazia na primeira execução (`ensure_directory_structure()`), então o pacote de distribuição continua pequeno. O usuário final povoa `%APPDATA%\IgnisRig\assets\` com os instaladores dele depois.

```
Fluxo completo:
  main.py  --(build_exe.bat)-->  dist\IgnisRig\IgnisRig.exe
                                        |
                                        v  (build_installer.bat, usa installer\IgnisRig.iss)
                          installer\output\IgnisRig_Setup.exe   <- distribua este arquivo
                                        |
                                        v  (usuário final executa)
                       %APPDATA%\IgnisRig\  (app instalado, pronto pra usar)
```

**Manutenção:** ao lançar uma nova versão, atualize `MyAppVersion` em `installer\IgnisRig.iss` e `APP_VERSION` em `core/settings.py` juntos, para manterem-se sincronizados.


## 4. Sistema de detecção inteligente de arquivos (novidade)

Instaladores baixados da internet quase sempre trazem a versão no nome do arquivo (`MSIAfterburnerInstaller466.exe`, `winrar-x64-723br.exe`, `EpicInstaller-20.1.4.msi`...). Exigir um nome exato tornaria o app frágil a cada atualização. Por isso, o IgnisRig agora resolve o arquivo de cada componente em três etapas, na ordem abaixo — a primeira que encontrar algo, vence:

1. **Nome exato** — o nome "canônico" configurado (ex.: `steam_setup.exe`).
2. **Palavra-chave** — qualquer arquivo na pasta cujo nome contenha uma palavra-chave associada ao componente (ex.: qualquer coisa com "steam" no nome, dentro de `assets/apps/steam/`).
3. **Arquivo único** — se a pasta é dedicada a um único componente (a maioria é) e há exatamente um instalador (`.exe`/`.msi`) nela, o IgnisRig assume que é esse, não importa o nome.

**Na prática: você só precisa jogar o instalador oficial na pasta certa.** Não é preciso renomear nada. O painel principal mostra o resultado com três símbolos:

| Símbolo | Significado |
|---|---|
| ● (verde) | Encontrado pelo nome exato esperado |
| ◐ (azul) | Encontrado automaticamente (palavra-chave ou arquivo único na pasta) |
| ○ (âmbar) | Nenhum arquivo correspondente encontrado |

### Conferência dos seus assets atuais

Testei o catálogo contra os arquivos que você enviou — todos foram reconhecidos automaticamente, mesmo com nomes de versão diferentes:

| Componente | Arquivo que você colocou | Como foi encontrado |
|---|---|---|
| PhysX System Software | `PhysX_9.23.1019_SystemSoftware.exe` | palavra-chave ("physx") — **também movi o arquivo de `drivers/nvidia/` para `runtimes/`, que é o local correto** (PhysX é um runtime, não um driver de vídeo) |
| Java Runtime | `Java Runtime.exe` | palavra-chave ("java") |
| Steam | `SteamSetup.exe` | palavra-chave ("steam") |
| Epic Games Launcher | `EpicInstaller-20.1.4.msi` | palavra-chave ("epic") — repare que é `.msi`, o motor já executa via `msiexec` automaticamente |
| Google Chrome | `ChromeSetup.exe` | palavra-chave ("chrome") |
| WinRAR | `winrar-x64-723br.exe` | palavra-chave ("winrar") |
| 7-Zip | `7z2602-x64.exe` | palavra-chave ("7z") |
| MSI Afterburner | `MSIAfterburnerInstaller466.exe` | palavra-chave ("afterburner") |
| HWMonitor | `hwmonitor_1.67.exe` | palavra-chave ("hwmonitor") |
| RivaTuner Statistics Server | `RTSSSetup737.exe` | palavra-chave ("rtss") |
| Discord | *(removido por você — arquivo pesado)* | quando quiser reativar, é só soltar o instalador oficial em `assets/apps/discord/`; qualquer `.exe` com "discord" no nome já é detectado |

Ainda **ausentes** no seu pacote atual (aparecem como ○ no painel, sem problema — o app funciona normalmente com eles vazios): driver NVIDIA, pacotes de DLLs (padrão/opcional), Discord, Utilitário de Drivers, DDU.

## 5. Onde conseguir os drivers e pacotes que faltam

### Drivers de vídeo (escolha o fabricante da sua placa)

| Fabricante | Onde baixar | Nome típico do arquivo |
|---|---|---|
| **AMD** | [amd.com/pt/support](https://www.amd.com/pt/support) → detectar automaticamente ou selecionar a placa manualmente → "AMD Software: Adrenalin Edition" | `amd-software-adrenalin-edition-*-minimalsetup-*.exe` (ou `setup.exe` dentro do pacote extraído) |
| **Intel** | [intel.com/content/www/us/en/download-center/home.html](https://www.intel.com/content/www/us/en/download-center/home.html) → "Intel Driver & Support Assistant" (detecção automática) ou o driver específico de Arc/Iris/UHD Graphics | `intel-dsa-*.exe` ou `igfx_win_*.exe` |
| **NVIDIA** | [nvidia.com/Download/index.aspx](https://www.nvidia.com/Download/index.aspx) → selecione o modelo da placa → "Game Ready Driver" (jogos) ou "Studio Driver" (produtividade/criação) | algo como `5xx.xx-desktop-win10-win11-64bit-international-dch-whql.exe` |

Coloque o arquivo baixado direto na pasta do fabricante (`assets/drivers/amd/`, `.../intel/` ou `.../nvidia/`) — a detecção por palavra-chave já reconhece qualquer nome que contenha "amd"/"radeon", "intel"/"arc", ou "nvidia"/"geforce"/"whql".

**Dica:** antes de trocar de fabricante de placa de vídeo (ex.: trocou de AMD para NVIDIA), vale rodar o **Display Driver Uninstaller (DDU)** — já incluso no catálogo em Utilitários e Diagnóstico — para remover resíduos do driver antigo. Baixe em [guru3d.com/files-details/display-driver-uninstaller-download.html](https://www.guru3d.com/files-details/display-driver-uninstaller-download.html) e use preferencialmente em Modo de Segurança.

### Pacotes de DLLs (`dll_pack_padrao.zip` / `dll_pack_opcional.zip`)

Aqui vale um alerta: sites de "pacotes de DLL" ou "corretores de DLL ausente" genéricos são uma fonte comum de malware e, muitas vezes, distribuem arquivos de forma irregular quanto a direitos autorais. **Recomendação:** na grande maioria dos casos, os componentes já cobertos em "Componentes Principais" (Visual C++, DirectX, .NET, OpenAL) resolvem praticamente todos os erros de DLL ausente — um pacote de DLLs avulso raramente é necessário.

Se ainda assim quiser montar os seus próprios `dll_pack_padrao.zip`/`dll_pack_opcional.zip`, o caminho seguro é extrair as DLLs você mesmo a partir de instaladores oficiais (ex.: o próprio pacote redistribuível do DirectX ou do Visual C++), nunca de sites de "DLL avulsa para download".

## 6. Fluxo do aplicativo

1. **Termo de Responsabilidade** (obrigatório na primeira execução) — aceite fica salvo, não é pedido de novo.
2. **Assistente de boas-vindas** — detecta automaticamente AMD/Intel/NVIDIA via PowerShell/WMI (em segundo plano, sem travar a interface), pergunta sobre DLLs e apps opcionais, e lembra suas últimas respostas.
3. **Painel principal** — tudo pré-configurado e editável, com os indicadores ●/◐/○ de cada item.
4. **Configurações** (ícone ⚙️) — preferências persistentes (pular assistente, confirmar antes de iniciar, aparência clara/escura, atalhos de pastas).

## 7. Interface: o que mudou nesta atualização

- **Visual atualizado**: paleta revisada com tons mais vivos de acento, cartões mais nítidos.
- **Animações suaves**: a janela abre com um fade-in; cada troca de tela (termo → assistente → painel → configurações) tem uma transição sutil de opacidade em vez de um corte seco; o indicador "● Executando" pulsa continuamente enquanto uma instalação está em andamento.
- **Enquadramento corrigido**: nomes de componentes longos (ex.: "Visual C++ Redist AIO (Todos em um)") agora quebram em duas linhas automaticamente dentro do painel, em vez de vazar para fora da barra lateral. A barra lateral também ficou um pouco mais larga (330px) e a janela um pouco maior (860×600) para dar mais espaço aos textos.
- **Indicador de status em 3 estados** (●/◐/○), explicado na seção 4 acima.
- Nova ferramenta no catálogo: **Display Driver Uninstaller (DDU)**.

## 8. Estrutura de pastas

```
ignisrig/
├── Instalador.bat / Instalador.ps1   # Instala dependências Python automaticamente
├── build_exe.bat                      # Compila main.py em dist\IgnisRig\IgnisRig.exe
├── installer/
│   ├── icon.ico                        # Ícone do aplicativo (gerado, tema "chama")
│   ├── IgnisRig.iss                    # Script do Inno Setup
│   └── build_installer.bat             # Compila IgnisRig.iss em Setup.exe
├── settings.json                      # Preferências do usuário (gerado automaticamente)
├── main.py                            # Janela principal
├── core/
│   ├── config.py                       # Catálogo de tarefas + motor de detecção inteligente
│   ├── settings.py                     # Configurações persistentes
│   └── task_manager.py                 # Motor de execução (subprocess/msiexec, extração, logging)
├── ui/
│   ├── animations.py                    # Fade de janela, transições, pulsação
│   ├── disclaimer.py                    # Termo de Responsabilidade
│   ├── settings_panel.py                # Painel de Configurações
│   ├── text_utils.py                    # Quebra de texto para checkboxes longos
│   └── wizard.py                        # Assistente de boas-vindas + detecção de GPU
├── assets/                              # Seus instaladores (ver seção 5)
├── logs/                                # Auditoria em disco (gerado automaticamente)
└── requirements.txt
```

## 9. Catálogo completo

| Categoria | Itens |
|---|---|
| Componentes Principais | Visual C++ (x64/x86), DirectX, .NET Framework 4.8.1, OpenAL |
| Drivers de Vídeo | AMD Adrenalin, Intel Driver & Support Assistant, NVIDIA Graphics Driver |
| DLLs | Pacote Padrão, Pacote Opcional |
| Runtimes Adicionais | .NET Framework 4.0, XNA Framework 4.0, VC++ ARM64, VC++ Redist AIO, NVIDIA PhysX, Java Runtime (JRE) |
| Aplicativos Populares | Steam, Epic Games Launcher, Discord, Google Chrome, WinRAR, 7-Zip |
| Utilitários e Diagnóstico | AIDA64, CrystalDiskInfo, MSI Afterburner, HWMonitor, RivaTuner Statistics Server, Display Driver Uninstaller (DDU), Utilitário de Drivers |

## 10. Adicionando novos componentes

Insira uma nova entrada em `TASKS` (`core/config.py`) com `filename` (nome canônico sugerido), `directory`, `silent_args`, `kind`, e opcionalmente `keywords=["..."]` para reforçar a detecção automática. Se a pasta for dedicada a esse único componente, nem precisa de `keywords` — a detecção por arquivo único já resolve sozinha.

## 11. Observações de segurança

- Nenhum comando é executado via shell (`shell=False`); toda extração de `.zip` valida contra *zip slip*; nada é copiado automaticamente para `System32`/`SysWOW64`.
- O `Instalador.bat` só instala pacotes Python listados em `requirements.txt` via `pip`.
- O Termo de Responsabilidade deixa claro que o usuário é responsável por verificar a procedência dos instaladores usados. **Este texto é um aviso ao usuário, não uma peça jurídica** — para distribuição pública, recomenda-se revisão por um advogado.
- A detecção "arquivo único em pasta dedicada" confia no conteúdo que você mesmo coloca nas pastas — mantenha cada pasta de `assets/` com apenas o instalador esperado, sem arquivos soltos de outras origens.
