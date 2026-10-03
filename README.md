# InterligandoDriveComGemini
import json
import time
import pandas as pd
from google.colab import drive
from google import genai
from google.genai import types
from google.colab import userdata

# Entra no Drive
print("Conectando ao Google Drive...")
drive.mount('/content/drive')

# Chave API
API_KEY = userdata.get('minhaChave')

ARQUIVO_ENTRADA = "/content/drive/MyDrive/produtos01.xlsx"
ARQUIVO_SAIDA = "/content/drive/MyDrive/produtos_preenchidos.xlsx"

TAMANHO_LOTE = 5           #Lotes para não estourar os Tokens por minuto
INTERVALO_ENTRE_LOTES = 12

#Inicializando o codigo
client = genai.Client(api_key=API_KEY)

try:
    # Tenta carregar a planilha de saída para continuar de onde parou
    df = pd.read_excel(ARQUIVO_SAIDA)
    print(f"--> Retomando processamento a partir do arquivo '{ARQUIVO_SAIDA}'")
except FileNotFoundError:
    try:
        # Se for a primeira execução, carrega o arquivo original de entrada
        df = pd.read_excel(ARQUIVO_ENTRADA)
        print(f"--> Carregando arquivo inicial '{ARQUIVO_ENTRADA}'")
    except Exception as e:
        print(f"❌ Erro: Não foi possível carregar 'produtos.xlsx' no Google Drive.\nDetalhes: {e}")
        raise SystemExit

# Garante a existência da coluna Peso_kg na planilha
if 'Peso_kg' not in df.columns:
    df['Peso_kg'] = None

# Aqui começa a consultar os itens
def consultar_pesos_gemini(lote_itens):
    prompt = f"""
    Atue como especialista em catálogo técnico e especificações de peças automotivas e industriais.
    Abaixo está uma lista de peças com ID, Marca e Código do Fabricante.
    Busque o PESO UNITÁRIO EM QUILOGRAMAS (kg) com embalagem para CADA UMA das peças da lista. Não invente dados absurdos, mas pode usar valores aproximados ou baseados em peças similares se necessário:

    {json.dumps(lote_itens, ensure_ascii=False, indent=2)}

    Retorne estritamente uma LISTA JSON contendo um objeto para cada 'id' informado, seguindo este formato exato:
    [
      {{
        "id": numero_do_id,
        "Peso_kg": número flutuante do peso estimado ou oficial em kg (ex: 0.350) ou null se totalmente desconhecido
      }}
    ]
    """

    max_tentativas = 5
    for tentativa in range(1, max_tentativas + 1):
        try:
            response = client.models.generate_content(
                model='gemini-3.6-flash',
                contents=prompt,
                config=types.GenerateContentConfig(
                    response_mime_type="application/json"
                )
            )
            return json.loads(response.text)
        except Exception as err:
            erro_msg = str(err)

            if "429" in erro_msg or "RESOURCE_EXHAUSTED" in erro_msg or "Quota" in erro_msg:
                print(f"\n⚠️ Limite de cota atingido (429). Tentativa {tentativa}/{max_tentativas} (Aguardando para reententar)...")
            else:
                print(f"\n⚠️ Instabilidade no servidor: {err}. Tentativa {tentativa}/{max_tentativas}...")

            if tentativa < max_tentativas:
                time.sleep(20 * tentativa)  # Pausa para limpar o limite de cota
            else:
                print(f"\n❌ O lote falhou após {max_tentativas} tentativas. Pulando lote com segurança para continuar...")
                return None
    return None

# Filtra apenas os índices das linhas cujo Peso_kg ainda está pendente
linhas_pendentes = df[df['Peso_kg'].isna() | (df['Peso_kg'].astype(str).str.strip() == "")].index.tolist()

total_pendentes = len(linhas_pendentes)
total_geral = len(df)

print(f"Total de itens na planilha: {total_geral}")
print(f"Itens pendentes de preenchimento: {total_pendentes}")

for i in range(0, total_pendentes, TAMANHO_LOTE):
    lote_indices = linhas_pendentes[i : i + TAMANHO_LOTE]

    lote_itens = []
    for idx in lote_indices:
        marca = str(df.at[idx, 'Marca']).strip()
        cod = str(df.at[idx, 'Cod_Fabricante']).strip()

        if marca and cod and marca.lower() != 'nan' and cod.lower() != 'nan':
            lote_itens.append({"id": int(idx), "marca": marca, "cod_fabricante": cod})

    if not lote_itens:
        continue

    progresso_atual = min(i + len(lote_itens), total_pendentes)
    print(f"[{progresso_atual}/{total_pendentes}] Consultando lote de {len(lote_itens)} peças...")

    resultados = consultar_pesos_gemini(lote_itens)

    if resultados and isinstance(resultados, list):
        for res in resultados:
            idx = res.get("id")
            if idx is not None and idx in df.index:
                df.at[idx, 'Peso_kg'] = res.get('Peso_kg')

    # Salva o arquivo no Google Drive a cada lote processado com sucesso ou falha tratada
    df.to_excel(ARQUIVO_SAIDA, index=False)
    time.sleep(INTERVALO_ENTRE_LOTES)

print("\n✅ Processamento concluído com sucesso!")
print(f"A planilha atualizada está disponível no seu Google Drive em: '{ARQUIVO_SAIDA}'")
