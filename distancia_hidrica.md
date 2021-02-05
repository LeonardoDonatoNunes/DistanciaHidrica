---
title: ""
author: ""
date: ""
output: 
  html_document:
    keep_md: true
    theme: simplex
encoding: iso-8859-1
---




<style type="text/css">
  body{
  font-size: 14pt;
  color: #B9BBB6;
  text-align: justify;
  background-color:black;
}
</style>


 &nbsp;  &nbsp;  &nbsp;    ![](C:/Users/user/Dropbox/R/ImagensRmarkdown/r.png){width=110px}    &nbsp;  &nbsp;  &nbsp;   ![](C:/Users/user/Dropbox/R/ImagensRmarkdown/rstudio.png){width=250px} ![](C:/Users/user/Dropbox/R/ImagensRmarkdown/ggplot.png){width=100px}       &nbsp; &nbsp; &nbsp;     ![](C:/Users/user/Dropbox/R/ImagensRmarkdown/qgis.png){width=100px}  &nbsp;  &nbsp;  &nbsp;   ![](C:/Users/user/Dropbox/R/ImagensRmarkdown/dplyr.png){width=100px}  









# Menor caminho entre dois pontos {.tabset}

Este documento foi criado para auxiliar quem precisa encontrar o menor caminho entre dois ou mais pontos dentro de um pol�gono, levando em considera��o os lim�tes da �rea.


Como exemplo, criei um pol�gono de rio, que cont�m ilhas e curvas. O objetivo � criar um caminho entre as ilhas e curvas que ligue dois pontos (como apresentado na **Figura 1**). Os pontos podem estar em um arquivo **.csv ** ou em um outro arquivo **.shp **. Neste caso preferi copiar as coordenadas para ter menos arquivos de entrada. selecionei um trecho de rio qualquer (n�o fiz quest�o de registrar o nome do rio para n�o desviar a aten��o), criei um arquivo **.shp ** e desenhei o contorno do rio e depois criei os buracos das ilhas (as ilhas n�o existiam no rio, eu inventei). Para cria��o do pol�gono utilizei o software QGIS.

<br />

<img src="distancia_hidrica_files/figure-html/plot_poligono_intro-1.png" style="display: block; margin: auto;" />
  


**Figura 1.** Pol�gono do rio e pontos de origem e de destino. O objetivo deste roteiro � criar um caminho entre os dois pontos que considere as margens do pol�gono (linha laranja).

<br />

O documento est� dividido em partes, que podem ser acessada nas abas da tabela de acordo com os �tens abaixo:

  1.1. Carregar os pacotes
  
  1.2. Carregar o pol�gono do rio
  
  1.3. Cria��o do grid e da camada raster
  
  1.4. Cria��o da camada de transi��o e do menor caminho
  
  1.5. Fun��o que calcula o tamanho do seguimento (dist�ncia entre os dois pontos)


<br />
<br />

## Pacotes

Pacotes para o tratamento dos dados:

```{.r .bg-info}
library(raster)
library(gdistance)
library(dplyr)
```

<br />
Pacotes para para o processamento dos dados geogr�ficos e cira��o do mapa.

```{.r .bg-info}
library(ggplot2)
library(ggthemes)
library(ggsn)
library(ggspatial)
library(sf)
```

<br />
<br />
<br />
<br />

## Carregar os dados

O primeiro passo � carregar o pol�gono do rio, para isso, utilisei a fun��o "shapefile" do pacote "raster". Depois armazenei os pontos de origem e de destino em duas vari�veis. Preferi copiar as coordenadas para ter um arquivo de entrada a menos. Mas essas coordenadas poderiam estar em um arquivo ou em outra camada. 


```{.r .bg-info}
# Carrega o poligono do rio no formato .shp
rio <- shapefile("rio.shp")

# Cria dois pontos que precisarão ser conectados
origem <- c(-49.16760,-20.30495)
destino <- c(-48.75058,-20.15693)
```

<br />
<br />
<br />
<br />

## Grid e raster

Em seguida � necess�rio criar uma camada no formato raster que ser� utilizada para criar a camada de transi��o. Para isso ser� necess�rio escolher o tamanho das c�lulas do grid. Lembrando que o tamanho � muito particular, dependende do n�vel de detalhes necess�rio para que toda a �rea contenha c�lulas. 



```{.r .bg-info}
# Cria o grid na 攼㸱rea do pol攼㹤gono
grid <- makegrid(rio, cellsize = 0.0005) # Tamanho da célula em unidades de mapa!

# O grid 攼㸹 um data.frame. para transformar em um  conjunto de dados espaciais utiliza-se a funcao SpatialPoints
grid <- SpatialPoints(grid, proj4string = CRS(proj4string(rio)))

# Extrair somente a parte do grid que intersepta o poligono do rio
grid <- grid[rio, ]

# Converte o grid novamente em um data.frame para criacaodo raster
grid_df <- as.data.frame(grid)

# Atribui um valor para as celulas que estao no rio (agua)
grid_df$value <- 1

# Cria o raser do data.frame
rio_raster <- rasterFromXYZ(grid_df)

plot(rio_raster)
points(x = -48.70708, y = -22.17826, col = "red", cex = 4)
```

