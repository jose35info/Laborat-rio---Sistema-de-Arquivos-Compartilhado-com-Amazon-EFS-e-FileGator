# Laboratório AWS --- Sistema de Arquivos Compartilhado com Amazon EFS e FileGator

![AWS](https://img.shields.io/badge/Cloud-AWS-orange) ![Amazon
Linux](https://img.shields.io/badge/OS-Amazon%20Linux%202023-blue)
![Storage](https://img.shields.io/badge/Storage-Amazon%20EFS-green)
![Access](https://img.shields.io/badge/Access-SSM%20Session%20Manager-purple)

## Visão geral

Este projeto implementa um ambiente com duas instâncias Amazon EC2
executando Apache, PHP e FileGator, conectadas ao mesmo sistema de
arquivos compartilhado Amazon EFS.

O objetivo é demonstrar como duas instâncias de aplicação podem acessar
os mesmos arquivos, permitindo que um arquivo enviado por uma instância
seja lido pela outra sem replicação manual.

O acesso administrativo às instâncias foi realizado pelo **AWS Systems
Manager Session Manager**, sem necessidade de SSH ou exposição de uma
porta SSH.

## Objetivos do laboratório

-   Criar e configurar um sistema de arquivos Amazon EFS.
-   Conectar duas instâncias EC2 ao mesmo EFS.
-   Configurar conectividade NFS pela porta TCP 2049.
-   Instalar e validar Apache e PHP.
-   Disponibilizar o FileGator nas duas instâncias.
-   Compartilhar o diretório de arquivos por meio do EFS.
-   Diagnosticar e corrigir problemas de rede, montagem e HTTP 403.
-   Validar o compartilhamento com upload de um arquivo pelo FileGator.

## Arquitetura

``` text
                         Usuário / Navegador
                                  |
                                  | HTTP (laboratório)
                    +-------------+-------------+
                    |                           |
             +------v------+             +------v------+
             | EC2-Web-01  |             | EC2-Web-02  |
             | Apache      |             | Apache      |
             | PHP         |             | PHP         |
             | FileGator   |             | FileGator   |
             +------+-----+             +------+------+
                    |                           |
                    |       NFS 4.1             |
                    +-------------+-------------+
                                  |
                         TCP 2049 / NFS
                                  |
                          +-------v-------+
                          |  Amazon EFS   |
                          | /arquivos     |
                          +---------------+
```

As duas instâncias montam o mesmo sistema de arquivos no caminho
`/mnt/efs`. O diretório compartilhado utilizado pelo FileGator é
`/mnt/efs/arquivos`.

## Serviços e recursos utilizados

  Recurso                       Configuração
  ----------------------------- -------------------------------------
  Região AWS                    `us-east-1`
  Sistema operacional           Amazon Linux 2023
  Armazenamento compartilhado   Amazon EFS
  ID do EFS                     `fs-0f5d109ed9288e8bd`
  Protocolo de montagem         NFS 4.1
  Porta NFS                     TCP 2049
  EC2-Web-01                    `172.31.15.112`
  EC2-Web-02                    `172.31.84.127`
  Security Group das EC2        `SG-EC2-JoseAntonioGLuz`
  Security Group do EFS         `SG-EFS-JoseAntonioGLuz`
  Acesso administrativo         AWS Systems Manager Session Manager
  Aplicação web                 Apache, PHP e FileGator
  Ponto de montagem             `/mnt/efs`
  Diretório compartilhado       `/mnt/efs/arquivos`

> Os endereços IP, nomes de recursos e identificadores acima documentam
> o laboratório original. Em outro ambiente, esses valores serão
> diferentes.

## Pré-requisitos

-   Conta AWS com permissões para criar EC2, EFS, Security Groups e IAM
    Roles.
-   Duas instâncias EC2 com conectividade de rede até os mount targets
    do EFS.
-   IAM Role associada às instâncias com a política
    `AmazonSSMManagedInstanceCore`.
-   Session Manager funcionando nas duas instâncias.
-   Regras de Security Group permitindo NFS entre as instâncias e o EFS.
-   Pacotes `nfs-utils`, Apache e PHP.

## Configuração de rede e segurança

A comunicação entre as instâncias e o EFS utiliza NFS na porta TCP 2049.

### Security Group das instâncias EC2

Foi necessário permitir tráfego de saída:

  Tipo   Protocolo     Porta Destino
  ------ ----------- ------- -----------------------
  NFS    TCP            2049 Security Group do EFS

### Security Group do EFS

Foi utilizada uma regra de entrada:

  Tipo   Protocolo     Porta Origem
  ------ ----------- ------- ------------------------
  NFS    TCP            2049 Security Group das EC2

**Importante:** referenciar o Security Group das instâncias como
origem/destino evita depender de endereços IP privados individuais.

## Etapas de implementação

### 1. Acesso às instâncias pelo Session Manager

As duas instâncias foram acessadas pelo AWS Systems Manager Session
Manager, sem SSH.

A Role IAM associada às instâncias foi:

`Role-EC2-SSM-JoseAntonioGLuz`

Política utilizada:

`AmazonSSMManagedInstanceCore`

### 2. Verificar o cliente NFS

Nas instâncias Amazon Linux 2023:

``` bash
rpm -q nfs-utils
```

Se não estiver instalado:

``` bash
sudo dnf install -y nfs-utils
```

### 3. Criar o ponto de montagem

Executar nas duas instâncias:

``` bash
sudo mkdir -p /mnt/efs
```

### 4. Verificar a resolução DNS do EFS

``` bash
getent hosts fs-0f5d109ed9288e8bd.efs.us-east-1.amazonaws.com
```

O DNS pode resolver para endereços IP diferentes em cada instância,
conforme o mount target da zona de disponibilidade.

### 5. Testar a porta NFS

O utilitário `nc` não estava instalado no ambiente. Foi utilizado o
recurso `/dev/tcp` do Bash:

``` bash
timeout 5 bash -c '</dev/tcp/ENDERECO_IP/2049' \
  && echo "PORTA 2049 ABERTA" \
  || echo "PORTA 2049 BLOQUEADA OU SEM RESPOSTA"
```

Substitua `ENDERECO_IP` pelo endereço resolvido para o mount target
correspondente.

### 6. Montar o EFS

Exemplo utilizado na EC2-Web-02:

``` bash
sudo mount -t nfs4 -o nfsvers=4.1 \
  fs-0f5d109ed9288e8bd.efs.us-east-1.amazonaws.com:/ \
  /mnt/efs
```

Na EC2-Web-01, a montagem também foi validada com:

``` bash
sudo mount -av
```

Verificar o resultado:

``` bash
df -hT | grep efs
```

O tipo de sistema de arquivos deve aparecer como `nfs4`.

### 7. Criar o diretório compartilhado

Na EC2-Web-01:

``` bash
sudo mkdir -p /mnt/efs/arquivos
sudo chmod 777 /mnt/efs/arquivos
```

A permissão `777` foi utilizada somente para facilitar o laboratório.
Não é recomendada para produção.

### 8. Instalar e validar Apache e PHP

Verificar os pacotes:

``` bash
rpm -q httpd php
```

Na EC2-Web-02, o Apache já estava instalado, mas o PHP não. Foi
executado:

``` bash
sudo dnf install -y php php-json php-mbstring php-xml php-gd php-fpm
```

Verificar os serviços:

``` bash
sudo systemctl status httpd
php -v
```

Se necessário, iniciar e habilitar o Apache:

``` bash
sudo systemctl enable --now httpd
```

### 9. Configurar o FileGator

O FileGator já estava presente em `/var/www/filegator` nas duas
instâncias.

O diretório de arquivos do FileGator foi vinculado ao EFS:

``` bash
sudo ln -s /mnt/efs/arquivos /var/www/filegator/repository
```

Se o link já existir, não execute o comando novamente sem antes
verificar o destino:

``` bash
ls -lah /var/www/filegator/
readlink -f /var/www/filegator/repository
```

O destino esperado é:

``` text
/mnt/efs/arquivos
```

### 10. Configurar o VirtualHost do Apache

Na EC2-Web-02, a ausência da configuração do VirtualHost causou resposta
HTTP 403.

Foi criado `/etc/httpd/conf.d/filegator.conf` com o conteúdo:

``` apache
<VirtualHost *:80>
    DocumentRoot /var/www/filegator/dist

    <Directory /var/www/filegator/dist>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/filegator_error.log
    CustomLog /var/log/httpd/filegator_access.log combined
</VirtualHost>
```

Validar a configuração:

``` bash
sudo apachectl configtest
```

Resultado esperado:

``` text
Syntax OK
```

Reiniciar o Apache:

``` bash
sudo systemctl restart httpd
sudo systemctl status httpd
```

Testar localmente:

``` bash
curl -I http://localhost
```

Resultado esperado:

``` text
HTTP/1.1 200 OK
```

## Problemas encontrados e soluções

  -------------------------------------------------------------------------
  Problema                  Causa identificada      Solução
  ------------------------- ----------------------- -----------------------
  EFS não aparecia no       Sistema de arquivos não Verificar rede e
  `df -hT`                  estava montado          executar a montagem NFS

  Conexão TCP 2049 falhava  Saída do Security Group Adicionar saída TCP
  na EC2-Web-01             da EC2 não permitia NFS 2049 com destino ao SG
                                                    do EFS

  `nc: command not found`   Ferramenta `nc` não     Usar teste Bash com
                            estava instalada        `/dev/tcp`

  EC2-Web-02 não listava o  EFS ainda não estava    Montar o EFS em
  EFS                       montado nessa instância `/mnt/efs`

  PHP ausente na EC2-Web-02 PHP não estava          Instalar os pacotes PHP
                            instalado               com `dnf`

  HTTP 403 no FileGator     VirtualHost específico  Criar `filegator.conf`,
                            do FileGator estava     validar e reiniciar
                            ausente                 Apache

  Arquivo novo não aparecia Ainda não havia sido    Enviar arquivo pelo
  no teste inicial          feito um novo upload    FileGator e conferir na
                            para validar o          outra EC2
                            compartilhamento        
  -------------------------------------------------------------------------

### Diagnóstico da conectividade NFS

Na EC2-Web-01, o teste para o mount target `172.31.15.118` inicialmente
falhou. A análise das regras mostrou que o Security Group permitia saída
TCP 80 e 443, mas não TCP 2049.

Após permitir NFS TCP 2049 com destino ao Security Group do EFS, o teste
retornou:

``` text
PORTA 2049 ABERTA
```

Em seguida, a montagem foi concluída com sucesso.

Na EC2-Web-02, o DNS do EFS resolveu para `172.31.84.6`. O teste da
porta 2049 também confirmou conectividade.

## Validação do compartilhamento

O teste final foi realizado com um upload pelo FileGator e a leitura do
arquivo pela segunda instância.

Na EC2-Web-02:

``` bash
ls -lah /mnt/efs/arquivos/
```

O arquivo abaixo apareceu no diretório compartilhado:

``` text
01-conteudo-do-curso.pdf
```

Tamanho observado: aproximadamente 380 KB.

Isso confirmou que o arquivo enviado pela aplicação estava armazenado no
EFS e podia ser acessado pela outra instância EC2.

## Checklist de validação

-   [x] Duas instâncias EC2 criadas e acessíveis pelo Session Manager.
-   [x] IAM Role com `AmazonSSMManagedInstanceCore`.
-   [x] Amazon EFS criado.
-   [x] Security Groups configurados para NFS TCP 2049.
-   [x] Cliente NFS disponível.
-   [x] EFS montado na EC2-Web-01.
-   [x] EFS montado na EC2-Web-02 durante a sessão.
-   [x] Apache ativo nas duas instâncias.
-   [x] PHP instalado nas duas instâncias.
-   [x] FileGator presente nas duas instâncias.
-   [x] Diretório do FileGator apontando para o EFS.
-   [x] HTTP 200 validado nas duas instâncias.
-   [x] Upload de arquivo validado e arquivo visível na EC2-Web-02.
-   [ ] Configurar e validar montagem persistente via `/etc/fstab` na
    EC2-Web-02.
-   [ ] Reiniciar as duas instâncias e validar a montagem automática.
-   [ ] Fazer upload pela EC2-Web-02 e confirmar leitura na EC2-Web-01.

## Melhorias recomendadas para produção

Este projeto é um laboratório educacional. Antes de utilizá-lo em
produção:

1.  Substituir `chmod 777` por permissões e propriedade restritas,
    adequadas ao usuário do Apache e à estratégia de acesso.
2.  Utilizar HTTPS com certificado TLS.
3.  Restringir o acesso HTTP/HTTPS aos clientes necessários.
4.  Manter NFS TCP 2049 permitido somente entre os Security Groups
    necessários.
5.  Configurar montagem persistente e testar o comportamento após
    reinicializações.
6.  Avaliar backups do EFS com AWS Backup e políticas de retenção.
7.  Proteger credenciais e configurações do FileGator.
8.  Monitorar disponibilidade, capacidade, throughput e custos.
9.  Avaliar criptografia em repouso e em trânsito conforme os requisitos
    do ambiente.

## O que foi aprendido

-   Integração entre Amazon EC2 e Amazon EFS.
-   Conceitos de armazenamento compartilhado e NFS 4.1.
-   Diagnóstico de conectividade por Security Groups.
-   Uso de AWS Systems Manager Session Manager sem SSH.
-   Administração de Linux com comandos de terminal.
-   Instalação e validação de Apache e PHP.
-   Configuração de VirtualHost no Apache.
-   Diagnóstico de erro HTTP 403.
-   Uso de links simbólicos para integrar aplicações a armazenamento
    compartilhado.
-   Testes práticos para comprovar compartilhamento de arquivos entre
    instâncias.

## Conclusão

O laboratório demonstrou como duas instâncias EC2 podem utilizar um
único sistema de arquivos Amazon EFS para compartilhar arquivos. Após
corrigir a regra de saída NFS na EC2-Web-01 e configurar o VirtualHost
ausente na EC2-Web-02, o FileGator respondeu com HTTP 200 nas duas
instâncias. O upload de `01-conteudo-do-curso.pdf` e sua visualização na
segunda EC2 comprovaram o compartilhamento.

O próximo passo é configurar a montagem persistente na EC2-Web-02 e
validar a recuperação automática após reinicialização.

------------------------------------------------------------------------

**Projeto de laboratório AWS --- EC2, Amazon EFS, Linux, Apache, PHP,
FileGator, IAM e Systems Manager.**
