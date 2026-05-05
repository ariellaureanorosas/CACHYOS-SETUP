# 🐧 CachyOS Setup Guide — GTX 1650 + Monitor 180Hz

> Guia completo e testado para configurar o CachyOS com NVIDIA GTX 1650 em modo **Reverse PRIME**, monitor externo a 180Hz com fluidez total.

**Hardware:** Lenovo IdeaPad Gaming 3i · Intel UHD (iGPU) · NVIDIA GeForce GTX 1650 Mobile  
**DE:** Cinnamon · **Shell:** Fish · **Kernel:** CachyOS (BORE)

---

## 📋 Índice

1. [Diagnóstico do Problema de Fluidez](#1-diagnóstico-do-problema-de-fluidez)
2. [Solução Definitiva — Reverse PRIME](#2-solução-definitiva--reverse-prime)
3. [Reversão de Emergência](#3-reversão-de-emergência)

---

## 1. Diagnóstico do Problema de Fluidez

A interface parecia travada a ~30fps mesmo com o monitor a 180Hz. Este diagnóstico identificou a causa raiz.

### 1.1 Verificar driver NVIDIA

```bash
nvidia-smi
```

> ✅ Resultado esperado: driver NVIDIA (595.71.05) instalado, GPU GeForce GTX 1650 reconhecida.

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

### 1.3 Identificar a causa raiz: modo híbrido Intel+NVIDIA

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

## 3. Reversão de Emergência

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

## Resumo

```
┌─────────────────────────────────────────────────────────────────┐
│  Problema     → Interface travada a ~30fps com monitor 180Hz    │
│  Causa raiz   → Modo híbrido Intel+NVIDIA (Optimus/PRIME)       │
│  Solução      → Reverse PRIME: NVIDIA como GPU principal        │
└─────────────────────────────────────────────────────────────────┘
```

| Componente | Versão/Pacote |
|------------|--------------|
| GPU | NVIDIA GeForce GTX 1650 Mobile |
| Driver NVIDIA | 595.71.05 |
| Kernel | linux-cachyos-bore |

---
<p align="center">
<em>Testado no CachyOS com Cinnamon · Lenovo IdeaPad Gaming 3i · Intel UHD + GTX 1650</em>
</p>
