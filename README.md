# Desafio 2 - AWS Step Functions: Processamento em Larga Escala no S3 com Step Functions

Este projeto demonstra o uso do modo **Mapa Distribuído** (Distributed Map) do AWS Step Functions para orquestrar um fluxo de trabalho de processamento de dados em duas fases:

1.  **Ingestão:** Copia um grande volume de dados meteorológicos de um bucket público da NOAA para um bucket S3 local.
2.  **Processamento:** Analisa todos os arquivos copiados em paralelo para encontrar a estação meteorológica com a temperatura mais alta registrada no mês.

O uso do Mapa Distribuído é ideal para processamento em paralelo em larga escala (milhões de arquivos), superando os limites de concorrência do estado de Mapa padrão.

<img width="795" height="1353" alt="stepfunctions_graph" src="https://github.com/user-attachments/assets/a52e9dd2-45cd-4aef-8422-fcfb4cfd98b8" />

## Arquitetura e Serviços Utilizados

* **AWS Step Functions:** Orquestra todo o fluxo de trabalho, gerenciando os estados, paralelismo, tratamento de erros e a lógica de paginação.
* **Amazon S3:** Usado como fonte dos dados (bucket público da NOAA), destino da cópia (bucket local) e para armazenar os resultados do processamento.
* **AWS Lambda:** Utilizada para a lógica de negócio principal:
    * Uma função (`ProcessNOAAData`) que analisa um arquivo S3 e encontra a temperatura mais alta.
    * Uma função (`Reducer`) que consolida os resultados parciais para encontrar o vencedor geral.

## Como o Sistema Funciona: Análise do Fluxo

O fluxo de trabalho (conforme o diagrama) é dividido nas seguintes etapas:

### 1. Verificação Inicial (CheckBucketContents)

O fluxo se inicia e a primeira etapa (`CheckBucketContents`) lista o conteúdo do nosso bucket S3 de destino.

Imediatamente após, um estado `Choice` (Escolha) toma uma decisão:

* **Caminho `Default` (Ex: Bucket Vazio):** Se o bucket estiver vazio ou os dados precisarem ser copiados, o fluxo segue para a Etapa 2 (Ingestão de Dados).
* **Caminho Alternativo (Ex: Dados já existem):** Se os dados já foram copiados anteriormente, o fluxo pula diretamente para a Etapa 4 (Processamento), economizando tempo e custos de cópia.

### 2. Etapa de Cópia/Ingestão (Distributed S3 copy NOA Data)

Esta fase é responsável por popular nosso bucket S3 com os dados da NOAA.

1.  **`ListObjectsInNoaBucket`:** O fluxo começa listando os objetos no bucket S3 público da NOAA.
2.  **`Distributed S3 copy NOA Data`:** Este é o primeiro **Mapa Distribuído**. Ele recebe a lista de objetos e inicia um fluxo de trabalho filho para cada item.
3.  **`Map state` (Interno):** Dentro do mapa distribuído, outro mapa (ou uma tarefa direta) executa a ação `CopyObject`, movendo o arquivo do bucket de origem (NOAA) para o nosso bucket de destino.
4.  **`Choice state (Has next page?)`:** A operação `ListObjectsV2` do S3 retorna no máximo 1000 itens por vez. Este estado de `Choice` verifica se a resposta contém um `NextContinuationToken` (indicando que há mais páginas de resultados).
    * **Se SIM:** O fluxo volta para a etapa `ListObjectsInNoaBucket with pagination`, passando o token para buscar a próxima página de arquivos.
    * **Se NÃO (Default):** Todos os arquivos foram copiados. O fluxo prossegue para a próxima etapa.

### 3. Etapa de Processamento (ProcessNOAAData)

Esta é a segunda e principal fase de processamento, também executada como um **Mapa Distribuído**.

1.  **`ProcessNOAAData`:** Este mapa itera sobre *todos os arquivos S3* que acabamos de copiar para o nosso bucket.
2.  **`Lambda Invoke`:** Para cada arquivo no bucket, uma função Lambda é invocada em paralelo. A lógica desta função é analisar o conteúdo do arquivo e encontrar a temperatura mais alta registrada *naquele arquivo específico*.
3.  **`Local de exportação`:** O resultado de cada execução da Lambda (ex: `{ "station": "ABC", "temp": 38.5 }`) é salvo automaticamente pelo Step Functions em um local de resultados no `S3_RESULTS_BUCKET`.

### 4. Etapa de Redução (Reducer)

Após o término *completo* do Mapa Distribuído (milhares de execuções da Lambda), uma última função é chamada:

1.  **`AWS Lambda: Invoke (Reducer)`:** Esta função "Redutora" tem uma única responsabilidade: ler todos os resultados parciais salvos no `S3_RESULTS_BUCKET`, compará-los e retornar o resultado final: a estação única com a temperatura mais alta de todo o conjunto de dados.

### 5. Fim

O fluxo de trabalho é concluído, tendo processado eficientemente um grande volume de dados para encontrar uma única resposta.
