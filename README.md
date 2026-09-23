# TCC Orchestrator

Orquestrador em **Docker Compose** que sobe, em um único comando, os três serviços que compõem o pipeline de classificação e roteirização de resíduos: a API de rotas (Node.js), a API do modelo de visão computacional (Python/Flask) e o pipeline de dados (Python). Ele cuida da rede interna, do volume compartilhado entre os containers e da ordem de inicialização, para que os três serviços conversem entre si sem configuração manual.

Este repositório é um dos componentes do projeto de TCC **Gestão de Resíduos Sólidos Urbanos (SWM)**, composto pelos seguintes repositórios:

| Repositório | Papel no fluxo |
|---|---|
| [APS_Android_SWM_Images](https://github.com/Gusta9s/APS_Android_SWM_Images) | Aplicativo mobile (Expo/React Native) usado para capturar e enviar as fotos dos resíduos descartados. |
| [TCC_workflow_data_SWM](https://github.com/Gusta9s/TCC_workflow_data_SWM) | Pipeline em Python que baixa as imagens recebidas, trata os dados e orquestra as chamadas entre o modelo de classificação e o serviço de rotas. |
| [TCC_Model1_CNN_SWM](https://github.com/Gusta9s/TCC_Model1_CNN_SWM) | API com o modelo de visão computacional (YOLOv11s) que classifica o tipo de resíduo na imagem. |
| [TCC-Routing-Machine-SWM](https://github.com/Gusta9s/TCC-Routing-Machine-SWM) | Recebe origem/destino, calcula a rota, gera a imagem do mapa (Leaflet.js) e a salva em um volume Docker compartilhado. |
| **TCC_Orchestrator** *(este repositório)* | Orquestra, via Docker Compose, a subida integrada de todos os serviços (rede, volumes compartilhados e ordem de inicialização). |

## O problema

O pipeline completo depende de três serviços independentes, em duas linguagens diferentes (Node.js e Python), que precisam: (1) enxergar uns aos outros pela rede mesmo rodando em containers isolados, (2) compartilhar os arquivos de imagem gerados sem depender de um sistema de arquivos externo, e (3) subir em uma ordem específica — a API de rotas e a API do modelo precisam estar de pé e saudáveis *antes* do pipeline de dados começar a chamá-las, senão a primeira requisição falha por conexão recusada. Coordenar isso manualmente (abrir três terminais, subir cada serviço na ordem certa, esperar cada um "aquecer") é lento e sujeito a erro humano, especialmente ao reiniciar o ambiente do zero.

## A solução

O **Docker Compose** foi escolhido por descrever essa orquestração de forma declarativa em um único arquivo, sem exigir uma ferramenta de orquestração mais pesada (como Kubernetes), o que é desnecessário para três serviços rodando em uma única máquina de desenvolvimento/demonstração. Dois mecanismos do Compose resolvem diretamente o problema de ordenação e comunicação: `healthcheck` em cada serviço (a API de rotas e a API do modelo só são consideradas "prontas" quando respondem a uma requisição HTTP real) combinado com `depends_on: condition: service_healthy`, que faz o pipeline de dados só iniciar depois que as APIs das quais ele depende estiverem de fato respondendo — não apenas com o container criado. A comunicação entre os serviços usa os nomes dos serviços como hostname (`http://routing-api:3004`, `http://ml-api:3001`) através de uma rede Docker dedicada (`tcc-network`), e a troca de arquivos entre a API de rotas e o pipeline de dados usa um volume nomeado compartilhado (`shared_assets`), evitando que os containers precisem expor portas ou diretórios do host só para trocar uma imagem.

## O resultado

Com um único comando, `docker compose up --build`, os três serviços sobem na ordem correta e prontos para operar: a API de rotas primeiro, a API do modelo em seguida (aguardando a rede), e o pipeline de dados só começa a processar depois que ambas as APIs respondem ao healthcheck — eliminando as falhas de "conexão recusada" que aconteciam ao iniciar tudo manualmente. Os logs de cada serviço podem ser acompanhados de forma isolada e sequencial (em vez de misturados) com `docker compose logs -f pipeline-core ml-api routing-api`, o que facilita depurar em qual etapa do pipeline um problema ocorreu.

## Serviços orquestrados

| Serviço | Build a partir de | Porta | Função |
|---|---|---|---|
| `routing-api` | [`TCC-Routing-Machine-SWM`](https://github.com/Gusta9s/TCC-Routing-Machine-SWM) | 3004 | Gera a imagem da rota (Leaflet.js) e salva no volume compartilhado. |
| `ml-api` | [`TCC_Model1_CNN_SWM`](https://github.com/Gusta9s/TCC_Model1_CNN_SWM) | 3001 | Classifica a imagem do resíduo e, se a confiança for suficiente, aciona a `routing-api`. |
| `pipeline-core` | [`TCC_workflow_data_SWM`](https://github.com/Gusta9s/TCC_workflow_data_SWM) | — | Baixa a imagem mais recente, envia para a `ml-api` e lê o resultado gerado no volume compartilhado. |

Os três repositórios precisam estar clonados **lado a lado**, no mesmo diretório pai, pois o `docker-compose.yml` referencia o código-fonte de cada serviço por caminho relativo (`../TCC-Routing-Machine-SWM`, `../TCC_Model1_CNN_SWM`, `../TCC_workflow_data_SWM`).

## Segredos e configuração

- Credenciais e chaves de API (ex.: chave da Mapbox usada pela `routing-api`) são passadas como variáveis de ambiente no `docker-compose.yml` — em um ambiente real, elas devem vir de um arquivo `.env` (não versionado) ou de um gerenciador de segredos, e não ficar hardcoded no arquivo de compose.
- Os arquivos de configuração e segredos do pipeline de dados (`config.yaml`, diretório `repository` com os arquivos de contingência) são montados como volumes a partir do repositório `TCC_workflow_data_SWM`, mantendo os segredos fora da imagem Docker.

## Como executar

```bash
# 1. Clone os quatro repositórios de serviço lado a lado neste mesmo diretório pai:
#    TCC_Orchestrator/, TCC-Routing-Machine-SWM/, TCC_Model1_CNN_SWM/, TCC_workflow_data_SWM/

# 2. Abra o Docker Desktop e aguarde o Docker Engine ficar com status "running"

# 3. Na pasta do orquestrador, suba todos os serviços:
docker compose up --build
```

Para acompanhar os logs de forma organizada por serviço:

```bash
docker compose logs -f pipeline-core ml-api routing-api
```

## Estrutura do projeto

```
.
├── docker-compose.yml   # Definição dos 3 serviços, rede, volume compartilhado e ordem de inicialização
├── docs/                # Guias de execução e obtenção de logs
└── .env                 # Variáveis de ambiente locais (não deve conter segredos versionados)
```

## Tecnologias principais

- Docker Compose
- Docker (rede bridge e volumes nomeados)

## Autor

Gustavo de Almeida Pacheco — desenvolvido como parte do Trabalho de Conclusão de Curso (TCC) sobre Gestão de Resíduos Sólidos Urbanos.
