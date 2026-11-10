# APS 3 — Dossiê de Arquitetura Viva: Componente Notificações (FabLab Manager)

**Aluno:** Nome Sobrenome · **Curso:** Engenharia de Software (4º Período) · **UC:** Modelagem de Software

## 1. Funcionalidade crítica

**Disparo e entrega confiável de notificações de decisão de reserva** (aprovação/negação). Quando o administrador/técnico decide uma reserva, o Componente Reservas publica um evento. O Componente Notificações valida o destinatário, persiste a notificação, envia ao Serviço de Notificação externo com retry e acompanha a confirmação de entrega. O mesmo componente atende eventos de Estoque e Compras (interface `INotificacao` do C4 de componentes do projeto).

Rastreabilidade: RF25 (enviar notificações), regra de negócio 6 (negação com justificativa) e entidade de domínio **Notificação**.

> Nota: a seta `-)` do Mermaid representa mensagem assíncrona. No sistema real ela é publicada via RabbitMQ (ver ADR 0001 e C4).

## 2. Diagrama de Sequência

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador/Técnico
    participant Portal as PortalWeb
    participant Reservas as ReservaService
    participant Notif as NotificacaoService
    participant Repo as NotificacaoRepository
    participant Ext as ProvedorNotificacaoExterno

    Admin ->> Portal: negarReserva(reservaId, justificativa)
    Portal ->> Reservas: decidirReserva(DecisaoReservaDTO)
    Reservas -) Notif: receberEvento(EventoReservaDTO)
    Reservas -->> Portal: ReservaDecididaDTO
    Portal -->> Admin: confirmação da decisão

    Notif ->> Repo: salvar(Notificacao)
    Repo -->> Notif: NotificacaoId
    Notif ->> Notif: validarDestinatario(EventoReservaDTO)

    alt destinatário válido
        Notif ->> Repo: atualizarStatus(NotificacaoId, ENFILEIRADA)
        Notif ->> Notif: montarPayload(EventoReservaDTO)

        par Envio ao provedor externo
            loop até ACEITO ou 3 tentativas
                Notif ->> Ext: enviar(PayloadNotificacaoDTO)
                alt resposta ACEITO
                    Ext -->> Notif: RespostaEnvioDTO(ACEITO)
                    Notif ->> Repo: atualizarStatus(NotificacaoId, AGUARDANDO_CONFIRMACAO)
                else resposta ERRO e tentativa ≤ 2
                    Ext -->> Notif: RespostaEnvioDTO(ERRO)
                    Notif ->> Repo: registrarTentativa(NotificacaoId, tentativa)
                else resposta ERRO e tentativa = 3
                    Ext -->> Notif: RespostaEnvioDTO(ERRO)
                    Notif ->> Repo: atualizarStatus(NotificacaoId, FALHA)
                end
            end
        and Auditoria
            Notif ->> Repo: registrarAuditoria(NotificacaoId, EventoReservaDTO)
        end

        alt status = AGUARDANDO_CONFIRMACAO
            Ext -) Notif: confirmarEntrega(ConfirmacaoEntregaDTO)
            Notif ->> Repo: atualizarStatus(NotificacaoId, ENTREGUE)
        else status = FALHA
            Notif -) Portal: publicarAlertaFalha(AlertaFalhaDTO)
        end
    else destinatário inválido
        Notif ->> Repo: atualizarStatus(NotificacaoId, DESCARTADA)
    end
