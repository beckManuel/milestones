# **Guia Prático: systemd, Comunicação entre Processos, Sockets, Virtualização e Docker**

---

## **1. Introdução ao systemd**

O `systemd` é um sistema de inicialização e gestor de serviços utilizado na maioria das distribuições Linux modernas.Ele é responsável por iniciar, parar e monitorizar serviços, ativar daemons e gerir dependências entre serviços, fornecendo uma maneira mais estruturada e eficiente de fazer a gestão de targets, serviços, sockets e dependências.

O systemctl é uma interface prática para interagir com o systemd, permitindo gerir o ciclo de vida de serviços, verificar dependências, configurar inicializações automáticas e depurar falhas.

**Principais Funcionalidades do systemctl**

- **Gestão de serviços**:

		•  **Iniciar**: systemctl start nome_do_serviço – Inicia um serviço manualmente.

		•  **Parar**: systemctl stop nome_do_serviço – Envia sinais (como SIGTERM) para terminar o serviço.

		•  **Reiniciar**: systemctl restart nome_do_serviço – Para e inicia novamente o serviço.

		•  **Recarregar configuração**: systemctl reload nome_do_serviço – Atualiza configurações sem reiniciar o processo.

2.  **Consulta de status**:

		•  systemctl status nome_do_serviço – Mostra o estado do serviço, logs recentes e informações como o PID.

3.  **Configuração de serviços no arranque**:

		•  **Ativar**: systemctl enable nome_do_serviço – Configura o serviço para iniciar automaticamente no arranque.

		•  **Desativar**: systemctl disable nome_do_serviço – Remove o serviço da sequência de arranque.

4.  **Gestão de dependências**:

		•  Garante que serviços dependentes são iniciados ou parados na ordem correta.

5.  **Logs e depuração**:

		•  O systemctl integra-se com o journalctl para facilitar a consulta de logs:

		•  journalctl -u nome_do_serviço – Mostra logs específicos de um serviço.

### **Funcionamento do systemd**
O `systemd` é baseado no conceito de unidades (**units**), que podem representar serviços, sockets, mounts, timers, targets, entre outros.

#### **Principais Tipos de Unidades:**
1. **Service Units (`*.service`)**:
   - Representam serviços ou daemons.
   - Exemplo: `nginx.service` para gerir o servidor NGINX.

2. **Target Units (`*.target`)**:
   - Agrupam várias unidades para atingir um estado desejado.
   - Exemplo: `multi-user.target`.

3. **Socket Units (`*.socket`)**:
   - Representam sockets para comunicação entre processos.

4. **Timer Units (`*.timer`)**:
   - Substituem o `cron`, permitindo a execução de tarefas em horários programados.

