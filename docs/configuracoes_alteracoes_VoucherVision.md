# Configurações e Alterações Fundamentais no VoucherVision

Para viabilizar o funcionamento da transcrição automatizada, foi necessário, primeiro, corrigir dois problemas que impediam o próprio programa de funcionar — independentemente de qualquer configuração de modelo — e, em seguida, configurá-lo com os parâmetros descritos mais adiante. Em três desses pontos de configuração, a opção desejada não estava disponível na interface por padrão, sendo necessário editar diretamente o código-fonte do programa para habilitá-la — essas alterações estão detalhadas logo após a configuração correspondente.

## Pré-requisitos fundamentais (sem os quais o sistema não funciona)

Antes de qualquer configuração de modelo, dois problemas estruturais precisaram ser corrigidos para que o VoucherVision sequer executasse ou reconhecesse as credenciais de acesso.

### 1. Incompatibilidade de bibliotecas Python

Na primeira execução, o programa quebrava na inicialização ao carregar o modelo de leitura de texto manuscrito (TrOCR), etapa que ocorre independentemente da configuração escolhida pelo usuário. A causa era a versão `5.15.1` da biblioteca `transformers`, recente demais e incompatível com o tokenizer utilizado pelo TrOCR.

**Correção:** downgrade das bibliotecas `transformers`, `tokenizers` e `huggingface-hub` para versões estáveis e compatíveis entre si (`4.46.3`, `0.20.3` e `0.26.2`, respectivamente), fixadas também no arquivo `requirements.txt` do projeto para evitar recorrência do problema.

### 2. Lógica de autenticação incompatível com o tipo de chave disponível

Mesmo com a chave de API do Google AI Studio corretamente cadastrada no `PRIVATE_DATA.yaml`, o programa retornava o erro "No API key found". A causa era que o código original só reconhecia autenticação via **Vertex AI** — método que exige três credenciais simultâneas (conta de serviço, projeto e localização) — ignorando por completo o tipo de chave simples disponível.

**Arquivo alterado:** `vouchervision/utils_VoucherVision.py`

Foram realizadas cinco edições: nas linhas 715 e 733, a chave `GOOGLE_PALM_API` passou a ser lida do arquivo de credenciais (e, alternativamente, de variável de ambiente); na linha 748, foi criada uma variável interna indicando se essa chave está presente; nas linhas 765–766, foi adicionado um caminho de autenticação alternativo, utilizado sempre que as credenciais completas do Vertex AI não estiverem configuradas; e na linha 291, a verificação geral de chave válida passou a considerar também essa nova forma de autenticação, além da credencial do Vertex AI.

## Configuração utilizada

- **Prompt version:** `SLTPvM_default.yaml`
- **LLM (interpretação do texto):** `Gemini 3.1 Flash Lite`
- **API Method (OCR — leitura da imagem):** `Gemini-3.1-Flash-Lite`
- **LeafMachine2 Label Collage:** opção "Use original images for transcriptions", utilizando diretamente as imagens de etiquetas já recortadas e separadas previamente pelo modelo YOLO
- **Validation Tools:** todas as opções habilitadas, **exceto** "Enable Google Search Grounding with Gemini-2.5-Pro"
- **LLM Cost Calculator:** mantido nos valores padrão do programa
- **Processing Options → Number of CPU Workers:** `1`

## Justificativa da escolha do modelo Gemini 3.1 Flash Lite

Tanto para a etapa de OCR quanto para a etapa de interpretação (LLM), optou-se pelo modelo **Gemini 3.1 Flash Lite** por oferecer o maior limite gratuito de solicitações diárias (RPD) entre os modelos testados — até **500 solicitações por dia**, contra apenas 20 solicitações diárias de modelos alternativos como o Gemini 3.5 Flash. Essa diferença foi decisiva, já que o volume de imagens processado ao longo dos testes esgotava rapidamente a cota diária de modelos com limites mais restritos, interrompendo o processamento.

## Alterações no código-fonte necessárias

### 1. Habilitar "Gemini 3.1 Flash Lite" como opção de LLM

Na versão original do programa, a interface exibia uma opção de nome semelhante ("Gemini 3 Flash Lite", sem o ".1"), porém essa opção não estava de fato implementada — ao ser selecionada, o programa retornava erro de "modelo não suportado". Constatou-se que o modelo corretamente implementado no sistema (`Gemini 3.1 Flash Lite`) simplesmente não constava na lista de opções exibida.

