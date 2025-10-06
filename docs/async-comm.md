# Por que comunicação assíncrona

Na forma de tradicional de comunicação síncrona, o cliente faz uma requisição ao servidor e espera a resposta. Durante esse tempo, o cliente fica bloqueado, ou seja, não pode fazer mais nada até que a resposta seja recebida. Isso pode ser um problema em aplicações web, onde o tempo de resposta do servidor pode ser alto, e o usuário pode ficar frustrado esperando.

Na comunicação assíncrona, o cliente faz uma requisição ao servidor e não espera pela resposta imediatamente. Em vez disso, o cliente pode continuar a executar outras tarefas enquanto aguarda a resposta do servidor. Isso melhora a experiência do usuário, pois a interface do aplicativo pode permanecer responsiva, mesmo que o servidor esteja demorando para responder.

Alguns cenários onde a comunicação assíncrona é benéfica incluem:

- Processamento de tarefas demoradas: em aplicações web, algumas tarefas podem levar muito tempo para serem concluídas, como o processamento de imagens ou a geração de relatórios. Nesses casos, a comunicação assíncrona permite que o usuário continue a usar a aplicação enquanto a tarefa está sendo processada em segundo plano.
- Notificações em tempo real: em aplicações que exigem atualizações em tempo real, como chats ou sistemas de monitoramento, a comunicação assíncrona permite que o servidor envie notificações para o cliente assim que novos dados estiverem disponíveis, sem que o cliente precise fazer requisições constantes ao servidor.
- Integração com serviços externos: em aplicações que dependem de serviços externos, como APIs de terceiros, a comunicação assíncrona permite que o cliente faça requisições a esses serviços sem bloquear a interface do usuário, melhorando a experiência do usuário.
- Envio de e-mails: em aplicações que enviam e-mails, a comunicação assíncrona permite que o envio de e-mails seja feito em segundo plano, sem bloquear a interface do usuário.
- Upload e conversão de arquivos: em aplicações que permitem o upload e conversão de arquivos, a comunicação assíncrona permite que o usuário continue a usar a aplicação enquanto o arquivo está sendo processado em segundo plano.
- Processamento de relatórios: em aplicações que geram relatórios, a comunicação assíncrona permite que o usuário continue a usar a aplicação enquanto o relatório está sendo gerado em segundo plano.

## Exemplo de comunicação síncrona bloqueada

Faremos um exemplo simples, usando um servidor FastAPI para ilustrar a comunicação síncrona bloqueada. Inicialmente, vamos criar um diretório para o projeto, e abrir no VSCode. Dentro do diretório do projeto, criaremos um ambiente virtual e instalaremos o FastAPI e o Uvicorn:

=== "Linux"

    ``` bash
    python -m venv venv
    source venv/bin/activate
    pip install fastapi uvicorn
    ```

=== "Windows"

    ``` bash
    python -m venv venv
    .\venv\Scripts\activate
    pip install fastapi uvicorn
    ```

Crie uma pasta chamada `src` e adicione um arquivo `main.py`, com o seguinte conteúdo.:

```python title="./src/main.py" linenums="1"
from fastapi import FastAPI

app = FastAPI()


@app.get("/sync-process/")
def sync_process():
    import time

    time.sleep(10)
    return {"message": "Processamento longo concluído"}
```

Em seguinte, iniciamos o servidor com o comando:

```bash
uvicorn src.main:app --reload
```

Para testar, acesse a documentação Swagger UI em `http://localhost:8000/docs`. Clique no endpoint `/sync-process/` e depois em "Try it out" e "Execute". Você verá que a requisição demora 10 segundos para ser concluída, e durante esse tempo, o navegador fica bloqueado, ou seja, você não pode fazer mais nada até que a resposta seja recebida.

Caso você tivesse feito um cliente API REST, por exemplo em um framework JavaScript como Vue.js ou React, a interface do usuário ficaria bloqueada durante esse tempo, o que não é uma boa experiência para o usuário.
