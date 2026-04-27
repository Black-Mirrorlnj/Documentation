# 🎮 Garrys Mod — Score API

> Back-end REST desenvolvido em **Spring Boot** para integração com servidores de **Garry's Mod**.  
> Registra jogadores, partidas, pontuações, estatísticas, eventos de morte, ranking global e tempo de jogo em tempo real via addon Lua.

---

## 📋 Sumário

- [Visão Geral](#visão-geral)
- [Diagrama de Arquitetura](#diagrama-de-arquitetura)
- [Diagrama de Fluxo de uma Requisição](#diagrama-de-fluxo-de-uma-requisição)
- [Diagrama do Banco de Dados](#diagrama-do-banco-de-dados)
- [Módulos do Sistema](#módulos-do-sistema)
- [Endpoints da API](#endpoints-da-api)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)

---

## Visão Geral

O sistema funciona como uma ponte entre o servidor de jogo (Garry's Mod) e um banco de dados MySQL.  
Um **addon escrito em Lua** rodando dentro do servidor de jogo envia requisições HTTP para esta API sempre que eventos importantes acontecem — entrada de jogador, morte, fim de partida, pontuação, etc.

A API está dividida em **três módulos principais**, cada um responsável por um tipo de usuário do sistema:

| Módulo | Responsabilidade |
|---|---|
| **Player** | Jogadores registrados com Steam ID, partidas, pontuações, estatísticas e ranking |
| **Usuario** | Administradores e operadores do sistema com login/senha |
| **Visitante** | Jogadores anônimos sem cadastro, rastreados por sessão |

---

## Diagrama de Arquitetura

O diagrama abaixo mostra a estrutura completa do sistema, desde o cliente até o banco de dados.

```mermaid
graph TD
    GMOD["🎮 Garry's Mod Addon (Lua)\nEnvia eventos via HTTP"]

    AUTH["🔐 AutenticacaoAddonService\nValida server_key · Registra log de autenticação"]

    GMOD -->|"HTTP + Authorization header"| AUTH

    subgraph API["🌐 API REST — com.score.garrys (Spring Boot)"]

        subgraph PLAYER["📦 Módulo Player"]
            PC["Controllers\n/jogadores · /partidas · /matches\n/pontuacoes · /ranking · /estatisticas\n/eventos/morte"]
            PS["Services\nJogadorService · PartidaService\nMatchLifecycleService · PontuacaoService\nEstatisticaService · RankingGlobalService\nEventoMorteService"]
            PR["Repositories\nJogadorRepo · PartidaRepo · MatchRepo\nPontuacaoRepo · EstatisticaRepo\nRankingGlobalRepo · EventoMorteRepo\nServidorRepo · AddonRegistradoRepo"]
            PE["Entities\nJogador · Partida · Match · Pontuacao\nEstatistica · RankingGlobal · EventoMorte\nServidor · AddonRegistrado"]
            PC --> PS --> PR --> PE
        end

        subgraph USUARIO["👤 Módulo Usuario"]
            UC["Controllers\n/usuarios · /usuarios/login\n/usuarios/rounds/sync"]
            US["Services\nUsuarioService · RoundSyncService"]
            UR["Repositories\nUsuarioRepository\nLogSincronizacaoRoundRepository"]
            UE["Entities\nUsuario · LogSincronizacaoRound"]
            UC --> US --> UR --> UE
        end

        subgraph VISITANTE["👁️ Módulo Visitante"]
            VC["Controllers\n/visitantes · /visitantes/entrada\n/visitantes/saida · /visitantes/ranking\n/visitantes/servers · /visitantes/playtime"]
            VS["Services\nVisitanteService · ServerService\nPlaytimeService"]
            VR["Repositories\nVisitanteRepository · ServerRepository\nPlaytimeCheckpointRepository\nLogPlaytimeRepo · LogAuditoriaServidorRepo"]
            VE["Entities\nVisitante · Server · PlaytimeCheckpoint\nLogPlaytime · LogAuditoriaServidor"]
            VC --> VS --> VR --> VE
        end

        SHARED["🛡️ Shared\nApiResponse · GlobalExceptionHandler\nUnauthorizedException"]
    end

    AUTH --> PC
    AUTH --> UC
    AUTH --> VC

    PE -->|"JPA / Hibernate"| DB[("🗄️ MySQL\ngmod database")]
    UE -->|"JPA / Hibernate"| DB
    VE -->|"JPA / Hibernate"| DB
```

---

## Diagrama de Fluxo de uma Requisição

O fluxo abaixo mostra o que acontece quando o addon registra um **evento de morte** no servidor.

```mermaid
sequenceDiagram
    participant LUA as Garry's Mod Addon
    participant CTR as EventoMorteController
    participant AUTH as AutenticacaoAddonService
    participant SVC as EventoMorteService
    participant REP as EventoMorteRepository
    participant DB as MySQL

    LUA->>CTR: POST /eventos/morte\n{ matchId, killerId, victimId, weapon }
    CTR->>AUTH: autenticar(Authorization, "/eventos/morte")
    AUTH->>DB: SELECT server_key FROM addon_registrado
    DB-->>AUTH: server_key válida ✓
    AUTH-->>CTR: autenticado

    CTR->>SVC: registrar(dto)
    SVC->>REP: save(eventoMorte)
    REP->>DB: INSERT INTO eventos_morte
    DB-->>REP: id gerado
    REP-->>SVC: EventoMorte salvo
    SVC-->>CTR: EventoMorte
    CTR-->>LUA: 200 OK · ApiResponse { success, data }
```

---

## Diagrama do Banco de Dados

Tabelas principais e seus relacionamentos.

```mermaid
erDiagram
    jogadores {
        int id PK
        varchar steam_id UK
        varchar nome
        datetime ultimo_login
        timestamp criado_em
    }

    partidas {
        int id PK
        varchar mapa
        datetime data_inicio
        datetime data_fim
    }

    matches {
        int id PK
        int server_id FK
        varchar mapa
        varchar modo
        datetime start_timestamp
        datetime end_timestamp
        varchar status
        datetime criado_em
    }

    pontuacoes {
        int id PK
        int jogador_id FK
        int partida_id FK
        int score_inicial
        int score_final
    }

    estatisticas {
        int id PK
        int jogador_id FK
        int kills
        int deaths
        int dinheiro
        int nivel
        int experiencia
        int tempo_jogado
    }

    ranking_global {
        int jogador_id PK
        int pontos
        int posicao
    }

    eventos_morte {
        int id PK
        int match_id FK
        varchar killer_id
        varchar victim_id
        varchar weapon
        timestamp timestamp
    }

    servidores {
        int id PK
        varchar nome
        varchar server_key UK
        boolean ativo
        datetime criado_em
    }

    usuarios {
        int id PK
        varchar nome
        varchar email UK
        varchar login UK
        varchar senha
        varchar perfil
        boolean ativo
        datetime criado_em
    }

    visitante {
        int id_visitante PK
        varchar nome_usuario
        int kills
        datetime horario_entrada
        datetime horario_saida
    }

    playtime_checkpoints {
        int id PK
        varchar steam_id
        int match_id FK
        int tempo_acumulado
        varchar motivo
        datetime criado_em
    }

    servers {
        int id PK
        varchar server_name
        varchar ip
        int port
        varchar steam_id
        boolean ativo
        datetime criado_em
    }

    jogadores ||--o{ pontuacoes : "tem"
    jogadores ||--|| estatisticas : "possui"
    jogadores ||--|| ranking_global : "aparece em"
    partidas ||--o{ pontuacoes : "contém"
    servidores ||--o{ matches : "hospeda"
    matches ||--o{ eventos_morte : "registra"
    matches ||--o{ playtime_checkpoints : "gera"
```

---

## Módulos do Sistema

### 📦 Módulo Player

Responsável pelos jogadores **registrados** que possuem Steam ID.

| Classe | Responsabilidade |
|---|---|
| `JogadorService` | Cadastra jogador por Steam ID. Ao criar, inicializa automaticamente `Estatistica` e `RankingGlobal` zerados |
| `MatchLifecycleService` | Valida se o servidor está ativo e cria uma `Match` com status `in_progress`. Registra `LogMatch` |
| `PontuacaoService` | Cria pontuação por partida e finaliza atualizando kills/deaths/score na `Estatistica` |
| `EstatisticaService` | Lê e atualiza kills, deaths, dinheiro, nível e experiência de cada jogador |
| `EventoMorteService` | Registra cada morte com killer, vítima e arma usada. Protegido por autenticação de addon |
| `RankingGlobalService` | Expõe o ranking ordenado por pontos |

### 👤 Módulo Usuario

Responsável pelos **administradores** da plataforma.

| Classe | Responsabilidade |
|---|---|
| `UsuarioService` | CRUD completo com validação de login e e-mail únicos. Autenticação simples por login/senha |
| `RoundSyncService` | Sincroniza dados de rounds vindos do addon. Autenticado via `Authorization` header |

### 👁️ Módulo Visitante

Responsável pelos jogadores **anônimos** (sem cadastro).

| Classe | Responsabilidade |
|---|---|
| `VisitanteService` | Registra entrada/saída com hora e kills via stored procedures do MySQL |
| `ServerService` | Cadastra servidores de jogo com validação de admin via header `X-Admin-User` |
| `PlaytimeService` | Registra checkpoints de tempo de jogo por Steam ID |

---

## Endpoints da API

### Player

| Método | Rota | Descrição |
|---|---|---|
| `POST` | `/jogadores` | Cadastra jogador e inicializa estatísticas |
| `GET` | `/jogadores` | Lista todos os jogadores |
| `GET` | `/jogadores/{id}` | Busca jogador por ID |
| `POST` | `/matches` | Cria uma nova partida (match) |
| `POST` | `/partidas` | Cria partida simplificada |
| `GET` | `/partidas` | Lista partidas |
| `PUT` | `/partidas/{id}/finalizar` | Finaliza partida |
| `POST` | `/pontuacoes` | Cria pontuação |
| `GET` | `/pontuacoes/partida/{id}` | Lista pontuações de uma partida |
| `PUT` | `/pontuacoes/finalizar` | Finaliza pontuação com kills/deaths |
| `GET` | `/estatisticas/jogador/{id}` | Busca estatísticas de um jogador |
| `PUT` | `/estatisticas/jogador/{id}` | Atualiza estatísticas |
| `GET` | `/ranking` | Ranking global de pontos |
| `POST` | `/eventos/morte` 🔐 | Registra evento de morte em partida |

### Usuario

| Método | Rota | Descrição |
|---|---|---|
| `POST` | `/usuarios` | Cria novo usuário |
| `GET` | `/usuarios` | Lista usuários |
| `GET` | `/usuarios/{id}` | Busca por ID |
| `PUT` | `/usuarios/{id}` | Atualiza usuário |
| `DELETE` | `/usuarios/{id}` | Remove usuário |
| `POST` | `/usuarios/login` | Autenticação por login/senha |
| `POST` | `/usuarios/rounds/sync` 🔐 | Sincroniza dados de rounds |

### Visitante

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/visitantes` | Lista visitantes ativos |
| `GET` | `/visitantes/ranking` | Ranking de kills de visitantes |
| `POST` | `/visitantes/entrada` | Registra entrada no servidor |
| `PUT` | `/visitantes/saida/{id}` | Registra saída com kills |
| `DELETE` | `/visitantes/{id}` | Remove visitante |
| `POST` | `/visitantes/servers` 🔑 | Cadastra servidor (requer `X-Admin-User`) |
| `POST` | `/visitantes/playtime` 🔐 | Registra checkpoint de playtime |

> 🔐 Requer header `Authorization` com server_key válida  
> 🔑 Requer header `X-Admin-User`

---

## Tecnologias Utilizadas

| Tecnologia | Versão | Uso |
|---|---|---|
| Java | 17+ | Linguagem principal |
| Spring Boot | 3.x | Framework web |
| Spring Data JPA | 3.x | Persistência e repositórios |
| Hibernate | 6.x | ORM / mapeamento objeto-relacional |
| MySQL | 8.x | Banco de dados relacional |
| Lombok | latest | Redução de boilerplate |
| Bean Validation | 3.x | Validação de DTOs |
| Maven | 3.x | Gerenciamento de dependências |

---

## Banco de Dados

O banco `gmod` é criado em MySQL com:

- **Tabelas** mapeadas via JPA (`@Entity`)
- **Stored Procedures** para operações de Visitante (`registrar_entrada`, `registrar_saida`, `ranking_kills`)
- **Triggers** automáticos para log de ações em `jogadores` e `visitante`
- **Índices** para otimizar queries de ranking e busca por nome

---

