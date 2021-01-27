---
title: "Distância Hídrica -  Menor caminho entre curvas e ilhas"
author: "Leonardo Nunes"
date: "1/25/2021"
output: 
  html_document:
    keep_md: true
    number_sections: true
    theme: simplex
---






# Menor caminho {.tabset}

Este documento foi criado para auxiliar quem precisa encontrar o menor caminho entre dois ou mais pontos dentro de um polígono, levando em consideração os limítes da área.


Como exemplo, criei um polígono de rio, que contém ilhas e curvas. O objetivo é criar um caminho entre as ilhas e curvas que ligue dois pontos (como apresentado na **Figura 1**). Os pontos podem estar em um arquivo **.csv ** ou em um outro arquivo **.shp **. Neste caso preferi copiar as coordenadas para ter menos arquivos de entrada. selecionei um trecho de rio qualquer (não fiz questão de registrar o nome do rio para não desviar a atenção), criei um arquivo **.shp ** e desenhei o contorno do rio e depois criei os buracos das ilhas (as ilhas não existiam no rio, eu inventei). Para criação do polígono utilizei o software QGIS.

<br />

<img src="distancia_hidrica_files/figure-html/plot_poligono_intro-1.png" style="display: block; margin: auto;" />
  


**Figura 1.** Polígono do rio e pontos de origem e de destino. O objetivo deste roteiro é criar um caminho entre os dois pontos que considere as margens do polígono (linha roxa).

<br />

O documento está dividido em partes, que podem ser acessada nas abas da tabela de acordo com os ítens abaixo:

  1.1. Carregar os pacotes
  
  1.2. Carregar o polígono do rio
  
  1.3. Criação do grid e da camada raster
  
  1.4. Criação da camada de transição e do menor caminho
  
  1.5. Função que calcula o tamanho do seguimento (distãncia entre os dois pontos)


<br />
<br />

## Pacotes

Pacotes para o tratamento dos dados:

```r
library(raster)
library(gdistance)
library(dplyr)
```

<br />
Pacotes para criar a vizsualização do mapa:

```r
library(ggplot2)
library(ggthemes)
library(ggsn)
library(ggspatial)
```

<br />
<br />
<br />
<br />

## Carregar os dados

O primeiro passo é carregar o polígono do rio, para isso, utilisei a função "shapefile" do pacote "raster". Depois armazenei os pontos de origem e de destino em duas variáveis. Preferi copiar as coordenadas para ter um arquivo de entrada a menos. Mas essas coordenadas poderiam estar em um arquivo ou em outra camada. 


```r
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

Em seguida é necessário criar uma camada no formato raster que será utilizada para criar a camada de transição. Para isso será necessário escolher o tamanho das células do grid. Lembrando que o tamanho é muito particular, dependende do nível de detalhes necessário para que toda a área contenha células. 



```r
# Cria o grid na área do polígono
grid <- makegrid(rio, cellsize = 0.0005) # Tamanho da célula em unidades de mapa!

# O grid é um data.frame. para transformar em um  conjunto de dados espaciais utiliza-se a funçao SpatialPoints
grid <- SpatialPoints(grid, proj4string = CRS(proj4string(rio)))

# Extrair somente a parte do grid que intersepta o poligono do rio
grid <- grid[rio, ]

# Converte o grid novamente em um data.frame para criação do raster
grid_df <- as.data.frame(grid)

# Atribui um valor para as células que estão no rio (água)
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

## Camada de transição

Para criar o caminho em um raster é necessário que seja criado uma camada de transição. Para isso, utilizei a função "transition", do pacote "gdistance". Para quem já possui um raster da área desejada, ou preferiu criar o raster com outro software, basta carregar o raster a partir desta etapa. 



```r
# Transforma o raster em uma camada de transiÃ§Ã£o
rio_tr <- transition(rio_raster, mean, directions = 8)
rio_tr <- geoCorrection(rio_tr, "c")
```

O menor caminho é criado com a função 'shortestPath' também do pacote 'ggdistance'. Após a criação do caminho, para que a camada seja utilziada no gráfico é necessário salver o arquivo para criar os 4 arquivos que compões um shapefile.


```r
# Função que cria o menor caminho (pacote gdistance)
caminho <- shortestPath(rio_tr, origem, destino, output = "SpatialLines")

# Salva um arquivo temporário pra criar todos os arquivos de um shapefile
shapefile(caminho, "Temp.shp", overwrite=TRUE)
caminho <- shapefile("Temp.shp")
```

<br />

A seguir a instruçao de como foi feito o plot que é apresentado na introdução.


```r
# Armazena os limites da área do rio para rosa dos ventos 
xmin <- extent(rio)[1]
xmax <- extent(rio)[2]
ymin <- extent(rio)[3]
ymax <- extent(rio)[4]

# Cria o gráfico para visualizar o caminho criado
ggplot() +
  geom_polygon(data = rio, aes(x = long, y = lat, group = group, fill = group), show.legend = F, col = "black") +
  
  geom_point(data = NULL, aes(x = origem[1], y = origem[2]), col = "red", size = 4) +
  geom_text(data = NULL, aes(x = origem[1], y = origem[2], label = "Origem"),
            size = 4, vjust = 3) +
  
  geom_point(data = NULL, aes(x = destino[1], y = destino[2]), col = "green", size = 4) +
  geom_text(data = NULL, aes(x = destino[1], y = destino[2], label = "Destino"), 
            size = 4, vjust = 2) +

  geom_path(data = caminho, aes(x = long, y = lat, group = group), col = "purple", lwd = 1.3) +
  
  scale_fill_manual(values = c("0.1" = alpha("lightblue", alpha = 0.3),
                               "0.2" = "#f5f5f2","0.3" = "#f5f5f2","0.4" = "#f5f5f2",
                               "0.5" = "#f5f5f2","0.6" = "#f5f5f2","0.7" = "#f5f5f2")) +
  
  coord_fixed() +
  labs(x = NULL, 
         y = NULL, 
         title = "Menor distância entre dois pontos", 
         subtitle = NULL, 
         caption = paste0("Distância calculada: ", round(SpatialLinesLengths(caminho, longlat = T), 2), " km")) +
  
    north(x.min = xmin, x.max = xmax, y.min = ymin, y.max = ymax, location = "bottomright", scale = 0.2, symbol = 3)  +

  annotation_scale(plot_unit = "km") +
  theme_minimal() +
  
  theme(
    text = element_text(family = "Ubuntu Regular", color = "#22211d"),
    axis.line = element_blank(),
    axis.text.x = element_blank(),
    axis.text.y = element_blank(),
    axis.ticks = element_blank(),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    # panel.grid.minor = element_line(color = "#ebebe5", size = 0.2),
    panel.grid.major = element_line(color = "#ebebe5", size = 0.2),
    panel.grid.minor = element_blank(),
    plot.background = element_rect(fill = "#f5f5f2", color = NA), 
    panel.background = element_rect(fill = "#f5f5f2", color = NA), 
    legend.background = element_rect(fill = "#f5f5f2", color = NA),
    panel.border = element_blank()
  )
```

<img src="distancia_hidrica_files/figure-html/plot_caminho_intro-1.png" style="display: block; margin: auto;" />
<br />
<br />
<br />
<br />

## Distância hidrica
Para calcular a distância hídrica utilizei a fução "SpatialLinesLengths" do pacote "sp". Importante notar que foi utilizado o argumento "latlong = T", para retornar a distância em km.



```r
SpatialLinesLengths(caminho, longlat = T)
```

```
## [1] 100.4559
```

<br />
<br />
<br />
<br />
