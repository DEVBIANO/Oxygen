[![DOI](https://zenodo.org/badge/1335994787.svg)](https://doi.org/10.5281/zenodo.21970976)
#  Oxygen

> **Air-Gapped, Local AI-Powered Security Logic & Cryptography Analyzer**
# Oxygen — Arquitetura e Fundamentação Técnica (V1)

# 1. Visão geral

Oxygen é uma ferramenta de infraestrutura para **análise estática de código e correção automatizada de vulnerabilidades**, desenhada para operar **100% offline**, em **CPU comum** (sem GPU dedicada), voltada a ambientes air-gapped de alta segurança (governo, defesa, setor bancário).

O ineditismo está na combinação específica de quatro propriedades que, juntas, nenhuma ferramenta hoje no mercado entrega de forma Clara:

1. Correção automatizada (não só detecção/triagem)
2. Multi-linguagem via AST (não travada a uma linguagem, como Bandit é a Python)
3. 100% offline, com mecanismo formal de atualização auditável
4. Implementada numa linguagem com segurança de memória formalmente provada

# 2. O problema que resolve (a lacuna real)

O mercado atual se divide em três grupos, e nenhum cobre o cenário do Oxygen:

| Categoria | Exemplos | Limitação |
| --- | --- | --- |
| SAST em nuvem, com correção | GitHub Copilot Enterprise, Qodo/CodiumAI | Dependem de nuvem — inviável em rede air-gapped |
| Scanners leves, sem correção | Semgrep, Trivy, SonarQube | Só apontam o problema, não geram o patch |
| IA local, mas escopo estreito | SecureFixAgent (2025), SPVR, VulnHunter | Ou são presos a uma linguagem (Bandit = só Python), ou dependem de LLM em nuvem (SPVR usa ChatGPT-4), ou fazem só triagem, não correção (VulnHunter) |

**A dor real:** instituições que são proibidas por lei de usar nuvem não têm hoje uma ferramenta que detecte em múltiplas linguagens **e** corrija **e** funcione sem internet **e** seja auditável quanto à atualização de suas regras de detecção.

# 3. Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                         OXYGEN CLI                            │
│                     (núcleo em Rust)                          │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────┐      ┌──────────────────────────────────┐
│  1. PARSER (AST)    │      │  Motor de Regras (YAML)          │
│  Tree-sitter        │─────▶│  Queries em S-expressions        │
│  multi-linguagem    │      │  mapeadas a CWEs específicos     │
└───────────────────┘      └──────────────────────────────────┘
        │                                    │
        │                                    ▼
        │                         ┌─────────────────────┐
        │                         │ Nó vulnerável detectado│
        │                         └─────────────────────┘
        │                                    │
        │                                    ▼
        │                    ┌───────────────────────────────┐
        │                    │ 2. EXTRAÇÃO DE MICRO-CONTEXTO   │
        │                    │ (10-15 linhas ao redor do nó,   │
        │                    │  guiado pela estrutura da AST — │
        │                    │  método validado pelo SPVR)     │
        │                    └───────────────────────────────┘
        │                                    │
        │                                    ▼
        │                    ┌───────────────────────────────┐
        │                    │ 3. CORREÇÃO VIA SLM LOCAL       │
        │                    │ Qwen2.5-Coder / DeepSeek-Coder  │
        │                    │ quantizado (GGUF, Q4_K_M)       │
        │                    │ via llama-cpp-rs, CPU-only      │
        │                    └───────────────────────────────┘
        │                                    │
        ▼                                    ▼
┌─────────────────────────────────────────────────────────────┐
│  4. LOOP DE VALIDAÇÃO SINTÁTICA                                │
│  Tree-sitter reparseia a sugestão em memória.                 │
│  Válida → segue para revisão. Inválida → retenta (até 3x).    │
│  Esgotou tentativas → reporta como "requer revisão manual".   │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  5. PORTÃO DE REVISÃO (HUMAN-IN-THE-LOOP)                      │
│  Patch sintaticamente válido é apresentado como sugestão.     │
│  Aplicação automática só em modo explícito de baixo risco.    │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  6. RELATÓRIO FINAL                                            │
│  Patch (aplicado ou sugerido) + explicação + confiança        │
└─────────────────────────────────────────────────────────────┘
```

# 3.1 Por que a revisão humana é obrigatória, não opcional

O portão de revisão (etapa 5) não é uma cautela genérica — é resposta direta a evidência recente de que validação sintática não implica correção funcional nem de segurança. Um estudo da 1Password avaliou 6.080 patches de segurança gerados por modelos de ponta (ChatGPT-5.5 e Claude Opus 4.8) e encontrou que apenas ~27% remediaram completamente a falha sem alterar o comportamento da aplicação, com 20,1% dos casos “corrigindo” a vulnerabilidade mas quebrando o comportamento esperado do sistema. A Anthropic, consultada no estudo, recomendou manter humanos no loop e tornar a verificação baseada em execução, não apenas em inspeção sintática (fonte: CSO Online, https://www.csoonline.com/article/4206598/human-oversight-is-still-critical-as-ai-patching-tools-miss-security-risks.html).

Isso é consistente com a literatura acadêmica de reparo automático de programas, que descreve o problema do “patch overfitting” (patch que passa na validação mas não resolve o problema real) como sem solução completa, concluindo que a resposta definitiva é envolver o humano no loop antes de aplicar qualquer correção (fonte: https://arxiv.org/pdf/2211.12787). O próprio pipeline de correção automática de vulnerabilidades do Google, usando Gemini, gera patches explicitamente “para revisão humana”, não para aplicação automática (fonte: https://research.google/pubs/ai-powered-patching-the-future-of-automated-vulnerability-fixes/).

**Implicação de design:** por padrão, o Oxygen nunca aplica um patch automaticamente em ambientes de produção. A validação sintática (etapa 4) é uma condição necessária, mas não suficiente — ela só qualifica o patch para apresentação ao revisor humano. Aplicação automática sem revisão é um modo explícito, opcional, e recomendado apenas para cenários de baixo risco (ex: pipelines de CI com suíte de testes própria como validação adicional).

# 3.2 Por que essa divisão em duas responsabilidades importa

A separação entre **motor de regras (detecção)** e **modelo de linguagem (correção)** não é só uma escolha de design — é a resposta direta a um dos pontos mais difíceis levantados na revisão do projeto: como uma ferramenta 100% offline evita ficar desatualizada?

- O **motor de regras** é o único componente que precisa saber sobre vulnerabilidades novas. Por isso, é o único que precisa de atualização.
- O **LLM de correção** nunca dependeu de conhecer CVEs específicas — ele trabalha em cima de padrões estruturais de código seguro (como reescrever uma concatenação de string insegura em uma query parametrizada), que não mudam com a mesma frequência que a lista de vulnerabilidades conhecidas.

Essa divisão significa que apenas uma fração pequena e bem definida do sistema precisa de manutenção externa — o que reduz drasticamente a superfície de “defasagem” que uma ferramenta air-gapped normalmente sofreria.

# 4. Mecanismo de atualização offline (o que resolve a defasagem)

Segue de perto o modelo de gerenciamento de patches para redes segmentadas descrito no **NIST SP 800-40 Rev. 4**:

1. **Zona de aquisição** (máquina com internet, fora do ambiente protegido) coleta novas definições de vulnerabilidades e traduz em novas regras YAML/queries Tree-sitter.
2. As regras são empacotadas e **assinadas digitalmente** (hash/assinatura, ex. SHA-256).
3. O pacote é transferido para dentro do ambiente air-gapped por **mídia controlada** (pendrive, DVD — conforme política de segurança da instituição).
4. O Oxygen **verifica a assinatura** antes de aceitar o pacote. Assinatura inválida → atualização recusada.
5. O modelo de correção (SLM) **não participa desse processo** — continua o mesmo, sem necessidade de retraining ou reconexão.

# 5. Por que Rust (argumento corrigido)

O ineditismo não é mais “Rust reduz a memória do modelo” — isso foi corrigido depois da revisão técnica. Os dois argumentos válidos que sustentam a escolha de Rust são:

- **Overhead da camada de ferramentas, não do modelo:** o parsing via Tree-sitter, a orquestração do loop de validação e a manipulação de AST rodam com menos overhead e menos memória em Rust do que rodariam em Python — o que importa porque, num cenário de CPU compartilhada com o SLM, cada MB e cada ciclo de CPU que a ferramenta economiza sobra para a inferência do modelo. (Caso real de referência: migração de um analisador estático de Java para Rust, 3x mais performance e 10x menos memória.)
- **Segurança de memória formalmente provada (RustBelt):** uma ferramenta que corrige vulnerabilidades de segurança não pode, ela mesma, introduzir uma nova superfície de ataque por bugs de memória (buffer overflow, use-after-free). O RustBelt oferece a primeira prova formal e verificada por máquina de que o sistema de tipos do Rust garante ausência desse tipo de falha — um argumento de confiabilidade da ferramenta, não de velocidade.

# 6. Viabilidade em CPU comum (o que resolve o ponto de latência)

A literatura recente já demonstra que SLMs de código (1.5B–3B parâmetros) quantizados em 4 bits (GGUF, Q4_K_M) rodando via llama.cpp atingem throughput de **20 a 50 tokens/segundo em CPUs comuns**, sem GPU dedicada — faixa considerada interativa para geração de sugestões de patch. Essa faixa orienta a escolha do modelo padrão do Oxygen (SLM de até 3B parâmetros) e será validada empiricamente com medição própria durante o desenvolvimento da PoC.

# 7. Stack tecnológica

| Componente | Tecnologia |
| --- | --- |
| Linguagem do núcleo | Rust |
| Parsing / AST | Tree-sitter (multi-linguagem) |
| Motor de regras | YAML + queries em S-expressions |
| Inferência do LLM | llama-cpp-rs (ou Ollama como alternativa), modelos GGUF quantizados |
| Modelos-alvo | Qwen2.5-Coder-1.5B/3B, DeepSeek-Coder (quantização Q4_K_M) |
| Atualização de regras | Pacotes assinados, transferência por mídia controlada (alinhado ao NIST SP 800-40) |
| Interface | CLI |

# 8. Limitações conhecidas e mitigações

Nenhuma arquitetura resolve tudo de uma vez. As limitações abaixo foram identificadas propositalmente, para que o texto da proposta as reconheça em vez de deixá-las como pontos cegos.

**Escalabilidade das regras manuais.** Regras Tree-sitter/YAML escritas à mão sofrem, por definição, de cobertura limitada e escalabilidade baixa à medida que crescem o número de CWEs e de linguagens suportadas — uma limitação bem documentada na literatura de análise estática. Ao mesmo tempo, abordagens puramente baseadas em deep learning têm dificuldade de generalizar para dados fora do domínio de treino e costumam ser pouco interpretáveis, o que também as torna inadequadas sozinhas (fonte: https://arxiv.org/html/2601.18844v1). É exatamente por isso que o Oxygen usa uma abordagem híbrida (regras + LLM) em vez de só regras: a cobertura de regras pode ser expandida ao longo do tempo com auxílio de IA para generalizar padrões — isso é tratado como trabalho futuro explícito, não como lacuna ignorada.

**Ausência de fine-tuning no modelo de correção.** O SecureFixAgent — o trabalho mais próximo do Oxygen — usa fine-tuning via LoRA sobre um dataset curado para melhorar a precisão da correção. A versão inicial do Oxygen usa um SLM genérico apenas com engenharia de prompt, sem ajuste fino. Fine-tuning específico para os CWEs cobertos é declarado como próximo passo de pesquisa, não como parte da PoC inicial.

**“Multi-linguagem” é uma capacidade estrutural, ainda não uma validação empírica.** A PoC cobre um único CWE em uma única linguagem. A capacidade multi-linguagem vem do uso do Tree-sitter (que já suporta dezenas de gramáticas), mas isso é uma propriedade arquitetural demonstrável, não algo validado empiricamente até que mais linguagens sejam efetivamente testadas — o texto da proposta deve manter essa distinção explícita.

**O caso da Datadog é evidência de viabilidade, não experimento controlado.** A migração Java→Rust da Datadog é um caso real de produção, útil para mostrar que o ganho de Rust em ferramentas de análise estática é real — mas não é um benchmark científico comparando exatamente a mesma carga de trabalho de segurança em condições controladas, ao contrário dos papers de latência em CPU citados na seção 6.

**Gestão de chaves do mecanismo de atualização.** O modelo de atualização offline (seção 4) resolve o “como” transferir regras com segurança, mas pressupõe uma infraestrutura de chaves (PKI) já gerenciada pela instituição cliente — gestão de chaves em si segue práticas padrão de PKI institucional e não é foco da pesquisa.

## 9. Trabalhos relacionados (posicionamento)

| Trabalho | O que faz | Onde o Oxygen se diferencia |
| --- | --- | --- |
| SySeVR (Li et al.) | Detecção de vulnerabilidades via deep learning sobre representação de AST | Base conceitual para usar AST como representação — Oxygen aplica isso à correção, não só detecção |
| SPVR | Sintaxe → prompt → correção via LLM | Valida o método de extração guiada por sintaxe, mas usa LLM em nuvem (ChatGPT-4) — Oxygen leva isso para local/offline |
| SecureFixAgent (2025) | Loop local detectar-corrigir-validar com Bandit + LLM | Bandit é específico para Python — Oxygen é multi-linguagem via Tree-sitter, e endereça o cenário air-gapped institucional que o paper não cobre |
| VulnHunter | Scanner offline multi-linguagem com triagem via LLM local | Faz triagem (avalia exploitabilidade), não gera correção — e é escrito em Python |
| cargo-cola | Auditoria de dependências Rust com LLM-assisted triage | Escopo restrito a crates Rust; no modo air-gapped não roda o LLM |

# 10. Próximos passos

1. Implementar o parser + motor de regras para CWE-89
2. Integrar llama-cpp-rs com um modelo quantizado de 1.5B–3B
3. Construir o loop de validação sintática
4. Rodar benchmark de latência/throughput em CPU comum
5. Avaliar contra o dataset de validação (Juliet/OWASP Benchmark)
6. Consolidar métricas e revisar a proposta para submissão de Pesquisa Acadêmica e ter base robusta pra implementar em produção
