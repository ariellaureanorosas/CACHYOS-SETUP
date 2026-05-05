<div align="center">

```
 ██████╗ █████╗  ██████╗██╗  ██╗██╗   ██╗ ██████╗ ███████╗
██╔════╝██╔══██╗██╔════╝██║  ██║╚██╗ ██╔╝██╔═══██╗██╔════╝
██║     ███████║██║     ███████║ ╚████╔╝ ██║   ██║███████╗
██║     ██╔══██║██║     ██╔══██║  ╚██╔╝  ██║   ██║╚════██║
╚██████╗██║  ██║╚██████╗██║  ██║   ██║   ╚██████╔╝███████║
 ╚═════╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝   ╚═╝    ╚═════╝ ╚══════╝
```

### `GTX 1650 · 180Hz · Reverse PRIME · macOS Look`

**Guia completo para CachyOS com NVIDIA em notebook híbrido — do diagnóstico ao sistema finalizado.**

---

![CachyOS](https://img.shields.io/badge/CachyOS-Arch_Based-blue?style=flat-square&logo=archlinux&logoColor=white)
![Cinnamon](https://img.shields.io/badge/DE-Cinnamon-green?style=flat-square)
![NVIDIA](https://img.shields.io/badge/GPU-GTX_1650-76B900?style=flat-square&logo=nvidia&logoColor=white)
![Kernel](https://img.shields.io/badge/Kernel-BORE-orange?style=flat-square)
![Shell](https://img.shields.io/badge/Shell-Fish-lightblue?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-purple?style=flat-square)

</div>

---

## O Problema

> Interface travada a ~30fps com monitor externo de 180Hz conectado via HDMI.

Em notebooks com arquitetura híbrida Intel + NVIDIA (Optimus/PRIME), a **Intel controla todas as saídas de vídeo** — inclusive o HDMI. Como a iGPU Intel não sincroniza corretamente a 180Hz, o resultado é uma experiência visual degradada, mesmo com o driver NVIDIA instalado e funcionando.

**A solução:** reverter o controle para a NVIDIA como GPU primária do servidor X (*Reverse PRIME*), mantendo a Intel apenas para renderizar a tela interna do notebook.

---

## Hardware

| Componente | Especificação |
|-----------|--------------|
| 💻 Notebook | Lenovo IdeaPad Gaming 3i |
| 🔵 iGPU | Intel UHD Graphics (CometLake-H GT2) |
| 🟢 dGPU | NVIDIA GeForce GTX 1650 Mobile / Max-Q |
| 🖥️ Monitor externo | 1920×1080 @ 180Hz (HDMI) |
| 🐧 Distro | CachyOS |
| 🖥️ DE | Cinnamon |
| 🐟 Shell | Fish |

---

## O que este guia cobre

```
📌 Diagnóstico     → identificar por que a interface trava a 180Hz
⚡ Reverse PRIME   → NVIDIA como GPU principal, Intel como secundária
🍎 Aparência       → tema macOS completo (WhiteSur, Plank, Cinnamenu)
💻 VS Code         → instalação AUR + extensões Python + configurações
🚀 Performance     → kernel BORE, Gamemode, Ananicy, PowerMizer
🧹 Limpeza         → pacotes órfãos, cache, logs, duplicados
🔊 Sons            → mapeamento completo dos eventos do Cinnamon
🔤 Fontes          → JetBrains Mono, Inter, Nerd Fonts e mais
🆘 Reversão        → como desfazer tudo se algo der errado
```

---

## Início Rápido

> Se o seu sistema já está em modo híbrido Intel+NVIDIA e a interface trava, comece aqui.

**1. Confirme a causa raiz:**
```bash
glxinfo | grep -i "server glx vendor"
# Se retornar "SGI" → Intel está no controle → siga o guia
```

**2. Force a NVIDIA como GPU principal:**
```bash
sudo nvidia-xconfig && systemctl reboot
```

**3. Crie a configuração Reverse PRIME:**
```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/10-hybrid-reverse-prime.conf
# → veja o conteúdo completo no guia principal
```

**4. Ative a tela do notebook:**
```bash
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --output eDP-1-1 --auto --right-of HDMI-0
```

**Resultado:** monitor externo a 180Hz com fluidez total + tela do notebook ativa.

---

## Documentação Completa

| Seção | Descrição |
|-------|-----------|
| [📋 Diagnóstico](CACHYOS-SETUP.md#1-diagnóstico-do-problema-de-fluidez) | Identificar e entender o problema |
| [⚙️ Reverse PRIME](CACHYOS-SETUP.md#2-solução-definitiva--reverse-prime) | Configuração completa da GPU |
| [🍎 Aparência macOS](CACHYOS-SETUP.md#3-personalização-visual--aparência-macos) | WhiteSur, Plank, cursores, fontes |
| [💻 VS Code](CACHYOS-SETUP.md#4-visual-studio-code) | Instalação e extensões |
| [🚀 Performance](CACHYOS-SETUP.md#5-otimizações-de-desempenho) | Kernel, Gamemode, Ananicy |
| [🧹 Limpeza](CACHYOS-SETUP.md#6-limpeza-do-sistema) | Cache, órfãos, logs |
| [🔊 Sons](CACHYOS-SETUP.md#7-sons-do-cinnamon) | Mapeamento de eventos |
| [🔤 Fontes](CACHYOS-SETUP.md#8-fontes-adicionais) | Instalação de fontes |
| [🆘 Reversão](CACHYOS-SETUP.md#9-reversão-de-emergência) | Desfazer em caso de emergência |
| [🔵 Bluetooth](CACHYOS-SETUP.md#10-configuração-do-bluetooth) | bluez, bluez-utils e Blueman |

---

## Stack Final

<div align="center">

| Categoria | Escolha |
|-----------|---------|
| **Driver** | NVIDIA 595.71.05 |
| **Kernel** | linux-cachyos-bore |
| **Modo GPU** | Reverse PRIME |
| **Tema GTK** | WhiteSur-light-solid |
| **Ícones** | WhiteSur |
| **Cursores** | McMojave |
| **Dock** | Plank + WhiteSur theme |
| **Menu** | Cinnamenu |
| **Fonte sistema** | Inter 10 |
| **Fonte código** | JetBrains Mono |
| **Editor** | VS Code (AUR) |

</div>

---

## Aviso

> Este guia foi escrito e testado especificamente no hardware descrito. BusIDs, nomes de saídas (`HDMI-0`, `eDP-1-1`) e versões de driver podem variar no seu sistema. Sempre confirme os valores com `lspci` e `xrandr` antes de aplicar as configurações.

Em caso de falha na inicialização, consulte a [seção de reversão de emergência](CACHYOS-SETUP.md#9-reversão-de-emergência).

---

<div align="center">

*Feito no CachyOS · Lenovo IdeaPad Gaming 3i · GTX 1650 · 180Hz*

</div>
