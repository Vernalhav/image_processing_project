# Desafio Visio  
Marcelo Isaias de Moraes Junior  
João Pedro Almeida Santos Secundino  
João Vitor Silva Ramos  
Victor Giovannoni Vernalha  

# Objetivo  
O objetivo desse projeto é unir as fotos tiradas de uma câmara de Neubauer contendo leveduras vivas e mortas e realizar a contagem de células em ambas as categorias.
A especificação completa do desafio está disponível [aqui](https://www.notion.so/Proposta-de-Projeto-336e8afb603447109116a61d147c0e09).

# Imagens
As imagens serão tiradas do dataset providenciado pela Visio e são no formato RGB com resolução 4k.
Elas estão organizadas em grupos de 25, correspondendo ao grid 5x5 dentro da câmara. As regiões comuns podem ter diferentes iluminações e rotações entre as imagens. 
Abaixo estão alguns exemplos de fotos:  

<img src="https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc5de2c11-1cfe-4815-94ee-6da952e64168%2F1Quad.jpg?table=block&id=c6104659-a9c0-4276-b231-b8507549f9bb&spaceId=c532495c-9c00-42ca-93eb-f43de25bdc5b&width=3070&userId=66ae02a8-0b98-4351-8841-80dc04e275f9&cache=v2" width=500>

<img src="https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F65e22a49-952c-4632-a2a2-bb1d47c36759%2F7Quad2.jpg?table=block&id=ca3ba6dc-c752-4b76-bafd-c61f8f7488ff&spaceId=c532495c-9c00-42ca-93eb-f43de25bdc5b&width=3070&userId=66ae02a8-0b98-4351-8841-80dc04e275f9&cache=v2" width=505>

# Reconstrução do grid 
<details>
  <summary>Representação ilustrativa da sanidade do time durante o processo</summary>
  <img src="https://i.imgur.com/dD3MM1b.gif?noredirect">
</details>

Para reconstruir a imagem completa tirada pela câmera, é necessário identificar as regiões que se sobrepõem nas imagens de entrada (que estão dispostas logicamente em um grid 5x5)
e aplicar transformações geométricas que sejam posicionadas corretamente.  
Nossa primeira tentativa foi usar template matching para identificar as intersecções entre as linhas delimitadoras do grid. Devido à rotação, ruído e possibilidade de oclusão dessa região, não obtivemos bons resultados.  
Depois disso, tentamos utilizar a classe Stitcher do OpenCV, que tenta buscar as regiões em comum entre as imagens e grudá-las corretamente. Como nossas imagens estão em um formato de grid (e não em uma linha da direita para a esquerda, como é comum em imagens panorâmicas), o Stitcher "out-of-the-box" não obteve um resultado satisfatório, mas percebemos que quando utilizávamos apenas duas imagens adjacentes horizontalmente, o resultado era bom.  
Descobrimos um programa standalone feito em Java chamado ImageJ que possui um plugin de stitching específico para imagens em grid, e esse funcionou perfeitamente: 

<img src="https://media.discordapp.net/attachments/691454909551214624/856630392588075008/unknown.png?width=691&height=670" width=600>

Estudamos o [paper do algoritmo implementado](https://academic.oup.com/bioinformatics/article/25/11/1463/332497?login=true) por esse programa, (e o [código correspondente](https://github.com/fiji/Stitching)), mas tivemos dificuldades ao tentar implementar a primeira etapa de phase correlation em Python (mesmo usando o OpenCV). Uma solução temporária não-ideal é utilizar o wrapper [PyImageJ](https://github.com/imagej/pyimagej), já que não roda corretamente no Google Colab e necessita da JVM.  

Como o stitching 2 a 2 do OpenCV estava funcionando bem, tentamos extrair os parâmetros da transformação que foi calculada durante o pipeline, mas também tivemos dificuldades ao implementá-lo. Utilizamos o feature detector ORB para extrair os keypoints das duas imagens adjacentes mascarando a região de interesse, depois utilizamos um knnMatcher para associar os keypoints entre as duas imagens. Associando-as, estimávamos a transfomração entre elas com 4 graus de liberdade (translação, rotação e escala). Os resultados não foram bons quando a diferença entre luminosidade era grande entre as imagens.

Por fim, tentamos identificar as extremidades do grid de cada foto (o que tem as intersecções de 3 linhas e 3 colunas). Fizemos isso através de um threshholding binário para explicitar as linhas do grid, aplicamos a operação morfológica de dilatação para unir as 3 linhas delimitadoras do grid e depois aplicamos a operação de open para removeras linhas internas do grid. Por fim, usamos a função `skeletonize` da biblioteca imutils, aplicamos um floodfill no quadrado central e usamos a função `findContours` do OpenCV para encontrar as extremidades do quadrado.

<img src="https://media.discordapp.net/attachments/691454909551214624/857343024299769866/pipeline.png?width=1440&height=640">

Por fim, associamos os pontos encontrados em cada imagem com as imagens adjacentes (por exemplo, associando o ponto superior direito da imagem da esquerda com o ponto superior esquerdo da imagem da direita) através de uma transformação euclidiana (que só utiliza rotação e translação). Para descobrir as transformações finais de cada imagem, compusemos as transformações locais anteriores.
Com isso, obtivemos um resultado satisfatório:

<img src="https://media.discordapp.net/attachments/691454909551214624/857340404545749033/unknown.png?width=740&height=670">


# Identificação das leveduras


# Classificação das leveduras

# Contagem das leveduras
