Linux agora está rodando no iPhone 7 e outros tipos de iPhone como iPhone X, 11,12,13 e com certeza no 14 e os próximos que virão pelos próximos anos...

A execução do postmarketOS Linux no iPhone 7 , finalmente consegui montar e executar imagens de sistema maiores e persistentes a partir da memória do sistema do iPhone. Portanto, usei a mesma técnica que o Corellium estava usando em sua compilação do Android para o iPhone 7. Além disso, o suporte de gravação efêmera para postmarketOS é obtido usando OverlayFS .

Preparando a imagem do postmarketOS

Vamos começar compilando a imagem base usando o utilitário postmarketOS pmbootstrap .

pmbootstrap init
# Work path [/home/onny/.local/var/pmbootstrap]
# Vendor: qemu
# Device codename: aarch64
# Kernel: virt
# User interface: weston
pmbootstrap install
Durante a inicialização, você pode deixar a maioria das variáveis ​​como estão. Como exemplo, estamos escolhendo Weston como interface do usuário. Após a instalação, temos que alterar uma configuração e executar o processo de instalação novamente.

pmbootstrap chroot -r
# vi /etc/xdg/weston/weston.ini # change one variable
# [...]
# backend=fbdev-backend.so
# [...]
pmbootstrap install

Temos que extrair o initramfs e adicionar nosso procedimento de montagem do sistema de arquivos personalizado no script init.

pmbootstrap initfs extract
~/.local/var/pmbootstrap/chroot_rootfs_qemu-aarch64/tmp/initfs-extracted/init
[...]
mount_root-partition

/bin/mkdir -p /mnt/apfs /mnt/ro /mnt/rw
/bin/mount -t apfs -o ro,relatime,vol=5 /dev/nvme0n1p1 /mnt/apfs
/sbin/losetup /dev/loop0 /mnt/apfs/qemu-aarch64.img -o 60817408 -r
/bin/mount -t ext4 -o ro /dev/loop0 /mnt/ro
/bin/mount -t tmpfs tmpfs /mnt/rw
/bin/mkdir -p /mnt/rw/data /mnt/rw/work
/bin/mkdir -p /sysroot
/bin/mount -t overlay -o lowerdir=/mnt/ro,upperdir=/mnt/rw/data,workdir=/mnt/rw/work overlay /sysroot

init="/sbin/init"
[...]

Existem duas variáveis ​​no trecho de código acima. Primeiro, há o parâmetro vol=5que especifica o volume APFS de destino que criaremos mais tarde. Se você já criou mais volumes personalizados em seu telefone, esse valor provavelmente é maior. Em segundo lugar , losetupespecifica um deslocamento -o 60817408que representa o deslocamento em bytes para a partição do sistema ext4 dentro da imagem. Você pode calcular esse deslocamento multiplicando o tamanho do setor e iniciar o setor usando fdisk.

Recompacte o initramfs para o kernel.

cd ~/.local/var/pmbootstrap/chroot_rootfs_qemu-aarch64/tmp/initfs-extracted/
sh -c "find . | cpio  --quiet -o -H newc | gzip -9 > /tmp/ramdisk.cpio.gz"
Compilando o kernel com ramdisk customizado

A parte seguinte é semelhante ao guia antigo, mas desta vez estamos usando a imagem initramfs pmbootstrapdiretamente.

pacman -S aarch64-linux-gnu-gcc 
cd /tmp
git clone https://github.com/corellium/linux-sandcastle.git
cd linux-sandcastle
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make hx_h9p_defconfig
cp /tmp/ramdisk.cpio.gz .
make -j4
./dtbpack.sh
lzma -z --stdout arch/arm64/boot/Image > arch/arm64/boot/Image.lzma
Imagem do sistema piscando e kernel

Tenha cuidado, as etapas a seguir são consideradas seguras de usar, mas isso ainda é experimental e pode bloquear seu telefone. Se vc for usar outras versões do Linux use por sua conta e risco, e se vc testar esse método e funcionar outra versão do Linux como alguns Debian, ubuntu, Arch Linux e etc... nesse Tutorial, me mande um email na conta redtube21002200@gmail.com quem sabe a gente pode tomar um café juntos, pois e claro precisamos de no mínimo outros 5 aparelhos telemovel iPhone no mínimo pra fazer outros teste, nesse iPhone 7 o Alpine Linux funcionou!

Isso não é “piscar” no sentido tradicional, mas agora vamos usar o bootrom exploit checkra1n para obter acesso root ssh no telefone. Coloque seu telefone no modo DFU e execute o seguinte comando:

checkra1n -cE
iproxy 2222 222 # leave this running while accessing via ssh
sshpass -p "alpine" ssh -p2222 root@localhost
Dentro do shell raiz do iPhone, vamos criar um novo volume APFS e montá-lo. Você precisa executar essas etapas apenas uma vez, basta remontar a partição se desejar excluir ou substituir a imagem do sistema existente.

newfs_apfs -A -v postmarketOS -e /dev/disk0s1
mkdir -p /tmp/mnt
mount -t apfs /dev/disk0s1s6 /tmp/mnt
O volume /dev/disk0s1s6deve ser o novo volume “postmarketOS”. Você pode verificar isso com /System/Library/Filesystems/apfs.fs/apfs.util -p /dev/disk0s1s6.

Agora podemos transferir a imagem do sistema dentro do novo volume usando scp.

sshpass -p "alpine"  scp -P2222 -v ~/.local/var/pmbootstrap/chroot_native/home/pmos/rootfs/qemu-aarch64.img root@localhost:/tmp/mnt/
Depois disso, desmonte a partição no iPhone e coloque-a novamente no modo DFU. Os comandos a seguir executarão o kernel do Linux e acionarão o processo de inicialização em nossa sessão gráfica do usuário :)

cd /tmp
git clone https://github.com/corellium/projectsandcastle
cd projectsandcastle/loader
make
checkra1n -cpE
./load-linux ../../linux-sandcastle/arch/arm64/boot/Image.lzma ../../linux-sandcastle/dtbpack
Se você deseja reinicializar em seu sistema postmarketOS, basta executar novamente os dois últimos comandos. As alterações feitas durante a execução do sistema serão perdidas na reinicialização e ainda não são persistentes.

Acesso shell via serial USB

Como tudo isso está em estado de desenvolvimento, é conveniente ter acesso serial/shell ao sistema em execução. Portanto, você deve adicionar CONFIG_USB_G_SERIALà configuração do kernel e anexar a seguinte linha ao arquivo inittab no sistema de arquivos raiz de destino postmarketOS.

/etc/inittab
ttyGS0::respawn:/sbin/getty -n -l /bin/sh ttyGS0 9600 linux
Durante a próxima inicialização, você poderá acessar seu telefone, por exemplo, com minicom, em /dev/ttyACM0.


A partir daqui, deve ser fácil habilitar o Bluetooth e o Wifi, pois já está implementado pela Corellium em seu Kernel personalizado .

Download do Alpine Linux: https://postmarketos.org/download/

Visite a página da CYBERAPT no Facebook pra mais atualizações: https://www.facebook.com/cyberapt

Site da CYBERAPT: https://cyberaptsecurity.wordpress.com/


Uma dica*:, se vc quer fazer ligações e mandar SMS com o Linux no iPhone vc pode usar números VOIP OTP e SMS por buffer semelhantes a motores escritos em python pra mandar SMS, e sim vc vai ter um número de operadora de telemovel no Linux sem Chips pois o número vai ser VOIP OTP e os SMS por pacote de buffer.