```

### Consistência mensagem × operação pública

| Linha de vida (B) | Operações públicas declaradas | Mensagens recebidas no diagrama |
|---|---|---|
| PortalWeb | `negarReserva(reservaId, justificativa)`, `publicarAlertaFalha(AlertaFalhaDTO)` | 1, `publicarAlertaFalha` |
| ReservaService (`IReservaService`) | `decidirReserva(DecisaoReservaDTO): ReservaDecididaDTO` | `decidirReserva` |
| NotificacaoService (`INotificacaoService`) | `receberEvento(EventoReservaDTO): void`, `validarDestinatario(EventoReservaDTO): boolean`, `montarPayload(EventoReservaDTO): PayloadNotificacaoDTO`, `confirmarEntrega(ConfirmacaoEntregaDTO): void` | as quatro |
| NotificacaoRepository (`INotificacaoRepository`) | `salvar(Notificacao): NotificacaoId`, `atualizarStatus(NotificacaoId, Status): void`, `registrarTentativa(NotificacaoId, int): void`, `registrarAuditoria(NotificacaoId, EventoReservaDTO): void` | as quatro |
| ProvedorNotificacaoExterno (`IProvedorNotificacao`) | `enviar(PayloadNotificacaoDTO): RespostaEnvioDTO` | `enviar` |

## 3. Diagrama de Atividades com Raias

```mermaid
flowchart TB
    classDef barra fill:#000,stroke:#000,color:#000,width:90px,height:8px;
    classDef decisao fill:#fff8dc,stroke:#8a6d00;
    classDef fim fill:#000,stroke:#000,color:#fff;

    subgraph R1["Raia 1 — Administrador/Técnico"]
        INI(("●"))
        A1["Decidir reserva<br/>(aprovar ou negar)"]
    end

    subgraph R2["Raia 2 — Sistema Back-End (Reservas e Notificações)"]
        B1["Registrar decisão<br/>ReservaService.decidirReserva"]
        B2["Publicar EventoReservaDTO<br/>(assíncrono)"]
        B3["Criar Notificação (CRIADA)"]
        D1{"Destinatário<br/>válido?"}
        B4["Descartar (DESCARTADA)"]
        B5["Enfileirar (ENFILEIRADA)"]
        FORK[" "]
        B6["Montar payload e<br/>enviar ao provedor"]
        D2{"Resposta do<br/>provedor?"}
        B7["Reagendar com backoff<br/>(REAGENDADA)"]
        B8["Marcar<br/>AGUARDANDO_CONFIRMACAO"]
        B9["Marcar FALHA"]
        B10["Registrar auditoria"]
        JOIN[" "]
        D3{"Status<br/>consolidado?"}
        B11["Marcar ENTREGUE"]
        B12["Publicar alerta de falha<br/>ao administrador"]
    end

    subgraph R3["Raia 3 — Serviço de Notificação (externo)"]
        E1["Processar envio e<br/>entregar ao destinatário"]
        E2["Emitir confirmação de<br/>entrega (webhook)"]
    end

    subgraph R4["Raia 4 — Solicitante"]
        S1["Receber notificação<br/>de aprovação/negação"]
    end

    FIMa(("◉"))
    FIMb(("◉"))
    FIMc(("◉"))

    INI --> A1 --> B1 --> B2 --> B3 --> D1
    D1 -->|"[destinatarioValido = true]"| B5
    D1 -->|"[destinatarioValido = false]"| B4 --> FIMa
    B5 --> FORK
    FORK --> B6
    FORK --> B10
    B6 --> E1 --> D2
    D2 -->|"[resposta = ACEITO]"| B8 --> JOIN
    D2 -->|"[resposta = ERRO e tentativa ≤ 2]"| B7 --> B6
    D2 -->|"[resposta = ERRO e tentativa = 3]"| B9 --> JOIN
    B10 --> JOIN
    JOIN --> D3
    D3 -->|"[status = AGUARDANDO_CONFIRMACAO]"| E2 --> B11 --> S1 --> FIMb
    D3 -->|"[status = FALHA]"| B12 --> FIMc

    class FORK,JOIN barra
    class D1,D2,D3 decisao
    class FIMa,FIMb,FIMc fim
```

## 4. Diagrama de Transição de Estados — entidade `Notificacao`

```mermaid
stateDiagram-v2
    [*] --> Criada : eventoRecebido / salvar()

    Criada --> Enfileirada : validarEvento [destinatarioValido] / atualizarStatus(ENFILEIRADA)
    Criada --> Descartada : validarEvento [destinatarioInvalido] / atualizarStatus(DESCARTADA)

    Enfileirada --> Enviando : iniciarEnvio [tentativa ≤ 3] / montarPayload()

    state Enviando {
        [*] --> Transmitindo
        Transmitindo --> AguardandoConfirmacao : respostaAceita [resposta = ACEITO] / atualizarStatus(AGUARDANDO_CONFIRMACAO)
    }

    Enviando --> Entregue : confirmacaoRecebida [aceitoPeloProvedor] / atualizarStatus(ENTREGUE)
    Enviando --> Reagendada : falhaEnvio [tentativa ≤ 2] / registrarTentativa()
    Enviando --> Falha : falhaEnvio [tentativa = 3] / publicarAlertaFalha()
    Reagendada --> Enviando : backoffExpirado [tentativa ≤ 3] / reenviar()

    Entregue --> [*]
    Falha --> [*]
    Descartada --> [*]

    Enfileirada : entry / registrarTimestampFila()
    Enfileirada : do / aguardarDespacho()
    Enfileirada : exit / removerDaFilaLogica()

    Transmitindo : entry / abrirSessaoHTTPS()
    Transmitindo : do / enviar(PayloadNotificacaoDTO)
    Transmitindo : exit / registrarLatencia()
```

**Invariantes protegidos:** (1) `Entregue`, `Falha` e `Descartada` são terminais, sem transição de saída; (2) `Entregue` só é alcançada após resposta `ACEITO` do provedor; (3) no máximo 3 tentativas; (4) confirmações duplicadas em estado terminal são ignoradas (idempotência por `eventoId`).

## 5. Visão Arquitetural C4 Nível 3 (Componentes)

Código em [`docs/architecture/c4-componentes.puml`](docs/architecture/c4-componentes.puml) (C4-PlantUML). Renderize com a extensão PlantUML do VS Code ou no plantuml.com e salve o PNG em `docs/architecture/c4-componentes.png`; depois referencie aqui:

`![C4 Componentes](docs/architecture/c4-componentes.png)`

## 6. Registro de Decisão Arquitetural

Veja [`docs/architecture/adr-0001.md`](docs/architecture/adr-0001.md).

## 7. Correspondência entre modelos

| Sequência | Atividades | Estado da `Notificacao` |
|---|---|---|
| `salvar` | Criar Notificação | Criada |
| `validarDestinatario` | Destinatário válido? | Criada → Enfileirada / Descartada |
| `par` + `enviar` | Fork → enviar / auditar | Enfileirada → Enviando |
| `registrarTentativa` | Reagendar com backoff | Reagendada |
| `atualizarStatus(FALHA)` + `publicarAlertaFalha` | Marcar FALHA + alerta | Falha |
| `confirmarEntrega` | Emitir confirmação (webhook) | Entregue |