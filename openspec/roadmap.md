# Roadmap de changes

Uma entrada por change. Ao rodar `/opsx:propose <id>`, use a entrada
correspondente como escopo. Não expanda além do que está em "Entra".

Ordem é dependência, não cronograma. As changes 1 a 6 não exigem hardware de
áudio e devem ser feitas primeiro.

---

## 1. `hooks-file-parser`

**Objetivo:** transformar um arquivo de texto em uma lista de hooks validada.

**Entra**
- Argumento posicional obrigatório: caminho do arquivo
- Uma linha não vazia = um hook
- Ignorar linhas em branco e linhas iniciadas por `#`
- Trim de espaços nas bordas
- Erro claro se: arquivo inexistente, ilegível, vazio, ou com menos de 3 hooks
- Erro claro se algum hook exceder 200 caracteres

**Não entra**
- Formatos além de texto plano
- Reescrita, normalização semântica ou edição do conteúdo do hook
- Recarga do arquivo em runtime

**Depende de:** nada

**Pronto quando:** testes cobrem arquivo válido, vazio, só comentários, hook
longo demais e arquivo inexistente.

---

## 2. `embedding-index`

**Objetivo:** vetorizar os hooks e responder consultas por similaridade.

**Entra**
- `fastembed` com `multilingual-e5-small`
- Vetorização de todos os hooks na inicialização
- Índice em memória, busca linear por cosseno
- API interna: `query(texto, k) -> Vec<(hook_id, score)>`
- Download ou empacotamento do modelo resolvido offline

**Não entra**
- ANN, HNSW, banco vetorial
- Persistência ou cache do índice
- Geração de perguntas-gatilho

**Depende de:** 1

**Pronto quando:** com fixture de 10 hooks conhecidos, consultas esperadas
retornam o hook correto em primeiro lugar.

---

## 3. `simulate-mode`

**Objetivo:** rodar a pipeline inteira sem áudio.

**Entra**
- Flag `--simulate <transcript.txt>`
- Cada linha do arquivo é injetada como se fosse um segmento transcrito
- Ritmo cronometrado configurável, padrão 5s por linha
- Ponto de injeção idêntico ao que o transcritor real usará

**Não entra**
- Qualquer captura de áudio
- Simulação de erro de transcrição

**Depende de:** 2

**Pronto quando:** a pipeline roda ponta a ponta em CI, sem dispositivo de
áudio e sem GPU.

---

## 4. `recommendation-loop`

**Objetivo:** decidir quais 3 hooks exibir a cada segmento.

**Entra**
- Janela deslizante de contexto (padrão 90s), em memória, sobrescrita
- Embedding do segmento concatenado às 2 falas anteriores
- Filtro por THRESHOLD
- Filtro por COOLDOWN de hook já exibido
- Ordenação por score, topK = 3
- DWELL: não substituir slot exibido há menos que o mínimo
- Todos os parâmetros configuráveis por flag, nunca hardcoded

**Não entra**
- Ajuste automático de qualquer parâmetro
- Aprendizado, feedback do usuário, histórico entre execuções

**Depende de:** 3

**Pronto quando:** com transcript simulado, a sequência de estados dos 3 slots
é determinística e verificável em teste.

---

## 5. `terminal-render`

**Objetivo:** exibir os 3 slots no terminal.

**Entra**
- `crossterm`, região fixa de 3 linhas mais moldura
- Amarelo sobre preto
- Redesenho no lugar, sem scroll, sem histórico
- Truncamento de hook longo demais para a largura do terminal
- Restauração do terminal ao sair

**Não entra**
- Animação, transição, cor por score
- Interatividade, navegação, seleção

**Depende de:** 4

**Pronto quando:** roda com `--simulate` e o quadro nunca produz scroll nem
deixa o terminal corrompido após Ctrl+C.

---

## 6. `debug-scores`

**Objetivo:** tornar a calibração possível.

**Entra**
- Flag `--debug <arquivo>` que escreve, por segmento: timestamp, texto
  transcrito, score de todos os hooks, decisão de cada slot
- Formato TSV
- Ativação explícita apenas; desligado é o padrão

**Não entra**
- Coleta automática, telemetria, envio de qualquer coisa

**Depende de:** 4

**Nota:** esta é a única exceção autorizada à regra de não escrever em disco, e
só sob flag explícita. Deixe isso claro na proposal.

**Pronto quando:** uma sessão simulada gera TSV analisável em planilha.

---

## 7. `audio-capture`

**Objetivo:** capturar áudio real e produzir segmentos de fala.

**Entra**
- WASAPI loopback (áudio do sistema) e capture (microfone), streams separados
- Apenas o loopback alimenta a pipeline; o microfone é capturado e descartado
- Silero VAD para delimitar segmentos, descartando silêncio
- Ring buffer lock-free entre callback e worker
- `--list-devices` e `--device <id>`
- Tratamento de troca de dispositivo em runtime

**Não entra**
- Diarização por modelo
- Gravação, mixagem, salvamento de áudio
- Transcrição do canal do microfone

**Depende de:** 3

**Atenção:** proibido alocar, travar lock, fazer I/O ou entrar em panic dentro
da callback. Revisão humana obrigatória nesta change.

**Pronto quando:** captura estável por 20 minutos com fone Bluetooth e após
troca de dispositivo no meio da sessão.

---

## 8. `whisper-transcription`

**Objetivo:** transcrever os segmentos de fala.

**Entra**
- `whisper-rs`, modelo `base` por padrão, `small` opcional
- Flag `--model base|small`
- Chunk configurável, padrão 5s
- Worker em thread separada, fila limitada com descarte do mais antigo sob
  pressão
- Aceleração por GPU quando disponível, fallback silencioso para CPU

**Não entra**
- Pós-processamento, pontuação, correção do texto
- Persistência da transcrição

**Depende de:** 7

**Pronto quando:** substitui `--simulate` sem alterar nada a jusante, e a
latência mediana entre fim da fala e exibição fica abaixo de 4s.

---

## 9. `packaging`

**Objetivo:** entregar um executável que roda em máquina limpa.

**Entra**
- Build release para Windows x64
- Estratégia de distribuição dos modelos (whisper e embedding)
- README com requisitos, uso, e a instrução de `alwaysOnTop` no Windows
  Terminal
- Verificação de que nenhum arquivo é criado fora dos modelos

**Não entra**
- Instalador, assinatura de código, auto-update
- Outras plataformas

**Depende de:** 8

**Pronto quando:** roda em máquina Windows sem toolchain Rust instalado.