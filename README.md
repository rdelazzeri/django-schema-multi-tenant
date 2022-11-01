# Django Multi Tenant project based on postgres schemas
Projeto para uma aplicação com multiplos clientes compartilhando o mesmo código e o mesmo banco de dados em schemas diferentes, utilizando este recurso do Postgresql

## Sequencia inicial:
> Esta sequencia foi adaptada do material original indicado nas referências.
> Outras opções de configuração devem ser consultadas na documentação do `django-tenant-schemas`

Criar ambiente
```
python -m venv venv 
.\venv\scripts\activate
```

Instalar dependências do projeto
```
python -m pip install --upgrade pip
pip install django, django-tenant-schemas
```

Criar o projeto
```
django-admin startproject multidb .
python manage.py startapp core

```

Criar um novo banco de dados postgreSQL
```
nome do db: schema
```

Fazendo alterações no `settings.py`

Altere a conexão com o banco de dados para usar o `Tenant_schemas`:
```
DATABASES = {
    'default': {
        'ENGINE': 'tenant_schemas.postgresql_backend',
        'NAME': 'schema',
        'USER': 'postgres',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '5432',
        # ..
    }
}
# ...
DATABASE_ROUTERS = ("tenant_schemas.routers.TenantSyncRouter",)
```

Adicioanar o middleware no todo das opções.
```
tenant_schemas.middleware.SuspiciousTenantMiddleware
```

```
...

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'core', #novo
]

```
## Criar o Tenent model
Este é o modelo onde contem as informações do Tenent, o modelo deve herdar de `TenantMixin`. 
Ele será criado dentro do app `core`: 
```
from django.db import models
from tenant_schemas.models import TenantMixin

class Client(TenantMixin):
    name = models.CharField(max_length=100)
    paid_until =  models.DateField()
    on_trial = models.BooleanField()
    created_on = models.DateField(auto_now_add=True)

    # default true, schema will be automatically created and synced when it is saved
    auto_create_schema = True
```


## Definir as aplicações compartilhadas e exclusivas:
```
SHARED_APPS = (
    'tenant_schemas',  # mandatory, should always be before any django app
    'core', # you must list the app where your tenant model resides in

    'django.contrib.contenttypes',

    # everything below here is optional
    'django.contrib.auth',
    'django.contrib.sessions',
    'django.contrib.sites',
    'django.contrib.messages',
    'django.contrib.admin',
)

TENANT_APPS = (
    'django.contrib.contenttypes',

    # your tenant-specific apps
    'myapp.hotels',
    'myapp.houses',
)

INSTALLED_APPS = (
    'tenant_schemas',  # mandatory, should always be before any django app

    'customers',
    'django.contrib.contenttypes',
    'django.contrib.auth',
    'django.contrib.sessions',
    'django.contrib.sites',
    'django.contrib.messages',
    'django.contrib.admin',
    'myapp.hotels',
    'myapp.houses',
)
```

Deve ser indicado em qual modelo está a vinculação com o Tenant?
```
TENANT_MODEL = "core.Client" # app.Model
```

** Hora de fazer as migrações para o app core:

```
python manage.py makemigrations core
```

> *Erros encontrados nessa etapa*
> `ImportError: cannot import name 'force_text' from 'django.utils.encoding' (C:\py\schema\venv\lib\site-packages\django\utils\encoding.py)`
> No django 4.0 não existe mais o `force_text` - Solução: procurar/substituir force_text em venv\lib\site-packages\django\utils\encoding.py por force_str
> `TypeError: __init__() got an unexpected keyword argument 'providing_args'`
> procurar: `post_schema_sync = Signal(providing_args=['tenant'])` em "venv\lib\site-packages\tenant_schemas\signals.py" substituir por: `post_schema_sync = Signal(['tenant'])`
> Estas alterações logo deverão estar incorporadas ao `django-tenent_schemas`

próxima etapa: 
`python manage.py migrate_schemas --shared`

> :warning: **Never use migrate as it would sync all your apps to public!**: Atenção!


** Criar os clientes
```
python manage.py shell

>>> cli = Client()                                                                        
>>> cli.name = "cli1" 
>>> cli.schema_name="cli1"                                     
>>> cli.domain_url="multidb1.local"                      
>>> cli.paid_until = "2023-06-01" 
>>> cli.on_trial = False
>>> cli.save()

>>> cli = Client()                                                                        
>>> cli.name = "cli2" 
>>> cli.schema_name="cli2"                                     
>>> cli.domain_url="multidb2.local"                      
>>> cli.paid_until = "2023-06-01" 
>>> cli.on_trial = False
>>> cli.save()

```

Em `settings.py` atualizar o `ALLOWED_HOSTS`
```
ALLOWED_HOSTS = ["multidb.local", "multidb1.local", "multidb2.local"]
```




## Referencias
https://django-tenant-schemas.readthedocs.io/en/latest/install.html
https://books.agiliq.com/projects/django-multi-tenant/en/latest/index.html