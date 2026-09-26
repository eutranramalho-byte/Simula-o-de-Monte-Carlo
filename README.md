# Metodologia para Análise de Impactos da Demanda de Eletropostos na Rede de Distribuição

Este repositório contém os códigos utilizados para reproduzir os resultados do artigo, organizados em três etapas: (1) conversão da rede elétrica real (BDGD/ANEEL) para o formato OpenDSS, (2) simulação de mobilidade urbana no Eclipse SUMO, e (3) simulação de Monte Carlo da demanda de recarga integrada ao OpenDSS.

## Requisitos gerais

- MATLAB (com o *Statistics and Machine Learning Toolbox*, usado para `tinv`, `prctile` e similares)
- [OpenDSS](https://sourceforge.net/projects/electricdss/) instalado, com a interface COM registrada
- [Eclipse SUMO](https://sumo.dlr.de/docs/index.html), incluindo a ferramenta `activitygen` e a interface `TraCI`
- Python (para o controle da simulação via TraCI)

---

## 1. `Codigo_BDGD2OPENDSS`

Converte os dados de rede elétrica extraídos da Base de Dados Geográficos da Distribuidora (BDGD/ANEEL) para o formato de scripts do OpenDSS (`.dss`).

### Como usar

1. Abra o script no MATLAB.
2. Defina apenas dois diretórios no início do código:
   - **Diretório de entrada**: pasta contendo as planilhas exportadas da BDGD (já filtradas para o alimentador de interesse).
   - **Diretório de saída**: pasta onde os arquivos `.dss` gerados serão salvos.
3. Execute o script. Os arquivos `.dss` do alimentador (linhas, transformadores, cargas, etc.) serão gerados automaticamente no diretório de saída, prontos para uso no OpenDSS.

---

## 2. Simulação de Mobilidade Urbana (Eclipse SUMO)

Gera as rotas de tráfego utilizadas para estimar a distância média percorrida pelos veículos elétricos de uso particular.

### Passo a passo

1. **Gere a demanda de tráfego com o `activitygen`**, a partir da descrição da população da região (número de habitantes, domicílios, distribuição etária, etc.), conforme os parâmetros de entrada do SUMO.
2. **Adicione os arquivos gerados ao mapa da cidade**: incorpore o arquivo de rotas/demanda produzido pelo `activitygen` ao arquivo do mapa da rede viária (obtido via OSM Web Wizard), de forma que o SUMO reconheça a demanda dentro da malha importada.
3. **Configure os diretórios no código de controle** (Python + TraCI):
   - Caminho do arquivo de configuração do SUMO (`.sumocfg`) com o mapa e a demanda já integrados.
   - Diretório de saída para os resultados (distâncias percorridas por veículo).
4. **Execute a simulação**: o script controla o SUMO via `TraCI`, executa a simulação de tráfego e calcula estatisticamente a distância média percorrida (e desvio-padrão) para uso posterior na estimativa do SoC inicial dos VEs.

> **Observação:** a distância média dos veículos de uso público (táxis) não é obtida pelo SUMO; ela é definida a partir de pesquisa de campo com motoristas locais e inserida diretamente como parâmetro na simulação de Monte Carlo (etapa 3).

---

## 3. Simulação de Monte Carlo (MATLAB + OpenDSS)

Realiza a simulação estocástica da demanda de recarga (EL, ES e ER), o dimensionamento iterativo de eletropostos rápidos por taxa de ocupação, e o cálculo do fluxo de potência em série temporal no OpenDSS.

### Configuração inicial

No início do script, defina:

| Variável | Descrição |
|---|---|
| `Ali` | Identificador do alimentador (usado para nomear pastas e arquivos de saída) |
| Diretório do alimentador OpenDSS | Caminho para o script mestre `.dss` gerado na etapa 1 |
| Diretório de saída (`pastaDestino`) | Pasta onde os resultados de cada cenário serão salvos |
| `caso` | Vetor com os cenários a simular (cada valor corresponde a uma pasta de saída própria, `caso<N>`) |
| `max_cenario` | Número máximo de iterações de Monte Carlo por cenário |
| Tolerância de convergência | Critério de parada: diferença entre a média móvel da tensão mínima diária da iteração atual e da anterior |
| Quantidade inicial de ERs | Número preliminar de eletropostos rápidos, ajustado iterativamente pelo algoritmo conforme a taxa de ocupação (30–70%) |

### Dados de entrada necessários

- Tabela de modelos de veículos elétricos (capacidade da bateria, potência máxima de recarga, autonomia) e sua representatividade no mercado.
- Tabela de modelos de eletropostos (EL, ES, ER), com potência e número de plugues.
- Distância média e desvio-padrão percorridos por veículos privados (SUMO, etapa 2) e por táxis (pesquisa de campo).
- Parâmetros das distribuições gaussianas de horário de início de recarga (EL, ES, ER).

### Execução

1. Certifique-se de que o OpenDSS está instalado e a interface COM acessível pelo MATLAB.
2. Ajuste os parâmetros da tabela acima no início do script.
3. Execute o script. Para cada cenário em `caso`, a simulação:
   - Sorteia os parâmetros estocásticos (horário de início, SoC inicial, duração de recarga) a cada iteração;
   - Calcula o fluxo de potência em 24h no OpenDSS;
   - Ajusta o número de ERs conforme a taxa de ocupação obtida;
   - Repete até atingir a convergência ou o número máximo de iterações.

### Saídas geradas

Para cada cenário, os resultados são salvos em `caso<N>/Dados_Alimentador<Ali>/`, incluindo (entre outros) `Tensao_minima_diaria_BT.mat`, `Tensao_minima_diaria_MT.mat`, `Demanda_Diario_Trafo_final.mat`, `P_Perdas_Final.mat`, `V_pior_caso.mat` e `V_melhor_caso.mat`. Esses arquivos são consumidos pelo script de análise de resultados (não incluído nesta etapa) para gerar os gráficos e tabelas do artigo.
