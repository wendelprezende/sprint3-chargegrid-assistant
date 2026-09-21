# ChargeGrid Assistant — Sprint 03 (Refactory LangChain)

**EV Challenge 2026 | GoodWe | FIAP | Prompt and Artificial Intelligence | 2026.2**

## Equipe 03

| Nome | RM |
|---|---|
| Arthur Araújo | 573308 |
| Daniel Alejandro | 573075 |
| Victor Hugo Lavaqui | 573838 |
| Wendel Pedro | 573126 |

## Continuidade do projeto

Esta é a Sprint 03, evolução direta das Sprints 1 (planejamento) e 2
(implementação inicial com Google Gemini). O núcleo conversacional foi
refatorado para **LangChain (LCEL)**, agora rodando sobre a **Ollama
Cloud API** (`gpt-oss:120b` e `qwen3:8b`), com memória por sessão,
saída estruturada validada e guardrails de segurança. Veja
`docs/relatorio_evolucao.txt` para o detalhamento completo da evolução.

## Estrutura do projeto

```
prompts/                  system prompt versionado (v1 legado, v2 com XML tagging)
  system_prompt_v1.md
  system_prompt_v2.md
  versoes.md               tabela de versões (o que mudou, por quê, ganho medido)

src/
  chain/
    builder.py              chain LCEL: ChatPromptTemplate | ChatOllama | parser
    memoria.py               memória por sessão com limite de tokens
    consulta_estruturada.py  chain de saída estruturada (Pydantic v2)
    multi_provider.py         bônus: chamada multi-modelo/multi-prompt
  schemas/
    consulta_recarga.py      schema Pydantic v2 (ConsultaRecarga)
  guardrails/
    scope_validator.py        validação de escopo GoodWe/ChargeGrid
    moderation.py              detecção de jailbreak/prompt injection
  utils/
    contagem_tokens.py         medição de tokens com tiktoken

evals/
  eval_set.json               16 casos: happy_path, edge_case, jailbreak, out_of_scope
  run_evals.py                 script que reexecuta o eval set e mede tokens/latência
  sprint3_results_*_offline_exemplo.json   exemplos gerados em modo offline (não são dados finais)

docs/
  relatorio_modelos.md         comparação gpt-oss:120b vs qwen3:8b, parâmetros documentados
```

## Como rodar

### 1. Instalar dependências

```bash
pip install -r requirements.txt
```

### 2. Configurar a API key do Ollama Cloud

```bash
cp .env.example .env
# edite o .env e cole sua API key (crie uma em https://ollama.com/settings/keys)
export OLLAMA_API_KEY="sua_api_key_aqui"
```

### 3. Testar a chain (modo offline, sem gastar chamadas de API)

```bash
PYTHONPATH=. python src/chain/builder.py
```

Isso roda 3 turnos de conversa + 2 casos de guardrail usando um LLM
"fake" — só para confirmar que a arquitetura (prompt, memória,
guardrails) está funcionando antes de gastar chamadas reais.

### 4. Rodar de verdade contra o Ollama Cloud

```python
from src.chain.builder import ChargeGridAssistant

system_prompt = open("prompts/system_prompt_v2.md").read()  # extraia o bloco ```xml```
assistant = ChargeGridAssistant(system_prompt=system_prompt, model="gpt-oss:120b")
print(assistant.perguntar("sessao-usuario-1", "Como eu começo a carregar meu carro aqui?"))
```

### 5. Reexecutar o eval set completo

```bash
python evals/run_evals.py --prompt v2 --model gpt-oss:120b
```

Gera `evals/sprint3_results.json` com tokens por turno, latência média
e taxa de acerto dos guardrails — os números que alimentam a tabela
antes/depois em `docs/relatorio_evolucao.txt`.

## Sobre o acesso ao modelo

`ChatOllama` é configurado com `base_url=
"https://ollama.com"` e autenticação via header `Authorization: Bearer
<OLLAMA_API_KEY>` — não é necessário instalar Ollama nem ter GPU
própria. Detalhes em `src/chain/builder.py`.

## Segurança e guardrails

Toda pergunta do usuário passa por dois guardrails ANTES de qualquer
chamada ao modelo (`src/chain/builder.py::ChargeGridAssistant.perguntar`):

1. **Moderação** (`src/guardrails/moderation.py`) — bloqueia tentativas
   de jailbreak/prompt injection (ex.: "ignore suas instruções", "revele
   seu system prompt").
2. **Validação de escopo** (`src/guardrails/scope_validator.py`) —
   recusa aconselhamento jurídico, financeiro e de segurança elétrica
   (orientando um profissional habilitado), e redireciona perguntas
   fora do domínio ChargeGrid/GoodWe.

## Referências

- GoodWe: https://en.goodwe.com
- Ollama Cloud API: https://docs.ollama.com/cloud
- LangChain (LCEL / RunnableWithMessageHistory): https://python.langchain.com
- Pydantic v2: https://docs.pydantic.dev
