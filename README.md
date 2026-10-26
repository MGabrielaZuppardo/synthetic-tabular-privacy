# synthetic-tabular-privacy

**Geração de dados tabulares sintéticos com GANs: fidelidade, utilidade e privacidade nos microdados do ENEM**

> **Status:** em desenvolvimento (planejamento do núcleo v1). Os resultados abaixo serão preenchidos conforme os experimentos forem concluídos.

---

## Motivação

Microdados educacionais contêm atributos sensíveis (renda, cor/raça, tipo de escola) e estão sujeitos à LGPD. Dados sintéticos costumam ser tratados como "seguros por padrão", mas essa suposição precisa ser **medida**, não assumida. Este projeto avalia, de ponta a ponta, o quanto dados sintéticos gerados a partir dos microdados do ENEM preservam qualidade estatística e utilidade preditiva, e quanto risco de privacidade ainda carregam.

## Perguntas de pesquisa

1. O treino adversarial (CTGAN) traz ganho real sobre TVAE e Gaussian Copula nos dados do ENEM?
2. Os dados sintéticos preservam utilidade preditiva (TSTR) e qualidade por subgrupos?
3. Qual o nível de risco de privacidade (DCR e inferência de pertencimento)?

## Escopo

### Núcleo (v1), com SDV

| Componente | Descrição |
|---|---|
| Dados | ENEM, amostra estratificada de 100k a 300k linhas (UF e faixa de renda), apenas participantes presentes em todas as provas |
| Modelos | Gaussian Copula (baseline), TVAE, CTGAN |
| Fidelidade | Marginais e correlações (SDMetrics) |
| Utilidade | TSTR com LightGBM/XGBoost (nota média acima/abaixo da mediana) |
| Privacidade | DCR (Distance to Closest Record) e ataque de inferência de pertencimento (MIA) |
| Subgrupos | Raça, renda e tipo de escola |

### Extensões

| Extensão | Conteúdo | Release alvo |
|---|---|---|
| E1: GAN do zero | Gerador e discriminador em PyTorch (MLP) | v1.5 |
| E2: Variantes | WGAN-GP e, se houver implementação confiável, CTAB-GAN | v1.5 |
| E3: Análise de treino | Curvas de perda, mode collapse, hiperparâmetros (épocas, pac, embedding), múltiplas sementes com IC | v1.5 |
| E4: Privacidade diferencial | Variante DP (ex.: DP-CTGAN ou PATE-GAN, validar referências) | v2.0 |
| E5: Benchmark Adult | Validação cruzada com resultados da literatura | v2.0 |

Se o tempo apertar, priorizar E1 e E3.

## Metodologia

- **Pipeline versionado com DVC**; nenhum dado real é versionado no Git.
- **Tratamento de colunas (variável experimental):** `CO_MUNICIPIO_*` e `CO_ESCOLA` são **mantidas** na configuração principal e comparadas com uma configuração de ablação que as remove e usa apenas UF ou região. Ambas rodam com os mesmos modelos e sementes, para medir o efeito das colunas em fidelidade, TSTR, DCR e MIA, em consumo de memória (VRAM) e no tempo de treino. A alta cardinalidade destas colunas é um risco conhecido para memória, qualidade e privacidade (são quase-identificadores).
- **Validação de domínio:** verificar valores fora de `[0, 1000]` nas notas geradas.
- **Rastreamento no MLflow**, com sementes fixas e intervalos de confiança nas métricas.
- **Edições do ENEM:** 2023, 2024 e 2025 (microdados do INEP). Os esquemas de colunas e os dicionários de variáveis podem diferir entre anos, então a ingestão harmoniza os nomes e as categorias antes da amostragem, e o ano de edição é mantido como coluna de controle para verificar mudanças de distribuição entre edições.

## Estrutura do repositório

