# Relatório Técnico de Implantação — Cellebrite Commander

## 1. Identificação do Projeto

| Campo | Informação |
|---|---|
| **Projeto** | Implantação Laboratorial do Cellebrite Commander |
| **Data da implantação** | 17/09/2026 |
| **Responsável** | Rodrigo Couto |
| **Finalidade** | Ambiente de testes e validação — não produção |
| **Plataforma de virtualização** | Microsoft Hyper-V |
| **Sistema operacional** | Ubuntu Server 22.04.5 LTS |

## 2. Resumo Executivo

Foi realizada a implantação do **Cellebrite Commander** em uma máquina virtual Ubuntu hospedada em Microsoft Hyper-V, com o objetivo de disponibilizar um ambiente controlado para testes e validação funcional da plataforma.

Durante a primeira execução do instalador, foi identificada uma falha relacionada à resolução de nomes do servidor. O FQDN `commander.lab.local` apontava para o endereço de loopback `127.0.1.1`, em vez do endereço de rede da máquina virtual. Essa configuração impediu que os serviços internos se comunicassem adequadamente durante o processo de bootstrap, afetando principalmente o provisionamento de componentes de autenticação e LDAP.

Após a correção da resolução local de nomes, a limpeza dos dados gerados na primeira tentativa e a reexecução do instalador, a implantação foi concluída com sucesso. Os containers essenciais ficaram operacionais e saudáveis. O bloqueio remanescente está no computador cliente: ele não consegue resolver o nome `commander.lab.local`, o que impede a conclusão do redirecionamento OAuth no navegador.

## 3. Objetivo da Implantação

O trabalho teve como objetivo instalar e validar o Cellebrite Commander em ambiente laboratorial, permitindo avaliar o comportamento da plataforma antes de qualquer eventual utilização em ambiente produtivo.

As atividades executadas abrangeram:

- Validação dos recursos da máquina virtual;
- Preparação do sistema operacional e da identidade do servidor;
- Instalação do Cellebrite Commander;
- Provisionamento e validação dos containers Docker;
- Diagnóstico de falhas de DNS e autenticação LDAP;
- Validação do portal web e do fluxo inicial de login.

## 4. Ambiente Implementado

| Componente | Configuração |
|---|---|
| Hypervisor | Microsoft Hyper-V |
| Sistema operacional | Ubuntu Server 22.04.5 LTS |
| Hostname inicial | `commander` |
| Hostname final | `commander.lab.local` |
| Endereço IP | `192.168.1.18/24` |
| Armazenamento | 210 GB |
| Processamento | 11 vCPU |
| Memória | 16 GB RAM |

O servidor foi configurado com recursos compatíveis com a finalidade laboratorial da implantação. Antes da instalação, foram verificados o sistema operacional, a capacidade de processamento, a memória disponível, o armazenamento e a conectividade de rede.

## 5. Validação Inicial da Máquina Virtual

A validação inicial confirmou que o servidor possuía os recursos planejados e estava apto a receber a aplicação.

### 5.1 Sistema operacional

```bash
hostnamectl
```

**Resultado observado:**

```text
Operating System: Ubuntu 22.04.5 LTS
```

### 5.2 Processamento

```bash
nproc
```

**Resultado observado:**

```text
11
```

### 5.3 Memória

```bash
free -h
```

**Resultado observado:** aproximadamente 15 GiB de memória RAM disponível.

### 5.4 Armazenamento

```bash
lsblk
```

**Resultado observado:**

```text
sda 210G
```

### 5.5 Rede

```bash
ip addr
```

**Resultado observado:**

```text
192.168.1.18/24
```

## 6. Configuração da Identidade do Servidor

O Cellebrite Commander depende de um nome de domínio totalmente qualificado (FQDN) consistente para a comunicação entre seus serviços e para a geração de URLs de acesso. Inicialmente, o servidor possuía apenas o hostname curto `commander`.

### 6.1 Estado inicial

```bash
hostname -f
```

**Resultado:**

```text
commander
```

### 6.2 Definição do FQDN

Foi definido o nome completo do servidor como `commander.lab.local`:

```bash
sudo hostnamectl set-hostname commander.lab.local
```

Em seguida, o arquivo `/etc/hosts` foi ajustado para associar o hostname curto e o FQDN ao servidor.

```bash
sudo nano /etc/hosts
```

**Configuração inicial:**

```text
127.0.0.1 localhost
127.0.1.1 commander
```

**Configuração aplicada inicialmente:**

```text
127.0.0.1 localhost
127.0.1.1 commander.lab.local commander
```

### 6.3 Validação da alteração

```bash
hostname
hostname -f
```

**Resultado:**

```text
commander.lab.local
commander.lab.local
```

Embora o hostname estivesse corretamente configurado, a associação do FQDN ao endereço `127.0.1.1` geraria uma falha posterior durante a instalação.

## 7. Preparação e Extração do Instalador

O instalador fornecido foi copiado para o diretório temporário do sistema para execução local.

