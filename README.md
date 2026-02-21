# **🐳** Docker: O Guia Essencial para Desenvolvedores (Do Conceito à Prática)

O Docker revolucionou a forma como desenvolvemos e entregamos software. Ele resolve um dos maiores pesadelos da nossa área: o famoso *"na minha máquina funciona"*.

Neste artigo, vamos explorar o que é o Docker, como ele funciona "por baixo do capô" e colocar a mão na massa com Dockerfiles e Docker Compose.

---

## 1. O que é Docker?

O Docker é uma ferramenta de virtualização de nível de sistema operacional, processo que chamamos de **containerização**. Diferente das máquinas virtuais tradicionais (VMs) que carregam um sistema operacional inteiro, o Docker isola a aplicação em **containers** que compartilham o mesmo Kernel do sistema operacional hospedeiro (seja Linux ou Windows via WSL).

**Por que utilizar?**
O principal motivo é a facilidade. Com Docker, não é necessário instalar dezenas de dependências ou configurar variáveis de ambiente complexas na sua máquina local. Basta definir um `Dockerfile`, e a aplicação rodará exatamente da mesma forma no seu computador, no computador do colega ou no servidor de produção.

### Arquitetura e Isolamento

Como o Docker utiliza o mesmo Kernel do host, ele é extremamente leve. O isolamento é garantido através de **Namespaces** (que separam o que o container "vê", como processos e rede) e **Cgroups** (que limitam recursos como CPU e RAM).

---

## 2. Imagens vs. Containers

Para entender Docker, precisamos distinguir dois conceitos fundamentais:

- **Imagens:** São pacotes estáticos (*read-only*) que contêm todo o ambiente, código e dependências. É como se fosse a "Classe" na programação orientada a objetos ou o código-fonte da infraestrutura.
- **Containers:** São as instâncias de execução de uma imagem. É o **processo vivo**. Ao iniciar um container, o Docker cria uma camada de escrita sobre a imagem e isola esse processo.

Dessa forma, conseguimos gerar containers de bancos de dados como **PostgreSQL**, ou aplicações como **Java** e **Angular**, permitindo que todo o ecossistema suba em segundos.

---

## 3. Registries (Onde guardamos as imagens?)

Os **Registries** são repositórios de imagens. O mais famoso é o **Docker Hub** (similar ao GitHub, mas para imagens). Empresas como AWS e Google também possuem seus próprios registries.
Basicamente, você constrói sua imagem localmente e faz o *push* para o Docker Hub, permitindo que qualquer outro servidor faça o *pull* e rode sua aplicação imediatamente.

---

## 4. Networking: Como os containers conversam?

O Docker possui um sistema de redes internas para garantir segurança.

- **Isolamento:** Se criarmos um `ContainerA` na `RedeA` e um `ContainerB` na `RedeB`, eles **não conseguem se enxergar**, garantindo segurança total.
- **Comunicação:** Apenas containers na mesma rede conseguem se comunicar diretamente pelo nome do container (DNS interno).

**Atenção ao Conflito de Portas:**
Dentro da rede do Docker, dois containers podem rodar na porta 8080 tranquilamente, pois têm IPs internos diferentes. O conflito só acontece se tentarmos mapear ambos para a **mesma porta da sua máquina real (host)**.

---

## 5. Persistência de Dados (Volumes)

Um detalhe crucial: Containers são **efêmeros**. Se você deletar um container de banco de dados, todos os dados somem.
Para resolver isso, usamos **Volumes**. Um volume é um diretório mapeado da sua máquina para dentro do container, garantindo que os dados do seu PostgreSQL ou MongoDB persistam mesmo se o container for destruído.

---

## 6. Mão na Massa: Criando um Dockerfile

O `Dockerfile` é a receita de bolo da sua imagem. Abaixo, um exemplo profissional para uma aplicação **Java Spring Boot**, utilizando uma técnica chamada **Multi-stage Build**.

**Por que Multi-stage?**
Note que usamos `AS build` na primeira etapa. Isso nos permite compilar o projeto com o Maven (que é pesado), mas na imagem final copiamos *apenas* o arquivo `.jar` gerado, descartando o código-fonte e o Maven. Isso torna a imagem final muito mais leve e segura.

Dockerfile