```
synthetic-tabular-privacy/
├── data/                  # versionado via DVC (conteúdo fora do Git)
│   ├── raw/
│   ├── processed/
│   └── synthetic/
├── src/
│   ├── ingestion/         # download e leitura dos microdados
│   ├── preprocessing/     # filtros, amostragem estratificada, remoção de colunas
│   ├── models/
│   │   ├── sdv_wrappers.py   # Gaussian Copula, TVAE, CTGAN
│   │   ├── gan_scratch.py    # E1: GAN em PyTorch
│   │   └── wgan_gp.py        # E2: WGAN-GP
│   ├── evaluation/
│   │   ├── fidelity.py       # SDMetrics
│   │   ├── tstr.py           # Train on Synthetic, Test on Real
│   │   ├── dcr.py            # Distance to Closest Record
│   │   ├── mia.py            # inferência de pertencimento
│   │   └── subgroups.py      # raça, renda, tipo de escola
│   └── api/               # API de geração (FastAPI)
├── notebooks/             # EDA e resultados
├── configs/               # hiperparâmetros e configurações de experimentos
├── tests/
├── Dockerfile
└── README.md
```

## Como executar

_Será preenchido após a implementação do núcleo._ Fluxo previsto:

```bash
# 1. ambiente
pip install -r requirements.txt

# 2. dados (obtidos via DVC / download dos microdados do INEP)
dvc pull

# 3. pipeline
dvc repro

# 4. API de geração
docker build -t synthetic-tabular-privacy .
docker run -p 8000:8000 synthetic-tabular-privacy
```

## Resultados

_A preencher._

| Modelo | Fidelidade (SDMetrics) | TSTR (AUC) | DCR (mediana) | MIA (AUC) |
|---|---|---|---|---|
| Gaussian Copula | – | – | – | – |
| TVAE | – | – | – | – |
| CTGAN | – | – | – | – |

## Cronograma

| Fase | Duração | Conteúdo |
|---|---|---|
| Núcleo | 4 semanas | Ingestão e EDA → treino e fidelidade → TSTR, DCR, MIA e subgrupos → engenharia e README |
| E1 + E3 | 2 semanas | GAN do zero e análise de treino |
| E2 | 1 semana | WGAN-GP / CTAB-GAN |
| E4 | 1 a 2 semanas | Privacidade diferencial |
| E5 | 3 a 5 dias | Adult como benchmark |

O núcleo já gera um projeto publicável; as extensões saem em releases incrementais (v1.0, v1.5, v2.0).

## Riscos e limitações

- **Termos de uso do INEP:** confirmar antes de publicar qualquer derivado dos dados.
- **Harmonização entre edições:** mudanças de esquema ou de codificação entre 2023 e 2025 podem introduzir inconsistências; serão documentadas e testadas na ingestão.
- **Colunas de alta cardinalidade (`CO_MUNICIPIO_*`, `CO_ESCOLA`):** podem estourar a VRAM, degradar a qualidade e elevar o risco de privacidade; por isso a ablação sem elas é obrigatória para comparação.
- **Categorias raras:** degradação esperada na geração; será documentada.
- **Variância entre execuções de GAN:** mitigada com múltiplas sementes e IC.
- **Escopo das extensões:** priorizar E1 e E3 se o tempo apertar.

## Aviso de privacidade

Dados sintéticos **não** são anonimização garantida. Os resultados de DCR e MIA deste projeto quantificam o risco empírico para os modelos e configurações testados, e não constituem garantia formal (exceto na variante com privacidade diferencial, quando houver). Não use os dados sintéticos gerados aqui como substitutos seguros de dados pessoais sem avaliação própria à luz da LGPD. Nenhum microdado real é distribuído neste repositório.

## Referências

> Validar antes de citar.

- Xu et al. (2019). _Modeling Tabular Data using Conditional GAN_ (CTGAN). NeurIPS.
- Stadler et al. (2022). Privacidade em dados sintéticos tabulares. USENIX Security (título exato a confirmar).
- Documentação do [SDV](https://docs.sdv.dev/sdv) e do [SDMetrics](https://docs.sdv.dev/sdmetrics).

## Licença

_A definir._ Atenção aos termos de uso dos microdados do INEP.
