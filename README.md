# 🫁 Oxygen

> **Air-Gapped, Local AI-Powered Security Logic & Cryptography Analyzer**

O **Oxygen** é uma ferramenta de linha de comando (CLI) focada em privacidade para auditoria automatizada de lógica de segurança e más práticas criptográficas. O projeto executa **100% offline e localmente**, combinando análise estática de código com modelos de linguagem quantizados (SLMs).

---

## ⚡ Principais Recursos

* **Privacidade Absoluta (Air-Gapped):** Todo o processamento ocorre no hardware local. Nenhum trecho de código-fonte é enviado para APIs de nuvem.
* **Motor Híbrido (AST + IA):** Mapeia a estrutura sintática do código via *Tree-sitter* e aciona o modelo local apenas nos pontos críticos.
* **Auditoria de Criptografia e Lógica:** Identifica vetores de inicialização (IV) reutilizados, modos de cifragem inseguros (AES-ECB), hashes fracos (MD5/SHA1) e falhas de autorização (IDOR/BOLA).
* **Zero Custo de API:** Roda utilizando executáveis locais (como Ollama / llama.cpp) sem gerar custos por token.
* **Saída Estruturada:** Gera relatórios no terminal, em JSON ou em padrão SARIF para esteiras de DevSecOps.

---

## 🏗️ Arquitetura do Sistema

```text
  [ Código-Fonte do Usuário ]
               │
               ▼
  [ Parser AST (Tree-sitter) ]  ──► Filtra funções, imports e vetores de risco
               │
               ▼
  [ Modelo Local (GGUF / Ollama) ] ──► Analisa intenção e lógica do trecho
               │
               ▼
  [ Validador / Relatório ] ──► CLI / JSON / SARIF
