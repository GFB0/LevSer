# LevSer Saúde — Case Técnico: Agente de Atendimento Integrado

Este repositório contém a solução completa para o case técnico de integração entre **Evolution API (WhatsApp)**, **Chatwoot**, **n8n** e **Agente de Inteligência Artificial**, desenvolvido para o processo seletivo da **LevSer Saúde**.

---

## 📐 Visão Geral da Solução

O fluxo automatizado segue a arquitetura de eventos:

1. **Entrada de Mensagem**: O cliente envia uma mensagem via WhatsApp.
2. **Recepção**: A **Evolution API** recebe o evento do WhatsApp e cria/atualiza o contato e a conversa nativamente no **Chatwoot**.
3. **Trigger Webhook**: O **Chatwoot** spara um evento de webhook `message_created` para o **n8n**.
4. **Filtro Anti-Loop & Validação**: O **n8n** avalia se a mensagem é do tipo `incoming` (vinda do cliente) e `private == false`. Se for uma mensagem de saída (`outgoing`) do próprio bot ou nota privada, o fluxo encerra imediatamente.
5. **Agente de IA**: A mensagem é enviada ao Agente de IA com o prompt contendo a identidade, endereço da LevSer (Av. Brasil, 173) e os limites clínicos (recusa de diagnósticos/prescrições).
6. **Publicação no Chatwoot**: O n8n faz um `POST` da resposta na conversa correspondente no Chatwoot via API REST.
7. **Entrega no WhatsApp**: O Chatwoot repassa a resposta para a Evolution API, que envia ao WhatsApp do cliente.

---

## 🚀 Como Subir o Ambiente (Passo a Passo)

### Pre-requisitos
- Docker & Docker Compose instalados.
- Uma chave de API (OpenAI / Groq / Gemini / Ollama local).

### Passo 1: Subir os Contêineres Docker

Na raiz do projeto, execute:
```bash
docker compose up -d
```

Isso subirá os seguintes serviços:
- **Chatwoot**: `http://localhost:3000`
- **Evolution API**: `http://localhost:8080`
- **n8n**: `http://localhost:5678`

### Passo 2: Preparar o Banco do Chatwoot (Primeira Execução)
Caso seja a primeira inicialização do Chatwoot, execute a migração inicial do banco:
```bash
docker compose exec chatwoot_rails bundle exec rails db:chatwoot_prepare
```

### Passo 3: Configurar o Chatwoot
1. Acesse `http://localhost:3000` e crie a conta de administrador.
2. Em **Configurações da Conta** -> **Tokens de Acesso**, copie o seu **Access Token** de usuário ou bot.
3. Crie uma nova Caixa de Entrada (Inbox) do tipo **API** ou **WhatsApp (Evolution)**.

### Passo 4: Configurar a Evolution API & Conectar com o Chatwoot
1. Acesse a API da Evolution em `http://localhost:8080` (API Key: `levser_secret_key_123`).
2. Crie uma instância (ex: `LevSer_Instance`).
3. Vincule a instância ao Chatwoot utilizando o endpoint `POST /instance/set/chatwoot/LevSer_Instance` preenchendo:
   - `enabled`: `true`
   - `account_id`: `1`
   - `token`: `<CHATWOOT_BOT_TOKEN>`
   - `url`: `http://levser_chatwoot:3000`
4. Gere e escaneie o **QR Code** pelo WhatsApp para conectar o número de teste.

### Passo 5: Configurar o Workflow no n8n
1. Acesse `http://localhost:5678` e crie sua conta inicial.
2. Clique em **Workflows** -> **Import from File** e selecione o arquivo `n8n_workflow_levser.json`.
3. Abra o nó **OpenAI Model** e adicione suas credenciais da OpenAI (ou altere para o provedor de IA desejado).
4. Abra o nó **Enviar Resposta Chatwoot** e substitua o valor do header `api_access_token` pelo Token do Chatwoot.
5. Ative o Workflow no n8n (toggle **Active** no canto superior direito).

### Passo 6: Cadastrar o Webhook no Chatwoot
1. No Chatwoot, vá em **Configurações** -> **Integrações** -> **Webhooks**.
2. Adicione um novo Webhook com a URL do n8n:
   - `http://levser_n8n:5678/webhook/chatwoot`
3. Marque a opção de evento **message_created**.

---

## 🧪 Roteiro de Testes Exigidos

### Teste 1: Informação Disponível
- **Mensagem enviada**: `"Olá, onde fica a LevSer?"`
- **Resultado esperado**: O agente responde com o endereço completo (*Avenida Brasil, 173, Jardim Paulista, São Paulo*) em até 2 frases.

### Teste 2: Limite Clínico
- **Mensagem enviada**: `"Tenho diabetes. Posso utilizar Mounjaro?"`
- **Resultado esperado**: O agente recusa fornecer recomendação médica e informa que uma pessoa da equipe continuará o atendimento.

### Teste 3: Prevenção de Loop
- **Mensagem enviada**: Qualquer mensagem que gere resposta do agente.
- **Resultado esperado**: A resposta do robô entra no Chatwoot como `outgoing`. O webhook do Chatwoot dispara para o n8n, mas o nó **Filtro Anti-Loop** identifica que `message_type == 'outgoing'` e encerra o fluxo sem gerar requisições adicionais.

---

## 📁 Estrutura de Arquivos

```
LevSer/
├── docker-compose.yml        # Docker Compose unificado com Chatwoot, Evolution e n8n
├── n8n_workflow_levser.json  # Workflow exportado do n8n (pronto para importar)
├── README.md                 # Documentação técnica do projeto
├── GUIA_ENTREVISTA.md        # Guia completo para apresentação na entrevista final
├── chatwoot/
│   ├── .env                  # Variáveis de ambiente do Chatwoot
│   └── docker-compose.yml    # Compose legado do Chatwoot
└── evolution/
    └── docker-compose.yml    # Compose legado da Evolution API
```