<img src="distancia_hidrica_files/figure-html/cria_grid_novamente-1.png" style="display: block; margin: auto;" />


<br />
<br />
<br />
<br />

## Camada de transi��o

Para criar o caminho em um raster � necess�rio que seja criado uma camada de transi��o. Para isso, utilizei a fun��o "transition", do pacote "gdistance". Para quem j� possui um raster da �rea desejada, ou preferiu criar o raster com outro software, basta carregar o raster a partir desta etapa. 

<br />


```{.r .bg-info}
# Transforma o raster em uma camada de transiÃ§Ã£o
rio_tr <- transition(rio_raster, mean, directions = 8)
rio_tr <- geoCorrection(rio_tr, "c")
```
<br />
O menor caminho � criado com a fun��o 'shortestPath' tamb�m do pacote 'ggdistance'. Utilizei a fun��o CRS para assinar um crs ao caminho.
<br />

```{.r .bg-info}
# Função que cria o menor caminho (pacote gdistance)
caminho <- shortestPath(rio_tr, origem, destino, output = "SpatialLines")
crs(caminho) <- CRS("+proj=longlat +datum=WGS84")
```

<br />

A seguir a instru�ao de como foi feito o plot que � apresentado na introdu��o.


```{.r .bg-info}
# Funcao que cria o menor caminho (pacote gdistance)
caminho <- shortestPath(rio_tr, origem, destino, output = "SpatialLines")
crs(caminho) <- CRS("+proj=longlat +datum=WGS84")

label_compr <- paste0("Dist攼㸲ncia calculada: ", round(SpatialLinesLengths(caminho), 2), " km")

rio <- st_as_sf(rio)
caminho <- st_as_sf(caminho, crs = CRS('+proj=longlat +datum=WGS84'))


# Armazena os limites da area do rio para rosa dos ventos 
xmin <- extent(rio)[1]
xmax <- extent(rio)[2]
ymin <- extent(rio)[3]
ymax <- extent(rio)[4]

ggplot() +
  geom_sf(data = rio, fill = "lightblue", color = "lightblue") +
  # Dados dos peixes
  geom_sf(data = caminho, col = "orange", lwd = 2) +
  
  geom_point(data = NULL, aes(x = origem[1], y = origem[2]), col = "red", size = 5) +
  geom_text(data = NULL, aes(x = origem[1], y = origem[2], label = "Origem"),
            size = 5, vjust = 3) +
  
  geom_point(data = NULL, aes(x = destino[1], y = destino[2]), col = "green", size = 5) +
  geom_text(data = NULL, aes(x = destino[1], y = destino[2], label = "Destino"), 
            size = 5, vjust = 2) +
  
  scalebar(dist = 5, dist_unit = "km", x.min =  xmin, x.max = xmax, y.min = ymin, y.max = ymax,
           transform = T, model = "WGS84", location = "bottomright", st.dist = 0.05, st.color = "grey50", box.fill = c("grey50", "white"), box.color = "grey50") +
  north(x.min =  xmin, x.max = xmax, y.min = ymin, y.max = ymax, scale = 0.12, anchor = c(x = -48.75, y = -20.39), symbol = 4) +
  
  annotate("text", x = -49.1, y = -20.43, label = label_compr, size = 5) +
  
  ylab("") +
  xlab("") +
  
  theme_base() +
  theme(axis.text = element_blank(),
        axis.ticks = element_blank(),
        plot.background = element_rect(fill  = "black"))
```

<img src="distancia_hidrica_files/figure-html/plot_caminho_intro-1.png" style="display: block; margin: auto;" />
<br />
<br />
<br />
<br />

## Dist�ncia hidrica
Para calcular a dist�ncia h�drica utilizei a fu��o "SpatialLinesLengths" do pacote "sp". Importante notar que foi utilizado o argumento "latlong = T", para retornar a dist�ncia em km.



```{.r .bg-info}
caminho <- shortestPath(rio_tr, origem, destino, output = "SpatialLines")
crs(caminho) <- CRS("+proj=longlat +datum=WGS84")
SpatialLinesLengths(caminho, longlat = T)
```

```
## [1] 100.4559
```

<br />
<br />
<br />
<br />

</br >
</br >

#

Leonardo Donato Nunes [![](C:/Users/user/Dropbox/R/ImagensRmarkdown/linkedin.png){width=40px}](https://www.linkedin.com/in/leonardo-donato-nunes-754aa5b8?lipi=urn%3Ali%3Apage%3Ad_flagship3_profile_view_base_contact_details%3BZeE8YK6ARgGfUOtWF8bkxg%3D%3D){style="float:right"} &nbsp;  &nbsp;  &nbsp;  &nbsp; [![](C:/Users/user/Dropbox/R/ImagensRmarkdown/email.png){width=40px}](leonardodonatonunes@yahoo.com.br){style="float:right"} &nbsp;  &nbsp;  &nbsp;  &nbsp; [![](C:/Users/user/Dropbox/R/ImagensRmarkdown/github.png){width=45px}](https://github.com/LeonardoDonatoNunes){style="float:right"} &nbsp;  &nbsp;

Brasil, 25 de Janeiro de 2021

</br >

 <a href="#top">Back to top</a>
