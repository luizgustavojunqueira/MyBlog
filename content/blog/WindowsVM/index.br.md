+++
date = '2025-10-03T07:20:59-04:00'
draft = true
title = 'VM Windows no Linux'
+++

Neste post, vou compartilhar e deixar registrado para meu uso no futuro
o processo de criação de uma máquina virtual Windows 11 no Linux usando QEMU/KVM.

<!--more-->

## Por que?

Bom, se você assim como eu, prefere usar alguma distro Linux como sistema principal ou, assim como eu, não gosta do Windows, mas precisa usar o Windows para algumas tarefas específicas, como trabalho, ou alguns softwares que só rodam no Windows, ou jogos que só rodam no Windows (apesar disso não ser mais tão real assim), ou até mesmo para desenvolvimento de software que precisa ser testado no Windows, uma máquina virtual é uma ótima solução.

## Requisitos

- Um computador com Linux instalado (de preferência uma distro que suporte KVM, como Ubuntu, Fedora, Arch, etc).

### QEMU/KVM

Como sou usuário de Arch Linux, vou usar o gerenciador de pacotes `pacman` para instalar os pacotes necessários. Se você estiver usando outra distro, adapte os comandos conforme necessário.

```bash
sudo pacman -S qemu-full libvirt virt-manager virt-viewer dnsmasq
```

- qemu-full: O emulador de hardware e hypervisor.
- libvirt: Biblioteca para gerenciar máquinas virtuais.
- virt-manager: Interface gráfica para gerenciar máquinas virtuais.
- virt-viewer: Ferramenta para visualizar máquinas virtuais.
- dnsmasq: Servidor DHCP e DNS leve.

### Windows 11 ISO

