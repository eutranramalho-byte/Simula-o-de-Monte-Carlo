# A Methodology for Analyzing the Impacts of Charging Station Demand on the Distribution Network

## Autores:
- Eutran de Jesus Ramalho
- Juan Carlos Galvis Manso
## Afiliação:
Universidade Federal de Ouro Preto

Este repositório contém os códigos utilizados para reproduzir os resultados do artigo *"A Methodology for Analyzing the Impacts of Charging Station Demand on the Distribution Network"* (E. J. Ramalho e J. C. Galvis Manso, UFOP). O fluxo de trabalho combina: (0) projeção da demanda e da penetração de VEs, (1) conversão da rede real (BDGD/ANEEL) para OpenDSS, (2) simulação de mobilidade urbana no Eclipse SUMO, (3) simulação de Monte Carlo da demanda de recarga integrada ao OpenDSS e (4) análise dos resultados.



## Visão geral do fluxo

```
Dados_Historicos.xlsx ──► [0] projecao_ajustes.m ──► anos dos cenários + fator de aumento de demanda
                                                              │
Planilhas BDGD (xlsx) ──► [1] Codigo_BDGD2OPENDSS.m ◄─────────┘  (fator_aumento)
                                   │  gera MainCode.dss + demais .dss
                                   ▼
Mapa OSM + activitygen ─► [2] calcularDistanciasPrivado.py (SUMO/TraCI)
                                   │  distâncias dos VEs privados
                                   ▼
                      [3] Simulacao_Monte_Carlo.m (MATLAB + OpenDSS via COM)
                                   │  arquivos .mat / .xlsx por caso
                                   ▼
        [4] Analizar_Resultados.m · Analise_Extremos.m · analise_ER.m · Analise_Sensibilidade.m
```

## Estrutura do repositório

| Arquivo | Etapa | Função |
|---|---|---|
| `projecao_ajustes.m` | 0 | Projeta a demanda do alimentador e a taxa de VEs na frota; obtém os anos de cada cenário e o fator de aumento de demanda |
| `Codigo_BDGD2OPENDSS.m` | 1 | Converte as planilhas da BDGD em scripts `.dss` |
| `calcularDistanciasPrivado.py` | 2 | Controla o SUMO via TraCI e calcula a distância diária percorrida pelos veículos privados |
| `Simulacao_Monte_Carlo.m` | 3 | Monte Carlo da demanda de recarga + dimensionamento iterativo de ERs + fluxo de potência 24 h no OpenDSS |
| `Analizar_Resultados.m` | 4 | Tensão mínima, carregamento de transformadores e perdas por cenário |
| `Analise_Extremos.m` | 4 | Pior e melhor casos de tensão mínima |
| `analise_ER.m` | 4 | Quantidade e taxa de ocupação dos eletropostos rápidos |
| `Analise_Sensibilidade.m` | 4 | Análise de sensibilidade (horário de recarga lenta e migração EL → ER) |

## Requisitos gerais

- **MATLAB** com *Statistics and Machine Learning Toolbox* (`tinv`, `prctile` etc.) e *Curve Fitting Toolbox* (usado na etapa 0 para ajuste de curvas)
- **OpenDSS** instalado, com a interface COM registrada (`OpenDSSEngine.DSS`). Como o controle é feito por `actxserver`, a simulação deve ser executada em **Windows**
- **Eclipse SUMO**, incluindo a ferramenta `activitygen` e a interface `TraCI`
- **Python 3** com as bibliotecas `pandas` e `traci` (`pip install pandas traci`; o pacote `traci` também acompanha a pasta `tools` do SUMO)

> **Atenção aos caminhos:** os scripts originais contêm caminhos absolutos do computador do autor (por exemplo `D:\1_UFOP\...` e `C:\Users\...`). Antes de executar, substitua-os pelos diretórios da sua máquina. Os pontos a editar estão indicados em cada etapa abaixo.

---

## Mapeamento entre casos do código e cenários do artigo

No código, os cenários são identificados pela variável `caso` (cada valor gera uma pasta de saída própria, `caso<N>`):

