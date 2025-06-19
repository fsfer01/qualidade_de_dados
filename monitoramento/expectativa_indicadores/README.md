## **Informações Gerais**
Esse exemplo de projeto de monitoramento de expectativa de indicadores tem como objetivo fazer de forma automática algumas validações que são feitas manualmente por usuários. Com isso, caso seja encontrado um comportamento fora do comúm, é gerado uma mensagem que pode ser enviada.



https://github.com/user-attachments/assets/c128ccea-4568-474a-b75b-278e1137cd6f



## **Objetivo:**
O principal objetivo é **identificar antecipadamente qualquer comportamento fora do padrão**, e com isso, realizarmos uma análise e caso tenha um erro, abrirmos um HELP ou comunicação para o time responsável.
![image](https://github.com/fsfer01/qualidade_de_dados/assets/78058494/1a694a7e-63f2-4625-8fa9-8ca2adba4ad8)



## **Funcionamento:**
1. Cria uma coluna com o nome dos dias da semana em português.
    ``` python
   # Convertendo a coluna 'dia' para datetime
   df['dia'] = pd.to_datetime(df['dia']) 

    # Extrair o nome do dia da semana em português
    df['dia_semana'] = df['dia'].dt.strftime('%A').map({
        'Monday': 'Segunda-feira',
        'Tuesday': 'Terça-feira',
        'Wednesday': 'Quarta-feira',
        'Thursday': 'Quinta-feira',
        'Friday': 'Sexta-feira',
        'Saturday': 'Sábado',
        'Sunday': 'Domingo'
    })
   
   ```
3. Calcula a média das colunas numéricas.
     ``` python
   media = df.select_dtypes(include=['int64', 'float64']).mean()  # Calcule a média de cada coluna int ou float.  
   ```
5. Carrega regras personalizadas de um arquivo YAML.
     ``` python
   with open(os.path.join(Path(__file__).parent.resolve(), 'utils','yaml', 'expectativa_indicadores.yaml'), 'r') as file:
     regras_personalizadas = yaml.safe_load(file)  
   ```
    ![image](https://github.com/fsfer01/qualidade_de_dados/assets/78058494/eb402e92-7e88-40f1-b0c0-8de5e6ae3299)


   
7. Verifica se os valores das colunas numéricas estão dentro dos limites definidos:
   
    A. **Geral**: Compara o valor atual com a média geral da coluna.
   
    B. **Por dia da semana**: Compara o valor atual com a média dos valores acumulados para aquele dia específico da semana (por exemplo, todas as segundas-feiras).
   ```python
   # Percorrer as colunas e os valores
    for col in df.columns:
        if df[col].dtype in ['int64', 'float64']:
            valores_coluna = df[col]
            media_coluna = media[col]
            
            # Verificar se a coluna tem uma regra personalizada definida
            if col in regras_personalizadas['colunas']:
                limite_inferior = regras_personalizadas['colunas'][col].get("limite_min", None)
                limite_superior = regras_personalizadas['colunas'][col].get("limite_max", None)
                media_dia_semana = regras_personalizadas['colunas'][col].get("media_dia_semana", False)
            
            else:
                # Se não houver regra personalizada, use os valores padrão
                limite_inferior = None
                limite_superior = None
                media_dia_semana = False
    
            # Lista para armazenar os alerts para esta coluna
            col_alerts = []
    
            # Verifique se cada valor está acima de limite_superior ou abaixo de limite_inferior da média.
            for indice, valor in enumerate(valores_coluna):
                if media_dia_semana:
                    # Calcular a média dos valores para o dia da semana atual
                    dia_semana_atual = df["dia_semana"].iloc[indice]
                    media_dia_semana_atual = df[df['dia_semana'] == dia_semana_atual][col].mean()
                    
                    if valor > media_dia_semana_atual * limite_superior or valor < media_dia_semana_atual * limite_inferior:
                        diferenca_percentual = ((valor - media_dia_semana_atual) / media_dia_semana_atual) * 100
                        alert_message = f'> • Em {df["dia"].iloc[indice].strftime("%Y-%m-%d")}, o valor da coluna {col} está {diferenca_percentual:.2f}% ({round(valor,2)}) comparando com a média de todas às {dia_semana_atual} ({round(media_dia_semana_atual,2)})'
                        col_alerts.append(alert_message)
                else:
                    if limite_inferior is not None and limite_superior is not None:
                        if valor > media_coluna * limite_superior or valor < media_coluna * limite_inferior:
                            diferenca_percentual = ((valor - media_coluna) / media_coluna) * 100
                            alert_message = f'> • Em {df["dia"].iloc[indice].strftime("%Y-%m-%d")}, o valor da coluna {col} está {diferenca_percentual:.2f}% ({round(valor,2)}) comparando com a média acumulada ({round(media_coluna, 2)})'
                            col_alerts.append(alert_message)
                    else:
                        # Se não houver limite definido, não faz a comparação
                        pass

   ```



9. Gera alertas se os valores estiverem fora dos limites definidos.
    ```python
    # Se houver alerts para esta coluna, adicione o cabeçalho e os alerts à lista principal de alerts
            if col_alerts:
                alerts.append(f'\n> Valores para {col}:')
                alerts.extend(col_alerts)
    ```
11. Se não houver alertas, retorna False.
    ```python
        if not alerts:
        # Se não houver alerts, retorna False para que o ShortCircuitOperator interrompa a execução.
        return False
    ```
## **Observações**:




