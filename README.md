# Troubleshooting: Partição NTFS não montada no Ubuntu

## Sobre o problema

Durante o uso de um sistema em **dual boot Ubuntu + Windows 10**, a
partição do Windows de aproximadamente **139 GB** não podia ser acessada
pelo Ubuntu.

Ao tentar abrir a unidade pelo gerenciador de arquivos, o sistema
apresentava um erro semelhante a:

``` text
Não foi possível acessar "Volume de 139 GB"

Error mounting /dev/nvme0n1p3:
wrong fs type, bad option, bad superblock,
missing codepage or helper program, or other error
```

O objetivo deste procedimento foi identificar a causa do problema e
recuperar a montagem da partição sem formatá-la ou apagar seus dados.

------------------------------------------------------------------------

## Ambiente

-   **Sistema operacional:** Ubuntu 26.04 LTS
-   **Sistema em dual boot:** Windows 10
-   **Armazenamento:** NVMe
-   **Partição afetada:** `/dev/nvme0n1p3`
-   **Sistema de arquivos:** NTFS
-   **Tamanho aproximado:** 139 GB

> Identificadores específicos da máquina, como hostname, usuário, UUIDs
> e endereços MAC, foram removidos desta documentação.

------------------------------------------------------------------------

## 1. Identificação das partições

O primeiro passo foi verificar as partições e seus sistemas de arquivos:

``` bash
lsblk -f
```

A estrutura relevante era:

``` text
nvme0n1
├─nvme0n1p1 vfat     FAT32   /boot/efi
├─nvme0n1p2
├─nvme0n1p3 ntfs             ~139 GB
├─nvme0n1p4 ntfs
└─nvme0n1p5 ext4             /
```

A partição afetada era:

``` text
/dev/nvme0n1p3
```

e estava formatada em **NTFS**.

------------------------------------------------------------------------

## 2. Verificação inicial do NTFS

Foi utilizada a ferramenta `ntfsfix` em modo de verificação, sem
alterações:

``` bash
sudo ntfsfix -n /dev/nvme0n1p3
```

Resultado relevante:

``` text
Mounting volume... OK
Processing of $MFT and $MFTMirr completed successfully.
Checking the alternate boot sector... OK
NTFS volume version is 3.1.
NTFS partition /dev/nvme0n1p3 was processed successfully.
```

### Interpretação

As estruturas básicas verificadas do NTFS estavam acessíveis.

O parâmetro `-n` foi utilizado para evitar alterações na partição
durante essa etapa.

------------------------------------------------------------------------

## 3. Tentativa de montagem manual

Foi criado um ponto de montagem:

``` bash
sudo mkdir -p /mnt/windows139
```

Em seguida, foi realizada uma tentativa de montagem utilizando o driver
`ntfs3`:

``` bash
sudo mount -t ntfs3 /dev/nvme0n1p3 /mnt/windows139
```

A tentativa falhou com uma mensagem genérica de erro de montagem.

------------------------------------------------------------------------

## 4. Análise dos logs do kernel

Para identificar a causa real, foi consultado o log do kernel:

``` bash
sudo dmesg | tail -50
```

Entre as mensagens encontradas estavam:

``` text
ntfs3(nvme0n1p3): It is recommended to use chkdsk.
ntfs3(nvme0n1p3): volume is dirty and "force" flag is not set!
```

### Diagnóstico

A mensagem `volume is dirty` indicava que o volume NTFS estava marcado
como inconsistente e que o próprio driver `ntfs3` recomendava a execução
do `chkdsk` no Windows.

Nesse ponto, não foi utilizada montagem forçada.

------------------------------------------------------------------------

## 5. Identificação da unidade no Windows

No Windows, foi utilizado o PowerShell para confirmar qual letra de
unidade correspondia à partição:

``` powershell
Get-Volume | Format-Table DriveLetter, FileSystemLabel, FileSystem, Size, HealthStatus
```

A unidade correspondente apresentava aproximadamente:

``` text
DriveLetter : C
FileSystem  : NTFS
Size        : 139259277312
HealthStatus: Warning
```

Também foi verificado o status do BitLocker:

``` powershell
manage-bde -status C:
```

O resultado indicou que a unidade estava:

``` text
Totalmente Descriptografado
Status de Proteção: Proteção Desativada
Status do Bloqueio: Desbloqueado
```

### Conclusão dessa etapa

O BitLocker não era a causa do problema.

Além disso, o Windows indicava que o volume precisava de reparo:

``` text
OperationalStatus : Full Repair Needed
HealthStatus      : Warning
```

------------------------------------------------------------------------

## 6. Tentativa de verificação no Windows

Foi inicialmente tentada uma verificação enquanto o Windows estava em
execução:

``` powershell
chkdsk C: /scan
```