| `caso` | Descrição | Penetração de VEs |
|---|---|---|
| 1 | Cenário 1 – caso base (somente cargas existentes) | 0% |
| 2 | Cenário 2 – inserção inicial (2027) | 10% |
| 3 | Cenário 3 – expansão (2054) | 50% |
| 4 | Cenário 4 – saturação (2088) | 100% |
| 5 | Cenário 5 – Cenário 4 com início da recarga lenta às 23 h | 100% |
| 6 a 10 | Sensibilidade: variação do horário médio de recarga em ELs (a partir do Cenário 4) | 100% |
| 11 a 14 | Sensibilidade: migração de VEs de EL para ER, com ES fixo em 18% (EL/ER = 65/17, 60/22, 55/27 e 50/32) | 100% |

A faixa de taxa de ocupação dos ERs (30% a 70%) é definida em `taxa_aco_min` e `taxa_aco_max`, no início de `Simulacao_Monte_Carlo.m`.

---

## 0. `projecao_ajustes.m` — Projeção de demanda e de penetração de VEs

Realiza duas projeções até o ano `ano_max` (padrão: 2100) e as cruza para definir os cenários:

1. **Crescimento natural da demanda do alimentador**: ajuste de curva sobre o histórico anual de demanda (coluna `JMLT308` do arquivo `Dados_Historicos.xlsx`).
2. **Participação de VEs na frota brasileira**: ajuste de curva sobre a taxa histórica de VEs (2014–2023, em % da frota total) e projeção da frota de automóveis da região (2013–2025).

A partir da taxa projetada, o script encontra o ano em que cada taxa de penetração-alvo ocorre (vetor `Taxas = [10; 50; 100]`, correspondentes aos Cenários 2, 3 e 4). Cruzando esses anos com a projeção de demanda, obtém-se o **fator de aumento de demanda** de cada cenário (`Aumento_Demanda_cenario`, relativo à demanda do ano-base). Esse fator deve ser informado na variável `fator_aumento` do script da etapa 1, que o aplica às cargas já existentes do alimentador.

### Como usar

1. Coloque o arquivo `Dados_Historicos.xlsx` (colunas `ANO` e a coluna com o nome do alimentador) no caminho lido no início do script e ajuste-o.
2. Verifique que as funções `createFit.m` e `createFit4.m` (geradas pelo *Curve Fitting Toolbox*, com `cftool`) estão na mesma pasta ou no *path* do MATLAB.
3. Execute o script. A tabela `Resultados` exibida no *Command Window* contém, por cenário: taxa de penetração, ano de ocorrência, fator de aumento de demanda, demanda projetada e número de automóveis na região.
4. O script também exporta as figuras em PDF para a pasta `Graficos_TCC_PDF_Final_2`. Para salvar a tabela em planilha, descomente a linha `writetable(Resultados, ...)`.

> Os valores do artigo para esta etapa são: Cenário 2 → 2027 (10%), Cenário 3 → 2054 (50%) e Cenário 4 → 2088 (100%).

---

## 1. `Codigo_BDGD2OPENDSS.m` — Conversão BDGD → OpenDSS

Converte os dados de rede elétrica extraídos da Base de Dados Geográficos da Distribuidora (BDGD/ANEEL) para scripts do OpenDSS (`.dss`). A conversão é baseada em H. da Silva (2022), referência [22] do artigo.

### Dados de entrada

Planilhas da BDGD exportadas (via QGIS) para `.xlsx`, **na pasta de trabalho do MATLAB** (ou no *path*): `CTMT`, `SSDMT`, `SSDBT`, `SEGCON`, `EQTRMT`, `UNTRMT`, `UCBT`, `UCMT`, `PIP`, `UNREMT` e `EQRE`. Os arquivos devem conter a rede completa da distribuidora, pois o script filtra o alimentador pelo código informado em `Ali`.

Também é lido o arquivo `perfil_agregado_carga_lenta.csv` (coluna `Potencia_Agregada_kW`, 24 valores horários), que fornece a curva de carga agregada das recargas lentas e é usada como *loadshape* dos ELs no cenário em questão. Ele é procurado dentro de `pasta_simulacao`.

### Parâmetros a configurar (início do script)

