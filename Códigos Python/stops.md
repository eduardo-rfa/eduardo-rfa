# Stops e Target Price

Esse código define Stop Loss e Stop Gain de ativos, com o intuito de realizar parte da posição em caso de perdas ou ganhos.

Inicialmente, definimos o Stop com base no Value-at-Risk e na volatilidade do ativo e, posteriormente, monitoramos diariamente os stops com base nos retornos de cada ativo.

Posteriormente, encerramos o código com a definição dos Target Prices (preços-alvo), baseados na volatilidade de cada ativo.

Obs.: código desenvolvido no Google Colab

## Importando Bibliotecas

```python
!pip install yfinance
import yfinance as yf
import pandas as pd
import numpy as np
import datetime as dt
import pytz
from datetime import date, datetime
```

## Definições iniciais

```python
# Definindo as datas
start = dt.datetime(2023,1,2) #pode mudar para a data que quiser
end = datetime.now(pytz.timezone('America/Sao_Paulo')).strftime("%Y-%m-%d") #data atual por fórmula

# Definindo os elementos para o dicionário que virará df

ativos = ['EWU', 'EWG', 'EWI', 'EWP', 'PGAL'] #escolha de ativos
posiçoes = ['V', 'V', 'V', 'V', 'V'] #importante seguir a ordem dos ativos acima. Ex.: se eu estou comprado em ARZZ3, o 1º elemento da lista 'posições' é a direção de Arezzo

cotaçoes = yf.download(ativos,start,end)['Adj Close'] #puxando o preço de fechamento dos ativos
apoio_r0 = cotaçoes.pct_change() #plotando os retornos
apoio_r1 = apoio_r0.ffill().tail(1) #removendo NaN (tratando o df de retornos)
apoio_r2 = apoio_r1.values.tolist() #aqui, após a conversão do df em lista, ele nos retorna uma lista de listas
retornos = [] #definindo retornos como lista
for lista in apoio_r2: #usando "for" para converter a lista de listas em uma lista única com a função "extend()"
    retornos.extend(lista)

cotaçoes_stop = yf.download(ativos, start, end)["Adj Close"]
cotaçoes_stop = cotaçoes_stop.reindex(columns = ativos)
cotaçoes_stop = cotaçoes_stop.ffill()

retornos_stop = cotaçoes_stop.pct_change().fillna(0)

# Stop com base em vol diária
vol_dia = {}
for etf in ativos:
  vol_dia[etf] = retornos_stop[etf].std() 

st_vol_dia = pd.DataFrame.from_dict(vol_dia, orient ='index')
st_vol_dia.columns = ['volatilidade diária']
st_vol_dia['ST Gain 1'] = st_vol_dia['volatilidade diária']
st_vol_dia['ST Loss 1'] = st_vol_dia['volatilidade diária'] * -1
st_vol_dia = st_vol_dia.drop(['volatilidade diária'], axis=1)

# Stop com base em var diário
var = {}
short_var = {}
for etf in ativos:
  var[etf] = np.percentile(retornos_stop[etf], 5)
  short_var[etf] = np.percentile(retornos_stop[etf], 95)

aux = pd.DataFrame.from_dict(var, orient ='index')
aux.columns = ['ST Loss 2']
aux2 = pd.DataFrame.from_dict(short_var, orient ='index')
aux2.columns = ['ST Gain 2']

stop_diario = pd.concat([st_vol_dia['ST Loss 1'], aux, st_vol_dia['ST Gain 1'], aux2], axis=1) #plotando e organizando o df de stop diário
stop_diario

# Convertendo as colunas do df em lista para utilizar no novo df posteriormente
stl1 = stop_diario['ST Loss 1'].tolist()
stl2 = stop_diario['ST Loss 2'].tolist()
stg1 = stop_diario['ST Gain 1'].tolist()
stg2 = stop_diario['ST Gain 2'].tolist()
```

## Monitoramento stops

