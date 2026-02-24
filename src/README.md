# Código da Aplicação

Esta pasta contém o código do seu agente financeiro.

## Estrutura Sugerida

```
src/
├── app.py              # Aplicação principal (Streamlit/Gradio)
├── agente.py           # Lógica do agente
├── config.py           # Configurações (API keys, etc.)
└── requirements.txt    # Dependências
```

## Exemplo de requirements.txt

```
streamlit
openai
python-dotenv
```

## Como Rodar

```bash
# Instalar dependências
pip install -r requirements.txt

# Rodar a aplicação
streamlit run app.py
```


## Código 

import pandas as pd
import json
import requests
import streamlit as st 


perfil = json.load(open('./data/perfil_investidor.json'))
transacoes = pd.read_csv('./data/transacoes.csv')
historico = pd.read_csv('./data/historico_atendimento.csv')
produtos = json.load(open('./data/produtos_financeiros.json'))

print("Dados carregados com sucesso!")

from ollama import chat

OLLAMA_URL = "http://localhost:11434/api/generate"
MODELO = "gpt-oss:latest"

# ======== MONTAR CONTEXTO PARA O MODELO ======== 
contexto = f"""
CLIENTE: {perfil['nome']}, {perfil['idade']} anos, perfil {perfil['perfil_investidor']}
OBJETIVO: {perfil['objetivo_principal']}
PATRIMONIO: R$ {perfil['patrimonio_total']:.2f} | RESERVA: R$ {perfil['reserva_emergencia_atual']:.2f}

TRANSACOES RECENTES:
{transacoes.to_string(index=False)}

ATENDIMENTOS ANTERIORES:
{historico.to_string(index=False)}

PRODUTOS FINANCEIROS DISPONIVEIS: 
{json.dumps(produtos, indent=2, ensure_ascii=False)}
"""

#======== PROMPT PARA O MODELO ======== 
SYSTEM_PROMPT = f"""
Você é ATLAS um agente financeiro inteligente especializado em finanças.
Seu objetivo principal é orientar usuários a respeito das melhores opções de investimento com base no perfil dos usuários.

REGRAS:
1. Sempre baseie suas respostas nos dados fornecidos
2. Nunca invente informações financeiras
3. Se não souber algo, admita e ofereça alternativas
4. Não responda perguntas a respeito de senhas.
5. Não responda peruntas foras da área de finanças 
6. Seja claro e objetivo, evite jargões técnicos 
7. Sempre que possível, ofereça opções de investimento alinhadas ao perfil do usuário 
8. Se o usuário tiver dúvidas, ofereça explicações simples e exemplos práticos 
9. Mantenha um tom amigável e acessível, mesmo ao explicar conceitos complexos 
10. Se o usuário tiver um perfil conservador, priorize opções de baixo risco e alta liquidez
11. Se o usuário tiver um perfil moderado, ofereça uma combinação de opções de baixo e médio risco, considerando seus objetivos e horizonte de investimento
12. Se o usuário tiver um perfil agressivo, sugira opções de médio a alto risco, alinhadas aos seus objetivos e horizonte de investimento
"""
#======== CHAMAR OLLAMA ======== 
def perguntar(msg): 
    prompt = f"""
{SYSTEM_PROMPT}

CONTEXTO:
{contexto}

PERGUNTA:
{msg}
"""

    try:
        r = requests.post(
            OLLAMA_URL,
            json={
                "model": MODELO,
                "prompt": prompt,
                "stream": False
            },
            timeout=120
        )

        r.raise_for_status()  # lança erro se status != 200

        data = r.json()
        if 'response' in data:
            return data['response']
        else:
            print("Erro na resposta da API:")
            print(data)
            return None

    except requests.exceptions.RequestException as e:
        print("Erro na requisição:", e)
        print("Resposta bruta:", r.text if 'r' in locals() else "")
        return "Erro ao conectar com o modelo."

#======= INTERFACE SIMPLES COM STREAMLIT ======== 
st.title("ATLAS - Seu Agente Financeiro Inteligente")
st.write("Olá! Sou o ATLAS, seu agente financeiro inteligente. Estou aqui para ajudar-lo a tomar as melhores decisões de investimento com base no seu perfil e objetivos. Pergunte-me qualquer coisa sobre finanças e investimentos!")

if conversa := st.chat_input("Digite sua pergunta aqui..."):
    resposta = perguntar(conversa)
    st.chat_message("user").write(conversa)
    with st.spinner("ATLAS está pensando..."):
        st.chat_message("assistant").write(resposta) 
