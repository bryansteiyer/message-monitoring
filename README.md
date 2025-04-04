# message-monitoring

Este projeto implementa uma automação usando n8n + Supabase para:

- Envio de mensagens via Callbell
- Monitoramento dos status das mensagens (enqueued, sent, delivered, read, failed, mismatch e deleted)
- Registro de erros HTTP e lógicos
- Armazenamento de dados em buffer e migração em lote para tabelas principais
- Exclusão de dados do buffer após migração de dados

---

## Ferramentas/Tecnologias usadas

- [n8n](https://n8n.io)
- [Supabase](https://supabase.com)
- [Postman](https://www.postman.com)

---

## Estrutura

### Fluxo 1: Disparo de Mensagem + Registro no Supabase
![Fluxo 1](./assets/fluxo_1.jpeg)

1. Escuta um Webhook com informações como `nome` e `telefone` do contato  
   (Pode ser um Google Forms, Typeform, etc.)
2. Faz a requisição para `POST /messages/send` na API da Callbell  
   - **Sucesso:** Mapeia os dados em um Set e grava as informações em `status_mensagens_buffer` no Supabase (incluindo erros lógicos, se houver)
   - **Erro HTTP:** Mapeia os dados em um Set e grava as informações em `log_mensagens_buffer` no Supabase

---

### Fluxo 2: Webhook Message Status Updated (Callbell)
![Fluxo 2](./assets/fluxo_2.jpeg)

1. Escuta o webhook `message_status_updated` da Callbell  
2. Consulta o `uuid` da mensagem recebida em `status_mensagens_buffer`  
3. Atualiza a tabela com base na seguinte lógica:

- Se status for `enqueued`, não faz nada  
- Se status for `sent`, as colunas `status_2` e `data_status_2` são atualizadas  
- Se status for `delivered`, as colunas `status_3` e `data_status_3` são atualizadas  
- O status final é registrado nas colunas `status_4` e `data_status_4`  
- Se o status final for `failed`, registra-se o motivo da falha na coluna `motivo_falha`  

---

### Fluxo 3: Migração dos dados dos buffers para as tabelas reais
![Fluxo 3](./assets/fluxo_3.jpeg)

`log_mensagens` e `log_mensagens_buffer`

1. A cada 5 minutos os dados das tabelas `log_mensagens_buffer` e `status_mensagens_buffer` são lidos  
2. Os dados são inseridos nas tabelas reais `log_mensagens` e `status_mensagens`  
3. É feita a exclusão dos dados salvos nas tabelas buffer
