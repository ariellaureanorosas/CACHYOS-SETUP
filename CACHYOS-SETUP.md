# 🐧 CachyOS Setup Guide — GTX 1650 + Monitor 180Hz + Aparência macOS

> Guia completo e testado para configurar o CachyOS com NVIDIA GTX 1650 em modo **Reverse PRIME**, monitor externo a 180Hz com fluidez total, personalização visual inspirada no macOS e otimizações de desempenho.

**Hardware:** Lenovo IdeaPad Gaming 3i · Intel UHD (iGPU) · NVIDIA GeForce GTX 1650 Mobile  
**DE:** Cinnamon · **Shell:** Fish · **Kernel:** CachyOS (BORE)

---

## 📋 Índice

1. [Diagnóstico do Problema de Fluidez](#1-diagnóstico-do-problema-de-fluidez)
2. [Solução Definitiva — Reverse PRIME](#2-solução-definitiva--reverse-prime)
3. [Personalização Visual — Aparência macOS](#3-personalização-visual--aparência-macos)
4. [Visual Studio Code](#4-visual-studio-code)
5. [Otimizações de Desempenho](#5-otimizações-de-desempenho)
6. [Limpeza do Sistema](#6-limpeza-do-sistema)
7. [Sons do Cinnamon](#7-sons-do-cinnamon)
8. [Fontes Adicionais](#8-fontes-adicionais)
9. [Reversão de Emergência](#9-reversão-de-emergência)

---

## 1. Diagnóstico do Problema de Fluidez

A interface parecia travada a ~30fps mesmo com o monitor a 180Hz. Este diagnóstico identificou a causa raiz.

### 1.1 Verificar driver NVIDIA

```bash
nvidia-smi
```

> ✅ Resultado esperado: driver NVIDIA (595.71.05) instalado, GPU GeForce GTX 1650 reconhecida.  
> ❌ `nvidea-smi` → erro de digitação, corrija para `nvidia-smi`.

### 1.2 Verificar taxa de atualização do monitor

```bash
xrandr
```

Procure a linha do monitor externo. Exemplo de saída correta:

```
HDMI-1-0 connected 1920x1080+0+0 (normal left inverted right x axis y axis) 527mm x 296mm
   1920x1080    180.00*+  ...
```

O `*` indica a taxa ativa. Se aparecer `180.00*`, os 180Hz estão ativos — o problema não é a taxa configurada.

### 1.3 Tentativa: fixar clock mínimo da GPU (PowerMizer)

Hipótese: a GPU reduzia o clock para 300MHz quando ociosa, causando atrasos.

```bash
sudo nvidia-smi -lgc 700,2100
```

> Isso fixa o clock entre 700MHz e 2100MHz. Pode ajudar, mas não resolve o problema principal.

### 1.4 Tentativa: ForceFullCompositionPipeline (falhou)

```bash
nvidia-settings --assign CurrentMetaMode="HDMI-1-0: 1920x1080_180 +0+0 { ForceFullCompositionPipeline = On }"
```

> ❌ Erro: `Error resolving target specification`. Os nomes do xrandr não correspondem aos nomes internos do driver NVIDIA nesse modo.

### 1.5 Identificar a causa raiz: modo híbrido Intel+NVIDIA

```bash
glxinfo | grep -i "server glx vendor"
```

**Se a saída for:**
```
server glx vendor string: SGI
```

O servidor GLX está usando o driver Intel (modesetting), **não a NVIDIA**. O sistema está em modo Optimus/PRIME, onde a Intel controla todas as saídas de vídeo — inclusive o HDMI. A Intel não sincroniza corretamente a 180Hz, gerando a sensação de 30fps.

Confirme com:

```bash
xrandr --listproviders
```

```
Provider 0: id: 0x... cap: 0xb, Source Output, Sink Output, Sink Offload name: modesetting
Provider 1: id: 0x... cap: 0x2, Sink Output name: NVIDIA-G0
```

```bash
lspci -k | grep -A 2 -E "(VGA|3D)"
```

```
00:02.0 Intel Corporation CometLake-H GT2 [UHD Graphics] → driver: i915
01:00.0 NVIDIA Corporation TU117M [GeForce GTX 1650 Mobile] → driver: nvidia
```

> **Conclusão:** a Intel está no controle. É necessário colocar a NVIDIA como GPU principal via **Reverse PRIME**.

---

## 2. Solução Definitiva — Reverse PRIME

A estratégia é forçar a NVIDIA como GPU principal do servidor X e usar a Intel apenas para renderizar a tela interna do notebook (reverse PRIME).

### 2.1 Gerar xorg.conf inicial

```bash
sudo nvidia-xconfig
systemctl reboot
```

> ✅ Após reiniciar, o monitor externo funcionará com fluidez total a 180Hz.  
> ⚠️ A tela do notebook ficará preta (conectada à Intel, que agora está inativa).  
> ✅ O `nvidia-settings` passará a mostrar a aba "X Server Display Configuration".

### 2.2 Criar configuração personalizada (Reverse PRIME)

Remova o xorg.conf genérico e crie um arquivo dedicado:

```bash
sudo rm /etc/X11/xorg.conf
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/10-hybrid-reverse-prime.conf
```

Antes de preencher o arquivo, obtenha os BusIDs das GPUs:

```bash
lspci | grep -E "VGA|3D"
```

```
00:02.0 VGA compatible controller: Intel Corporation CometLake-H GT2 [UHD Graphics]
01:00.0 3D controller: NVIDIA Corporation TU117M [GeForce GTX 1650 Mobile]
```

- Intel `00:02.0` → BusID `PCI:0:2:0`
- NVIDIA `01:00.0` → BusID `PCI:1:0:0`

Cole o conteúdo abaixo no arquivo (ajuste os BusIDs se necessário):

```xorg
Section "ServerLayout"
    Identifier "layout"
    Screen 0 "nvidia" 0 0
    Inactive "intel"
EndSection

Section "Device"
    Identifier "nvidia"
    Driver "nvidia"
    BusID "PCI:1:0:0"
    Option "AllowEmptyInitialConfiguration"
    Option "Coolbits" "28"
EndSection

Section "Device"
    Identifier "intel"
    Driver "modesetting"
    BusID "PCI:0:2:0"
    Option "kmsdev" "/dev/dri/card0"
    Option "AllowEmptyInitialConfiguration" "True"
EndSection

Section "Screen"
    Identifier "nvidia"
    Device "nvidia"
    Option "AllowEmptyInitialConfiguration" "True"
EndSection

Section "Screen"
    Identifier "intel"
    Device "intel"
EndSection
```

Reinicie:

```bash
systemctl reboot
```

Após reiniciar, confirme os providers:

```bash
xrandr --listproviders
```

```
Provider 0: name: NVIDIA-0
Provider 1: name: modesetting
```

### 2.3 Ativar a tela do notebook via Reverse PRIME

```bash
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --output eDP-1-1 --auto --right-of HDMI-0
```

> ⚠️ O nome da saída interna pode variar. Rode `xrandr` para confirmar (pode ser `eDP-1-1`, `eDP-1`, etc.).  
> O monitor HDMI também mudou de nome para `HDMI-0` (antes era `HDMI-1-0`).

### 2.4 Ativar ForceFullCompositionPipeline

Agora com a NVIDIA como principal, o comando funciona:

```bash
nvidia-settings --assign CurrentMetaMode="HDMI-0: 1920x1080_180 +0+0 { ForceFullCompositionPipeline = On }"
```

Para salvar permanentemente:
1. Abra o `nvidia-settings`
2. Vá em **X Server Display Configuration**
3. Clique em **Advanced**
4. Marque **Force Full Composition Pipeline** para o monitor HDMI
5. Clique em **Save to X Configuration File**

### 2.5 Serviço systemd — travar clock mínimo da GPU

Evita que a GPU reduza para 300MHz e cause stutter:

```bash
sudo nano /etc/systemd/system/nvidia-lockclk.service
```

```ini
[Unit]
Description=Travar clock mínimo da NVIDIA para fluidez
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/nvidia-smi -lgc 700,2100
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now nvidia-lockclk.service
```

### 2.6 Script de autostart — ativar tela do notebook no login

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/dual-screen.sh
```

```bash
#!/bin/bash
sleep 2
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --output eDP-1-1 --auto --right-of HDMI-0
```

```bash
chmod +x ~/.config/autostart/dual-screen.sh
```

Adicione nos aplicativos de inicialização do Cinnamon:
> **Menu → Aplicativos de inicialização → Adicionar**  
> Nome: `Dual Screen`  
> Comando: `/home/SEU_USUARIO/.config/autostart/dual-screen.sh`

---

## 3. Personalização Visual — Aparência macOS

### 3.1 Instalar dependências

```bash
sudo pacman -S gnome-screenshot git sassc glib2
```

### 3.2 Tema GTK WhiteSur

```bash
git clone https://github.com/vinceliuice/WhiteSur-gtk-theme.git --depth=1
cd WhiteSur-gtk-theme
./install.sh -t purple -o solid -i arch
```

| Flag | Descrição |
|------|-----------|
| `-t purple` | Cor de destaque roxa (estilo macOS) |
| `-o solid` | Janelas sólidas, sem transparência |
| `-i arch` | Ajustes específicos para Arch/CachyOS |

### 3.3 Ícones WhiteSur

```bash
cd ~
git clone https://github.com/vinceliuice/WhiteSur-icon-theme.git --depth=1
cd WhiteSur-icon-theme
./install.sh
```

### 3.4 Cursores McMojave

```bash
cd ~
git clone https://github.com/vinceliuice/McMojave-cursors.git --depth=1
cd McMojave-cursors
sudo cp -r McMojave* /usr/share/icons/
```

### 3.5 Fonte Inter

```bash
sudo pacman -S ttf-inter
```

### 3.6 Dock Plank

```bash
sudo pacman -S plank
```

Copie o tema WhiteSur para o Plank:

```bash
cd ~/WhiteSur-gtk-theme
cp -r src/other/plank/theme-WhiteSur* ~/.local/share/plank/themes/
```

Configure o Plank:

```bash
plank --preferences
```

Marque **Icon Zoom** e selecione o tema `WhiteSur`.

Adicione o Plank aos aplicativos de inicialização:
> **Menu → Aplicativos de inicialização → Adicionar**  
> Nome: `Plank`  
> Comando: `plank`

### 3.7 Menu Cinnamenu

> **Clique direito no painel → Miniaplicativos → aba Download → procurar "Cinnamenu" → Instalar**

Remova o menu padrão e adicione o Cinnamenu ao painel.

### 3.8 Aplicar temas nas Configurações do Sistema

**Configurações do Sistema → Temas:**

| Elemento | Valor |
|----------|-------|
| Temas da área de trabalho | `WhiteSur-light` (ou dark) |
| Controles | `WhiteSur-light-solid` |
| Janelas | `WhiteSur-light-solid` |
| Ícones | `WhiteSur` |
| Cursores | `McMojave-cursors` |

**Configurações do Sistema → Fontes:**

| Tipo | Fonte | Tamanho |
|------|-------|---------|
| Padrão | Inter | 10 |
| Documento | Inter | 10 |
| Terminal | Inter Mono (ou monospace) | 10 |
| Dica de renderização | Subpixel (LCD) ou Leve | — |

### 3.9 Mover painel para o topo (estilo macOS)

> **Clique direito no painel → Mover → clique na borda superior da tela**  
> **Clique direito no painel → Configurações do painel → marcar "Bloquear painel"**

### 3.10 Limpar repositórios clonados

```bash
cd ~
rm -rf WhiteSur-gtk-theme WhiteSur-icon-theme McMojave-cursors
```

Verificação no fish shell:

```fish
for dir in WhiteSur-gtk-theme WhiteSur-icon-theme McMojave-cursors
    test -d $dir; and echo "$dir ainda existe"; or echo "$dir removida"
end
```

---

## 4. Visual Studio Code

### 4.1 Diferença entre os métodos de instalação

| Método | Pacote | Observação |
|--------|--------|------------|
| `pacman` | `code` | Versão open source (VSCodium) |
| AUR (`paru`) | `visual-studio-code-bin` | Versão oficial Microsoft ✅ Recomendado |
| Flatpak | `com.visualstudio.code` | Isolado, exige config extra de temas |

> Para manter integração com o tema WhiteSur, use a versão AUR.

### 4.2 Instalar via paru

```bash
paru -S visual-studio-code-bin
```

> Pressione `q` para sair do visualizador do PKGBUILD, confirme com `s` (sim).

### 4.3 Instalar fonte JetBrains Mono

```bash
sudo pacman -S ttf-jetbrains-mono
```

### 4.4 Instalar extensões

```bash
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension ms-python.black-formatter
code --install-extension ms-python.isort
code --install-extension ms-python.flake8
code --install-extension formulahendry.code-runner
code --install-extension streetsidesoftware.code-spell-checker
code --install-extension streetsidesoftware.code-spell-checker-portuguese-brazilian
code --install-extension PKief.material-icon-theme
code --install-extension KevinRose.vsc-python-indent
code --install-extension omaressam.om-theme
```

---

## 5. Otimizações de Desempenho

### 5.1 Kernel com agendador BORE

Abra o **CachyOS Kernel Manager** e instale o kernel `linux-cachyos-bore`. Reinicie após a instalação.

### 5.2 Governador de CPU

```bash
# Temporário (até reiniciar)
sudo cpupower frequency-set -g performance
```

### 5.3 Modo de persistência da NVIDIA

```bash
sudo nvidia-persistenced
sudo systemctl enable nvidia-persistenced
```

### 5.4 PRIME Render Offload

Para executar aplicativos diretamente na GPU NVIDIA:

```bash
prime-run nome_do_aplicativo
```

### 5.5 Metapacote de jogos

```bash
sudo pacman -S cachyos-gaming-meta
```

> Inclui Gamemode, MangoHud, Steam e bibliotecas de compatibilidade.

### 5.6 Gamemode

```bash
# Terminal
gamemoderun ./jogo

# Steam (opções de inicialização do jogo)
gamemoderun %command%
```

### 5.7 Ananicy (prioridade automática de processos)

```bash
sudo pacman -S ananicy
sudo systemctl enable --now ananicy
```

---

## 6. Limpeza do Sistema

### 6.1 Pacotes órfãos

```bash
# Listar órfãos
pacman -Qtdq

# Remover (revise a lista antes de confirmar!)
sudo pacman -Rns $(pacman -Qtdq)
```

### 6.2 Cache do pacman

```bash
# Manter apenas as 3 últimas versões
sudo paccache -rk3

# Remover pacotes desinstalados do cache
sudo paccache -ruk0
```

### 6.3 Cache do paru (AUR)

```bash
paru -Scc

# Limpar clones antigos
rm -rf ~/.cache/paru/clone/*
```

### 6.4 Logs do sistema (journald)

```bash
# Verificar uso atual
journalctl --disk-usage

# Manter apenas os últimos 2 dias
sudo journalctl --vacuum-time=2d

# Limitar tamanho máximo a 500MB
sudo journalctl --vacuum-size=500M
```

### 6.5 Cache do usuário

```bash
# Remover arquivos não acessados há mais de 100 dias
find ~/.cache/ -type f -atime +100 -delete

# Limpar thumbnails
rm -rf ~/.cache/thumbnails/*

# Limpar lixeira
rm -rf ~/.local/share/Trash/*
```

### 6.6 Detectar arquivos duplicados com rmlint

```bash
sudo pacman -S rmlint

# Simulação (não remove nada, só lista)
rmlint ~/
```

### 6.7 Script de manutenção da comunidade

```bash
# Fish shell — usar pipe ao invés de substituição de processo
curl -sL https://raw.githubusercontent.com/7uisu/cachyos-maintenance/main/cachyos-maintenance.sh | bash

# Alternativa: salvar e executar
curl -sL https://raw.githubusercontent.com/7uisu/cachyos-maintenance/main/cachyos-maintenance.sh \
  -o /tmp/cachyos-maintenance.sh && bash /tmp/cachyos-maintenance.sh
```

> ⚠️ No **fish shell**, a sintaxe `$(...)` não funciona em todos os contextos. Use pipes ou salve em arquivo intermediário.

---

## 7. Sons do Cinnamon

> **Configurações do Sistema → Som → Sons do sistema**

| Evento | Arquivo de som |
|--------|---------------|
| Iniciando o Cinnamon | `service-login.oga` |
| Saindo do Cinnamon | `service-logout.oga` |
| Mudando espaço de trabalho | `complete.oga` |
| Abrindo novas janelas | `dialog-information.oga` |
| Fechar janelas | `complete.oga` |
| Minimizando janelas | `bell.oga` |
| Maximizando janelas | `bell.oga` |
| Desmaximizando janelas | `bell.oga` |
| Janelas em mosaico/encaixes | `camera-shutter.oga` |
| Inserindo um dispositivo | `device-added.oga` |
| Removendo um dispositivo | `device-removed.oga` |
| Mostrar notificações | `message.oga` |
| Alterando volume | `audio-volume-change.oga` |

---

## 8. Fontes Adicionais

### 8.1 Fontes para desenvolvimento

```bash
sudo pacman -S \
  ttf-jetbrains-mono \
  ttf-firacode-nerd \
  ttf-cascadia-code \
  ttf-iosevka-nerd \
  ttf-hack \
  noto-fonts-emoji
```

### 8.2 Fontes do Windows (opcional)

```bash
paru -S ttf-ms-win11-auto
```

### 8.3 Fontes da Apple (opcional)

```bash
paru -S apple-fonts
```

### 8.4 Verificar instalação

```bash
fc-list | grep "JetBrains Mono"
```

---

## 9. Reversão de Emergência

Se o sistema não iniciar ou o monitor não funcionar após as configurações:

**1.** Na tela de login (ou boot), pressione `Ctrl + Alt + F2` para abrir um terminal virtual.

**2.** Faça login com seu usuário.

**3.** Remova o arquivo de configuração:

```bash
sudo rm /etc/X11/xorg.conf.d/10-hybrid-reverse-prime.conf

# Se existir um xorg.conf na raiz de X11, remova também
sudo rm -f /etc/X11/xorg.conf
```

**4.** Reinicie:

```bash
systemctl reboot
```

> ✅ O sistema voltará ao modo híbrido original, com a Intel controlando tudo. Perda de fluidez a 180Hz, mas o sistema volta a funcionar normalmente.

---

## Resumo do que foi feito

```
┌─────────────────────────────────────────────────────────────────┐
│  Problema     → Interface travada a ~30fps com monitor 180Hz    │
│  Causa raiz   → Modo híbrido Intel+NVIDIA (Optimus/PRIME)       │
│  Solução      → Reverse PRIME: NVIDIA como GPU principal        │
│  Extras       → Aparência macOS, VS Code, fontes, limpeza       │
└─────────────────────────────────────────────────────────────────┘
```

| Componente | Versão/Pacote |
|------------|--------------|
| GPU | NVIDIA GeForce GTX 1650 Mobile |
| Driver NVIDIA | 595.71.05 |
| Kernel | linux-cachyos-bore |
| DE | Cinnamon |
| Tema GTK | WhiteSur-light-solid |
| Ícones | WhiteSur |
| Cursores | McMojave-cursors |
| Fonte sistema | Inter 10 |
| Fonte código | JetBrains Mono |
| Dock | Plank + tema WhiteSur |
| Menu | Cinnamenu |

---

*Testado no CachyOS com Cinnamon · Lenovo IdeaPad Gaming 3i · Intel UHD + GTX 1650*
