# Desafio Visio  
Marcelo Isaias de Moraes Junior  
João Pedro Almeida Santos Secundino  
João Vitor Silva Ramos  
Victor Giovannoni Vernalha  

# Objetivo  
O objetivo desse projeto é unir as fotos tiradas de uma câmara de Neubauer contendo leveduras vivas e mortas e realizar a contagem de células em ambas as categorias.
A especificação completa do desafio está disponível [aqui](https://www.notion.so/Proposta-de-Projeto-336e8afb603447109116a61d147c0e09).

# Arquivos  
`stitcher.ipynb`: Responsável por receber as 5x5 imagens e criar a imagem unificada do grid.  
`dataset_creator.ipynb`: Responsável por extrair as features das imagens do dataset de classificação e criar o dataset.
`detect_and_classify.ipynb`: Responsável por receber a imagem do grid, detectar, classificar e contar as células mortas e vivas.  

**OBS:** utilizamos um Drive compartilhado para armazenar as imagens de entrada, assim como os datasets auxiliares que criamos para classificação. Como montamos o Drive no Colab, não será possível rodar os exemplos sem ter acesso a essa pasta. Mesmo assim, os arquivos `stitcher` e `detect_and_classify` já estão com os exemplos pré-executados, então é possível ver a saída produzida (não geramos exemplos para o `dataset_creator` porque ele só criaa o dataset de classificação).

# Imagens
As imagens serão tiradas do dataset providenciado pela Visio e são no formato RGB com resolução 4k.
Elas estão organizadas em grupos de 25, correspondendo ao grid 5x5 dentro da câmara. As regiões comuns podem ter diferentes iluminações e rotações entre as imagens. 
Abaixo estão alguns exemplos de fotos:  

<img src="https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc5de2c11-1cfe-4815-94ee-6da952e64168%2F1Quad.jpg?table=block&id=c6104659-a9c0-4276-b231-b8507549f9bb&spaceId=c532495c-9c00-42ca-93eb-f43de25bdc5b&width=3070&userId=66ae02a8-0b98-4351-8841-80dc04e275f9&cache=v2" width=500>

<img src="https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F65e22a49-952c-4632-a2a2-bb1d47c36759%2F7Quad2.jpg?table=block&id=ca3ba6dc-c752-4b76-bafd-c61f8f7488ff&spaceId=c532495c-9c00-42ca-93eb-f43de25bdc5b&width=3070&userId=66ae02a8-0b98-4351-8841-80dc04e275f9&cache=v2" width=505>

# Metodologia e resultados
## Reconstrução do grid
Para reconstruir a imagem completa tirada pela câmera, é necessário identificar as regiões que se sobrepõem nas imagens de entrada (que estão dispostas logicamente em um grid 5x5)
e aplicar transformações geométricas para que sejam posicionadas corretamente na imagem resultante.

### Identificação do grid
Para atingir esse objetivo, tentamos identificar as extremidades do grid de cada foto (as 4 intersecções de 3 linhas e 3 colunas). Fizemos isso através de um threshholding binário para explicitar as linhas do grid; aplicamos a operação morfológica de dilatação para unir as 3 linhas delimitadoras do grid e depois aplicamos a operação de open para remover as linhas internas do grid. Por fim, usamos a função `skeletonize` da biblioteca imutils para centralizar as linhas externas do grid, aplicamos um floodfill no quadrado central e depois usamos a função `findContours` do OpenCV para encontrar as extremidades do quadrado. Ordenamos as coordenadas das 4 extremidades de forma horária, começando pelo canto superior esquerdo.  
O processo completo está ilustrado abaixo:

<img src="https://media.discordapp.net/attachments/691454909551214624/857343024299769866/pipeline.png?width=1440&height=640">

### Calculando a transformação entre as imagens
Agora com as coordenadas dos quadrados de cada imagem, associamos os pontos encontrados em cada imagem com as imagens adjacentes, estimando a transformação euclidiana de 3 graus de liberdade (rotação e translação) através da função `estimateTransform('euclidean')` da biblioteca `skelarn`. Para cada imagem, computamos a transformação relativa entre ela e as imagens adjacentes à esquerda (left) e acima (up), armazenando cada uma em sua própria matriz.

Para calcular a transformação global de cada imagem na figura resultante, aplicamos o seguinte algoritmo:
```python
img[0, 0].global_t = identity;
for i in [1..n]:
    imgs[i, 0].global_t = imgs[i - 1, 0].global_t * imgs[i, 0].up

for j in [1..m]:
    imgs[0, j].global_t = imgs[0, j - 1].global_t * imgs[0, j].left
  
for i in [1..n]:
    for j in [1..m]:
        global_up = imgs[i - 1, j].global_t * imgs[i, j].up
        global_left = imgs[i, j - 1].global_t * imgs[i, j].left
        imgs[i, j].global_t = t * global_up + (1 - t) * global_left
```
Ou seja, computamos a transformação global à esquerda e acima, e decidimos a transformação global à partir de uma combinação dessas duas. Nossos experimentos indicam que o valor `t = 0.5` é o ideal para essa aplicação.

### Sobreposição das imagens
Calculadas as transformações globais, criamos uma imagem vazia de tamanho `5n x 5m` e usamos a função `warpAffine` do OpenCV com a transformação global de cada imagem.
Embora seja possível perceber algumas descontinuidades nos quadrados mais distantes do grid, estamos satisfeitos com o resultado:

<img src="https://media.discordapp.net/attachments/439158826483056660/863551234127822868/unknown.png?width=740&height=670">
<img src="https://media.discordapp.net/attachments/439158826483056660/863550605091930112/unknown.png?width=740&height=670">


## Identificação das células
Para identificar as células, o seguinte pipeline foi proposto: (1) conversão do espaço de cores de BGR para LAB; (2) soma dos canais LA em um único canal;  (3) aplicação do HoughCircles na imagem resultante e posterior remoção de círculos sobrepostos e demarcação dos mesmos na imagem original. O primeiro passo foi proposto por os canais L e A extraírem características importantes relacionadas à colocação da imagem e às bordas das células. A imagem resultante do passo 2 e as células identificadas nessa imagem são demonstradas nas imagens a seguir.

<img src="https://media.discordapp.net/attachments/691454909551214624/857390542916419634/final.png?width=1440&height=519">

Este modelo se mostrou pouco propício a problemas de superposição de células como pode ser observado a seguir.

<img src="https://media.discordapp.net/attachments/691454909551214624/857390977874264074/super.png">

## Classificação e contagem das células
Para a classificação de leveduras, foi criado um dataset a partir da sequência 1 de imagens disponibilizadas para os times. O software utilizado para criar as anotações das classes foi o CVAT e as células anotadas com bounding boxes foram recortadas e organizadas em dois diretórios: um para as células vivas e outro para as células mortas. A partir deste dataset foi treinado um modelo K-nearest-neighbours. Nele, cada célula é representada por um vetor de features e por um label. As features escolhidas para representarem as células foram as provenientes dos bins dos histogramas de cores das imagens do dataset. O resultado da classificação por este método é demonstrado na imagem a seguir.

<img src="https://cdn.discordapp.com/attachments/691454909551214624/863655390779670558/unknown.png">

Por fim, é durante a classificação das células que a contagem acontece. O número de células vivas e mortas, assim como suas respectivas porcentagens são mostrados ao fim do processamento da imagem de entrada. Um exemplo da saída produzida é ilustrado abaixo:
```cs
Número total de células: 971
Número total de células mortas: 844 (86.92%)
Número total de células vivas: 127 (13.08%)
```



# Outras tentativas ao longo do desenvolvimento

## Reconstrução do grid 
<!-- <details>
  <summary>Representação ilustrativa da sanidade do time durante o processo</summary>
  <img src="https://i.imgur.com/dD3MM1b.gif?noredirect">
</details> -->

Nossa primeira tentativa foi usar template matching para identificar as intersecções entre as linhas delimitadoras do grid. Devido à rotação, ruído e possibilidade de oclusão dessa região, não obtivemos bons resultados.  
Depois disso, tentamos utilizar a classe Stitcher do OpenCV, que tenta buscar as regiões em comum entre as imagens e grudá-las corretamente. Como nossas imagens estão em um formato de grid (e não em uma linha da direita para a esquerda, como é comum em imagens panorâmicas), o Stitcher "out-of-the-box" não obteve um resultado satisfatório; além disso, o algoritmo .  
Descobrimos um programa standalone feito em Java chamado ImageJ que possui um plugin de stitching específico para imagens em grid, e esse funcionou perfeitamente: 

<img src="https://media.discordapp.net/attachments/691454909551214624/856630392588075008/unknown.png?width=691&height=670" width=600>

Estudamos o [paper do algoritmo implementado](https://academic.oup.com/bioinformatics/article/25/11/1463/332497?login=true) por esse programa, (e o [código correspondente](https://github.com/fiji/Stitching)), mas tivemos dificuldades ao tentar a etapa de phase correlation em Python (mesmo usando o OpenCV). Caso o nosso resultado não seja bom o suficiente para a empresa, uma possível solução não-ideal é utilizar o wrapper [PyImageJ](https://github.com/imagej/pyimagej) e gerar a foto final chamando a função do plugin.

Como o stitching 2 a 2 do OpenCV estava funcionando bem, tentamos extrair os parâmetros da transformação relativa estimada pelo OpenCV para cada par de imagens, mas essa funcionalidade só está presente na biblioteca em C++ ([Stitcher.cameras](https://docs.opencv.org/4.5.2/d2/d8d/classcv_1_1Stitcher.html#af9097e2b658dc1d3d57763c3fefd40aa)). Com isso, tentamos replicar o pipeline do OpenCV para podermos obter os parâmetros de transformação: utilizamos o feature detector ORB para extrair os keypoints das duas imagens adjacentes mascarando a região de interesse, depois utilizamos um knnMatcher para associar os keypoints entre as duas imagens. Associando-as, estimávamos a transfomração entre elas com 4 graus de liberdade (translação, rotação e escala). Os resultados não foram bons quando a diferença entre luminosidade era grande entre as imagens, já que o matching de keypoints associava regiões muito diferentes da imagem. Para melhorar essa solução, nós provavelmente iríamos pegar mais keypoints e aplicar um RANSAC (como [nesse artigo](https://towardsdatascience.com/image-panorama-stitching-with-opencv-2402bde6b46c)), mas não optamos por seguir esse caminho.


## Identificação e classificação de células de levedura
Para esta etapa, foram testados diversos modelos de detecção. Todos estes modelos buscavam evidenciar as céluas na imagem por meio da sua filtragem, canais de cores, thresholding, detecção de bordas etc. Os primeiros modelos, descartados ainda no início do projeto, exerciam as funções de identificar as células e classificá-las ao mesmo tempo por meio da segmentação da imagem por cores. Os modelos seguintes focaram na separação entre as etapas de detecção e classificação por meio da identificação de todas as células de forma indiscriminada seguida da classificação das mesmas por meio de modelos de classificação treinados em um dos <em>datasets</em> disponibilizados. 

### Primeiros experimentos
Nestes modelos, as células vivas e mortas eram identificadas por meio de operações nos canais de cor da imagem seguidas de detecção de componentes conectadas para a identificação como pode ser visualizado na imagem a seguir. Na imagem, da esquerda para a direita, são mostradas as máscaras para a identificação das células mortas e vivas, respectivamente. Esta identificação foi realizada por meio da seleção das componentes conectadas que respeitavam um limiar (A) de área associado à área média das células, ou seja, apenas as componentes que possuíssem área maior que (A) foram reconhecidas como células. Este limiar foi definido de forma empírica.

<img src="https://media.discordapp.net/attachments/691454909551214624/857355744215433266/first_method.png?width=710&height=701">

Apesar de identificar células separadas de forma satisfatória, o modelo peca quando as células encontram-se próximas uma das outras, como pode ser percebido a seguir:

<img src="https://media.discordapp.net/attachments/766087042199191562/857360667916369960/erro.png?width=710&height=701">

Isto acontece justamente pela técnica utilizada utilizar apenas informações de cor RGB. Os modelos seguintes, então, buscaram trabalhar com outros tipos de informações tais como textura e bordas, assim como também objetivaram a separação das etapas de detecção e classificação.

### Identificação de leveduras
Os modelos a serem explicados utilizam uma metodologia diferente da anterior: ao invés de identificarem as células de cada classe separadamente, as células são identificadas em conjunto para que sejam submetidas, posteriormene, a um modelo de classificação.

### Primeiro Modelo de Identificação
O primeiro modelo realiza um pré-processamento para a remoção do canal vermelho e para o aumento do contraste da imagem. Em seguida é realizada a filtragem da imagem por 16 filtros de gabor (mostrados abaixo) e soma as melhores ativações é utilizada como máscara.

<img src="https://media.discordapp.net/attachments/766087042199191562/857368984486936576/gabor_results.png?width=1006&height=701">

A máscara criada pela seleção das melhores ativações dos filtros de gabor é processada para eliminar o ruído e em seguida é somada ao threshold aplicado inicialmente. A máscara resultante desta operação é demonstrada a seguir (da esquerda para a direita: imagem inicial pré-processada, ativações de gabor pré-processadas, thresholding, resultado da aplicação da máscara na imagem original). 

<img src="https://media.discordapp.net/attachments/766087042199191562/857370708396474368/bbb.png?width=1920&height=309">

O resultado destas etapas (última imagem mostrada acima) é submetido ao algoritmo de Canny para a deteção de bordas. O resultado do algoritmo é somado ao canal vermelho da imagem original para que as suas bordas sejam ainda mais explicitadas. Esta última etapa é seguida da conversão da imagem resultante para tons de cinza. 

<img src="https://media.discordapp.net/attachments/766087042199191562/857375456176635914/cccccc.png">

Após este pré-processamento, a imagem resultante (última imagem mostrada acima) é submetida ao algoritmo HoughCircles, o qual tem a finalidade de encontrar os padrões circulares das células e retornas as suas respectivas coordenadas. Os círculos encontrados são pré-processados para evitar superposições e a imagem final é demarcada com os círculos. As imagens abaixo apresentam, respectivamente, as demarcações antes e após o pré-processamento dos círculos encontrados.

<img src="https://media.discordapp.net/attachments/691454909551214624/857378260681883678/final.png">

Como pode ser observado na imagem a seguir, este modelo é menos propício a problemas na superposição de células se comparado ao modelo mostrado anteriormente.

<img src="https://media.discordapp.net/attachments/691454909551214624/857381039999811594/trash.png">

O seu maior problema, porém, é o tempo levado para realizar a detecção. Por este motivo, este modelo fora descartado.


Abaixo, é possível comparar os resultados, para a mesma imagem, da detecção de cada um dos modelos criados, sendo o terceiro modelo o utilizado neste projeto e detalhado na seção de metodologia.

<img src="https://media.discordapp.net/attachments/691454909551214624/857395306627203112/comparison.png?width=1440&height=347">

O modelo da primeira imagem tende a identificar conjuntos de células como se fossem uma única e, por este motivo, foi descartado. Já o modelo apresentado na imagem central é o que apresentou o melhor resultado, sendo ele o que detecta corretamente o maior número de células na imagem. Porém este modelo é o mais ineficiente em termos de tempo e custo computacional e por este motivo ele foi descartado. Por fim, o modelo apresentado na última imagem, o qual foi escolhido para ser o modelo utilizado no projeto, tende a ignorar algumas células na borda da imagem, problema que pode ser resolvido, em boa parte, quando as imagens são unidas em grid. 

## Classificação
Três características foram escolhidas para a criação destes vetores: (1) concatenação das ativações de filtros de gabor sobre as imagens de células para a extração de características topológicas, (2) histograma de cor de cada imagem de célula e (3) extração de descritores Haralick. O primeiro vetor de features testado foi criado a partir da concatenação de (1) e (2), o segundo tomando-se apenas (2) e o terceiro concatenando-se (2) e (3). A tabela a seguir mostra os resultados para alguns modelos treinados. A métrica utilizada para comparar os modelos no dataset de teste foi a F1 score para cada classe devido à natureza desbalanceada do dataset.


| Modelo/features | F1 (mortas) | F1 (vivas) |
|-----------------|-------------|------------|
| Knn/(2)         | 0.83        | 0.97       |
| Knn/(1) e (2)   | 0.83        | 0.97       |
| Knn/(2) e (3)   | 0.76        | 0.95       |
| SVM/(2)         | 0.83        | 0.97       |
| SVM/(1) e (2)   | 0.83        | 0.97       |
| SVM/(2) e (3)   | 0.83        | 0.97       |

Como pode ser observado, os classificadores apresentaram resultados semelhantes e isto se deu devido ao tamanho do dataset de testes (41 imagens para teste), o que limita o poder de generalização dos modelos. Sendo assim, por sua simplicidade, o modelo escolhido foi o KNN baseado em features de cor (Knn/(2)). 
