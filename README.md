# doc-antifrauce-ai
Projeto DIO Analise de Documentos Anti-fraude com AzureAI

## Análise de Documentos Anti-Fraude com Azure AI (Python) - Passo a Passo Simplificado

Este guia simplificado demonstra como utilizar o Azure AI para análise anti-fraude de documentos, focando em rapidez e praticidade. A imagem fornecida serve como referência para os tipos de informações que podem ser extraídas, como detalhes do remetente e itens de um pedido.

**Objetivo:** Extrair informações relevantes de documentos (como o da imagem) e realizar verificações básicas para identificar possíveis fraudes.

**Serviços Azure Utilizados:**

*   **Azure AI Document Intelligence:** Extração de texto e estrutura de documentos (principal).
*   **Azure AI Language (Opcional):** Análise de sentimento ou identificação de entidades (pode ser adicionado posteriormente para aprimorar a análise).

**Pré-requisitos:**

*   Conta Azure ativa.
*   Python instalado.
*   Bibliotecas Azure AI instaladas: `pip install azure-ai-formrecognizer azure-core`
*   Variáveis de ambiente configuradas (endpoints e chaves de API para Document Intelligence): `DOCUMENT_INTELLIGENCE_ENDPOINT`, `DOCUMENT_INTELLIGENCE_KEY`.

**Passo a Passo:**

1.  **Extração de Dados do Documento:**

    *   Utilize o Azure AI Document Intelligence para extrair dados estruturados do documento. Neste exemplo, utilizaremos o modelo `prebuilt-invoice` (já treinado) para agilizar o processo.  Para maior precisão, dependendo do tipo de documentos que tem, poderá ser mais acertado treinar o seu próprio modelo.
    *   **Código Python:**

```python
import os
from azure.core.credentials import AzureKeyCredential
from azure.ai.formrecognizer import DocumentAnalysisClient

endpoint = os.environ["DOCUMENT_INTELLIGENCE_ENDPOINT"]
key = os.environ["DOCUMENT_INTELLIGENCE_KEY"]

document_analysis_client = DocumentAnalysisClient(
    endpoint=endpoint, credential=AzureKeyCredential(key)
)

def extract_invoice_data(document_path):
    """Extrai dados de faturas usando o modelo prebuilt-invoice."""
    try:
        with open(document_path, "rb") as f:
            poller = document_analysis_client.begin_analyze_document("prebuilt-invoice", document=f)
        result = poller.result()

        invoice_data = {}

        # Extrair informações do remetente (Shipped From)
        for field, value in result.documents[0].fields.items():
            if value:
                invoice_data[field] = value.value if value.value else value.content


        # Extrair itens da fatura (Details)
        items = []
        for item in result.documents[0].fields.get("Items",[]).value:
            item_data = {}
            for k,v in item.value.items():
                item_data[k] = v.value if v.value else v.content
            items.append(item_data)

        invoice_data["items"] = items

        return invoice_data

    except Exception as e:
        print(f"Erro ao extrair dados da fatura: {e}")
        return None

# Exemplo de uso
document_path = "caminho/para/sua/fatura.pdf"  # Substitua pelo caminho do seu documento
invoice_data = extract_invoice_data(document_path)

if invoice_data:
    print("Dados da Fatura Extraídos:")
    for key, value in invoice_data.items():
        if key == "items":
            print ("\nItems: ")
            for item in value:
                print (f"\t {item}")
        else:
            print(f"\n{key}: {value}")
else:
    print("Falha ao extrair dados da fatura.")
```

2.  **Verificações Anti-Fraude Simplificadas:**

    *   Implementar verificações básicas com base nos dados extraídos.  Esta é a parte onde você define suas próprias regras anti-fraude.  Aqui estão alguns exemplos:
        *   **Verificar Consistência:** Comparar o nome e endereço do remetente com registros existentes (banco de dados interno, lista negra, etc.).
        *   **Analisar Valores Atípicos:** Verificar se os valores da fatura (total, quantidade de itens) estão dentro de um intervalo razoável com base em dados históricos.
        *   **Verificar Itens Suspeitos:** Procurar por descrições de itens que podem ser indicativos de fraude (e.g., termos relacionados a atividades ilegais).
    *   **Código Python (exemplos):**

```python
def perform_fraud_checks(invoice_data):
    """Realiza verificações anti-fraude básicas."""
    flags = []

    # Checar a quantidade total
    if invoice_data.get("InvoiceTotal", 0) > 10000:
        flags.append("Alerta: Valor total da fatura acima do limite.")

    # Verificar se o nome do remetente está na lista negra
    blacklist = ["Wesley Smith", "Empresa Suspeita"]
    if invoice_data.get("VendorName") in blacklist:
        flags.append("Alerta: Remetente está na lista negra.")

    # Verificar itens suspeitos
    suspeitos = ["ilegalsubstance", "hack tool"]
    for item in invoice_data.get("items", []):
      for k,v in item.items():
        if any(word in v.lower() for word in suspeitos):
          flags.append("Alerta: Item suspeito encontrado.")
          break

    return flags

# Exemplo de uso:
fraud_flags = perform_fraud_checks(invoice_data)

if fraud_flags:
    print("\nAlertas de Fraude:")
    for flag in fraud_flags:
        print(f"- {flag}")
else:
    print("\nNenhum alerta de fraude detectado.")
```

**3.  Apresentação dos Resultados:**

*   Combine os resultados da extração de dados e as verificações anti-fraude em um relatório ou painel.

**Considerações e Simplificações:**

*   **Modelo Pré-Treinado:** A utilização do `prebuilt-invoice` simplifica o processo, mas pode não ser ideal para todos os tipos de documentos.  Se precisar de maior precisão, considere treinar um modelo personalizado com seus próprios documentos de exemplo.
*   **Verificações Anti-Fraude:** As verificações anti-fraude são simplificadas e dependem do seu conhecimento do domínio e dos padrões de fraude que você deseja detectar.  Amplie e personalize essas verificações com base nas suas necessidades.
*   **Análise de Linguagem:** A análise de sentimento e a identificação de entidades podem ser adicionadas usando o Azure AI Language para identificar padrões de linguagem suspeitos.  Essas análises aprimoradas podem levar mais tempo, mas podem melhorar a precisão da detecção de fraude.
*   **Banco de Dados:** Para comparações mais robustas (e.g., verificar a consistência do remetente), você precisará integrar sua solução a um banco de dados onde você armazena informações de remetentes conhecidos, listas negras, etc.

**Próximos Passos (Melhorias Opcionais):**

*   **Treinamento de Modelo Personalizado:** Se você tiver muitos documentos de um tipo específico, treinar um modelo personalizado no Azure AI Document Intelligence pode aumentar significativamente a precisão da extração de dados.
*   **Integração com Outras Fontes de Dados:** Combine os resultados com outras fontes de dados (e.g., informações de transações, dados de clientes) para obter uma visão mais completa e identificar padrões de fraude mais complexos.
*   **Automação:** Automatize o processo de análise de documentos à medida que eles chegam (e.g., monitorando uma pasta, integrando com um sistema de gerenciamento de documentos).

Este guia oferece uma base para criar uma solução de análise de documentos anti-fraude com o Azure AI.  Personalize e expanda este exemplo para atender às suas necessidades específicas.  Lembre-se de considerar o custo dos serviços Azure ao projetar sua solução.