#### **Estrutura de um ficheiro de unidade (unit file)**
Os ficheiros de unidade são usados pelo `systemd` para descrever serviços, sockets, targets, entre outros. Geralmente estão localizados em:
	
	 service: Define serviços a serem iniciados e geridos.
	 socket: Gere sockets (Unix Domain Sockets ou TCP/IP) e permite a ativação baseada em sockets (Socket-based Activation).
	- /lib/systemd/system/ (configurações padrão do sistema)
	
	-  Configurações personalizadas:
	-  /etc/systemd/system/**: Para configurações criadas pelo utilizador ou administrador.
	-  /etc/nginx/: Contém ficheiros de configuração do servidor web NGINX.
	-  /opt/: Usado para instalar aplicações adicionais fora do sistema base.


#### **Fluxo de Trabalho do systemd**
1. **Definir o Target Padrão:**
   - O `systemd` começa pelo ficheiro `/etc/systemd/system/default.target`, que é um link simbólico para o target inicial.
   - Exemplo: `default.target -> multi-user.target`.

2. **Processar Dependências:**
   - O `systemd` lê o target especificado e todas as dependências associadas.
   - Apenas as unidades referenciadas por este target (e as suas dependências) são ativadas.

3. **Iniciar Unidades:**
   - O `systemd` utiliza diretivas como `After=` e `Requires=` para iniciar as unidades na ordem correta.

---
#### **O que são Targets no systemd?**
- Um **target** no `systemd` é uma forma de agrupar unidades relacionadas para alcançar um determinado estado do sistema.
- Targets são equivalentes aos **runlevels** em sistemas de inicialização mais antigos, mas mais flexíveis.

#### **Exemplo de Targets:**
- **`graphical.target`**: Representa o estado onde o sistema está pronto para um ambiente gráfico (interface gráfica).
- **`multi-user.target`**: Representa um estado em que o sistema está pronto para múltiplos utilizadores via linha de comandos.

#### **Como os Targets Gerem Unidades?**
- Targets são definidos em ficheiros de unidade (ex.: `graphical.target`).
- Cada target tem uma lista de unidades que devem ser ativadas para alcançar esse estado.
  - **`WantedBy=`**: Lista de unidades que o target deseja ativar.
  - **`RequiredBy=`**: Lista de unidades que o target precisa para funcionar (se falharem, o target falha).

•  **Diretivas Importantes**:

	• Requires=: O serviço depende obrigatoriamente de outra unidade. Se a unidade requerida falhar, o serviço também falhará.

	• Wants=: Indica dependências opcionais. Se a unidade falhar, o serviço continua.

	• After=:	Garante que o serviço só será iniciado após outra unidade estar completamente ativa.

#### **Fluxo Completo de Acontecimentos**
	1.  O sistema é ligado, e o kernel inicia o systemd.
	2.  O systemd identifica o _target_ inicial (ex.: multi-user.target).
	3.  O systemd analisa o diretório do _target_ (multi-user.target.wants/) para identificar os serviços associados.
	4.  Cada serviço é processado, com as suas dependências resolvidas (ex.: network.target).
	5.  Os serviços são iniciados na ordem correta, considerando diretivas como Requires=, Wants= e After=.
	6.  O systemd monitoriza os serviços em execução e aplica políticas de recuperação, se necessário.
---

#### **Daemons**

- São processos que correm em segundo plano, normalmente iniciados no boot do sistema. Os daemons em sistemas Linux possuem as seguintes características:
- **Execução em Background**:
	- Um daemon é projetado para operar em segundo plano, sem interação direta com um terminal de utilizador.
- **Desacoplamento de Terminais**:
	- Durante a inicialização, o daemon dissocia-se do terminal ao qual estava originalmente ligado. Isto envolve fechar os _file descriptors_ associados a stdin, stdout, e stderr e redirecioná-los (geralmente para /dev/null).
- **Execução Contínua**:
	- Os daemons operam continuamente para prestar serviços enquanto o sistema está ativo. São frequentemente iniciados no arranque do sistema e executados até serem explicitamente terminados ou até o sistema ser desligado.
-  **PID Fixado ou Registrado**:
	 - Muitos daemons registam o seu identificador de processo (PID) em ficheiros específicos, como em /var/run, para permitir que outros processos ou gestores de serviços interajam com eles.
 -  **Sem Interação Direta com Utilizadores**:
	- Os daemons não possuem interfaces diretas com os utilizadores; em vez disso, comunicam através de _sockets_, filas de mensagens, ou outros mecanismos de IPC.

-  **Exemplo**:

-  `nginx`: O daemon que processa pedidos HTTP. 

-  `nginx.service`: O serviço gerido pelo `systemd` que controla o daemon.





### **Exemplos Práticos**
#### **Configurar um Target Padrão**
Para alterar o target inicial:
```bash
sudo systemctl set-default graphical.target
```
Isto altera o link simbólico `default.target` para apontar para `graphical.target`.

#### **Ver Unidades de um Target**
Para ver as unidades associadas a um target:
```bash
systemctl list-dependencies multi-user.target
```

#### **Ativar um Serviço no Arranque**
Se quiseres que um serviço seja ativado sempre que um target específico for atingido:
1. Adicionar a unidade ao `WantedBy` de um target:
   ```bash
   sudo systemctl enable nginx.service --now
   

#### **Comandos Frequentes com systemd**

- **Ativar uma unidade:**
  ```bash
  systemctl enable nome-da-unidade
  ```
- **Iniciar/Parar/Verificar status de uma unidade:**
  ```bash
  systemctl start nome-da-unidade
  systemctl stop nome-da-unidade
  systemctl status nome-da-unidade
  ```
- **Ver logs:**
  ```bash
  journalctl -u nome-da-unidade
  ```
- **Recarregar configurações:**
  ```bash
  systemctl daemon-reload
  ```

- **Exemplo de ficheiro de serviço:**

```ini
[Unit]
Description=Exemplo de Serviço
After=network.target

[Service]
ExecStart=/usr/bin/exemplo
Restart=always

[Install]
WantedBy=multi-user.target
```
---

## **2. Comunicação entre Processos (Inter-Process Communication - IPC)**

A comunicação entre processos (IPC) é essencial para que processos separados possam trocar informações de maneira eficiente. Aqui estão as principais formas de IPC abordadas:

#### **1. Pipes**

- **Unidirecionais** e utilizados para comunicação entre processos relacionados (pai e filho).
- **Vantagem**: Simples de usar.
- **Limitação**: Não permite comunicação bidirecional.

#### **2. FIFOs (Named Pipes)**

- Extensão dos pipes, permitindo comunicação entre processos não relacionados.
- **Exemplo**: Um processo escreve para um FIFO enquanto outro lê.

#### **3. Memória Partilhada**

- Permite que múltiplos processos acessem a mesma área de memória.
- **Vantagem**: Muito eficiente para grandes volumes de dados.
- **Limitação**: Requer mecanismos de sincronização (ex.: semáforos).

#### **4. Sockets de Domínio Unix**

- Usados para comunicação entre processos na mesma máquina.
- **Forma**: Identificados por caminhos no sistema de ficheiros (ex.: `/tmp/socket`).

#### **5. Semáforos**

- Utilizados para sincronizar o acesso a recursos partilhados entre processos.
- **Vantagem**: Garante exclusividade no uso de recursos.

---

## **3. Sockets (Unix e de Rede)**

#### **Sockets de Domínio Unix**

- **Forma de endereço**: Caminhos no sistema de ficheiros.
- **Escopo**: Comunicação local (na mesma máquina).
- **Vantagem**: Mais rápidos, pois não utilizam a pilha de rede.

#### **Sockets de Rede (Internet Sockets)**

- **Forma de endereço**: Combinação de endereço IP e porta (ex.: `192.168.1.10:8080`).
- **Escopo**: Comunicação entre máquinas diferentes.
- **Vantagem**: Permite sistemas distribuídos.

---

## **4. Virtualização e Hipervisores**

#### **Tipos de Hipervisores**

- **Tipo 1 (Bare-metal)**:
  - Executa diretamente no hardware.
  - **Exemplos**: VMware ESXi, Xen, Microsoft Hyper-V.
  - **Vantagem**: Alta eficiência e desempenho.

- **Tipo 2 (Hospedado)**:
  - Executa sobre um sistema operativo hospedeiro.
  - **Exemplos**: VirtualBox, VMware Workstation.
  - **Vantagem**: Fácil configuração.

#### **Suporte de Processadores à Virtualização**

- **Intel VT-x** e **AMD-V**: Fornecem instruções específicas para acelerar operações de virtualização.
- **Benefícios**: Redução de overhead e melhoria de desempenho.

#### **Paravirtualização**

- **Definição**: Técnica onde o sistema operativo convidado é modificado para interagir diretamente com o hipervisor.
- **Vantagem**: Maior eficiência em comparação com a virtualização completa.
- **Exemplo**: Xen com suporte paravirtualizado.

---

## **5. Docker e o Sistema de Ficheiros overlay2**
#### **Definição**
O Docker é um sistema de contentores que permite a execução de aplicações isoladas no mesmo kernel do sistema host.
#### **Características dos Contentores**
1.  **Isolamento**:
- Os contentores são isolados uns dos outros, mas partilham o kernel do sistema host.
- Cada contentor tem o seu próprio sistema de ficheiros e interface de rede.
2.  **Imagens e Contentores**:
- As imagens servem como base para os contentores.
- Os contentores adicionam uma camada adicional sobre a imagem base, onde as alterações são registadas.
3.  **Processos**:
- Cada contentor vê apenas os seus próprios processos, mas o host pode visualizar todos os processos em execução.
- Configurações de namespaces garantem o isolamento entre contentores.

#### **Docker e Camadas (Layers)**

O Docker organiza imagens e contêineres em camadas de sistema de ficheiros. Cada camada é imutável (read-only), exceto a camada superior, que é de leitura-escrita (read-write) e exclusiva ao contêiner em execução.

- **Camadas Partilhadas**: Camadas idênticas são armazenadas apenas uma vez no disco, permitindo deduplicação.
- **Exemplo**: Se dois contêineres usam a mesma imagem base (`alpine:3.18`), a camada base será armazenada uma única vez.

#### **overlay2**

- É o driver de armazenamento mais comum no Docker moderno. O OverlayFS permite reaproveitar camadas para múltiplos contentores, reduzindo o consumo de espaço.

- **Funcionamento**:
  - Camadas read-only são empilhadas para formar o sistema de ficheiros da imagem.
  - A camada superior (read-write) permite que o contêiner faça alterações sem modificar as camadas inferiores.
- **Vantagens**:
  - Eficiência no uso de espaço.
  - Rápida criação de contêineres com base em imagens existentes.
 
  #### **Funcionamento**
- **Diretórios Principais**:
	- **lowerdir**: Camadas read-only fornecidas pela imagem.
	- **upperdir**: Camada read-write onde as alterações são registadas.
	- **mergedir**: Apresenta uma visão unificada do sistema de ficheiros.


#### **Componentes**
-  **dockerd**: O daemon principal do Docker que gere as operações.
-  **containerd**: Um runtime responsável pela gestão de contentores.
-  **runc**: Executa operações de baixo nível, como a criação de namespaces e configuração de isolamento.
#### **Camadas de Gestão**
-  **Alto Nível**:
	- Gere imagens e fornece assistência aos contentores. 
- **Baixo Nível**:
	- Interage com o kernel para configurar isolamento de processos, rede, etc.

#### **Dockerfiles: Estrutura e Práticas Recomendadas**
  O Dockerfile é o ficheiro de configuração que descreve os passos para construir imagens Docker. Aqui estão algumas boas práticas e explicações detalhadas:

1. **`FROM`**:
   - Define a imagem base a ser usada. Exemplo: `FROM node:alpine`.

2. **`WORKDIR`**:
   - Define o diretório de trabalho para as instruções subsequentes. Exemplo:
     ```dockerfile
     WORKDIR /home/node/app
     ```

3. **`COPY`**:
   - Copia ficheiros ou diretórios do sistema host para a imagem do contêiner. Exemplo:
     ```dockerfile
     COPY src/ ./
     ```
   - Separar o `COPY package.json` do resto dos ficheiros pode otimizar o cache do Docker:
     ```dockerfile
     COPY src/package.json ./
     RUN npm install
     COPY src/ ./
     ```

4. **`RUN`**:
   - Executa comandos na construção da imagem. Executa comandos e grava alterações na imagem final Combinar instruções relacionadas reduz o número de camadas:
     ```dockerfile
     RUN npm install && chown -R node:node .
     ```

5. **`CMD` e `ENTRYPOINT`**:
   -Define o comando a ser executado no início do contentor. Exemplo:
     ```dockerfile
     CMD ["node", "app.js"]
     ```
     
#### **Build Cache**
- Se uma instrução foi executada anteriormente, o Docker usa o cache para acelerar a construção.
#### **Camadas**:
- Cada instrução pode criar uma nova camada no sistema de ficheiros.
- Camadas são armazenadas em `/lib/docker/overlay2`.
#### **Volumes no Docker**

Os **volumes** são um mecanismo no Docker para persistir dados fora do ciclo de vida de um contentor, permitindo que os dados sejam preservados mesmo após o contentor ser removido. Eles também facilitam o compartilhamento de dados entre contentores.
Os volumes são ideais para:
- Persistência de dados (ex.: bases de dados).
- Partilha de ficheiros entre contentores.
- Facilitar backups e restaurações.

Bind mounts são úteis para:
- Aceder a diretórios específicos no host.
- Partilhar configurações ou dados do host com contentores.

#### **Como os Volumes Funcionam?**
1. **Armazenamento Externo:**
   - Os volumes são armazenados fora do sistema de ficheiros exclusivo de um contentor.
   - Ficam localizados numa área gerida pelo Docker, geralmente em `/var/lib/docker/volumes`.

2. **Persistência de Dados:**
   - Os dados escritos num volume não são apagados quando o contentor é removido.
   - Permitem que os dados sejam utilizados por novos contentores.

3. **Partilha entre Contentores:**
   - Vários contentores podem aceder ao mesmo volume, facilitando o compartilhamento de dados.

---



#### **Execução**

-  **`docker run`**: Cria e executa um novo contentor.
-  **`docker start`**: Executa um contentor já existente.
O Dockerfile é o ficheiro de configuração utilizado para construir imagens Docker. Aqui estão algumas boas práticas e explicações detalhadas:

---

## **6. Docker Compose: Organização e Práticas Recomendadas**

O Docker Compose é uma ferramenta que permite definir e executar aplicações multi-container. As aplicações são especificadas em ficheiros `docker-compose.yml`, que organizam serviços, redes e volumes.

#### **Funcionalidades**

1.  **Nomes na Rede**:
- Facilita a comunicação entre contentores usando nomes em vez de endereços IP.
2.  **Redes Personalizadas**:
- Garante isolamento entre grupos de contentores.

#### **Estrutura do Ficheiro docker-compose.yml**
Exemplo básico:

```yaml
version: "3.9"
services:
  app:
    image: node:alpine
    build:
      context: ./
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

---

## **7. Namespaces no Linux**


Os namespaces permitem particionar recursos do kernel para isolar processos:

- Cada conjunto de processos vê apenas os seus próprios recursos (processos, rede, sistema de ficheiros, etc.).
#### **Exemplo**
-  **Network Namespace**: Cada contentor pode ter uma interface de rede virtual isolada.
---
