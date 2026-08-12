# hooks-cli

**Versão:** 0.1
**Data:** 2026-08-12

---

## 1. Resumo

CLI para Windows que escuta uma conversa em andamento e exibe, no terminal, até 3 lembretes relevantes extraídos de um arquivo de texto fornecido pelo usuário.

Os lembretes ("hooks") são fatos que o usuário já conhece e quer ter na ponta da língua no momento certo. Não são falas prontas para leitura.

Processamento 100% local. Nenhum dado persistido.

---

## 2. Problema

Em conversas de alto valor (venda, entrevista, negociação, apresentação), a pessoa esquece de mencionar fatos que preparou. O material de apoio existe, mas está em um documento que não dá para consultar durante a conversa.

Soluções existentes (Gong, Clari Copilot, Cluely) geram texto para o vendedor repetir, exigem nuvem e assinatura, e são desenhadas para times de vendas com CRM.

---

## 3. Usuário e cenário

**Usuário:** profissional individual que prepara conversas com antecedência e já tem material escrito.

**Cenário:**
1. Antes da call, escreve ou cola seus fatos-chave em um `.txt`, um por linha.
2. Roda o CLI apontando para o arquivo.
3. Entra na call normalmente.
4. Durante a conversa, olha o terminal de relance quando quer.
5. Encerra com Ctrl+C. Nada fica.

---

## 4. Escopo v1

### Dentro

- Windows 10/11, x64
- Captura de microfone e áudio do sistema
- Transcrição local
- Vetorização local do arquivo de hooks
- Recomendação de até 3 hooks simultâneos no terminal
- Binário único, sem instalador

### Fora

- macOS, Linux
- Interface gráfica e overlay sobre outras janelas
- Persistência de qualquer natureza (transcrição, histórico, cache de sessão)
- Configuração além de flags de linha de comando
- Edição de hooks pelo app
- Login, conta, telemetria
- Resumo pós-conversa
- Qualquer chamada de rede em tempo de execução

---

## 5. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF-01 | Receber caminho de arquivo de texto como argumento obrigatório |
| RF-02 | Interpretar cada linha não vazia como um hook independente |
| RF-03 | Ignorar linhas em branco e linhas iniciadas por `#` |
| RF-04 | Gerar embedding de cada hook na inicialização |
| RF-05 | Falhar com mensagem clara se o arquivo não existir, estiver vazio ou tiver menos de 3 hooks |
| RF-06 | Capturar microfone e áudio do sistema como streams separados |
| RF-07 | Detectar segmentos de fala por VAD, descartando silêncio |
| RF-08 | Transcrever apenas o canal do interlocutor (áudio do sistema) |
| RF-09 | Manter janela deslizante de contexto dos últimos 90 segundos transcritos |
| RF-10 | Recalcular recomendações a cada segmento de fala encerrado |
| RF-11 | Exibir até 3 hooks, ordenados por score decrescente |
| RF-12 | Exibir hook apenas acima do threshold de similaridade |
| RF-13 | Manter hook exibido por tempo mínimo antes de substituí-lo |
| RF-14 | Suprimir hook já exibido durante janela de cooldown |
| RF-15 | Redesenhar região fixa do terminal, sem scroll |
| RF-16 | Encerrar limpo em Ctrl+C, liberando dispositivos de áudio |

---

## 6. Requisitos não funcionais

| ID | Requisito |
|---|---|
| RNF-01 | Latência entre fim da fala e exibição do hook ≤ 4s |
| RNF-02 | Nenhuma escrita em disco além de logs de erro opcionais |
| RNF-03 | Nenhuma conexão de rede durante a execução |
| RNF-04 | Transcrição existe apenas em memória, sobrescrita pela janela deslizante |
| RNF-05 | Consumo de RAM ≤ 4 GB com modelo padrão |
| RNF-06 | Funcionar sem GPU, com degradação aceitável de latência |
| RNF-07 | Distribuição como executável único mais arquivos de modelo |

---

## 7. Arquitetura

**Linguagem:** Rust

**Componentes:**

| Camada | Biblioteca |
|---|---|
| Captura de áudio | `cpal` ou `wasapi` (WASAPI loopback + capture) |
| Detecção de voz | `voice_activity_detector` (Silero VAD) |
| Transcrição | `whisper-rs` (whisper.cpp), modelo `base` ou `small` |
| Embedding | `fastembed` com `multilingual-e5-small` |
| Terminal | `crossterm` |

**Fluxo:**

```
arquivo.txt → split por linha → embeddings (N × 384) → índice em memória
                                                              ↓
áudio do sistema → VAD → chunks de fala → whisper → texto → embedding → topK → terminal
                                                       ↓
                                              janela deslizante 90s
```

**Notas de implementação:**

- Microfone e áudio do sistema são dispositivos distintos. Isso separa os interlocutores sem modelo de diarização.
- O canal do microfone é capturado mas não transcrito na v1. Reservado para evolução (detectar que o usuário já mencionou o hook).
- Ring buffer lock-free entre a callback de áudio e o worker de transcrição. **Proibido alocar, travar lock ou entrar em panic dentro da callback.**
- Índice vetorial é busca linear. Com centenas de hooks, é irrelevante otimizar.

