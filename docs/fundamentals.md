# Fundamentos da comunicação assíncrona

Vamos agora entender alguns fundamentos básicos sobre comunicação assíncrona, e os principais elementos envolvidos nesse tipo de comunicação.

Na comunicação assíncrona, os principais elementos envolvidos são:

- Broker de mensagens: é um componente que gerencia a fila de mensagens, com a função de armazenar mensagens ou tarefas na fila. O _broker_ de mensagens pode ser um sistema de mensagens como RabbitMQ, Redis, ou outro sistema de mensagens que suporte filas.
- Worker: é um componente que consome as mensagens da fila de mensagens, e processa as tarefas solicitadas pelo cliente. O worker pode ser executado em um processo separado, ou em um servidor dedicado, e pode ser escalado horizontalmente para lidar com um grande volume de requisições. Entre os principais exemplos para Python estão Celery, RQ (Redis Queue), Dramatiq, entre outros.
- Backend de resultados: é um componente que armazena os resultados das tarefas processadas pelos workers. O backend de resultados pode ser um banco de dados, um sistema de cache, ou outro tipo de armazenamento persistente. Um exemplo comum é o Redis, mas pode ser usado também bancos relacionais como PostgreSQL, MySQL, SQLite, entre outros.

Podemos fazer uma analogia com um sistema de troca de correspondências via correios tradicional. Nesse caso:

- Broker de mensagens: seria a caixa de correio do remetente, onde as cartas são depositadas para serem enviadas
- Worker: seriam os carteiros, que pegam as cartas das filas e as entregam
- Backend de resultados: seria um sistema de rastreamento de correspondências, onde o remetente pode verificar se a carta foi entregue e quando.

## Celery - O orquestrador de tarefas assíncronas

O Celery é uma biblioteca Python que facilita a implementação de comunicação assíncrona em aplicações. Ele é um orquestrador de tarefas assíncronas, que permite que você defina tarefas que podem ser executadas em segundo plano, sem bloquear a execução do código principal da aplicação.

O Celery é altamente escalável, e pode ser usado em aplicações de pequeno a grande porte. Ele suporta múltiplos _brokers_ de mensagens, incluindo RabbitMQ, Redis, Amazon SQS, entre outros. Além disso, o Celery suporta múltiplos _backends_ de resultados, incluindo Redis, bancos relacionais como PostgreSQL, MySQL, SQLite, entre outros.

O Celery é fácil de usar, e possui uma API simples e intuitiva. Ele permite que você defina tarefas como funções Python normais, e fornece mecanismos para agendar a execução dessas tarefas em segundo plano. O Celery também suporta a execução de tarefas periódicas, o que é útil para tarefas que precisam ser executadas em intervalos regulares.

!!! note "O BackgroundTasks do FastAPI"

    No caso do uso do FastAPI, poderíamos fazer uso da função `BackgroundTasks`, que permite a execução de tarefas em segundo plano, mas essa abordagem é limitada, pois as tarefas são executadas no mesmo processo do servidor web, o que pode levar a problemas de desempenho e escalabilidade. Além disso, o `BackgroundTasks` não oferece suporte a filas de mensagens, o que limita a capacidade de lidar com um grande volume de requisições.

## Redis - O corretor de mensagens

O Redis é um sistema de armazenamento de dados em memória, que pode ser usado como um _broker_ de mensagens para comunicação assíncrona. Ele é altamente escalável, e pode lidar com um grande volume de requisições. O Redis é fácil de usar, e possui uma API simples e intuitiva. Ele suporta múltiplos tipos de dados, incluindo strings, listas, conjuntos, hashes, entre outros.

O Redis é uma escolha popular como _broker_ de mensagens para comunicação assíncrona, devido à sua alta performance e escalabilidade. Ele é capaz de lidar com um grande volume de mensagens, e pode ser usado em aplicações de pequeno a grande porte.

!!! note "RabbitMQ - Outra opção de broker de mensagens"

    Uma alternativa ao Redis como _broker_ de mensagens é o RabbitMQ, que é um sistema altamente escalável, e pode lidar com um grande volume de requisições. Ele suporta múltiplos protocolos de mensagens, incluindo AMQP, MQTT, entre outros. O RabbitMQ é uma escolha popular para aplicações que exigem alta confiabilidade e durabilidade das mensagens.

### Usando um docker compose para subir o Redis

Para facilitar o uso do Redis como _broker_ de mensagens, podemos usar um arquivo `docker-compose.yml` para subir uma instância do Redis em um contêiner Docker. O arquivo `docker-compose.yml` pode ser o seguinte:

```yaml title="docker-compose.yml" linenums="1"
services:
  redis:
    image: redis:8
    container_name: redis
    command: redis-server --requirepass myStrongPassword
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
```

Para subir o Redis, basta executar o comando:

```bash
docker-compose up
```

Agora, quando formos configurar o Celery, podemos usar o endereço `redis://:myStrongPassword@localhost:6379/0` para conectar ao Redis, onde `myStrongPassword` é a senha definida no comando de inicialização do Redis. Este é uma forma simples, apenas para testes locais. Em um ambiente de produção, é importante configurar o Redis com mais segurança, incluindo o uso de senhas fortes, configuração de firewalls, entre outras medidas de segurança.
