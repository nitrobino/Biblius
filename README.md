# Biblius

```python
import pandas as pd
import numpy 
from fuzzywuzzy import process

def data_show(DF,score_sort=True):
    '''
    Aplicar à dataframe para cada função que retorne uma dataframe. Isto vai eliminar do dataframe retornado os parâmetros
    contidos na lista Dropar.
    
    Para além disto tivémos que ter o cuidado de olhar para esta linha de código na busca_geral -> Lista.append((df[o1].str.contains(t1))&(df[o2].str.contains(t2)))
    Podíamos ter o caso do livro X escrito pelo Sakurai fosse aprovado e o livro Y do mesmo autor rejeitado. Quando corremos a linha mencionada, se X e Y forem
    suficientemente semelhantes entre si, pode ser devolvido o livro Y mesmo que este não tenha sido aprovado. Para isso é que estamos a utilizar
    o index_linhas. Vamos ver onde estão os livros que entraram para a dtaframe com score 0 e removemos esses livros do resultado da pesquisa.
    '''
#     Dropar = ['Score','ISBN10','ISBN13','Editora','Edicao']
    Dropar = ['ISBN10','ISBN13','Editora','Edicao','Score']
    if score_sort is True:
        Resultado = DF.sort_values(by=['Score'],ascending=False)
        index_linhas = Resultado[ Resultado['Score'] == 0 ].index
        Resultado.drop(index_linhas , inplace=True)
        Resultado = Resultado.drop(columns=Dropar)
    else:
        Resultado = DF.copy()
        Resultado = Resultado.drop(columns=Dropar)
    return Resultado

def busca_geral(pesquisa,mscore=75):
    '''
    Função utilizada no processamento de strings da pesquisa rápida.
    
    Argumentos
    pesquisa: string introduzida pelo utilizador.
    mscore: score mínimo de correspondência entre strings para um dado livro ser aprovado como resultado da pesquisa.
    
    A variável result contém dois túpulos do tipo:
    ( String na base de dados , score , Coluna correspondente na base de dados ) , um para Título e outro para Autores
    
    Se os parâmetros de pesquisa não forem aprovados, ou seja, se nenhum tiver um score satisfatório, podemos abandonar
    a pesquisa. A última linha retorna um dataframe vazio, porque iremos-nos certificar que a string 'por2357' não entra
    na nossa dataframe.
    
    Returns: Uma série do pandas com o valor lógico True/False de cada uma das linhas da data frame.
    Adicionalmente, altera todos os scores do catálogo para zero e reatribui scores aos livros quecumprem ascondições desejadas.
    '''
    df = pd.read_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo')
    dft = df.transpose()
    Lista = [] #onde iremos guardar as series de True/False
    
    # Comparar a pesquisa com as entradas Titulo e Autores dum livro
    for i in range(len(df)):
        df.at[i,'Score'] = 0
        result = process.extractBests(pesquisa,dft[i][0:2])
        t1 , s1 , o1 = result[0][0] , result[0][1] , result[0][2]
        t2 , s2 , o2 = result[1][0] , result[1][1] , result[1][2]
#         print(t1 , s1 , o1)
#         print(t2 , s2 , o2)
        # Aprovar ou rejeitar de acordo com o score
        if (s2>=mscore) or (s1>=mscore):
            df.at[i,'Score'] = s1+s2
            Lista.append((df[o1].str.contains(t1))&(df[o2].str.contains(t2)))
    # Juntar os resultados aprovados
    if len(Lista)>=1:
        for i in range(len(Lista)-1):
            Lista[i+1] |= Lista[i]
        final_res = data_show(df.loc[Lista[-1]]).values.astype(str)
        return final_res.tolist()
    else:
        final_res = data_show(df.loc[df['Edicao'].astype(str).str.contains('por2357')]).values.astype(str)
        return final_res.tolist()

def busca_geral_short(DF,pesquisa,coluna,mscore=80,renovar_scores=0):
    '''
    Função muito semelhante a busca_geral. Utilizada para a pesquisa restrita.
    Não produz efeitos nos scores (ainda).
    Se renovar_scores=0 (default)os scores serão renovados. Se for 1, os scores serão somados aos anteriores. 
    '''
    # Especificar as colunas para process.extractBests
    column_index = DF.columns.get_loc(coluna)
        
    # Queremos uma sub-dataframe da original mas indexada de 0 ao tamanho da sub-dataframe
    DF.index = pd.RangeIndex(len(DF.index))
    
    DF.index = range(len(DF.index))
    DFT = DF.transpose()
    Lista = [] #onde iremos guardar as series de True/False
    
    # Proceder apenas se o parâmetro de pesquisa não for vazio
    if pesquisa=='':
        return DF[coluna].str.contains('')

    # Comparar a pesquisa com as entradas Titulo e Autores dum livro
    for i in range(len(DF)):
        DF.at[i,'Score'] *= renovar_scores
        result = process.extractBests(pesquisa,DFT[i][column_index:column_index+1])
        t1 , s1 , o1 = result[0][0] , result[0][1] , result[0][2]
        # Aprovar ou rejeitar de acordo com o score
        if s1>=mscore:
            DF.at[i,'Score'] += s1
            Lista.append(DF[o1].str.contains(t1))
    
    # Juntar os resultados aprovados
    if len(Lista)>=1:
        for i in range(len(Lista)-1):
            Lista[i+1] |= Lista[i]
        return Lista[-1]
    else:
        return DF[coluna].str.contains('por2357')


def busca_restrita(titulo='',autores='', edicao='',ano='',editora='',tipo='',isbn10='', isbn13='',mscore=80):
    '''
    O utilizador decide que parâmetros quer pesquisar. Os parâmetros introduzidos serão pesquisados
    na base de dados pela ordem: Edicao, Ano, Editora, Autores, Titulo.
    
    Os últimos três utilizam o fuzzywuzzy para um score de aceitação mscore (=85 por default).
    '''
    df = pd.read_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo')
    if (isbn10 != '') or (isbn13 !=''):
        Lista = (df['ISBN13'].astype(str).str.contains(isbn13))&(df['ISBN10'].astype(str).str.contains(isbn10))
        final_res = data_show(df[Lista]).values.astype(str)
        return final_res.tolist()
    else:
        df_Ed = df.copy()
        df_Ed = df.loc[df['Tipo'].str.contains(tipo)] if tipo!='' else df_Ed
        df_Ed = df_Ed.loc[df_Ed['Edicao'].astype(str).str.contains(edicao)] if edicao!='' else df_Ed
        df_Ed = df_Ed.loc[df_Ed['Ano'].astype(str).str.contains(ano)] if ano!='' else df_Ed
        df_Ed , rscore1 = [df_Ed.loc[(busca_geral_short(df_Ed,editora,'Editora',mscore,renovar_scores=0))] , 1 ]  if editora!='' else [ df_Ed , 0]
        df_Ed , rscore2 = [df_Ed.loc[(busca_geral_short(df_Ed,autores,'Autores',mscore,rscore1))] , 1 ] if autores!='' else [df_Ed,0]
        rscore = 1 if (rscore1+rscore2)!=0 else 0
        df_Ed = df_Ed.loc[(busca_geral_short(df_Ed,titulo,'Titulo',mscore,rscore))] if titulo!='' else df_Ed
        final_res = data_show(df_Ed,score_sort=True).values.astype(str)
        return final_res.tolist()

def add_book(cota,quantidade,quantidade1,boolada,categorias=[]):
    Catalogo = pd.read_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo_cotas.csv')
    '''
    Adiciona um livro se a cota já se encontra na base de dados e cria caso contrário.
    '''
    COTAs = Catalogo.loc[Catalogo['Cota']==cota]
    if boolada == 1:
        # Livro já existente
        if COTAs.shape[0]==1:
            new_Catalogo = Catalogo.copy()
            indice = COTAs.index[0]
            Existentes = new_Catalogo.at[indice,'Existentes']
            Disponiveis = new_Catalogo.at[indice,'Disponiveis']
            new_Catalogo.at[indice,'Existentes'] = Existentes+quantidade
            new_Catalogo.at[indice,'Disponiveis'] = Disponiveis+quantidade
            new_Catalogo.to_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo_cotas.csv', index = False)
            return ''
        # Livro não existente
        else:
            Titulo,Autores,Edicao,Ano_Edicao,Editora,ISBN10,ISBN13 = categorias
            linha = {
                'Titulo':Titulo,
                'Autores':Autores,
                'Edicao':Edicao,
                'Ano':Ano_Edicao,
                'Editora':Editora,
                'Existentes':quantidade,
                'Disponiveis':quantidade,
                'ISBN10':ISBN10,
                'ISBN13':ISBN13,
                'Cota':Catalogo.shape[0]+1,
                'Score':0
            }
            new_Catalogo = Catalogo.append(linha,ignore_index=True)
            new_Catalogo.to_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo_cotas.csv', index = False)
            return ''
    else:
        new_Catalogo = Catalogo.copy()
        indice = COTAs.index[0]
        Existentes = new_Catalogo.at[indice,'Existentes']
        Disponiveis = new_Catalogo.at[indice,'Disponiveis']
        if Existentes < Disponiveis + quantidade1:
            return 'Maximo de Disponiveis Atingido!'
        elif 0 > Disponiveis + quantidade1:
            return 'Minimo de Disponiveis Atingido!'
        else:
            new_Catalogo.at[indice,'Existentes'] = Existentes
            new_Catalogo.at[indice,'Disponiveis'] = Disponiveis+quantidade1
            new_Catalogo.to_csv('/Users/paularibeiro/Desktop/3º ano/LAC/catalogo_cotas.csv', index = False)
            return ''

```