---

## 8. Algoritmo de recomendação

Para cada segmento de fala transcrito:

1. Gerar embedding do segmento concatenado à janela de contexto recente (últimas 2 falas).
2. Calcular similaridade de cosseno contra todos os hooks.
3. Descartar hooks abaixo de `THRESHOLD`.
4. Descartar hooks exibidos há menos de `COOLDOWN`.
5. Ordenar por score e tomar os 3 primeiros.
6. Não substituir slot cujo hook está exibido há menos de `DWELL`.

**Parâmetros iniciais (a calibrar):**

| Parâmetro | Valor inicial |
|---|---|
| `THRESHOLD` | 0,72 |
| `DWELL` | 4 s |
| `COOLDOWN` | 120 s |
| Janela de contexto | 90 s |
| Chunk de transcrição | 5 s |

Estes números são chute informado. A calibração real é item de escopo, não detalhe de implementação.

**Postura de erro:** três slots permitem trocar precisão por recall. Slot vazio é preferível a hook irrelevante, mas o custo de um falso positivo aqui é baixo — o usuário ignora e segue.

---

## 9. Interface

```
recomendador <arquivo> [--model base|small] [--threshold 0.72] [--device <id>]
recomendador --list-devices
```

**Saída durante execução:**

```
┌─ ouvindo ──────────────────────────────────────────┐
│                                                    │
│  › conversão de 100k em experimento X, base 250k   │
│  › churn caiu 18% no trimestre seguinte            │
│  › piloto rodou 6 semanas com 3 squads             │
│                                                    │
└────────────────────────────────────────────────────┘
```

Amarelo sobre preto. Região fixa, redesenhada no lugar. Sem histórico, sem scroll.

**Limitações conhecidas do terminal:**

- Aparece em compartilhamento de tela.
- Fica atrás da janela da call por padrão. Mitigação: `alwaysOnTop: true` no `settings.json` do Windows Terminal. Instrução de README, não código.

Ambas desaparecem se e quando o overlay entrar em escopo.

---

## 10. Critérios de aceite

| ID | Critério |
|---|---|
| CA-01 | Arquivo com 30 hooks carrega e vetoriza em menos de 10s |
| CA-02 | Em call de 20 min, latência mediana entre fim da fala e exibição ≤ 4s |
| CA-03 | Nenhum arquivo criado no disco durante a sessão, verificado por monitor de I/O |
| CA-04 | Nenhuma conexão de saída durante a sessão, verificado por firewall |
| CA-05 | Em roteiro de teste com 10 perguntas-gatilho conhecidas, o hook correto aparece entre os 3 em pelo menos 7 |
| CA-06 | Nenhum hook pisca ou é substituído antes de 4s de exibição |
| CA-07 | Ctrl+C encerra em menos de 2s sem deixar processo órfão |
| CA-08 | Funciona com fone Bluetooth e com dispositivo trocado no meio da sessão |

---

## 11. Riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| Transcrição em CPU não fecha em tempo real na máquina alvo | Bloqueante | Spike de 2 dias antes do compromisso de prazo |
| Qualidade do arquivo de hooks é ruim ou insuficiente | Produto parece quebrado | Mínimo de 20 hooks; orientação de redação no README |
| Similaridade direta falha (pergunta não parece com o fato) | Recall baixo | 3 slots mitigam; se insuficiente, gerar perguntas-gatilho por hook em pré-processamento |
| Loopback WASAPI instável com Bluetooth ou troca de dispositivo | Falha em uso real | CA-08; teste em hardware real, não em CI |
| Carga cognitiva — ler durante a fala atrapalha | Rejeição do produto | 3 linhas, sem animação, sem scroll; validar em conversa real |
| Terminal visível em compartilhamento de tela | Constrangimento | Documentado; overlay em fase futura |

---

## 12. Fases e estimativa

Execução via spec-driven development com agente de código. A estimativa reflete isso: blocos determinísticos comprimem, verificação em hardware real não.

### Fase 1 — Spike (2 dias)

Captura WASAPI + whisper `base` em CPU, medindo latência real na máquina alvo. Entregável: número, não código de produção.

**Critério de continuidade:** latência de transcrição de chunk de 5s abaixo de 3s em CPU.

### Fase 2 — Build (6–8 dias)

| Bloco | Dias |
|---|---|
| Spec detalhada | 1,0 |
| CLI, parse do arquivo | 0,3 |
| Áudio, VAD, ring buffer | 1,5 |
| Transcrição | 0,5 |
| Embedding, índice, topK | 0,3 |
| Loop de recomendação e render | 0,7 |
| Empacotamento | 0,7 |
| Calibração com conversas reais | 3,0–5,0 |

**Total: 8–10 dias úteis**, incluindo o spike.

A calibração é o maior item e o menos compressível. Não é implementação — é rodar conversas reais, observar e ajustar os parâmetros da seção 8.

---

## 13. Evolução possível (não orçada)

- Overlay gráfico com exclusão de captura de tela
- Transcrição do canal do microfone para suprimir hook já mencionado
- Geração de perguntas-gatilho por hook em pré-processamento
- macOS