Você pode baixar a ISO do Windows 11 diretamente do site da Microsoft: [Download Windows 11](https://www.microsoft.com/software-download/windows11)

### VirtIO Drivers

Para que a integração entre o Windows e o QEMU/KVM funcione corretamente, você precisará dos drivers VirtIO. 

Isso pois o Windows 11 não reconhece nativamente os dispositivos virtuais criados pelo QEMU/KVM, como o disco e a placa de rede.  

Além disso, por meio desses drivers, você terá melhor desempenho com performance quase nativa, visto que o Windows vai saber que está rodando em uma máquina virtual e vai otimizar o uso dos recursos.

Para instalar no Arch Linux, você pode usar o seguinte comando:

```bash
sudo pacman -s virtio-win
```

Ou você pode baixar diretamente do site oficial: [VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/)

## Configuração do Host

Antes de começãr a configurar a máquina virtual, primeiro vamos configurar o libvirt para permitir que o usuário atual possa gerenciar as máquinas virtuais sem precisar de privilégios de superusuário.


```bash
sudo usermod -aG libvirt $(whoami)
newgrp libvirt
```

Agora, vamos iniciar o serviço do libvirt e habilitá-lo para iniciar automaticamente na inicialização do sistema:

```bash
sudo systemctl enable --now libvirtd
```

### Configuração do libvirt

Durante a instalação da VM na minha máquina, tive alguns problemas com a questão de acesso a rede e também de permissões de usuário, então tive que fazer as seguintes alterações:

Em `/etc/libvirt/qemu.conf`, descomente a linha `user` e altere o valor para o seu usuário

```conf
...

# The user for QEMU processes run by the system instance. It can be
# specified as a user name or as a user id. The qemu driver will try to
# parse this value first as a name and then, if the name doesn't exist,
# as a user id.
#
# Since a sequence of digits is a valid user name, a leading plus sign
# can be used to ensure that a user id will not be interpreted as a user
# name.
#
# Some examples of valid values are:
#
#       user = "qemu"   # A user named "qemu"
#       user = "+0"     # Super user (uid=0)
#       user = "100"    # A user named "100" or a user with uid=100
#
user = "seu_usuario"

...
```

Além disso, precisei mudar o backend do firewall para usar iptables, em `/etc/libvirt/network.conf`


```conf
...

# Master configuration file for the network driver.
# All settings described here are optional - if omitted, sensible
# defaults are used.

# firewall_backend:
#
#   determines which subsystem to use to setup firewall packet
#   filtering rules for virtual networks.
#
#   Supported settings:
#
#     iptables - use iptables commands to construct the firewall
#     nftables - use nft commands to construct the firewall
#
#   If firewall_backend isn't configured, libvirt will choose the
#   first available backend from the following list:
#
#     [nftables, iptables]
#
#   If no backend is available on the host, then the network driver
#   will fail to start, and an error will be logged.
#
#   (NB: switching from one backend to another while there are active
#   virtual networks *is* supported. The change will take place the
#   next time that libvirtd/virtnetworkd is restarted - all existing
#   virtual networks will have their old firewalls removed, and then
#   reloaded using the new backend.)
#
firewall_backend = "iptables"
...
```


Caso você não tenha o `iptables` instalado, você pode instalar com o seguinte comando:

```bash
sudo pacman -S iptables
```

## Configuração da Máquina Virtual

Agora vamos para a parte interessante, que é configurar a instalar a máquina virtual.

Uma vez que tenha tudo instalado e confgurado, você pode iniciar o `virt-manager` (Gerenciador de Máquinas Virtuais) a partir do menu do seu sistema ou digitando `virt-manager` no terminal ou pesquisando no seu launcher favorito.

Se tudo deu certo, você verá uma interface assim:

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/blog_windowsvm_virt-manager.png" alt="Virt-Manager" />

Agora, clique no ícone de criar uma nova máquina virtual e siga os seguintes passos:

{{% steps %}}

### Criar máquina Virtual

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-createvm.png" alt="Virt-Manager" />

Aqui, mantenha as opções padrão e clique em "Forward".

### Selecionar ISO

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-selectiso.png" alt="Virt-Manager" />

Nessa segunda tela, encontre a ISO do Windows 11 que você baixou anteriormente clicando em "Browse" e depois em "Browse Local".

Após selecionar a ISO, clique em "Forward".

### Configurar Memória e CPU

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-configmem.png" alt="Virt-Manager" />

Aqui escolha a quantidade de memória RAM e CPU que você quer alocar para a máquina virtual.

Recomendo no mínimo 8GB de RAM e 4 núcleos de CPU para uma boa performance.

Depois de configurar, clique em "Forward".

### Configurar Disco Rígido

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-configdisk.png" alt="Virt-Manager" />

Aqui você pode escolher o tamanho do disco rígido virtual. Recomendo pelo menos 64GB para o Windows 11, mas se você planeja instalar muitos programas, considere aumentar esse valor, principalmente se for usar para jogos.


Também há a possibilidadde de usar um disco inteiro existente com acesso direto, via Passthrough, mas isso é um assunto para outro post e é mais avançado.

### Finalizar Configuração

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-configfinish.png" alt="Virt-Manager" />

Agora de um nome para a máquina virtual e marque a opção "Customize configuration before install" para fazer algumas configurações adicionais.


{{% /steps %}}

## Configurações Adicionais


No menu Overview, mude o firmware para UEFI x86_64: /usr/share/edk2/x64/OVMF_CODE.secboot.4m.fd

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-overview.png" alt="Virt-Manager" />

Caso você não tenha o OVMF instalado, você pode instalar com o seguinte comando:

```bash
sudo pacman -S edk2-ovmf
```

Clique em "Apply".


Agora vamos adicionar os drivers VirtIO que você baixou anteriormente.

Vá em `Add Hardware` > `Storage` > `Select or create custom storage` > `Browse Local` e selecione o arquivo ISO dos drivers VirtIO. Depois mude o device type para `CDROM device` e clique em `Finish`.

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-virtiodisk.png" alt="Virt-Manager" />

Também é interessante trocar o `bus type` do disco rígido para `VirtIO` para melhorar a performance. Vá em `SATA Disk 1` e mude o `bus type` para `VirtIO`. Clique em `Apply`.

Por último, vamos fazer a emulação do chip TPM, que é necessário para o Windows 11.

Deve ter por padrão um menu escrito `TPM vNone`. Vá nele, abra o menu de opções avançadas e selecione no dropdown de versão `2.0`. Depois clique em `Apply`.

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/virt-manager-tpm.png" alt="Virt-Manager" />

Agora pode iniciar a instalação clicando em `Begin Installation`.

## Instalação do Windows 11


Ao iniciar a instalação, você verá na tela escrito para apertar qualquer tecla para iniciar a partir do CD/DVD. Aperte qualquer tecla e será redirecionado para a tela de instalação do Windows.

Faça a configuração inicial de idioma, teclado e clique em `Next`.

Na aba de `Product Key` caso você tenha uma chave, insira ela aqui. Caso contrário, clique em `I don't have a product key` para continuar a instalação sem a chave.

Na seleção de imagem selecione `Windows 11 Home`.

### Instalação dos Drivers VirtIO

Na tela de seleção de disco, como escolhemos usar o `bus type` virtio, o Windows não reconhece o disco rígido, então precisamos carregar os drivers VirtIO.

Caso apareça o disco (no caso de você ter mantido SATA) o disco vai aparecer, mas ainda assim, instale os drivers pois são utilizados para outros dispositivos também.

Para isso, vá em `Load Driver`.

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/windows-install-loaddriver.png" alt="Virt-Manager" />

Abra o drive de CD-ROM e selecione a pasta `viostor\w11\amd64` e clique em `OK`.

Desabilite a opção de mostrar apenas drivers compatíveis e clique `install`.

Depois vá novamente em Load Driver > Browse e selecione a pasta `NetKVM\w11\amd64` e clique em `OK` e instale.

Agora pode clicar em `Next` e continuar a instalação normalmente.

Pode ser que o processo demore um pouco, então tenha paciência. Na minha máquina demorou cerca de 15 minutos para completar ir para a parte de configuração inicial do Windows.

Nessa parte, siga os passos para configurar o Windows como preferir.

Se aparecer uma tela de erro dizendo que não é possível conectar a internet, provavelmente o driver faltou alguma configuração relacionada a rede mencionada anteriormente, então volte e veja se está tudo certo.

Caso tenha ido para uma tela de verifiação de atualizações, muito provavelmente deu tudo certo, e é só esperar mais um pouco.

Como a Microsoft é chata, vai pedir para você logar em uma conta Microsoft ou criar uma e até o momento não descobri se é possível pular essa etapa, então crie uma conta Microsoft ou use uma que você já tenha.

## Configuração Pós-Instalação

Uma vez finalmente com o windows instalado, vamos fazer algumas configurações adicionais para melhorar a performance e a integração com o Linux.

Abra o explorador de arquivos e vá até o drive de CD-ROM onde estão os drivers VirtIO.

Execute o instalador do `virtio-win-gt-x64.msi` e siga os passos para instalar todos os drivers.

Depois instale o `virtio-win-guest-tools` que vai instalar o QEMU Guest Agent, que é um serviço que permite a comunicação entre o host e a máquina virtual, permitindo funcionalidades como desligamento seguro, sincronização de tempo, etc.

Uma vez instalados, você pode ir no menu do virt-manager `View` -> `Scale to Display` e ativar `Auto resize VM with window` para que a resolução da máquina virtual se ajuste automaticamente ao tamanho da janela.

Agora pode desligar a máquina para que possamos fazer mais algumas configurações.

### Configurações no Host

Agora que já temos o Windows instalado, vamos fazer algumas configurações adicionais no host para melhorar a performance.

Primeiramente, podemos remover CD-ROM com os drivers VirtIO, já que não precisamos mais deles. Para isso, selecione o CD-ROM na lista de hardware e clique em `Remove`. Provavelmente é o SATA CDROM 2.

Caso você queria usar uma camera, como no meu caso, estou fazendo a instalação em um notebook, para adicionar a camera a VM, basta ir em `Add Hardware` > `USB Host Device` e encontrar o seu dispositivo e adicioná-lo.

Além disso, pelo menos no meu notebook, tive problemas com entrada de audio, então tive que fazer a seguinte configuração na aba `Overview` editando manualmente o XML de configuração, trocando a parte de audio por essa configuração:

```XML
<sound model="ich9">
  <codec type="duplex"/>
  <audio id="1"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
</sound>
<audio id="1" type="pipewire" runtimeDir="/run/user/1000">
  <input name="alsa_input.pci-0000_00_1f.3-platform-skl_hda_dsp_generic.HiFi__Mic1__source" streamName="VM-Microphone" latency="100"/>
  <output name="default" streamName="VM-Audio" latency="15000"/>
</audio>
```

Como uso pipewire no meu sistema, essa configuração funciona perfeitamente, mas se você usa pulseaudio, pode tentar mudar o type para `pulseaudio` e ver se funciona.

Para definir o `name` do input, você pode usar o comando `pactl list sources short` para listar as fontes de audio disponíveis e encontrar a que corresponde ao seu microfone. (No caso eu fui testando até achar a correta).

Depois de fazer essas alterações, clique em `Apply`.

Agora você deve ter uma máquina virtual Windows 11 rodando no seu Linux com QEMU/KVM, com boa performance e integração.

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/WindowsVM/blog_windowsvm_windows11.png" alt="Windows 11" />


## Considerações Finais

Bom, a ideia desse post era deixar registrado o processo que segui para instalar o Windows 11 no Linux usando QEMU/KVM, para que eu possa consultar no futuro caso precise fazer novamente e para deixar disponível para quem mais possa precisar visto que eu mesmo tive dificuldades para encontrar um guia completo e atualizado sobre o assunto.

No futuro pretendo fazer mais posts sobre configurações avançadas, como passthrough de GPU, USB, CPU pinning, etc.

Além disso, quero deixar claro que esse processo pode variar dependendo do hardware e da distribuição Linux que você está usando, então é sempre bom consultar a documentação oficial do QEMU/KVM e do VirtIO para mais detalhes.

Também, obviamente, como é uma máquina virtual, não espere performance igual a de um sistema nativo, mas com as configurações corretas, a performance pode ser muito boa e suficiente para a maioria das tarefas.

No meu caso, tenho uma máquina relativamente potente, com a seguinte configuração:

<img src="https://luizgustavojunqueira-personalblog.s3.us-east-1.amazonaws.com/my_hardware.png" alt="Hardware" />

Então consigo deixar 16GB de RAM e 8 núcleos de CPU para a máquina virtual, o que é mais do que suficiente para o meu uso, enquanto ainda consigo usar o Linux normalmente.
