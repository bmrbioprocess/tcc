# TCC
## 🧩 Arquitetura, Lógica e Etapas do Pipeline

O pipeline foi modularizado em blocos independentes para garantir a escalabilidade e facilitar a manutenção do código. Abaixo está o detalhamento técnico de cada etapa acionada pelo orquestrador:

### 📥 Etapa 1: Aquisição
* **Objetivo:** Coleta primária dos dados a partir de identificadores.
* **Lógica de Programação e Pontos-Chave:**
  * **Manipulação de Dados:** Utiliza a biblioteca `pandas` para ler a planilha Excel inicial e iterar sobre a coluna de DOIs.
  * **Requisições Web:** Emprega bibliotecas como `requests` ou `urllib` para acessar os repositórios dos artigos.
  * **Tratamento de Exceções (Error Handling):** Ponto crítico do script. Implementa blocos `try-except` para lidar com links quebrados (Erro 404), bloqueios de acesso (Erro 403) ou tempos de limite de conexão (Timeouts), garantindo que o orquestrador não pare de funcionar se um download falhar.
  * **Controle de Taxa (Rate Limiting):** Uso de pausas (`time.sleep`) entre as requisições para evitar o bloqueio do IP pelos servidores acadêmicos.

### 🧹 Etapa 2: Normalização
* **Objetivo:** Limpeza e padronização do texto bruto.
* **Lógica de Programação e Pontos-Chave:**
  * **Extração:** Uso de bibliotecas de manipulação de PDF (como `PyMuPDF`, `pdfplumber` ou similares) ou HTML para converter os documentos baixados em texto puro (strings).
  * **Expressões Regulares (Regex):** Aplicação massiva da biblioteca `re` para identificar e remover ruídos, como cabeçalhos, rodapés, números de página, referências bibliográficas e quebras de linha indesejadas.
  * **Pré-processamento NLP:** Conversão do texto para caixa baixa (lowercasing), remoção de caracteres especiais e preparo da tokenização inicial para otimizar o consumo de memória na próxima fase.

### 🧠 Etapa 3: Inferência (SciBERT)

* **Objetivo:** Reconhecimento de Entidades Nomeadas (NER) focadas em jargão científico.
* **Lógica de Programação e Pontos-Chave:**
  * **Carregamento do Modelo:** Integração com a biblioteca `transformers` (Hugging Face) para instanciar o modelo SciBERT com os pesos ajustados pelo processo de *Fine-Tuning*.
  * **Tokenização Específica:** O texto normalizado é fatiado usando o tokenizador próprio do SciBERT, que compreende a morfologia de termos acadêmicos.
  * **Processamento em Lotes (Batching):** Para evitar estouro de memória (Out-Of-Memory/OOM), o script divide textos longos em blocos menores (ex: 512 tokens) antes de passá-los pelo modelo.
  * **Extração de Tags:** A saída do modelo (geralmente no formato BIO - Begin, Inside, Outside) é processada para identificar com precisão onde uma entidade começa e termina no texto.

### 🏗️ Etapa 4: Pós-processamento e Estruturação
* **Objetivo:** Organização lógica das entidades extraídas.
* **Lógica de Programação e Pontos-Chave:**
  * **Reconstrução de Palavras:** O SciBERT frequentemente divide palavras complexas em sub-tokens (ex: "micro" e "##biologia"). Este script contém a lógica para concatenar esses fragmentos de volta em palavras legíveis.
  * **Filtragem de Confiança (Thresholding):** Descarta predições do modelo que possuam um grau de certeza muito baixo, reduzindo falsos positivos.
  * **Mapeamento de Relações:** Estrutura as entidades encontradas em dicionários Python (`dict`) ou formato JSON, criando relacionamentos entre autores, metodologias e resultados dentro do mesmo artigo.

### 🔗 Etapa 5: Enriquecimento
* **Objetivo:** Adicionar metadados e ranqueamento científico.
* **Lógica de Programação e Pontos-Chave:**
  * **Consumo de APIs Externas:** Realiza chamadas automatizadas à API do **CrossRef** (enviando o DOI) para recuperar metadados confiáveis: ano exato de publicação, contagem de citações, autores e nome da revista.
  * **Cálculo do Methodi Ordinatio:** Implementação da lógica matemática para ranquear a relevância dos artigos. O script processa as variáveis coletadas e aplica a fórmula InOrdinatio:

