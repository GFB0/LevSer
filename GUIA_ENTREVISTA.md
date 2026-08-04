# Roteiro de Apresentação — Entrevista Final LevSer Saúde

Este guia foi elaborado para orientar a apresentação do case técnico na entrevista final com a equipe da LevSer Saúde.

---

## 🎙️ Parte 1: Apresentação Pessoal (2 a 3 minutos)

### 1. Formação e Momento Acadêmico
> *"Olá! Sou o Guilherme. Atualmente estou cursando [Seu Curso/Faculdade] no [Seu Semestre/Período]. Tenho focado meus estudos na área de Inteligência Artificial, automação de processos e desenvolvimento de integrações orientadas a eventos."*

### 2. Experiências e Projetos Relevantes
> *"Tenho experiência prática no desenvolvimento de agentes inteligentes, consumo e criação de APIs REST, webhooks e orquestração de workflows utilizando ferramentas como n8n, Docker, Python e ecossistemas LLM (OpenAI, LangChain, etc.)."*

### 3. Interesse pela Vaga na LevSer Saúde
> *"O que me atrai na LevSer é a oportunidade de aplicar Inteligência Artificial diretamente na melhoria da experiência em saúde humana, criando atendimentos mais ágeis, precisos e seguros, sem perder o cuidado ético e acolhedor que o setor de saúde exige."*

### 4. Tipo de Problema que Mais Gosta de Resolver
> *"Gosto de resolver desafios de integração de sistemas complexos — conectar plataformas heterogêneas (como WhatsApp, Chatwoot e n8n), estruturar fluxos robustos contra falhas e garantir que a IA atue com limites bem claros e previsíveis."*

---

## 💻 Parte 2: Apresentação do Case Técnico (Demonstração Prática)

Mantenha a tela compartilhada com as seguintes janelas/abas abertas:
1. **n8n**: Workflow visual (`LevSer Saúde - Agente de Atendimento WhatsApp`) e aba **Executions**.
2. **Chatwoot**: Painel de conversas/inbox com a interação registrada.
3. **WhatsApp Web / Celular**: Tela do chat onde as mensagens foram enviadas.
4. **Evolution API Manager / Postman**: Instância ativa.

---

### 📋 Respostas para os 9 Tópicos do Case Técnico

#### 1. Demonstração do fluxo funcionando de ponta a ponta
- **Ação na entrevista**: Envie uma mensagem ao vivo pelo WhatsApp (ex: *"Olá, qual o endereço da LevSer?"*).
- **O que mostrar**:
  - A mensagem aparecendo instantaneamente no **Chatwoot**.
  - A execução verde acendendo no **n8n**.
  - A resposta sendo entregue de volta no **WhatsApp** e gravada na conversa do **Chatwoot**.

#### 2. Como preparou o ambiente
- **Explicação**:
  - *"Criei uma arquitetura containerizada em **Docker** utilizando o `docker-compose`. Unifiquei o **Chatwoot** (Rails, Postgres e Redis), a **Evolution API** e o **n8n** em uma mesma rede virtual (`levser_net`)."*
  - *"Isso permitiu que os serviços conversassem localmente usando nomes de serviço DNS internos (ex: `http://levser_chatwoot:3000` e `http://levser_n8n:5678`), eliminando a necessidade de expor portas desnecessárias ao ambiente externo."*

#### 3. Como conectou Evolution API, Chatwoot, n8n e o agente
- **Explicação**:
  - **Evolution API ➔ Chatwoot**: Configurada via integração nativa da Evolution. A Evolution intercepta as mensagens do WhatsApp e sincroniza automaticamente os contatos e conversas com o Chatwoot.
  - **Chatwoot ➔ n8n**: Configurado um Webhook no Chatwoot para disparar eventos de `message_created` para a URL do n8n.
  - **n8n ➔ Agente de IA**: Utilizado nó de modelo de linguagem no n8n configurado com o prompt com as regras da LevSer.
  - **n8n ➔ Chatwoot**: Nó de `HTTP Request` no n8n realizando `POST` na REST API do Chatwoot (`/api/v1/accounts/1/conversations/{id}/messages`) para enviar a resposta como `outgoing`.
  - **Chatwoot ➔ WhatsApp**: A integração nativa do Chatwoot com a Evolution API envia a mensagem de volta para o número do cliente.

#### 4. Como disponibilizou os endpoints e webhooks
- **Explicação**:
  - *"Como os contêineres estavam na mesma rede Docker (`levser_net`), a comunicação entre Chatwoot, Evolution API e n8n ocorreu de forma direta via HTTP interno."*
  - *"Para testes externos ou túneis locais quando necessário, utilizei soluções de tunelamento como **ngrok** / **Cloudflare Tunnel** para expor a porta do n8n ao webhook do Chatwoot."*