| Variável | Descrição |
|---|---|
| `Ali` | Código do alimentador (campo `COD_ID` da camada CTMT). Ex.: `'JMLT308'` |
| `freq` | Frequência do sistema (60 Hz) |
| `mvasc3`, `mvasc1` | Potência de curto-circuito trifásica e monofásica da subestação (MVA) |
| `Dia_analise` | Tipo de dia analisado: `'UT'` (dias úteis), `'SAB'` ou `'DOM'`. O artigo usa `'UT'` |
| `fator_aumento` | Fator multiplicativo aplicado às cargas existentes, obtido na etapa 0 para o ano do cenário (`1` = demanda atual) |
| `numero_caso` | Caso/cenário (ver tabela acima). Define a taxa de penetração (`PROB_EL`) usada para sortear os ELs |
| `pasta_destino` | Diretório de saída dos arquivos `.dss` |
| `pasta_simulacao` | Pasta que contém `perfil_agregado_carga_lenta.csv` do caso |

### Execução e saída

Execute o script. Serão gerados no diretório de saída os arquivos `.dss` do alimentador (linhas, *linecodes*, transformadores, cargas, *loadshapes*, coordenadas das barras etc.) e o arquivo mestre **`MainCode.dss`**, que referencia todos os demais por `Redirect`. Esse é o arquivo compilado pelo OpenDSS na etapa 3.

> Para o Monte Carlo, o diretório de saída deve seguir a estrutura esperada pela etapa 3: `Resultados\caso<N>\Alimentador<Ali>\MainCode.dss`.
> O script define a taxa `PROB_EL` apenas para os casos 2, 3 e 4; para o caso base (1), use `PROB_EL = 0`.

---

## 2. Simulação de Mobilidade Urbana (Eclipse SUMO) — `calcularDistanciasPrivado.py`

Gera as rotas de tráfego utilizadas para estimar a distância média diária percorrida pelos veículos elétricos de uso particular.

### Passo a passo

1. **Importe o mapa da cidade** com o *OSM Web Wizard* do SUMO, obtendo a malha viária da região de estudo.
2. **Gere a demanda de tráfego com o `activitygen`**, a partir da descrição da população (número de habitantes, domicílios, distribuição etária, taxa de motorização, empregos etc.). No estudo de caso: 83.791 habitantes (IBGE) e 25.240 domicílios.
3. **Integre a demanda ao mapa**: adicione o arquivo de rotas gerado pelo `activitygen` ao arquivo de configuração do mapa, resultando em um `.sumocfg` com rede e demanda.
4. **Configure o script Python** (início de `calcularDistanciasPrivado.py`):
   - `SUMO_HOME`: diretório de instalação do SUMO;
   - `sumo_config_file`: caminho do `.sumocfg` do passo 3;
   - `SIMULATION_DURATION_SEC`: duração da simulação em segundos (padrão: 100000 s, o que cobre o dia completo de viagens).
5. **Execute** `python calcularDistanciasPrivado.py`. O script abre o `sumo-gui` via `TraCI`, acompanha cada veículo do tipo `passenger` e, ao final de cada viagem, soma a distância percorrida ao identificador-base do veículo (as viagens de um mesmo indivíduo são agregadas).

### Saída

O arquivo `resultados_veiculos.csv`, com as colunas `ID_Veiculo`, `Distancia_Total_km` e `Tempo_Final_s`. A partir dele, calcule a **média e o desvio-padrão** da distância diária. No artigo: média de **14,85 km** e desvio-padrão de **7,20 km**.

Na etapa 3, o Monte Carlo lê as distâncias em uma planilha `Resultados_privado.xlsx`; portanto, converta/salve o conteúdo de `resultados_veiculos.csv` nesse formato.

> **Observação:** a distância dos veículos de uso público (táxis) **não** é obtida pelo SUMO. Ela vem de pesquisa de campo com motoristas locais (média de **230,59 km**, desvio-padrão de **67,66 km**, no artigo) e é fornecida ao Monte Carlo pela planilha `Resultados_taxi.xlsx`.

---

## 3. Simulação de Monte Carlo (MATLAB + OpenDSS) — `Simulacao_Monte_Carlo.m`