**Arquivo alterado:** `vouchervision/model_maps.py`

A entrada quebrada `'Gemini 3 Flash Lite'` foi substituída por `'Gemini 3.1 Flash Lite'` em três listas internas do arquivo:
- Linha 118 — lista de modelos exibidos na interface (`MODELS_GOOGLE`);
- Linha 284 — dicionário de verificação de credenciais (`has_key`);
- Linha 362 — dicionário de custo/flags do modelo.

### 2. Habilitar "Gemini-3.1-Flash-Lite" como opção de OCR

De forma semelhante ao item anterior, o motor `Gemini-3.1-Flash-Lite` já possuía suporte implementado internamente para a etapa de OCR, mas não aparecia entre as opções selecionáveis na tela "Select the OCR Method(s)".

**Arquivo alterado:** `app.py`

Foram feitas duas inclusões:
- Linha 2333 — lista `options`, que define as opções exibidas no menu suspenso da interface: adicionado o item `"Gemini-3.1-Flash-Lite"`;
- Linha 2419 — dicionário `OCR_options`, que traduz o nome exibido na tela para o identificador interno usado nas chamadas de API: adicionada a entrada `"Gemini-3.1-Flash-Lite": "gemini-3.1-flash-lite"`.

### 3. Fixar "Number of CPU Workers" em 1

O campo "Number of CPU workers", disponível na interface, não controlava de fato a quantidade de chamadas simultâneas feitas à API do Gemini durante a transcrição.

**Arquivo alterado:** `vouchervision/utils_VoucherVision.py`

Na linha 1883, o número de processos paralelos (*workers*) era definido de forma fixa no código:
```python
num_workers = min(16, len(valid_img_paths))
```
Esse valor ignorava a configuração feita na interface e disparava simultaneamente até 16 chamadas à API (ou o total de imagens do lote, o que fosse menor), o que ultrapassava o limite de requisições por minuto (RPM) da cota gratuita do Google. A linha foi alterada para processar as imagens de forma sequencial (um *worker* por vez), evitando o excesso de chamadas simultâneas.

## Observação sobre o "LeafMachine2 Label Collage"

A opção "Use original images for transcriptions" foi mantida em vez do recorte automático de etiquetas pelo LeafMachine2, uma vez que as imagens de entrada já haviam sido previamente recortadas e segmentadas por um modelo YOLO em uma etapa anterior do fluxo de trabalho, tornando o recorte adicional do LeafMachine2 desnecessário.

## Observação sobre o "Google Search Grounding"

A opção "Enable Google Search Grounding with Gemini-2.5-Pro" foi a única desabilitada entre as ferramentas de validação. Essa ferramenta ativa uma busca automática na internet para verificação cruzada de informações, porém consome uma cota de uso gratuita separada e, no caso da conta utilizada, essa cota estava zerada — o que fazia com que **toda** chamada à API falhasse imediatamente por excesso de cota, independentemente do modelo selecionado ou da conta Google utilizada. A desativação dessa ferramenta foi a correção que resolveu a maior parte dos erros de limite de uso enfrentados durante os testes.

## Correções adicionais de robustez e integridade dos dados

Após a estabilização da autenticação, dos modelos e da cota de uso, surgiram outras falhas pontuais ao longo de execuções com lotes maiores, relacionadas a instabilidades momentâneas dos servidores do Google (erros HTTP 503, indicando alta demanda temporária). As correções a seguir não alteram a configuração do programa, mas tornam o sistema mais resiliente a esse tipo de falha.

### 4. Erro que descartava uma etiqueta inteira do lote sem gerar registro

Quando uma chamada à API falhava por qualquer motivo após esgotar as tentativas automáticas, o programa interrompia o processamento daquela etiqueta com o erro `UnboundLocalError: local variable 'response_candidate_sanitized' referenced before assignment`, descartando a etiqueta inteira do resultado final, sem nenhuma linha de registro na planilha (nem mesmo vazia).

**Arquivo alterado:** `vouchervision/utils_VoucherVision.py`

A causa foi identificada em um bloco de inicialização de variáveis (próximo à linha 88), no qual uma variável auxiliar (`response_candidate_sanitized`) deixou de receber um valor padrão — diferente das demais variáveis daquele mesmo bloco, que já tinham esse cuidado. A correção consistiu em adicionar essa variável à lista de inicializações padrão, garantindo que ela sempre exista, independentemente do resultado da chamada à API. Com isso, uma etiqueta que falhe passa a gerar uma linha vazia na planilha (preservando seu registro, e permitindo reprocessamento posterior), em vez de ser descartada silenciosamente.