**Arquivo recebido:**

```text
CMS_10.10.0.112.run.tar
```

**Local de armazenamento temporário:**

```text
/tmp
```

Após a confirmação da presença do arquivo, o pacote foi extraído:

```bash
cd /tmp
sudo tar -xvf CMS_10.10.0.112.run.tar
```

**Arquivo gerado:**

```text
CMS_10.10.0.112.run
```

## 8. Primeira Execução da Instalação

O instalador foi iniciado utilizando os parâmetros necessários para o ambiente de testes.

```bash
sudo ./CMS_10.10.0.112.run
```

| Parâmetro | Valor selecionado |
|---|---|
| Aceite da licença | `2 - I accept the agreement` |
| FQDN | `commander.lab.local` |
| Módulo Triage | `Y` |
| SSL | `Y` |
| Confirmação da instalação | `Y` |

Durante essa execução, o instalador interrompeu o processo com a mensagem abaixo:

```text
Installation Failed

An error occurred during installation:
please make sure that the DNS had been properly setup
```

## 9. Diagnóstico da Falha de DNS

A mensagem do instalador indicava uma possível inconsistência na resolução de nomes. Foram então realizados testes para confirmar qual endereço estava sendo associado ao FQDN configurado.

```bash
getent hosts commander.lab.local
```

**Resultado:**

```text
127.0.1.1 commander.lab.local
```

```bash
nslookup commander.lab.local
```

**Resultado:**

```text
Address: 127.0.1.1
```

```bash
ping commander.lab.local
```

**Resultado:**

```text
127.0.1.1
```

### 9.1 Causa raiz

Foi constatado que o FQDN do servidor resolvia para `127.0.1.1`, um endereço local de loopback do Ubuntu, em vez de resolver para o endereço real da interface de rede da máquina virtual:

```text
192.168.1.18
```

Essa condição impedia que componentes executados em containers se comunicassem corretamente usando o FQDN do servidor. Como consequência, o bootstrap de microsserviços e componentes de autenticação não foi concluído de forma consistente.

## 10. Correção da Resolução Local

O arquivo `/etc/hosts` foi corrigido para associar o FQDN ao endereço IP real da máquina virtual.

```bash
sudo nano /etc/hosts
```

**Configuração corrigida:**

```text
127.0.0.1 localhost
192.168.1.18 commander.lab.local commander
```

Após a alteração, foram repetidos os testes de resolução:

```bash
getent hosts commander.lab.local
```

```text
192.168.1.18 commander.lab.local
```

```bash
ping commander.lab.local
```

```text
PING commander.lab.local (192.168.1.18)
```

A partir desse momento, o nome do servidor passou a resolver para o endereço acessível pelos serviços internos e pelos equipamentos da rede local.

## 11. Verificação dos Serviços Criados na Primeira Tentativa

Mesmo com a interrupção reportada pelo instalador, parte dos serviços havia sido criada e iniciada. A verificação foi realizada com:

```bash
sudo docker ps -a
```

Foram identificados, entre outros, os seguintes containers:

- `upload-service`
- `authenticator`
- `cache-service`
- `analytics-service`
- `publish-service`
- `admin-service`
- `license-service`
- `dus-service`
- `configuration`
- `opendj`
- `ums`
- `postgres_sql`
- `elasticsearch`
- `nginx`

Essa evidência confirmou que a falha ocorreu após o início do provisionamento. Portanto, seria necessário remover os dados persistentes antes de executar novamente a instalação, evitando reutilizar uma configuração parcialmente criada com o FQDN incorreto.

## 12. Diagnóstico da Falha de Login

Foi realizada uma tentativa de acesso inicial ao portal utilizando as credenciais previstas para o ambiente.

```text
Usuário: commander
Senha: Commander123!
```

**Resultado:**

```text
Authentication Failed
```

Para verificar a origem da falha, foram analisados os logs do serviço de autenticação:

```bash
sudo docker logs authenticator --tail 100
```

**Mensagens identificadas:**

```text
User commander cannot be found in LDAP

Account commander unable to login
```

Também foi encontrada uma tentativa de conexão para o endereço incorreto:

```text
connect ECONNREFUSED 127.0.1.1:443
```

Esses registros confirmaram que alguns componentes haviam sido inicializados usando a associação incorreta do FQDN. O problema de login não era uma falha de credencial, mas uma consequência do provisionamento parcial realizado antes da correção do DNS local.

## 13. Limpeza da Instalação Parcial

Antes da nova instalação, os containers e volumes da tentativa anterior foram removidos. Essa etapa foi necessária para assegurar que os serviços seriam provisionados novamente utilizando a configuração corrigida.

### 13.1 Parada dos containers

```bash
cd /opt/Cellebrite-Mobile-Synchronization/CentralManagementSystem/scripts

sudo /usr/local/bin/docker-compose \
-f infieldManagementSetup.yml down
```

### 13.2 Remoção dos volumes

```bash
sudo bash remove-docker-volumes.sh
```

## 14. Segunda Execução da Instalação