Realiza a simulação estocástica da demanda de recarga (EL, ES e ER), o dimensionamento iterativo de eletropostos rápidos por taxa de ocupação e o fluxo de potência em série temporal (24 h, passo de 1 h) no OpenDSS.

### Dependências do script

- Função **`alocarEletropostos.m`** (deve estar no *path* do MATLAB): sorteia os pontos de conexão, o horário de início, o SoC inicial e a duração das recargas, e calcula a demanda e a taxa de ocupação dos ERs (Equações 1 a 5 do artigo). Gera os arquivos `Eletropostos_EL_<Ali>.dss`, `Eletropostos_ES_<Ali>.dss` e `Eletropostos_ER_<Ali>.dss` na pasta do caso.
- Saídas da etapa 1 (`MainCode.dss` de cada caso).

### Configuração inicial (início do script)

| Variável | Descrição |
|---|---|
| `Ali` | Identificador do alimentador (nomeia pastas e arquivos de saída) |
| `caso` | Vetor com os casos a simular (ex.: `[2;3;4;5]`, `[6;7;8;9;10]`, `[11;12;13;14]`) |
| `qtd_ER` | Quantidade inicial (preliminar) de eletropostos rápidos, ajustada iterativamente |
| `taxa_aco_min`, `taxa_aco_max` | Faixa de taxa de ocupação dos ERs (padrão 0,3 e 0,7) |
| `max_cenario` | Número máximo de iterações de Monte Carlo por caso |
| `tolerancia` | Critério de convergência (diferença entre as médias móveis da tensão mínima diária de iterações consecutivas) |
| `stepsize_h`, `n_pontos` | Passo (1 h) e número de pontos da simulação diária (24) |
| Caminhos de leitura/escrita | Planilhas `UNTRMT`, `UCBT`, `SSDMT`, `EQTRMT`, `CTMT`, `TTEN`, `VEs`, `Resultados_privado` e `Resultados_taxi`; variáveis `pasta_cenario`, `pasta_capacidades`, `arquivo_tensoes`, `pasta_monitor` e `pastaDestino` |

> No código fornecido, `max_cenario = 100` e `tolerancia = 1e-7`. O artigo reporta convergência com diferença inferior a 1e-6 ou no máximo 200 iterações; ajuste esses valores para reproduzir exatamente os números publicados.

### Dados de entrada necessários

- **`VEs.xlsx`**: modelos de VEs (capacidade da bateria, potência máxima de recarga, autonomia) e representatividade no mercado (Tabela I do artigo; 12 modelos mais emplacados no Brasil, segundo a ABVE).
- Modelos de eletropostos EL, ES e ER, com potência e número de plugues (Tabela II).
- **`Resultados_privado.xlsx`** (SUMO, etapa 2) e **`Resultados_taxi.xlsx`** (pesquisa de campo): distâncias percorridas.
- **`TTEN.xlsx`**: tabela de domínio de tensões da BDGD.
- Parâmetros das distribuições gaussianas de horário de início de recarga: EL (média 18 h, σ = 3 h), ES (média 15 h, σ = 3 h) e ER (dois picos: 12,2 h com σ = 3,8 h e 21,0 h com σ = 2,4 h, estimados a partir dos dados de ocupação do aplicativo Tupi Recarga).
- Distribuição dos VEs por modalidade de recarga: 70% lenta (EL), 18% semirrápida (ES) e 12% rápida (ER). A frota máxima do alimentador (100% de penetração) é de 3.618 veículos leves.

### Execução

1. Certifique-se de que o OpenDSS está instalado e acessível por COM (`actxserver('OpenDSSEngine.DSS')`).
2. Execute a etapa 1 para cada caso a simular, gerando o `MainCode.dss` na pasta correspondente.
3. Ajuste os parâmetros e caminhos descritos acima.
4. Execute o script. Para cada caso em `caso`, em cada iteração:
   - sorteia os parâmetros estocásticos (pontos de conexão, horário de início, SoC inicial, duração de recarga);
   - compila o circuito e calcula o fluxo de potência de 24 h no OpenDSS;
   - registra tensão mínima diária (MT e BT), carregamento dos transformadores, demanda e perdas;
   - atualiza o pior e o melhor caso (menor e maior tensão mínima diária em BT) e salva os respectivos `.dss` e tensões;
   - ajusta o número de ERs: reduz se a menor taxa de ocupação for inferior a 30% e aumenta se a maior for superior a 70%;
   - repete até a convergência ou até `max_cenario`.

