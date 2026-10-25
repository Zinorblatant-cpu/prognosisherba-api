# prognosisherba-api

API do **PrognosisHerba**: transforma a previsão de crescimento da grama
em um cronograma de poda para as equipes da rodovia.

- Ingere a planilha de previsões publicada pela IA de crescimento.
- Deriva os locais que vão cruzar o limiar de poda, cada um com prazo,
  prioridade e dificuldade.
- Roda o **HoroHerba**, um modelo de programação linear inteira
  (PuLP/CBC), que decide qual equipe poda qual local em qual dia.
- Agrupa as regiões por perfil de crescimento (k-means).
- Guarda a alocação publicada para o app das equipes, que marca os
  locais como concluídos.

Faz parte de um conjunto de repositórios:

| Repositório | Papel |
|---|---|
| **prognosisherba-api** (este) | Previsões → otimização → alocação publicada |
| [prognosisherba-painel](https://github.com/Zinorblatant-cpu/prognosisherba-painel) | Painel de planejamento do gestor |
| [prognosisherba-podadores](https://github.com/Zinorblatant-cpu/prognosisherba-podadores) | App das equipes em campo |
| [prognosisherba-docs](https://github.com/Zinorblatant-cpu/prognosisherba-docs) | Documentação completa: arquitetura e modelos matemáticos |

A formulação do modelo de otimização está em
[`MODELO_MATEMATICO.md`](MODELO_MATEMATICO.md).

## Rodando

Python 3.11 ou mais novo.

```bash
pip install -r requirements.txt
uvicorn server:app --host 127.0.0.1 --port 8002 --reload
```

```bash
curl http://127.0.0.1:8002/health
# {"status":"ok"}
```

Documentação interativa em `http://127.0.0.1:8002/docs`.

## Configuração

Copie `.env.example` para `.env` (o `.env` não vai para o Git). Todas as
variáveis têm valor padrão. Sem o arquivo, a API usa o CSV de previsões
versionado em `data/`.

| Variável | Padrão | O que faz |
|---|---|---|
| `GRAMA_API_URL` | API de produção | Endereço da planilha de previsões |
| `GRAMA_API_KEY` | vazio | Chave de leitura. **Vazia = ingestão desligada** |
| `GRAMA_SYNC_TTL_S` | `900` | Intervalo mínimo entre buscas (15 min) |
| `GRAMA_SYNC_TIMEOUT_S` | `30` | Timeout HTTP; se estourar, usa o CSV local |

Se a ingestão falhar (rede, timeout, planilha corrompida), a API registra
um aviso e continua com o último CSV bom.

## Rotas

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/health` | Verificação de saúde |
| `GET` | `/parametros` | Limiar de poda e capacidade diária |
| `GET` | `/previsoes` | Altura prevista por região, semana a semana |
| `POST` | `/previsoes/gerar-alocacao` | Deriva os locais e roda o solver sobre todo o horizonte |
| `POST` | `/gerar-alocacao` | Roda o solver sobre um lote informado, num mês fixo |
| `GET` | `/clusterizacao?k=N` | Agrupa as regiões por rota, altura e tendência |
| `POST` | `/alocacao/publicar` | Publica a alocação para o app das equipes |
| `GET` | `/alocacao/atual` | Última alocação publicada com o status de cada local |
| `POST` | `/alocacao/locais/concluir` | Marca ou desmarca um local como concluído |

## Estrutura

```
server.py            rotas FastAPI e schemas Pydantic
horoprognosis.py     modelo de programação linear inteira (PuLP/CBC)
previsoes.py         CSV → locais de poda (prazo, prioridade, dificuldade)
fonte_previsoes.py   ingestão da planilha .xlsx da IA
clusterizacao.py     tendência por ciclo de crescimento + k-means
persistencia.py      SQLite da alocação publicada
data/                CSVs de previsão e podadores.db
tests/               180 testes
```

## Testes

```bash
pytest
```

Sem mock do solver nem do banco: o CBC resolve de verdade e o SQLite
grava de verdade. A suíte zera `GRAMA_API_KEY`, então nunca acessa a rede.

## Hospedagem

Precisa de um processo Python rodando
`uvicorn server:app --host 0.0.0.0 --port $PORT` e de **disco
persistente** para `data/podadores.db`. Em hospedagem com disco efêmero,
a alocação publicada se perde a cada reinício.
