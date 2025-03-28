# Ponderada Semana 8 - Como avaliar a integração do sistema do projeto (baixo acoplamento e alta coesão)

## Como poderíamos usar a Engenharia Simultânea no Projeto?

Como parte de reflexão desta atividade ponderada, pensei em como meu grupo poderia ter implementado a engenharia siultânea para desenvolvermos as integrações internas e externas dentro do nosso projeto. Nesse sentido, esta arquitetura foi pensada de forma que diferentes partes do sistema possam ser desenvolvida em paralelo e ao implementá-la, poderíamos seguir as tarefas dessa forma, por exemplo: 
* Enquanto eu estava desenvolvendo o `feedback_service`, os outros integrantes do grupo poderiam ir desenvolvendo os outros serviços do projeto, como o `onboarding_service`, etc.
* A integração com o Google Sheets é feita via API, então não iria bloquear outros desenvolvimentos do backend.
* O uso de métricas e roteamento para Prometheus/Grafana pode ser feito ainda por outras pessoas do time em paralelo também.

Com essa separação, é possível ter a presença das entregas contínuas, de testes automatizados isolados e uma flexbilidade na priorização de partes do projeto que estamos desenvolvendo.

## Contexto do Projeto
Pensando agora no projeto que meu grupo está desenvolvendo para a Rappi, nós estamos trabalhando dentro do app dos entregadores da Rappi, especificamente na vertente do Turbo 10. De acordo com isso, nosso grupo focou em 5 direcionadores de negócio:
- Onboarding do Entregador
- Alocação de Entregadores
- Tela de Ganhos
- Alertas e notificações
- Feedbacks dos Entregadores
Eu em específico fiquei responsável por desenvolver o **serviço de feedback dos entregadores**, que tem como objetivo permitir que eles relatem as dificuldades técnicas de forma estruturada.

## Integração Interna
Dado o meu requisito funcional geral (registro de feedback dos entregadores), ele pode ser conectado com um serviço interno de **classificação desses feedbacks**. Esse serviço iria receber os feedbacks e os classificaria como "Financeiro", "Problemas Técnicos", "Atendimento", etc. 

Dado esse contexto, abaixo há um diagrama de como seria essa arquitetura:
```
┌────────────────────────────┐
│      Entregador App        │
│ Envia feedback pela API    │
└──────────────┬─────────────┘
               │
               ▼
       ┌──────────────┐
       │ feedback_service │◄──────┐
       └─────┬────────────┘       │
             │                   │
             ▼                   │
 ┌─────────────────────────┐     │
 │ classification_service  │─────┘
 │ Classifica categoria do │
 │ feedback por palavras   │
 └─────────────────────────┘
```

Também deixo um trecho de código de como poderia ser essa integração interna (dado toda a existência do código geral do serviço de feedback). No `feedback_service`:
```

import requests

def get_category_from_service(response_text):
    try:
        response = requests.post("http://localhost:9000/classify", json={"text": response_text})
        return response.json().get("category", "Outros")
    except:
        return "Outros"
```

Agora no próprio serviço de classificação dos feedbacks, `classification_service`:
```
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

CATEGORIES = {
  "pagamento": "Financeiro",
  "erro": "Problemas Técnicos",
  "aplicativo": "Problemas Técnicos",
  "taxa": "Financeiro",
  "bônus": "Recompensas"
}

class FeedbackText(BaseModel):
    text: str

@app.post("/classify")
def classify(feedback: FeedbackText):
    text = feedback.text.lower()
    for word, category in CATEGORIES.items():
        if word in text:
            return {"category": category}
    return {"category": "Outros"}
```

Com essa separação, nós conseguiríamos ter:
* **Melhor escalabilidade**, uma vez que os outros serviços podem usar essa categorização de outras formas
* **Maior testabilidade**, dado que terão os testes unitários específicos para o classificador
* **Presença da Engenharia simultânea**, uma vez que esse microsserviço (e outros) pode ser desenvolvido em paralelo por outra pessoa do grupo.

## Integração Externa
O serviço de feedback se conecta diretamente ao Google Sheets para armazenar os dados recebidos. Essa integração permite que os membros não técnicos do time (por exemplo o time de operação) acessem os feedbacks em tempo real, podendo, ainda, usar os dados desse Google Sheets para outros itens operacionais, como análise de dados, entre outros. 

Diagrama de como seria (e como está ocorrendo atualmente):
```
┌────────────────────────────┐
│ feedback_service           │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐
│ Google Sheets API          │
│ Armazena feedbacks         │
└────────────────────────────┘
```
Trecho de código dessa integração:
```
def save_feedback_to_sheets(motoboy_id: int, response: str, category: str):
    spreadsheet = client.open("Feedbacks Entregadores")
    sheet = spreadsheet.sheet1
    sheet.append_row([motoboy_id, response, category])
```
Essa integração inclui controle de qualidade a partir desses aspectos:
* Verificação de credenciais
* Retries com delay
* Versionamento das libs usadas
* Armazenamento de logs e execução da verificação na aba "Verificação de Qualidade" da planilha 
