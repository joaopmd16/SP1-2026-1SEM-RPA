# Documento Técnico — Sprint 1
## Automação Inicial para Coleta, Registro e Atualização de Dados de Ativos
### Monitoramento Preditivo de Motores Elétricos Industriais

---

## 1. Arquitetura da Automação

```
┌──────────────────────────────────────────────────────────────┐
│                    CAMADA DE FONTES                           │
│  [Sensores IoT]   [Arquivos CSV]   [Entrada Manual]          │
└────────────┬──────────────┬───────────────┬─────────────────┘
             │              │               │
             ▼              ▼               ▼
┌──────────────────────────────────────────────────────────────┐
│              CAMADA DE INGESTÃO (data_collector.py)           │
│  - Coleta unificada de múltiplas fontes                      │
│  - Geração de hash SHA-256 para idempotência                  │
│  - Normalização de unidades e cabeçalhos                     │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│           CAMADA DE TRANSFORMAÇÃO (transformer.py)            │
│  - Coerce de tipos (str → float)                             │
│  - Validação de limites físicos por ativo                    │
│  - Detecção de anomalias operacionais                        │
│  - Separação válidos / rejeitados                            │
└─────────────────────────────┬────────────────────────────────┘
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  tabela ativos │  │ tabela leituras  │  │ tabela historico │
│  (cadastro)    │  │ (fatos/séries)   │  │ (auditoria)      │
└────────────────┘  └──────────────────┘  └──────────────────┘
             │                │                ▼
             └────────────────┘        ┌──────────────────┐
                              │        │ log_execucoes    │
                              ▼        └──────────────────┘
                    ┌──────────────────────┐
                    │  SCHEDULER (batch)   │
                    │  Ciclos periódicos   │
                    │  Retry / Idempot.    │
                    └──────────────────────┘
```

---

## 2. Fluxo de Dados

### Entrada → Processamento → Saída

| Etapa | Componente | Entrada | Saída |
|-------|-----------|---------|-------|
| Coleta IoT | `SimuladorSensorIoT` | Parâmetros do ativo | Dict com leitura + hash |
| Coleta CSV | `ler_csv()` | String CSV legado | Lista de dicts normalizados |
| Transformação | `PipelineTransformacao` | Lista bruta | Válidos + Rejeitados |
| Persistência | `RepositorioAtivos` | Dicts válidos | Registros no SQLite |
| Auditoria | `_registrar_historico()` | Dados antes/depois | JSON em historico_atualizacoes |
| Scheduler | `executar_ciclo_batch()` | — | Ciclo completo automatizado |

### Campos normalizados (padrão interno)

```
temperatura_c    → float, °C
vibracao_mm_s   → float, mm/s  (ISO 10816)
corrente_a      → float, Ampères
tensao_v        → float, Volts
rpm             → float, rotações/minuto
fator_potencia  → float, 0.0–1.0
```

---

## 3. Tecnologias Utilizadas

| Tecnologia | Versão | Papel |
|-----------|--------|-------|
| Python | 3.10+ | Linguagem base das automações |
| SQLite | 3.x (stdlib) | Banco de dados estruturado |
| sqlite3 | stdlib | Driver de acesso ao SQLite |
| csv / io | stdlib | Leitura de fontes legadas |
| hashlib | stdlib | SHA-256 para idempotência |
| logging | stdlib | Rastreabilidade de execuções |
| json | stdlib | Serialização do histórico |
| dataclasses | stdlib | Regras de validação tipadas |
| tabulate | PyPI | Formatação do dashboard |
| Google Colab | — | Ambiente de execução |

---

## 4. Justificativas das Decisões Técnicas

### 4.1 Por que SQLite?

**SQLite foi escolhido intencionalmente para esta sprint**, não por limitação. Razões:

1. **Zero infraestrutura**: Roda dentro do Colab sem servidor, sem configuração de rede, sem credenciais. Isso acelera o desenvolvimento e elimina dependências externas na fase de prototipação.

2. **ACID completo**: Suporte a transações com rollback automático garante consistência dos dados mesmo em falhas parciais.

3. **WAL mode**: `PRAGMA journal_mode = WAL` permite leituras concorrentes sem travar escritas — fundamento para escalabilidade futura.

4. **Migração trivial**: Todo o acesso ao banco é feito via `context manager` + SQL padrão ANSI. A troca para PostgreSQL em produção é apenas uma mudança de connection string via SQLAlchemy.

5. **Versionável**: Um arquivo `.db` é auditável, portátil e pode ser commitado como evidência de execução.