```python
# Crie um df com os ativos, retornos e stops dos ativos
df = pd.DataFrame({'Ativo': ativos, 'Retorno': retornos, 'Stop Gain 1': stg1, 'Stop Gain 2': stg2, 'Stop Loss 1': stl1, 'Stop Loss 2': stl2, 'Posição': posiçoes})

for x in range(len(df)): #range(len(df)) é usada para percorrer todas as linhas do df e verificar se o retorno de cada ativo é > ou < que seu stop
  if df.loc[x, 'Posição'] == 'C': #caso a posição seja comprada (C)
    if df.loc[x, 'Retorno'] <= df.loc[x, 'Stop Loss 2']:
        print(f"{df.loc[x, 'Ativo']}: Stop Loss 2 (realizar 25% da posição)")
    elif df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 2'] and df.loc[x, 'Retorno'] <= df.loc[x, 'Stop Loss 1']:
        print(f"{df.loc[x, 'Ativo']}: Stop Loss 1 (realizar 15% da posição)")
    elif df.loc[x, 'Retorno'] >= df.loc[x, 'Stop Gain 2']:
        print(f"{df.loc[x, 'Ativo']}: Stop Gain 2 (realizar 80% da posição)")
    elif df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 2'] and df.loc[x, 'Retorno'] >= df.loc[x, 'Stop Gain 1']:
        print(f"{df.loc[x, 'Ativo']}: Stop Gain 1 (realizar 50% da posição)")
    elif df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 2'] and df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 1']:
        print(f"{df.loc[x, 'Ativo']}: Manter posição")
    elif df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 1'] and df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 2']:
        print(f"{df.loc[x, 'Ativo']}: Manter posição")
    else:
        pass
  else: #caso a posição seja vendida (V)
    if df.loc[x, 'Retorno'] <= df.loc[x, 'Stop Loss 2']:
        print(f"{df.loc[x, 'Ativo']}: Stop Gain 2 (realizar 80% da posição)")
    elif df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 2'] and df.loc[x, 'Retorno'] <= df.loc[x, 'Stop Loss 1']:
        print(f"{df.loc[x, 'Ativo']}: Stop Gain 1 (realizar 50% da posição)")
    elif df.loc[x, 'Retorno'] >= df.loc[x, 'Stop Gain 2']:
        print(f"{df.loc[x, 'Ativo']}: Stop Loss 2 (realizar 25% da posição)")
    elif df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 2'] and df.loc[x, 'Retorno'] >= df.loc[x, 'Stop Gain 1']:
        print(f"{df.loc[x, 'Ativo']}: Stop Loss 1 (realizar 15% da posição)")
    elif df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 2'] and df.loc[x, 'Retorno'] > df.loc[x, 'Stop Loss 1']:
        print(f"{df.loc[x, 'Ativo']}: Manter posição")
    elif df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 1'] and df.loc[x, 'Retorno'] < df.loc[x, 'Stop Gain 2']:
        print(f"{df.loc[x, 'Ativo']}: Manter posição")
    else:
        pass
```

## Check Target Price

```python
start2 = dt.datetime(2023,1,2) #pegando uma janela de 3 meses
cotaçao2 = yf.download(ativos, start2, end)["Adj Close"] #puxando os preços novamente
cotaçao2 = cotaçao2.ffill()
cotaçao2.loc['Média'] = cotaçao2.mean() #adicionando nova linha com a média

target = cotaçao2.tail(2).T #selecionando os 2 últimos dados (média dos preços e último preço) e transpondo
target.columns = ['Último Preço', 'Média dos Preços'] #ajeitando os nomes das colunas
target['Posição'] = posiçoes

retornos2 = cotaçao2.pct_change().fillna(0) #plotando os retornos novamente e tratando com fillna(0)

# Volatilidade (os "for" são necessários porque vamos fazer para cada ativo, chamando de x)
dic_vol = {}
dic_vol_aux = {}
for x in ativos:
  dic_vol[x] = retornos2[x].std() #volatilidade de cada ativo com for
  if target.loc[x, 'Posição'] == 'C':
    dic_vol_aux[x] = 1 - retornos2[x].std() #volatilidade de cada ativo com for - 1
  else:
    dic_vol_aux[x] = 1 + retornos2[x].std() #mesma situação da anterior, mas pensando numa pos. vendida
for x in ativos:
  target.loc[x, 'Volatilidade'] = str(round(dic_vol[x] * 100, 2)) + '%' #plotando a volatilidade e adicionando formatação p/ display
  target.loc[x, 'Target Price'] = target.loc[x, 'Média dos Preços'] * dic_vol_aux[x] #plotando o Target Price como a média dos preços * (1 + vol do ativo)
target

dic_preço = {}
#Fazendo um for para rodar todos os ativos e um if dentro para receber o retorno de se o Target Price foi batido ou não
for x in ativos:
  if target.loc[x, 'Último Preço'] >= target.loc[x, 'Target Price'] and target.loc[x, 'Posição'] == 'C':
    dic_preço[x] = "Target Price batido"
  elif target.loc[x, 'Último Preço'] < target.loc[x, 'Target Price'] and target.loc[x, 'Posição'] == 'C':
    dic_preço[x] = "Target Price não foi batido"
  elif target.loc[x, 'Último Preço'] <= target.loc[x, 'Target Price'] and target.loc[x, 'Posição'] == 'V':
    dic_preço[x] = "Target Price batido"
  elif target.loc[x, 'Último Preço'] > target.loc[x, 'Target Price'] and target.loc[x, 'Posição'] == 'V':
    dic_preço[x] = "Target Price não foi batido"
dic_preço
```
