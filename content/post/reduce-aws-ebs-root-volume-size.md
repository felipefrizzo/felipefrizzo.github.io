+++
Tags = ["devops", "aws", "ebs", "storage"]
Categories = ["DevOps", "AWS", "EBS", "Storage", "Resize"]
date = "2020-01-29T11:02:27-03:00"
title = "Como diminuir o tamanho do volume raiz do EC2"
author = "Felipe Frizzo"
Description = ""
+++

Os volumes EBS (Elastic Block Store) da Amazon são fáceis de usar e expandir, mas são difíceis de diminuir quando o tamanho aumenta. Aqui vou deixar um dica de como diminuir o tamanho do EBS. Eu vou assumir que você queira diminuir o volume raiz da sua EC2.  

Primeiramente você vai precisar parar a instância que deseja fazer a alteração no tamanho do disco. Após essa etapa crie um snapshot do atual volume e crie um volume do tipo `General Purpose SSD` a partir deste snapshot.  

Crie outro volume do tipo `General Purpose SSD` com o tamanho desejado  

Adicione esses dois novos volumes a instância:  

* `/dev/sda1` para o atual volume  
* `/dev/xvdf` para o volume com o tamanho desejado  
* `/dev/xvdg` para o volume feito a partir do snapshot do atual volume  

*Observação*
Para os novos tipos de discos da AWS eles podem se chamar por /dev/nvme1n1 ao inves de /dev/sda1, por exemplo.  

Agora com todos os volumes adicionados ligue a instância novamente e faça acesse-a via SSH.  

Crie os seguintes diretórios para poder "montar" os discos que foram adicionados  
`mkdir /source /target`  

Crie o sistema de arquivos ext4 para o novo volume  
`mkfs.ext4 /dev/xvdf`  

"Monte" o disco para o seguinte diretorio  
`mount -t ext4 /dev/xvdf /target`  

Essa parte é bem importante, o sistema de arquivo precisa do e2label para que o linux reconheça e possa dar boot, use `e2label /dev/sda1` no principal disco da instância para ver o que deveria ser.  
`e2label /dev/xvdf cloudimg-rootfs`  

Agora monte o disco criado a partir do snapshot  
`mount -t ext4 /dev/xvdg /source`  

Copie os arquivos para o disco alvo  
`rsync -ax /source/ /target`  
`Nota: não há "/" para "/target". Além disso, pode haver alguns erros sobre links simbólicos e atributos, mas o redimensionamento ainda foi bem-sucedido`

E por último atualize as configurações do `Grub` procure e substitua o `uuid` no arquivo `/target/boot/grub/grub.cfg`  

```shell
$ blkid
/dev/sda1: LABEL="cloudimg-rootfs" UUID="651cda91-e465-4685-b697-67aa07181279" TYPE="ext4" PARTUUID="eeaf5908-01"
/dev/xvdg: LABEL="cloudimg-rootfs" UUID="651cda91-e465-4685-b697-67aa07181279" TYPE="ext4" PARTUUID="eeaf5908-01"
/dev/xvdf: LABEL="cloudimg-rootfs" UUID="370f8787-c234-487b-aea1-b4e9e794721f" TYPE="ext4"
```  

"Desmonte" os discos  
`umount /target`  
`umount /source`  

Agora volte para o console da AWS pare a instância e remova todos os volumes. Adicione o volume redimensionado na instância com o seguinte `/dev/sda1`. Inicie a instância e ele deverá "bootar"
