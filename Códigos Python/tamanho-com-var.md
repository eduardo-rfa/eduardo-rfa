# Tamanhos com VaR e Avaliação de Volume

Esse código calcula os tamanhos financeiros de entrada em posições do fundo fictício da Impactus UFRJ com base no Value-at-Risk dos ativos. As definições a serem feitas são as quantidades de posições médias, pequenas e grandes que o fundo pretende negociar ao longo da semana, além de um VaR Target, disponível nas políticas de risco das gestoras de fundos de investimento. Todas essas definições são feitas de maneira estratégica pelos gestores e analistas que cuidam do fundo.

Além disso, estabelecemos uma avaliação de volume, de modo a verificar o volume máximo do ativo que pode ser negociado no dia. Como trabalhamos com um fundo fictício, é importante estabelecer limites de volume para negociação. No nosso caso, o limite corresponde a 40% da média do volume do ativo negociado nos últimos 20 dias.

## Importando bibliotecas

```python
!pip install yfinance
import yfinance as yf
import pandas as pd
import numpy as np
import datetime as dt
import pytz
from datetime import date, datetime
```
## Escolha de ativos e dfs iniciais

```python
#Escolhendo os ativos
ativos = ['EWG', 'EWP', 'EWI', 'SOYB']

#Definindo as posições e pesos (OBS.: colocar as posições sempre na mesma ordem em posiçao, tam_posiçao e peso)
posiçao = {'EWG': 'V  ',	'EWP': 'V', 'EWI': 'V', 'SOYB': 'C'} # C = comprado; V = vendido
tam_posiçao = {'EWG': 'M',	'EWP': 'M', 'EWI': 'M', 'SOYB': 'M'} # P = pequena; M = média; G = grande

#Definindo as datas
start = dt.datetime(2018,1,2) #pode mudar para a data que quiser
end = datetime.now(pytz.timezone('America/Sao_Paulo')).strftime("%Y-%m-%d") #data atual por fórmula

#Definindo as cotações
cotaçoes = yf.download(ativos,start,end)['Adj Close'] #puxando o preço de fechamento dos ativos
cotaçoes

retorno_base = cotaçoes.pct_change() #plotando os retornos
retornos = retorno_base.fillna(0) #removendo NaN (tratando o df de retornos)
retornos

#Definindo os volumes
volume_i = yf.download(ativos, start, end)["Volume"]
volume_m = volume_i.fillna(0).tail(20) #janela de 20 dias
volume_m
```

## Posições e estratégias

```python
#Definindo as posições (pequenas, médias e grandes) - para isso, é preciso definir a quantidade média de ações negociadas em um certo período de tempo, e avaliar estrategicamente qual parte dessas são médias, pequenas e grandes
pequena = 2
media = 3
grande = 2
 
#Estabelecendo pesos para cada posição
peso_p = 1
peso_m = 2*peso_p #uma média equivale a 2 pequenas
peso_g = 4*peso_p #uma grande equivale a 4 pequenas
 
#Definindo o PL
PL = 100404643.50
 
#Definindo o VaR Target
var_target = 0.015
 
# % do PL em "risco" com base no VaR Target
pct_pl_risco = PL*var_target
 
#Definindo os tamanhos padrão
padrao_pequena = pct_pl_risco / (pequena*peso_p + media*peso_m + grande*peso_g)
padrao_media = peso_m*padrao_pequena
padrao_grande = peso_g*padrao_pequena

#Criando uma lista
tam_padrao = [padrao_pequena,padrao_media,padrao_grande]
```

## Tamanho c/ Var e avaliação de volume

```python
#VaR Histórico X% (posso garantir que não perderei mais que (resultado do var)% em um dia com X% de certeza)

def formatar_tamanho(valor):
    return "R$ {:,.10}".format(valor) #definindo a formatação do tamanho
def formatar_percentual(valor):
    return "{:.2f}%".format(valor*100) #definindo a formatação do percentual

confiança_var = 95 #Var 95%
var = {}
for x in posiçao:
  if posiçao[x] == 'C':
    var[x] = np.percentile(retornos[x], 100 - confiança_var)
  else:
    var[x] = np.percentile(retornos[x], confiança_var) #pega o kº elemento da série de retornos ordenados

tamanho = {}

#Calculando o tamanho para cada ativo, considerando o tipo de posição desejada e o Value-at-Risk
for x in tam_posiçao:
  if tam_posiçao[x] == 'P':
    tamanho[x] = abs(padrao_pequena/var[x])
  elif tam_posiçao[x] == 'M':
    tamanho[x] = abs(padrao_media/var[x])
  else:
    tamanho[x] = abs(padrao_grande/var[x])

tamanho = pd.DataFrame.from_dict(tamanho, orient='index') 
tamanho.columns = ['Financeiro c/ VaR'] 

#Incluindo o % do PL
for x in ativos:
  tamanho.loc[x, '% PL'] = tamanho.loc[x, 'Financeiro c/ VaR'] / PL #criando coluna de % do PL que o financeiro representa
tamanho['Volume Médio'] = volume_m.mean() #criando coluna de volume médio
limite = 0.4 #parte limite da qtd. máxima a ser negociada

#Incluindo a avaliação de volume
for x in ativos:
  tamanho.loc[x, 'Qtd. máxima'] = tamanho.loc[x, 'Volume Médio'] * limite #criando coluna de qtd. máxima
tamanho['Financeiro c/ VaR'] = tamanho['Financeiro c/ VaR'].apply(formatar_tamanho) #aplicando a formatação que definimos
tamanho['% PL'] = tamanho['% PL'].apply(formatar_percentual) #aplicando a formatação que definimos
tamanho
```