Após a correção da resolução de nomes e a remoção da instalação parcial, o instalador foi executado novamente:

```bash
sudo ./CMS_10.10.0.112.run
```

Como os binários da aplicação permaneceram no sistema, o instalador apresentou a seguinte mensagem:

```text
Commander will be upgraded
```

Na prática, a execução reutilizou a estrutura já instalada e recriou os componentes necessários com a configuração de FQDN corrigida.

| Parâmetro | Valor aplicado |
|---|---|
| FQDN | `commander.lab.local` |
| Módulo Triage | `Y` |
| SSL | `Y` |

## 15. Conclusão da Instalação

A segunda execução foi concluída com sucesso.

```text
Setup has finished successfully
```

**Informações apresentadas pelo instalador:**

| Item | Valor |
|---|---|
| IP | `192.168.1.18` |
| Zone | `commander.lab.local` |

## 16. Validação Pós-Instalação

Após a conclusão, os serviços foram verificados para confirmar o estado operacional da plataforma.

```bash
sudo docker ps
```

Todos os containers esperados estavam em execução, com indicadores de estado como:

```text
Up
Healthy
```

Também foi validado o serviço principal do Commander:

```bash
sudo systemctl status commander.service
```

**Resultado:**

```text
Active: active (exited)
```

O estado observado é compatível com um serviço de orquestração que conclui sua rotina de inicialização e delega a operação contínua aos containers Docker.

## 17. Bloqueio Remanescente no Acesso pelo Cliente

Após o login inicial, o navegador redirecionou para a URL abaixo:

```text
https://commander.lab.local/redirect_uri
```

No computador cliente, o navegador apresentou:

```text
DNS_PROBE_FINISHED_NXDOMAIN
```

### 17.1 Análise

No servidor, o FQDN está funcionando corretamente:

```text
commander.lab.local -> 192.168.1.18
```

Contudo, o notebook corporativo utilizado no acesso não possui um registro DNS interno para esse nome. Além disso, não há permissão administrativa para incluir uma entrada local no arquivo `hosts` da estação.

Como o fluxo de autenticação OAuth redireciona o navegador para o FQDN configurado no Commander, o acesso não pode ser concluído enquanto o computador cliente não conseguir resolver esse nome.

## 18. Estado Atual da Implantação

| Item | Status | Observação |
|---|---|---|
| Ubuntu instalado | ✅ | Sistema operacional validado |
| Commander instalado | ✅ | Instalação concluída com sucesso |
| Docker operacional | ✅ | Containers provisionados |
| PostgreSQL operacional | ✅ | Serviço disponível em container |
| OpenDJ operacional | ✅ | Serviço LDAP disponível em container |
| NGINX operacional | ✅ | Camada de acesso web disponível |
| SSL configurado | ✅ | Configurado durante a instalação |
| Containers saudáveis | ✅ | Estados `Up` e `Healthy` observados |
| Portal web acessível | ✅ | Acesso inicial disponível |
| Login inicial | ✅ | Fluxo iniciado com sucesso |
| Redirecionamento OAuth | ❌ | Dependente da resolução DNS do cliente |
| Resolução DNS no cliente | ❌ | Registro ausente no notebook corporativo |

## 19. Próximos Passos Recomendados

### Opção 1 — Criar um registro DNS interno

Criar um registro DNS acessível à estação cliente:

```text
commander.lab.local -> 192.168.1.18
```

Essa é a opção recomendada quando o ambiente será utilizado por mais de uma estação ou mantido por um período maior.

### Opção 2 — Adicionar entrada no arquivo `hosts` da estação cliente

Em uma estação com permissão administrativa, adicionar:

```text
192.168.1.18 commander.lab.local
```

Essa alternativa é adequada para um teste pontual e controlado.

### Opção 3 — Utilizar uma estação de testes com privilégios locais

Utilizar uma máquina de testes que permita a alteração do arquivo `hosts` ou a configuração de DNS. Isso possibilita validar integralmente o fluxo do portal sem depender de mudanças na infraestrutura corporativa.

## 20. Conclusão

A implantação do Cellebrite Commander foi concluída com sucesso em ambiente Ubuntu 22.04 sobre Hyper-V. O principal incidente ocorreu na primeira execução, quando o FQDN `commander.lab.local` foi associado ao endereço `127.0.1.1`. Essa configuração não era adequada para a comunicação entre os componentes internos da plataforma e afetou o provisionamento de serviços LDAP e autenticação.

Após a correção da resolução local para `192.168.1.18`, a limpeza da instalação parcial e a nova execução do instalador, os serviços do Commander passaram a operar normalmente. A plataforma está instalada, os containers estão saudáveis e o portal está disponível.

O único impedimento para a validação completa do fluxo de autenticação no navegador está fora do servidor: a estação cliente precisa resolver o nome `commander.lab.local`. Com a criação de um registro DNS interno ou uma entrada no arquivo `hosts` de uma estação de testes, o ambiente estará apto para a validação funcional completa.