#### 5. Como identificou mensagens recebidas e enviadas
- **Explicação**:
  - *"Analisando a estrutura do payload de JSON enviado pelo webhook do Chatwoot no evento `message_created`, identifiquei o campo `message_type`:"*
    - `incoming`: Mensagem enviada pelo cliente (WhatsApp -> Chatwoot).
    - `outgoing`: Mensagem enviada pelo atendente/bot (Chatwoot -> WhatsApp).
    - `private`: Mensagem de nota interna na conversa.
  - *"Além disso, o campo `private` (booleano) indica se é um comentário interno da equipe."*

#### 6. Como evitou loops e respostas duplicadas (Filtro Anti-Loop)
- **Explicação**:
  - *"O maior risco em integrações de atendimento via webhook é a mensagem gerada pelo próprio bot disparar novamente o webhook do Chatwoot, criando um loop infinito de auto-respostas."*
  - *"Resolvi isso inserindo um nó de **Filtro (IF)** logo após o Webhook no n8n que valida três condições obrigatórias:"*
    1. `message_type == 'incoming'`
    2. `private == false`
    3. `event == 'message_created'`
  - *"Se o evento for uma mensagem de saída (`outgoing`) gerada pelo próprio bot, a condição avalia como Falsa e o n8n encerra a execução **imediatamente**, sem chamar a IA e sem enviar mensagem."*

#### 7. Principais dificuldades encontradas
- **Explicação**:
  - Mapear corretamente os campos do payload do Chatwoot dentro das expressões do n8n (ex: extrair `conversation.id` e `account.id`).
  - Garantir a autenticação correta da API do Chatwoot utilizando o `api_access_token` no header.
  - Garantir que a resposta da IA respeitasse estritamente o limite de **duas frases** e as diretrizes clínicas sem inventar informações.

#### 8. Como investigou e resolveu os problemas
- **Explicação**:
  - Inspeção dos logs brutos do webhook no painel de **Execuções do n8n** para entender a estrutura exata do JSON recebido.
  - Utilização do Postman para testar os endpoints da API REST do Chatwoot isoladamente antes de integrar no nó HTTP Request.
  - Testes sistemáticos de prompt engineering e ajuste de parâmetros (`temperature = 0.3`) no modelo de linguagem para assegurar respostas curtas e objetivas.

#### 9. O que melhoraria caso a solução fosse utilizada em produção
- **Explicação**:
  1. **Persistência de Memória de Conversa**: Implementar armazenamento de histórico de contexto (ex: Postgres / Redis no n8n / LangChain) para que o agente lembre de mensagens anteriores na mesma sessão.
  2. **Transbordo Humano Automatizado**: Criar uma regra onde, caso a mensagem seja classificada com dúvida clínica ou solicitação humana, o n8n atribua automaticamente a conversa a uma fila de atendentes humanos no Chatwoot (`assignee_id` / `status: open`) e pause o robô.
  3. **Monitoramento e Alertas (DLQ)**: Adicionar nó de tratamento de erros no n8n (*Error Trigger*) para notificar a equipe via Slack/Discord caso a API da OpenAI ou Chatwoot apresente indisponibilidade.
  4. **Segurança e Variáveis de Ambiente**: Mover os tokens de API para o gerenciador de credenciais nativo do n8n e utilizar segredos no Docker (`docker secrets` / `.env`).

---

## 🤖 Parte 3: Explicando o Uso de Inteligência Artificial

Quando questionado sobre o uso de ferramentas de IA durante o desenvolvimento do case:

- **Onde utilizou IA**:
  - Para acelerar a montagem do arquivo `docker-compose.yml` e checar sintaxe do JSON do workflow do n8n.
  - Para refinamento do prompt do sistema (*System Prompt*) visando garantir o limite de duas frases.
- **O que foi gerado**:
  - Boilerplate de configurações do Docker e modelos de payload de webhooks.
- **Como verificou o resultado**:
  - Testou cada contêiner e endpoint manualmente via `docker logs`, `curl` e Postman.
  - Executou os 3 testes do case enviando mensagens reais pelo WhatsApp e validando o comportamento na aba de execuções do n8n.
- **Decisões técnicas tomadas por você**:
  - Escolha da estratégia do nó de filtro anti-loop baseada no campo `message_type` do Chatwoot.
  - Arquitetura da rede interna no Docker para comunicação segura entre contêineres.