Porém, a execução foi bloqueada pelo sistema com:

``` text
Acesso negado
```

Como o volume do sistema estava marcado como necessitando de reparo, o
procedimento foi transferido para o **Ambiente de Recuperação do
Windows**.

------------------------------------------------------------------------

## 7. Reparo com CHKDSK

No Ambiente de Recuperação do Windows, foi executado:

``` cmd
chkdsk C: /f
```

O resultado final informou:

``` text
Não há problemas no sistema de arquivos.
Nenhuma ação necessária.
```

Também foi apresentado:

``` text
0 KB em setores defeituosos.
```

### Interpretação

O `chkdsk` concluiu a verificação do sistema de arquivos e não encontrou
problemas que exigissem uma nova ação.

A mensagem relacionada à transferência de mensagens para o log de
eventos não impediu a conclusão da verificação do sistema de arquivos.

------------------------------------------------------------------------

## 8. Teste após o reparo

De volta ao Ubuntu, a partição foi montada novamente:

``` bash
sudo mount -t ntfs3 /dev/nvme0n1p3 /mnt/windows139
```

Dessa vez, o comando terminou sem apresentar erro.

O `dmesg` também não apresentou novamente a mensagem:

``` text
volume is dirty
```

A partição passou a estar acessível pelo sistema.

------------------------------------------------------------------------

## 9. Validação

Para verificar o conteúdo da partição:

``` bash
ls -la /mnt/windows139
```

Também é possível confirmar o ponto de montagem com:

``` bash
findmnt /dev/nvme0n1p3
```

------------------------------------------------------------------------

## Resultado final

  Etapa                       Resultado
  --------------------------- --------------------------------------
  Identificação da partição   `/dev/nvme0n1p3`
  Sistema de arquivos         NTFS
  Tamanho                     \~139 GB
  Verificação inicial         Estruturas básicas acessíveis
  Diagnóstico do kernel       Volume marcado como `dirty`
  BitLocker                   Descartado como causa
  Estado no Windows           `Full Repair Needed`
  Reparo                      `chkdsk C: /f`
  Resultado do CHKDSK         Sem problemas no sistema de arquivos
  Montagem após o reparo      Funcionando

------------------------------------------------------------------------

## Conclusão

O problema de montagem não era causado por um sistema de arquivos
desconhecido, BitLocker ou simplesmente pelo gerenciador de arquivos do
Ubuntu.

O diagnóstico realizado pelo `dmesg` mostrou que a partição NTFS estava
marcada como **dirty** e recomendava o uso do `chkdsk`.

O procedimento utilizado foi:

``` text
Identificação
    ↓
Verificação do NTFS
    ↓
Tentativa de montagem
    ↓
Análise do dmesg
    ↓
Identificação do volume como "dirty"
    ↓
Verificação no Windows
    ↓
CHKDSK no Ambiente de Recuperação
    ↓
Nova montagem no Ubuntu
    ↓
Partição acessível
```

O caso demonstra um processo básico de **troubleshooting de sistemas**,
utilizando evidências do sistema operacional para identificar a causa
antes de realizar alterações.

------------------------------------------------------------------------

## Comandos utilizados

### Ubuntu

``` bash
lsblk -f

sudo ntfsfix -n /dev/nvme0n1p3

sudo mkdir -p /mnt/windows139

sudo mount -t ntfs3 /dev/nvme0n1p3 /mnt/windows139

sudo dmesg | tail -50

ls -la /mnt/windows139

findmnt /dev/nvme0n1p3
```

### Windows

``` powershell
Get-Volume | Format-Table DriveLetter, FileSystemLabel, FileSystem, Size, HealthStatus

manage-bde -status C:
```

### Windows --- Ambiente de Recuperação

``` cmd
chkdsk C: /f
```

------------------------------------------------------------------------

## Observação de segurança

Durante o diagnóstico, não foram utilizados comandos de formatação nem
ferramentas de reparo destrutivas no Linux.

Também não foi utilizada a opção de montagem forçada do NTFS antes da
identificação da causa.

Isso reduz o risco de alterar ou corromper dados enquanto o estado do
sistema de arquivos ainda não estava compreendido.

------------------------------------------------------------------------

## Aprendizados

Este troubleshooting permitiu praticar:

-   Identificação de partições com `lsblk`
-   Identificação de sistemas de arquivos
-   Montagem manual de volumes NTFS
-   Uso do driver `ntfs3`
-   Análise de mensagens do kernel com `dmesg`
-   Diagnóstico de volumes NTFS
-   Uso básico do `ntfsfix`
-   Identificação de volumes Windows pelo PowerShell
-   Verificação do estado do BitLocker
-   Uso do `CHKDSK`
-   Utilização do Ambiente de Recuperação do Windows
-   Processo de diagnóstico baseado em evidências