```docker
# 1. Estágio de Build (Cozinha)
FROM maven:3.9.6-eclipse-temurin-21 AS build

WORKDIR /app

# Copia apenas o pom.xml primeiro para baixar as dependências e ganhar tempo no cache
COPY pom.xml .
RUN mvn dependency:go-offline

# Copia o código fonte e gera o SNAPSHOT (o arquivo .jar)
COPY src ./src
RUN mvn clean package -DskipTests

# 2. Estágio de Execução (O que vai rodar no servidor)
FROM eclipse-temurin:21-jre-jammy

WORKDIR /app

# O Maven gera o arquivo com o nome: artifactId-version.jar
COPY --from=build /app/target/stock-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 🧩 6.1. O Arquivo docker-compose.yml

Enquanto o `Dockerfile` define a "receita" para construir **uma única imagem**, o `docker-compose.yml` (que também deve ficar na raiz do projeto) é o maestro responsável por orquestrar **múltiplos containers** que precisam trabalhar em conjunto.

Imagine que o seu projeto de portfólio — como a aplicação de gerenciamento de livros que você está desenvolvendo — não é apenas código Java. Ele precisa de um banco de dados **PostgreSQL** e, talvez, um serviço de cache ou mensageria. Em vez de subir cada um manualmente, o Compose une todos em um único ecossistema.

### Estrutura Prática (Exemplo Full-Stack)

Aqui está como o arquivo organiza os serviços para que eles se comuniquem automaticamente:

```docker
services:
  db: # Configurações do Banco de Dados
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: library_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5437:5432" # Mapeia a porta 5437 do PC para a 5432 do Docker (evita conflitos locais)

  app: # Configurações da nossa Aplicação Spring Boot
    build: . # Indica que o Dockerfile está na mesma pasta
    container_name: library-api
    ports:
      - "8080:8080"
    environment: # Variáveis de conexão com o banco e OAuth2 do Google
      - DATASOURCE_URL=jdbc:postgresql://db:5432/library_db
      - DATASOURCE_USERNAME=postgres
      - DATASOURCE_PASSWORD=postgres
      - GOOGLE_CLIENT_ID=${GOOGLE_ID} # Valor lido do arquivo .env por segurança
      - GOOGLE_CLIENT_SECRET=${GOOGLE_SECRET}
      - SPRING_PROFILES_ACTIVE=production
    depends_on:
      - db # Garante que a API só suba após o banco estar pronto
```

OBS: antes disso o DockerFile ja tem que estar configurado.

logo apos a configuracao de ambos so rodar o 

```docker
docker-compose up --build -d
```

para compilar o Docker-compose

### ## Organização dos Arquivos no Projeto

```docker
meu-projeto/
│
├── src/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
└── .dockerignore
```

🐳Dockerfile

Fica dentro da raiz da aplicação.

Ele é responsável por:

- Definir como a imagem será construída
- Instalar dependências
- Copiar código
- Gerar o artefato (.jar, por exemplo)
- Definir o comando inicial do container

Ele constrói **UMA imagem para UMA aplicação**.

Exemplo de uso:

```
docker build -t minha-api:1.0 .
```

## 🧩 docker-compose.yml

Também fica na raiz do projeto.

Ele é usado quando temos **múltiplos containers**.

Exemplo clássico:

- API Spring Boot
- PostgreSQL
- RabbitMQ

Ao invés de rodar 3 comandos `docker run`, usamos um único:

```
docker-compose up
```

### Gerando e Rodando a Imagem

Para transformar esse arquivo em uma imagem:

Bash

```docker
docker build -t minha-api-java:1.0 .
```

Para rodar a aplicação:

Bash

```docker
docker run -p 8080:8080 minha-api-java:1.0
```

### Entendendo o Mapeamento de Portas (`p`)

O comando `-p 8080:8081` cria um túnel:

- **8080 (Esquerda):** Porta do seu computador (Host).
- **8081 (Direita):** Porta interna do container.
Isso significa que você acessará a aplicação no navegador via `localhost:8080`, mas o tráfego será redirecionado para a porta 8081 do container.

---

## 7. Docker Compose: A Orquestração

Em ambientes profissionais com microsserviços, subir container por container é inviável. O **Docker Compose** nos permite definir múltiplos containers (API, Banco, Mensageria) em um único arquivo YAML.

**Dica de Ouro (`depends_on`):**
No Compose, podemos usar a instrução `depends_on` para garantir que nossa API Java só inicie depois que o container do Banco de Dados estiver pronto, evitando erros de conexão na inicialização.

---

## Cheat Sheet: Principais Comandos

Aqui está um resumo rápido para o seu dia a dia:

### Gerenciamento de Imagens

- `docker build .` - Constrói uma imagem a partir do Dockerfile.
- `docker pull [imagem]` - Baixa uma imagem do Docker Hub.
- `docker images` - Lista imagens locais.
- `docker rmi [imagem]` - Remove uma imagem.

### Gerenciamento de Containers

- `docker run [imagem]` - Cria e inicia um container.
    - `d`: Roda em background.
    - `p`: Mapeia portas.
    - `v`: Monta volumes.
    - `-name`: Nomeia o container.
- `docker ps` - Lista containers rodando.
- `docker ps -a` - Lista todos os containers (incluindo parados).
- `docker stop [container]` - Para um container.
- `docker rm [container]` - Remove um container.
- `docker logs [container]` - Vê os logs da aplicação.
- `docker exec -it [container] bash` - Entra no terminal do container (use `sh` se `bash` falhar).

### Networking

- `docker network ls` - Lista as redes.
- `docker network create [nome]` - Cria uma rede.
- `docker network connect [rede] [container]` - Conecta container à rede.

### Docker Compose

- `docker-compose up -d` - Sobe todo o ambiente em background.
- `docker-compose down` - Derruba e remove tudo.
- `docker-compose logs -f` - Acompanha os logs de todos os serviços.

---

### Docker Hub (Publicando sua imagem)

1. Login: `docker login`
2. Tag (se necessário): `docker tag imagem-local usuario/imagem:versao`
3. Enviar: `docker push usuario/imagem:versao`
