# Controle de status das tarefas

Agora, vamos explorar como monitorar o status das tarefas assíncronas usando Celery. Isso é crucial para entender o progresso das operações em segundo plano e fornecer feedback aos usuários. Para isso, vamos simular uma atividade que leva algum tempo para ser concluída, mas que pode ser controlada por meio de um status que informe o progresso.

## Criando uma tarefa com status

Nesse caso, vamos criar uma tarefa que simula um processamento longo e que atualiza seu status periodicamente. Para isso, vamos usar o backend de resultados do Celery, que permite armazenar e consultar o estado das tarefas.

```python title="./src/tasks.py" linenums="35"
@celery_app.task(bind=True)
def long_task_with_progress(self, total: int = 100):
    import time
    for i in range(total):
        time.sleep(1)
        self.update_state(state="PROGRESS", meta={"current": i, "total": total})
    return {"status": "completed"}
```

??? note "Versão completa do arquivo tasks.py"

    ```python title="./src/tasks.py" linenums="1" hl_lines="35-41"
    from app.config import celery_app, MAILTRAP

    client_mt = mt.MailtrapClient(token=MAILTRAP["token"])

    @celery_app.task
    def long_task():
        import time
        time.sleep(10)
        return "Tarefa concluída"
        celery_app = Celery(
            "tasks",
            broker="redis://localhost:6379/0",
            backend="redis://localhost:6379/0"
        )

    @celery_app.task
    def process_data_and_sending_email(data: str, to_email: str):
        import time

        time.sleep(10)

        subject = "Processamento concluído"
        body = f"O processamento dos dados foi concluído com sucesso. Dados processados: {data}"
        asyncio.run(send_email(to_email, subject, body))

    async def send_email(to_email: str, subject: str, body: str):
        mail = mt.Mail(
            sender=mt.Address(email="mailtrap@demomailtrap.co", name="Mailtrap Test"),
            to=[mt.Address(email=to_email, name="User")],
            subject=subject,
            text=body,
        )
        client_mt.send(mail)

    @celery_app.task(bind=True)
    def long_task_with_progress(self, total: int = 100):
        import time
        for i in range(total):
            time.sleep(1)
            self.update_state(state="PROGRESS", meta={"current": i, "total": total})
        return {"status": "completed"}
    ```

Note que a tarefa `long_task_with_progress` está configurada com o parâmetro `bind=True`, o que permite que a tarefa acesse seu próprio estado e atualize-o conforme necessário. A cada iteração do loop, a tarefa simula um atraso de 1 segundo e atualiza seu estado para "PROGRESS", incluindo informações sobre o progresso atual.

Essa informação é armazenada no backend de resultados do Celery, que pode ser consultado posteriormente para verificar o status da tarefa.

## Consumindo a terafa e verificando o status

Para consumir essa tarefa e verificar seu status, você pode usar o seguinte código:

```python title="./src/main.py" linenums="24"
@app.post("/process-data-with-status/")
def process_data_with_status(time: int = 100):
    task = long_task_with_progress.delay(time)
    return {"message": "Processamento iniciado", "task_id": task.id}


@app.get("/task-status/{task_id}")
def task_status(task_id: str):
    result = AsyncResult(task_id, app=celery_app)
    if result.state == "PENDING":
        return {"state": result.state, "progress": 0}
    if result.state == "PROGRESS":
        return {"state": result.state, "progress": result.info.get("percent", 0)}
    if result.state == "SUCCESS":
        return {"state": result.state, "result": result.result}
    return {"state": result.state, "info": str(result.info)}
```

??? note "Versão completa do arquivo main.py"

    O arquivo `main.py` completo deve ficar assim:

    ```python title="./src/main.py" linenums="1" hl_lines="2 4 25-39"
    from fastapi import FastAPI
    from celery.result import AsyncResult

    from app.tasks import long_task, process_data_and_sending_email, long_task_with_progress

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

    @app.post("/process-and-email/")
    def process_email(email: str, data: dict):
        task = process_data_and_sending_email.delay(data, email)
        return {"message": "Processamento iniciado", "task_id": task.id}

    @app.post("/process-data-with-status/")
    def process_data_with_status(time: int = 100):
        task = long_task_with_progress.delay(time)
        return {"message": "Processamento iniciado", "task_id": task.id}

    @app.get("/task-status/{task_id}")
    def task_status(task_id: str):
        result = AsyncResult(task_id, app=celery_app)
        if result.state == "PENDING":
            return {"state": result.state, "progress": 0}
        if result.state == "PROGRESS":
            return {"state": result.state, "progress": result.info.get("percent", 0)}
        if result.state == "SUCCESS":
            return {"state": result.state, "result": result.result}
        return {"state": result.state, "info": str(result.info)}
    ```

Note que a rota `/process-data-with-status/` inicia a tarefa `long_task_with_progress` e retorna o ID da tarefa. A rota `/task-status/{task_id}` permite consultar o status da tarefa usando seu ID. Dependendo do estado da tarefa, ela retorna informações diferentes, como o progresso atual ou o resultado final. Fazemos uso da classe `AsyncResult` do Celery para consultar o estado da tarefa. Essa consulta é feita ao backend de resultados configurado no Celery.

Esse _status_ também pode ser consultado diretamente no painel do Flower, que fornece uma interface web para monitorar as tarefas do Celery.