### Saídas geradas

Para cada caso, os resultados são salvos em `caso<N>/Dados_Alimentador<Ali>/`:

| Arquivo | Conteúdo |
|---|---|
| `Relatorio_ER.xlsx` | Histórico de ajuste dos ERs por iteração (quantidade, taxas de ocupação, decisão) |
| `Tensao_minima_diaria_BT.mat`, `Tensao_minima_diaria_MT.mat` | Tensão mínima por hora e iteração |
| `Tensao_minima_diaria_BT_final.mat`, `Tensao_minima_diaria_MT_final.mat` | Versões finais (após a convergência) |
| `Demanda_Diario_Trafo_3D.mat`, `Demanda_Diario_Trafo_final.mat` | Fator de demanda dos transformadores (hora × iteração) |
| `P_Perdas_final.mat`, `Q_Perdas_final.mat` | Perdas totais |
| `PT_Perdas_Trafo_final.mat`, `PT_Perdas_MT_final.mat`, `PT_Perdas_BT_final.mat` (e `QT_*`) | Perdas por componente (transformadores, MT e BT) |
| `P_Media.mat`, `Q_Media.mat` | Demanda média do alimentador |
| `V_pior_caso.mat`, `V_melhor_caso.mat` | Tensão mínima do pior e do melhor caso |
| `VOLTAGES_PIOR_CASO.csv`, `VOLTAGES_MELHOR_CASO.csv` | Tensões em todas as barras nos casos extremos |
| `Eletropostos_{EL,ES,ER}_{PIOR,MELHOR}.dss` | Alocação de eletropostos nos casos extremos |

---

## 4. Análise de resultados

Os scripts abaixo leem os arquivos `.mat`/`.xlsx` gerados na etapa 3 e produzem as figuras e tabelas do artigo. Para usá-los, **basta configurar os diretórios** (pasta de resultados dos casos e pasta de saída das figuras) no início de cada script.

| Script | Resultados do artigo reproduzidos |
|---|---|
| `Analizar_Resultados.m` | Perfil de tensão (Fig. 6), convergência do Monte Carlo (Fig. 7), estatísticas da tensão mínima com IC de 95% (Tabela III), sobrecarga de transformadores (Fig. 8) e perdas técnicas (Fig. 9) |
| `analise_ER.m` | Quantidade de ERs e taxa de ocupação (Fig. 10) e análise de viabilidade econômica — VPL, TIR e *payback* (Tabelas IV e V) |
| `Analise_Extremos.m` | Pior e melhor casos de tensão mínima (Tabela VI) e comparação das demandas dos VEs (Fig. 11) |
| `Analise_Sensibilidade.m` | Sensibilidade ao horário médio de recarga em ELs e à migração EL → ER (Fig. 12) |

O intervalo de confiança de 95% é calculado por `CI = x̄ ± t(0,975; N−1) · s/√N` (Equação 6 do artigo).

---

## Ordem recomendada de execução (resumo)

1. `projecao_ajustes.m` → obtém os fatores de aumento de demanda e os anos dos cenários.
2. SUMO (`OSM Web Wizard` + `activitygen`) e `calcularDistanciasPrivado.py` → distâncias dos VEs privados (e inclusão das distâncias dos táxis).
3. `Codigo_BDGD2OPENDSS.m` → um `MainCode.dss` por caso, com o `fator_aumento` correspondente.
4. `Simulacao_Monte_Carlo.m` → simulações de todos os casos.
5. Scripts de análise → figuras e tabelas.

## Citação

Se este código for útil para o seu trabalho, cite:

> E. de J. Ramalho e J. C. Galvis Manso, "A Methodology for Analyzing the Impacts of Charging Station Demand on the Distribution Network," Universidade Federal de Ouro Preto (UFOP), Minas Gerais, Brasil.

Contato: eutran.ramalho@aluno.ufop.edu.br · juancgalvis@ufop.edu.br