**Para produção**: PostgreSQL com particionamento por `coletado_em` na tabela `leituras` (que crescerá de forma acelerada) e TimescaleDB como extensão para séries temporais.

### 4.2 Por que idempotência via hash SHA-256?

Cada leitura gera um hash determinístico baseado em `ativo_id + timestamp + temperatura + corrente`. O banco usa `INSERT OR IGNORE` contra o campo `hash_registro UNIQUE`. Isso garante que:
- Reprocessar o mesmo arquivo CSV não duplica dados.
- Retries do scheduler não geram registros fantasma.
- A automação é segura para execução repetida (característica essencial de RPA).

### 4.3 Por que separar coleta, transformação e persistência em camadas?

Seguindo o princípio de **Single Responsibility** (SOLID):
- `SimuladorSensorIoT` apenas gera dados brutos.
- `PipelineTransformacao` apenas valida e normaliza — sem tocar no banco.
- `RepositorioAtivos` apenas persiste — sem conhecer a origem dos dados.

Isso permite trocar qualquer camada independentemente. Em sprints futuras, substituir o simulador por um cliente MQTT real não requer nenhuma alteração no pipeline ou na persistência.

### 4.4 Por que auditoria em tabela separada?

A tabela `historico_atualizacoes` armazena o estado `antes` e `depois` em JSON para qualquer INSERT/UPDATE crítico. Isso:
- Atende ao requisito de rastreabilidade.
- Permite reconstruir o estado de um ativo em qualquer ponto no tempo (event sourcing leve).
- Fundamenta o Digital Twin das próximas sprints.

### 4.5 Escalabilidade futura

A arquitetura atual está preparada para crescer nos seguintes eixos:

| Eixo | Situação atual | Escalada futura |
|------|---------------|----------------|
| Banco | SQLite (arquivo) | PostgreSQL + TimescaleDB |
| Ingestão | Simulador Python | MQTT broker (Mosquitto) + cliente paho-mqtt |
| Orquestração | Função Python | Apache Airflow DAG / Prefect flow |
| Volume | Milhares de registros | Particionamento por data na tabela leituras |
| Containerização | Colab | Docker Compose (app + db + broker) |

O único ponto de mudança para containerizar é o `DB_PATH` — que pode ser lido de variável de ambiente, isolando a configuração do código.

---

## 5. Estrutura de Arquivos

```
motor_rpa/
├── motores.db               ← Banco SQLite (ativos + leituras + histórico)
├── data/
│   └── legado_mtr001.csv   ← CSV de fonte legada (evidência)
├── logs/
│   └── execucao_YYYYMMDD.log  ← Log diário de execuções
└── evidencias_sprint1.zip  ← Pacote de entregáveis
```

---

## 6. Requisitos Funcionais — Atendimento

| RF | Descrição | Atendimento |
|----|-----------|-------------|
| RF1 | Ingerir dados de fonte simulada ou real | ✅ IoT simulado + CSV legado |
| RF2 | Transformação e normalização | ✅ `PipelineTransformacao` com regras físicas |
| RF3 | Registro em base estruturada | ✅ SQLite com schema relacional |
| RF4 | Rotina automatizada de atualização | ✅ `executar_ciclo_batch()` |
| RF5 | Histórico de atualizações | ✅ `historico_atualizacoes` + `log_execucoes` |
| RF6 | Rastreabilidade via logs | ✅ logging stdlib + tabela `log_execucoes` |

---

## 7. Como Executar no Google Colab

```python
# 1. Abrir o Colab: https://colab.research.google.com

# 2. Fazer upload do arquivo sprint1_motores_eletricos.py

# 3. Criar um novo notebook e na primeira célula:
!pip install tabulate --quiet

# 4. Na segunda célula, executar o script:
exec(open("sprint1_motores_eletricos.py").read())

# OU: Copiar cada célula do script (separadas por # %%)
#     em células individuais do Colab e executar em ordem.

# 5. Para baixar as evidências:
from google.colab import files
files.download("motor_rpa/evidencias_sprint1.zip")
```

---

## 8. Evidências de Execução Esperadas

Ao final da execução, o banco conterá:
- **3 ativos** cadastrados com dados completos.
- **~6.048 leituras** (3 motores × 2.016 leituras de 7 dias) + leituras batch.
- **~240–300 anomalias** detectadas (~4% de taxa simulada).
- **8+ entradas** no log de execuções das automações.
- **Histórico de auditoria** de todos os INSERT/UPDATE em ativos.

---

*Documento gerado automaticamente como parte dos entregáveis da Sprint 1.*
