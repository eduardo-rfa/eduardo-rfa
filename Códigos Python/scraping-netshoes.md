# Web Scraping (Netshoes)

Desenvolvi esse código dentro da Impactus UFRJ como treinamento para o Web Scraping. Na ocasião, o objetivo era obter como retorno um dataframe que reunisse todos os tênis masculinos do site da Netshoes e seus respectivos preços.

## Importando Bibliotecas

```python
%%shell
# Ubuntu no longer distributes chromium-browser outside of snap
#
# Proposed solution: https://askubuntu.com/questions/1204571/how-to-install-chromium-without-snap

# Add debian buster
cat > /etc/apt/sources.list.d/debian.list <<'EOF'
deb [arch=amd64 signed-by=/usr/share/keyrings/debian-buster.gpg] http://deb.debian.org/debian buster main
deb [arch=amd64 signed-by=/usr/share/keyrings/debian-buster-updates.gpg] http://deb.debian.org/debian buster-updates main
deb [arch=amd64 signed-by=/usr/share/keyrings/debian-security-buster.gpg] http://deb.debian.org/debian-security buster/updates main
EOF

# Add keys
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys DCC9EFBF77E11517
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 648ACFD622F3D138
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 112695A0E562B32A

apt-key export 77E11517 | gpg --dearmour -o /usr/share/keyrings/debian-buster.gpg
apt-key export 22F3D138 | gpg --dearmour -o /usr/share/keyrings/debian-buster-updates.gpg
apt-key export E562B32A | gpg --dearmour -o /usr/share/keyrings/debian-security-buster.gpg

# Prefer debian repo for chromium* packages only
# Note the double-blank lines between entries
cat > /etc/apt/preferences.d/chromium.pref << 'EOF'
Package: *
Pin: release a=eoan
Pin-Priority: 500


Package: *
Pin: origin "deb.debian.org"
Pin-Priority: 300


Package: chromium*
Pin: origin "deb.debian.org"
Pin-Priority: 700
EOF

!apt-get update
!apt-get install chromium chromium-driver
!pip3 install selenium

from selenium import webdriver
from selenium.webdriver.chrome.options import Options
import pandas as pd
import time
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
```

## Web Scraping Netshoes

```python
# Definições iniciais
url = "https://www.netshoes.com.br/busca/tenis-masculino?searchTermCapitalized=Tenis%20Masculino&nsCat=Artificial&page=1" #definindo o url
options = Options()
options.add_argument("--headless") #não abre a janela de browser que seria aberta no VS Code
options.add_argument("--no-sandbox") #economia de memória (código não vai fazer coisas desnecessárias)
driver = webdriver.Chrome("chromedriver", options=options) #abrindo o driver e passando as opções que criamos

df = pd.DataFrame() #definindo df como dataframe Pandas

#Fazendo um for para o código fazer o scraping em cada página, a partir de um loop
for i in range(1,239): #definir o range com base no número de páginas, considerando que o url inicial apresenta o "&page=1" e que a cada página que passa, só muda o nº no url
  #f" para fazer o scraping página por página, dado o range definido acima
  driver.get(f"https://www.netshoes.com.br/busca/tenis-masculino?searchTermCapitalized=Tenis%20Masculino&nsCat=Artificial&page={i}") #abrindo a pág.
  time.sleep(7) #Aguardando 7 segundos para carregar a página

  # XPaths
  tenis = driver.find_elements("xpath", "//div[@class = 'item-card__description__product-name']") #xpath do tênis
  preco = driver.find_elements("xpath", "//div[@class='price__list']//span[@class='haveInstallments']") #xpath do preço

  df_apoio = pd.DataFrame() #definindo df_apoio como dataframe Pandas
  for p in range(len(tenis)): #range(len(tenis)) vai nos retornar o nº de tênis que aparecem na página (no caso, 42). Assim, o código puxará todos os 42 tênis em cada página e seus respectivos preços
    driver.execute_script('window.scrollBy(0,window.innerHeight)') #scrolling para carregar os dados
    df_apoio = df_apoio.append({'Tênis': tenis[p].text, 'Preço': preco[p].text}, ignore_index=True) #append no df definido para pegar os tênis e preços
  df = pd.concat([df, df_apoio], ignore_index=True)
  # Código vai puxando as infos. com o df_apoio, que vai sendo concatenado gradativamente ao df principal - se não fizermos isso, cada scraping deletaria o df derivado da pág. anterior (ex.: df do scraping da pág. x+1 substitui o df da pág. x)
df
```