### 5. Baixa tolerância a falhas temporárias na etapa de OCR

A etapa de OCR possuía um mecanismo de repetição automática (até 3 tentativas, com espera progressiva entre elas) para lidar com falhas temporárias de rede, porém as mensagens de cada tentativa eram registradas apenas no terminal, não no arquivo de log da aplicação — dificultando o diagnóstico de falhas. Além disso, o número de tentativas se mostrou insuficiente em períodos de maior instabilidade dos servidores do Google.

**Arquivo alterado:** `vouchervision/OCR_Gemini.py`

O número de tentativas foi elevado de 3 para 5, e as mensagens de cada tentativa (sucesso, falha e tempo de espera) passaram a ser registradas no logger oficial da aplicação, em vez de exibidas apenas no terminal — tornando essas falhas visíveis no arquivo de log de cada execução.

### 6. Vazamento de dados entre etiquetas distintas ao reordenar a planilha

Com o objetivo de agrupar, na planilha final, as múltiplas etiquetas fotografadas de uma mesma exsicata (adicionando um passo de reordenação alfabética pelo nome do arquivo), uma primeira implementação desse recurso — que reordenava a planilha célula por célula a cada imagem processada — introduziu um efeito colateral grave: valores de campos opcionais (como `county`) passaram a ser copiados indevidamente de uma etiqueta para outra, sem relação entre si. O problema foi isolado e confirmado por comparação direta com execuções anteriores ao recurso, nas quais esse tipo de inconsistência nunca ocorria.

**Arquivo alterado:** `vouchervision/utils_VoucherVision.py`

A solução adotada foi reescrever a função de reordenação utilizando a biblioteca `pandas` (leitura da planilha inteira como tabela, ordenação e gravação completa de uma só vez), em vez de manipular célula por célula com `openpyxl`, e executá-la **uma única vez**, após o processamento de todas as imagens do lote — em vez de repeti-la a cada imagem individual, como na implementação original. Essa mudança eliminou por completo a inconsistência observada, confirmada em múltiplas execuções subsequentes sem nenhuma recorrência do problema.

## Padronizações aplicadas conforme o padrão Darwin Core

Para adequar os dados extraídos às convenções internacionais de nomenclatura e preenchimento de acervos biológicos (padrão Darwin Core), foi adicionada uma etapa de normalização automática, aplicada a cada etiqueta logo após a extração dos dados pelo modelo de linguagem.

**Arquivo alterado:** `vouchervision/utils_VoucherVision.py` (nova função `normalize_taxonomic_fields`, chamada antes da gravação de cada registro)

As seguintes regras passaram a ser aplicadas automaticamente:
- O epíteto específico (`specificEpithet`) é sempre convertido para letras minúsculas, conforme exige a nomenclatura botânica — corrigindo uma tendência do modelo de linguagem de capitalizar esse campo como se fosse um nome próprio, confirmada estatisticamente em mais da metade dos registros de um lote analisado;
- O nome do país (`country`) é padronizado para o inglês (por exemplo, "Brasil" é convertido para "Brazil"), e o código do país (`countryCode`, no padrão ISO 3166-1 alpha-2, como "BR") passou a ser preenchido automaticamente a partir dessa padronização;
- Datas totalmente desconhecidas, anteriormente representadas por um valor de preenchimento (`0000-00-00`), passaram a ser gravadas como campo vazio, conforme a convenção do Darwin Core, que não reconhece esse tipo de valor como data válida;
- O campo `basisOfRecord`, que descreve a natureza do registro, passou a ser preenchido automaticamente e de forma fixa como `"PreservedSpecimen"`, valor correto para todo espécime de herbário, eliminando variações de grafia que o modelo de linguagem produzia ao tentar preenchê-lo por conta própria.

Como os campos `countryCode` e `basisOfRecord` não faziam parte da estrutura de dados original do programa, também foi necessário incluí-los no arquivo de definição de campos usado para instruir o modelo de linguagem (`custom_prompts/SLTPvM_default.yaml`), tanto na lista de descrições de campos quanto na lista de mapeamento que determina as colunas exibidas na planilha final.
