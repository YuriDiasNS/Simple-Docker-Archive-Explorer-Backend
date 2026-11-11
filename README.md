[README.md](https://github.com/user-attachments/files/34698158/README.md)
# Simple Docker Archive Explorer — Backend

API REST para **navegar, inspecionar, baixar, enviar e remover arquivos** dentro de um diretório isolado (`workdir`), pensada para rodar em container Docker e expor com segurança um volume de arquivos.

![Java 21](https://img.shields.io/badge/Java-21-orange?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5.13-6DB33F?logo=springboot&logoColor=white)
![Maven](https://img.shields.io/badge/Maven-3.9-C71A36?logo=apachemaven&logoColor=white)
![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-85EA2D?logo=swagger&logoColor=black)

---

## Sumário

- [Visão geral](#visão-geral)
- [Funcionalidades](#funcionalidades)
- [Tecnologias](#tecnologias)
- [Arquitetura](#arquitetura)
- [Segurança](#segurança)
- [Começando](#começando)
- [Configuração](#configuração)
- [Endpoints](#endpoints)
- [Exemplos de uso](#exemplos-de-uso)
- [Estrutura do projeto](#estrutura-do-projeto)

---

## Visão geral

O backend expõe um único diretório raiz (por padrão `./workdir`) e permite operar sobre ele por HTTP. Todo caminho recebido do cliente passa por uma camada de validação que impede qualquer acesso fora dessa raiz, o que inclui *path traversal*, caminhos absolutos e *symlinks* maliciosos.

Ele funciona como a base para um explorador de arquivos web: o frontend consome a árvore de diretórios, mostra metadados e dispara downloads, uploads e remoções.

## Funcionalidades

- **Listagem em árvore** do diretório raiz, de forma recursiva
- **Metadados** de arquivos, diretórios e symlinks (tipo, MIME, tamanho, se é executável, se aponta para fora da raiz)
- **Download** de arquivos com limite de tamanho configurável (padrão: 50 MB)
- **Upload** multipart, com criação automática de subdiretórios e aviso de sobrescrita
- **Remoção** de arquivos e diretórios, inclusive recursiva
- **Respostas padronizadas** (`success`, `message`, `data`) e tratamento global de erros
- **Documentação OpenAPI / Swagger UI**

## Tecnologias

| Camada | Tecnologia |
| --- | --- |
| Linguagem | Java 21 |
| Framework | Spring Boot 3.5.13 (Web, Validation, Actuator) |
| Documentação | springdoc-openapi 2.8.15 (Swagger UI) |
| Utilitários | Apache Commons IO 2.15.1, Apache Tika Core 2.9.2 |
| Build | Maven (Maven Wrapper incluso) |

## Arquitetura

O projeto segue uma separação em camadas, com interfaces isolando o acesso ao sistema de arquivos:

```
Controller  →  Service  →  PathSanitizer (validação)  →  SecureFileSystem / SecureFileManager
 (HTTP)       (regras)      (segurança de caminhos)       (I/O real em disco)
```

| Camada | Responsabilidade |
| --- | --- |
| `controller` | Rotas HTTP e montagem das respostas |
| `service` | Regras de negócio; valida e sanitiza o caminho antes de tocar no disco |
| `security` | `PathSanitizer`, com todas as verificações de caminho e symlink |
| `filesystem` | Interfaces `SecureFileSystem` (leitura) e `SecureFileManager` (escrita) e suas implementações |
| `model` | DTOs de requisição/resposta e nós da árvore de arquivos |
| `exception` | Exceções de domínio e `GlobalExceptionHandler` |
| `config` | Propriedades (`explorer.*`) e agrupamento do OpenAPI |

## Segurança

A principal preocupação do projeto é impedir que o cliente saia do diretório raiz:

- **Path traversal:** o caminho é resolvido e normalizado, e deve continuar dentro da raiz (`../` é bloqueado → `403`)
- **Caminhos absolutos** são rejeitados
- **Symlinks:**
  - bloqueados em qualquer pai do caminho em `info`, `upload` e `delete`
  - bloqueados em qualquer ponto do caminho no `download`
  - exibidos na árvore como `symlink`, sem serem seguidos
  - `info` sinaliza `pointsOutsideRoot` quando o link aponta para fora da raiz
- **Raiz protegida:** a raiz não pode ser um symlink nem ser removida
- **Download seguro:** valida o caminho real (`toRealPath`), recusa diretórios, aplica o limite de tamanho e envia com `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff` e `Cache-Control: no-store`
- **Erros genéricos:** falhas inesperadas retornam `500` sem expor detalhes internos

> **Nota:** a API não implementa autenticação nem autorização.

## Começando

### Pré-requisitos

- JDK 21+
- Não é preciso instalar o Maven: o projeto inclui o Maven Wrapper (`mvnw`)

### Executando localmente

```bash
# Clonar
git clone https://github.com/<seu-usuario>/Simple-Docker-Archive-Explorer-Backend.git
cd Simple-Docker-Archive-Explorer-Backend

# Rodar em modo desenvolvimento
./mvnw spring-boot:run          # Linux / macOS
mvnw.cmd spring-boot:run        # Windows
```

A API sobe em `http://localhost:8080`.

### Gerando o JAR

```bash
./mvnw clean package
java -jar target/simple-docker-archive-explorer-backend-1.0.0.jar
```

## Configuração

Arquivo: `src/main/resources/application.yml`

| Propriedade | Padrão | Descrição |
| --- | --- | --- |
| `explorer.root-path` | `./workdir` | Diretório raiz exposto pela API (precisa existir ao iniciar) |
| `explorer.max-download-size` | `52428800` (50 MB) | Tamanho máximo, em bytes, de um download |
| `spring.servlet.multipart.max-file-size` | `10GB` | Tamanho máximo de um arquivo enviado |
| `spring.servlet.multipart.max-request-size` | `10GB` | Tamanho máximo de uma requisição de upload |

Qualquer propriedade pode ser sobrescrita por variável de ambiente, por exemplo:

```bash
EXPLORER_ROOTPATH=/data EXPLORER_MAXDOWNLOADSIZE=104857600 java -jar target/*.jar
```

## Endpoints

Documentação interativa (Swagger UI) disponível com a aplicação em execução, em `http://localhost:8080/swagger-ui/index.html`.

| Método | Rota | Descrição | Parâmetros |
| --- | --- | --- | --- |
| `GET` | `/api/explorer` | Lista a árvore do diretório raiz | — |
| `GET` | `/api/explorer/info` | Metadados de um arquivo/diretório | `path` (query, obrigatório) |
| `GET` | `/api/explorer/download` | Baixa um arquivo | `path` (query, obrigatório) |
| `POST` | `/api/manage/upload` | Envia um arquivo | `file` (multipart, obrigatório), `path` (query, diretório de destino, opcional) |
| `DELETE` | `/api/manage/delete` | Remove arquivo ou diretório | `path` via query **ou** body JSON `{"path": "..."}` |

Todos os caminhos são **relativos à raiz** (ex.: `documents/report.pdf`).

### Formato das respostas

```json
{
  "success": true,
  "message": "Diretório listado com sucesso",
  "data": { }
}
```

Em caso de erro, `success` é `false`, `data` é `null` e `message` descreve o problema.

| Status | Quando acontece |
| --- | --- |
| `400` | Parâmetro ausente/inválido, tentativa de baixar diretório, arquivo acima do limite, remover a raiz |
| `403` | Path traversal, caminho absoluto ou symlink bloqueado |
| `404` | Arquivo ou diretório não encontrado |
| `500` | Erro interno |

## Exemplos de uso

**Listar a árvore**

```bash
curl http://localhost:8080/api/explorer
```

```json
{
  "success": true,
  "message": "Diretório listado com sucesso",
  "data": {
    "root": {
      "name": "workdir",
      "type": "directory",
      "accessible": true,
      "error": null,
      "children": [
        { "name": "notas.txt", "type": "file", "accessible": true, "error": null }
      ]
    }
  }
}
```

**Informações de um arquivo**

```bash
curl "http://localhost:8080/api/explorer/info?path=notas.txt"
```

**Download**

```bash
curl -OJ "http://localhost:8080/api/explorer/download?path=notas.txt"
```

**Upload (para a raiz ou para um subdiretório)**

```bash
curl -F "file=@./foto.png" "http://localhost:8080/api/manage/upload?path=uploads/imagens"
```

**Remoção**

```bash
# via query param
curl -X DELETE "http://localhost:8080/api/manage/delete?path=temp/antigo.txt"

# via body JSON
curl -X DELETE http://localhost:8080/api/manage/delete \
  -H "Content-Type: application/json" \
  -d '{"path": "temp/antigo.txt"}'
```

**Tentativa de path traversal (bloqueada)**

```bash
curl "http://localhost:8080/api/explorer/info?path=../../etc/passwd"
# 403 → "Tentativa de acesso fora do diretório raiz"
```

## Estrutura do projeto

```
.
├── pom.xml
├── mvnw / mvnw.cmd
├── workdir/                         # diretório raiz padrão exposto pela API
└── src/main
    ├── resources
    │   ├── application.yml
    │   └── openapi.yaml
    └── java/com/yuridiasns/archiveexplorer
        ├── SimpleDockerArchiveExplorerBackendApplication.java
        ├── config/                  # ExplorerProperties, OpenApiConfig
        ├── controller/              # FileExplorerController, FileManagerController
        ├── service/                 # FileExplorerService, FileManagerService
        ├── security/                # PathSanitizer
        ├── filesystem/              # SecureFileSystem, SecureFileManager (+ implementações)
        ├── model/                   # ApiResponse, DTOs e nós da árvore
        └── exception/               # exceções de domínio e handler global
```
