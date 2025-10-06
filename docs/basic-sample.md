# Exemplo básico usando Celery com FastAPI e Redis

Neste exemplo, criaremos um novo endpoint na aplicação FastAPI que usará o Celery para processar uma tarefa em segundo plano. A tarefa será simular um processamento longo, como no exemplo anterior, mas agora sem bloquear o servidor.

## Instalação das dependências

Primeiro, precisamos instalar as dependências necessárias. Já vamos instalar tudo de uma vez. No ambiente virtual do projeto, instale o Celery:

```bash
pip install celery redis python-dotenv flower
```

??? note "Dependências instaladas"

    As dependências instaladas são:

    -  `celery`: A biblioteca principal para comunicação assíncrona.
    -  `redis`: O broker de mensagens que usaremos para gerenciar as filas de tarefas.
    -  `python-dotenv`: Para carregar variáveis de ambiente a partir de um arquivo `.env`.
    -  `flower`: Uma ferramenta para monitorar e gerenciar tarefas do Celery.

## Configuração do Celery

Vamos seguir os passos para configurar o Celery na aplicação.

### Configuração do ambiente

Inicialmente, vamos criar um arquivo `.env` na raiz do projeto, para armazenar as variáveis de ambiente necessárias para a configuração do Celery. O conteúdo do arquivo `.env` será o seguinte:

```env title=".env" linenums="1"
REDIS_URL=redis://:myStrongPassword@localhost:6379/0
```

Estamos definindo a variável `REDIS_URL`, que contém a URL de conexão com o Redis, incluindo a senha que definimos anteriormente.

### Definindo as variáveis de configuração e o Celery

Agora, criaremos um arquivo `config.py` dentro da pasta `src`, que conterá a configuração do Celery. O conteúdo do arquivo `config.py` será o seguinte:

```python title="./src/config.py" linenums="1"
import os
from dotenv import load_dotenv
from celery import Celery

load_dotenv()

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

celery_app = Celery("tasks", broker=REDIS_URL, backend=REDIS_URL)
```

Esse arquivo carrega as variáveis de ambiente do arquivo `.env` e define a variável `REDIS_URL`, que será usada para configurar o Celery. Caso a variável de ambiente não esteja definida, ele usará um valor padrão.

### Definindo o worker do Celery

Agora, criaremos um arquivo `worker.py` dentro da pasta `src`, que conterá a definição do worker do Celery. O conteúdo do arquivo `worker.py` será o seguinte:

```python title="./src/worker.py" linenums="1"
from app.config import celery_app
```

Nesse caso não precisamos adicionar mais nada, pois o worker do Celery já está configurado no arquivo `config.py`. Embora esse arquivo possa parecer redundante, ele é útil para iniciar o worker do Celery a partir da linha de comando.

## Criando a tarefa assíncrona

Agora, criaremos um arquivo `tasks.py` dentro da pasta `src`, que conterá a definição da tarefa assíncrona que será executada pelo Celery. O conteúdo do arquivo `tasks.py` será o seguinte:

```python title="./src/tasks.py" linenums="1"
from app.config import celery_app

@celery_app.task
def long_task():
    import time
    time.sleep(10)
    return "Tarefa concluída"
```

Essa tarefa simula um processamento longo, dormindo por 10 segundos antes de retornar uma mensagem indicando que a tarefa foi concluída. Note que essa mensagem de retorno não será diretamente visível para o cliente que fez a requisição, mas pode ser consultada posteriormente. Ela é enviada e armazenada no backend de resultados do Celery, que neste caso é o Redis.

### Atualizando o servidor FastAPI

Agora, vamos atualizar o arquivo `main.py` dentro da pasta `src`, para adicionar um novo endpoint que usará o Celery para processar a tarefa em segundo plano. O conteúdo atualizado do arquivo `main.py` será o seguinte:

```python title="./src/main.py" linenums="13"
@app.get("/async-process/")
def async_process():
    from app.tasks import long_task
    task = long_task.delay()
    return {"task_id": task.id, "status": "Tarefa iniciada"}
```

Esse novo endpoint `/async-process/` chama a tarefa `long_task` de forma assíncrona, usando o método `delay()`, que enfileira a tarefa para ser executada pelo Celery. O endpoint retorna imediatamente um JSON contendo o ID da tarefa e uma mensagem indicando que a tarefa foi iniciada.

??? note "Código completo do main.py"

    ```python title="./src/main.py" linenums="1"
    from fastapi import FastAPI

    from app.tasks import long_task

    app = FastAPI()

    @app.get("/sync-process/")
    def sync_process():
        import time

        time.sleep(10)
        return {"message": "Processamento longo concluído"}

    @app.get("/async-process/")
    def async_process():
        task = long_task.delay()
        return {"task_id": task.id, "status": "Tarefa iniciada"}
    ```

Note que o endpoint `/sync-process/` permanece inalterado, para que possamos comparar o comportamento dos dois endpoints. Ainda, no novo endpoint, importamos a tarefa `long_task` diretamente no escopo da função no exemplo, mas a importação poderia ser feita no início do arquivo, como no exemplo comentado.

??? danger "Importação dentro da função"

    Embora seja comum fazer a importação no início do arquivo, como no exemplo comentado, pode ser interessante fazer a importação dentro da função, para evitar problemas de importação circular em projetos maiores, quando o arquivo `tasks.py` pode acabar importando algo do `main.py`.

## Executando o exemplo

Agora, vamos executar o exemplo completo. Precisamos iniciar o servidor FastAPI, o worker do Celery e o Flower para monitorar as tarefas.

```bash
uvicorn src.main:app --reload
celery -A src.worker worker --loglevel=info
celery -A src.worker flower
```

Vamos explicar os comandos:

- `uvicorn src.main:app --reload`: Inicia o servidor FastAPI, com recarregamento automático. A aplicação estará disponível em `http://localhost:8000`. E a documentação Swagger UI em `http://localhost:8000/docs`.
- `celery -A src.worker worker --loglevel=info`: Inicia o worker do Celery, que ficará escutando as tarefas enfileiradas.
- `celery -A src.worker flower`: Inicia o Flower, que é uma interface web para monitorar as tarefas do Celery. Acesse em `http://localhost:5555`.

Vamos agora testar o novo endpoint `/async-process/`. Acesse a documentação Swagger UI em `http://localhost:8000/docs`, clique no endpoint `/async-process/`, depois em "Try it out" e "Execute". Você verá que a requisição retorna imediatamente, com o ID da tarefa e a mensagem indicando que a tarefa foi iniciada.

Caso você acesse a visualização do Flower em `http://localhost:5555`, verá a tarefa sendo processada pelo worker do Celery. Após 10 segundos, a tarefa será concluída, e você poderá ver o _status_ da tarefa atualizado na interface do Flower.
