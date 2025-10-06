# Envio de emails em segundo plano

Em aplicações web, o envio de emails é uma tarefa comum, mas que pode ser demorada, dependendo do provedor de email e da quantidade de emails a serem enviados. Se o envio de emails for feito de forma síncrona, ou seja, bloqueando a requisição do usuário até que o email seja enviado, isso pode levar a uma má experiência do usuário, que pode ficar esperando por um longo tempo.

Para resolver esse problema, podemos usar comunicação assíncrona para enviar emails em segundo plano, sem bloquear a requisição do usuário. Dessa forma, o usuário pode continuar a usar a aplicação enquanto o email está sendo enviado em segundo plano. Isso melhora a experiência do usuário, e também permite que a aplicação lide com um maior volume de requisições, já que o envio de emails não bloqueia o processamento das requisições.

Nos nossos exemplo, usaremos a ferramenta Mailtrap, que é um serviço de email para desenvolvimento e teste. O Mailtrap permite que você envie emails para uma caixa de entrada virtual, onde você pode visualizar os emails enviados, sem que eles sejam realmente entregues aos destinatários. Isso é útil para testar o envio de emails em aplicações web, sem correr o risco de enviar emails reais para usuários.

## Criando uma conta no Mailtrap

Para criar uma conta no Mailtrap, acesse o site [https://mailtrap.io/](https://mailtrap.io/) e clique em "Sign Up". Preencha o formulário de cadastro com seu nome, email e senha, e clique em "Create Account". Você também pode se cadastrar usando sua conta do GitHub, Google ou LinkedIn. Após criar a conta, você será redirecionado para o painel do Mailtrap.

Dentro da conta, você deve gerar um token de autenticação para usar na aplicação. Para isso, clique em "Settings" no menu lateral, e depois em "API Tokens". Em seguida, clique em "Add Token", e dê um nome para o token, como "Celery Email", e atribua a ele a permissão de _Account Admin_. Então, copie o token gerado, pois você precisará dele para configurar o envio de emails na aplicação.

## Configurando o mailtrap na aplicação

Inicialmente, vamos adicionar as variáveis de ambiente necessárias para configurar o Mailtrap. No arquivo `.env`, adicione as seguintes linhas:

```env title=".env" linenums="1"
REDIS_URL=redis://:myStrongPassword@localhost:6379/0

MAILTRAP_TOKEN=MEU_TOKEN_GERADO_ANTERIORMENTE
MAIL_FROM=no-reply@meucurso.ifc.dev
```

Substitua o valor de `MAILTRAP_TOKEN` pelo token gerado na sua conta do Mailtrap. A variável `MAIL_FROM` define o endereço de email que será usado como remetente nos emails enviados pela aplicação.

Em seguida, edite o arquivo `config.py` para adicionar as novas variáveis de ambiente:

```python title="./src/config.py" linenums="11"
MAILTRAP = {
    "token": os.getenv("MAILTRAP_TOKEN"),
    "from_email": os.getenv("MAIL_FROM"),
}

client_mt = mt.MailtrapClient(token=MAILTRAP["token"])
...
```

??? Note "Versão completa do arquivo config.py"

    O arquivo `config.py` completo deve ficar assim:

    ```python title="./src/config.py" linenums="1"
    import os
    from dotenv import load_dotenv
    from celery import Celery

    load_dotenv()

    REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

    MAILTRAP = {
        "token": os.getenv("MAILTRAP_TOKEN"),
        "from_email": os.getenv("MAIL_FROM"),
    }

    celery_app = Celery("tasks", broker=REDIS_URL, backend=REDIS_URL)
    ```

## Definindo a tarefa de envio de email

Agora, criaremos uma nova tarefa no Celery para enviar emails usando o Mailtrap. No arquivo `tasks.py`, import as variáveis necessárias no início do arquivo e instancie o cliente mailtrap:

```python title="./src/tasks.py" linenums="3"
import asyncio
import mailtrap as mt

from app.config import MAILTRAP

client_mt = mt.MailtrapClient(token=MAILTRAP["token"])
```

Em seguida, defina a tarefa `send_email` também dentro do arquivo `tasks.py`:

```python title="./src/tasks.py" linenums="12"
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
```

Note que a tarefa `process_data_and_sending_email` simula um processamento demorado, dormindo por 10 segundos, e depois chama a função `send_email` para enviar o email usando o Mailtrap. A função `send_email` cria um objeto `Mail` com os detalhes do email, e usa o cliente do Mailtrap para enviar o email. Note que a função `send_email` é uma função assíncrona, e por isso usamos `asyncio.run` para executá-la dentro da tarefa do Celery.

Note também que usamos um endereço de email fictício como remetente (`mailtrap@demomailtrap.co`). Esse endereço é aceito pelo Mailtrap para testes, mas em uma aplicação real, você deve usar um endereço de email válido e configurado corretamente. Contudo, é necessário que o domínio do email remetente seja válido, ou seja, que o domínio possua registros DNS corretos para evitar que os emails sejam marcados como spam pelos provedores de email.

## Atualizando o servidor FastAPI

Finalmente, vamos atualizar o servidor FastAPI para usar a nova tarefa de envio de email. No arquivo `main.py`, importe a nova tarefa no início do arquivo:

```python title="./src/main.py" linenums="19"
@app.post("/process-and-email/")
def process_email(email: str, data: dict):
    task = process_data_and_sending_email.delay(data, email)
    return {"message": "Processamento iniciado", "task_id": task.id}
```

??? Note "Código completo do main.py"

    O arquivo `main.py` completo deve ficar assim:

    ```python title="./src/main.py" linenums="1" hl_lines="3 19-22"
    from fastapi import FastAPI

    from app.tasks import long_task, process_data_and_sending_email

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
    ```