$$InOrdinatio = \left( \frac{IF}{1000} \right) + \alpha \cdot [10 - (Ano_{atual} - Ano_{publicacao})] + Citacoes$$

  * *(Onde IF é o Fator de Impacto, α é o peso atribuído pelo pesquisador, e o ano atual penaliza artigos muito antigos em favor dos mais recentes e citados).*

### 📊 Etapa 6: Apresentação
* **Objetivo:** Geração do banco de dados analítico final.
* **Lógica de Programação e Pontos-Chave:**
  * **Agregação de DataFrames:** O script utiliza o `pandas` para cruzar e unir (operações de *merge* e *join*) os dados estruturados do pós-processamento com as métricas calculadas no enriquecimento.
  * **Formatação de Saída:** Renomeação de colunas para termos amigáveis ao usuário final e tratamento de dados nulos (`NaN` ou `None`).
  * **Exportação:** Uso do método `to_excel` para gerar o arquivo final, garantindo a codificação correta (`utf-8`) para que não haja problemas com acentuação na leitura da planilha.
## 🔬 Treinamento e Validação do Modelo (SciBERT)

O desenvolvimento do modelo de Reconhecimento de Entidades Nomeadas (NER) ocorreu em uma esteira paralela ao pipeline principal de processamento. Antes de o orquestrador utilizar o modelo em produção na **Etapa 3 (Inferência)**, o SciBERT passou por um rigoroso processo de *fine-tuning* e validação metodológica.

### 📈 Avaliação de Desempenho
* **Objetivo:** Validação quantitativa e qualitativa da eficácia do modelo SciBERT após o treinamento em dados rotulados.
* **Justificativa Metodológica:** A escolha destas técnicas afasta-se da acurácia global tradicional (que costuma ser inflada em tarefas de NER devido ao excesso de palavras fora das entidades). O uso do *seqeval* garante rigor ao avaliar a entidade como um todo (esquema BIO). A Matriz de Confusão é essencial para diagnosticar se o modelo está confundindo conceitos científicos próximos (ex: classificando uma Metodologia como Resultado). Por fim, o UMAP atua como prova de aprendizado profundo: ele demonstra visualmente que a IA não está apenas memorizando palavras, mas sim compreendendo a distância semântica entre os termos no espaço vetorial.
* **Lógica de Programação e Pontos-Chave:**
  * **Métricas de Rótulos de Sequência (Seqeval):** Integração com o framework `seqeval` para extrair valores precisos de Precisão (Precision), Revocação (Recall) e F1-Score (Micro e Macro) no conjunto de teste. O cálculo respeita rigorosamente as fronteiras das entidades extraídas. A métrica harmônica F1, central para atestar o equilíbrio do modelo, é processada pela relação:

$$F_1 = 2 \cdot \frac{Precision \cdot Recall}{Precision + Recall}$$

  * **Análise de Erros (Matriz de Confusão):** Construção de matrizes de contingência utilizando `scikit-learn` e bibliotecas de plotagem (`matplotlib`/`seaborn`). 
  
  A matriz permite a inspeção direta da taxa de acertos na diagonal principal e expõe detalhadamente os falsos positivos e falsos negativos entre as diferentes categorias semânticas mapeadas na monografia.
  * **Projeção do Espaço Vetorial Latente (UMAP):** Aplicação do algoritmo não linear UMAP (*Uniform Manifold Approximation and Projection*) para redução de dimensionalidade. O script extrai os *embeddings* densos gerados pelas últimas camadas ocultas do SciBERT (vetores de 768 dimensões) e os comprime matematicamente para visualização gráfica em 2D.
  
  * **Interpretação Semântica Qualitativa:** O agrupamento (*clustering*) resultante do UMAP gera evidências visuais indispensáveis para a publicação. A separação clara dos *clusters* de dados atesta empiricamente que o modelo aprendeu a distinguir as classes de entidades no espaço semântico antes de ser integrado ao orquestrador principal.
