# Análise Somática com GATK4 Mutect2 (Tumor-Normal)
Este repositório fornece um tutorial prático para realizar a chamada de variantes somáticas utilizando a abordagem "Padrão Ouro" (emparelhamento Tumor-Normal) do GATK 4 Mutect2.
Para manter o processamento leve e rápido em ambientes de estudo, o foco deste guia será exclusivamente nos cromossomas 13 e 19 do genoma humano (referência hg19), com dados provenientes do UCSC Genome Browser.

## Pré-Requisitos

Antes de começar, certifique-se de que o seu ambiente tem os seguintes recursos prontos:

[x] Ambiente: GitHub Codespaces (Recomenda-se máquina de 4-cores para maior velocidade).

[x] Ferramentas Instaladas: samtools e gatk-4.2.2.0.

[x] Ficheiros BAM de Entrada:

Paciente 1: tumor_wp017.bam (Tumor) e tumor_wp018.bam (Normal)

Paciente 2: tumor_wp190.bam (Tumor) e tumor_wp191.bam (Normal)

[x] Recursos Populacionais: Ficheiro af-only-gnomad-chr13-chr19.vcf.gz (usado para filtrar mutações comuns na população).

### Passo 1: Obtendo os Ficheiros BAM (Amostras)

Os ficheiros de sequenciação brutos (`.bam` e `.bai`) são pesados e já se encontram preparados e alojados no repositório oficial do curso. 

Para aceder a estes dados, os alunos têm duas opções:

#### Opção A: Utilizar o GitHub Codespaces

Esta é a forma mais fácil e rápida, pois não requer instalação no seu computador.

1. Aceda ao repositório base: [renatopuga/somatico](https://github.com/renatopuga/somatico).
2. Clique no botão **`<> Code`**.
3. Mude para o separador **Codespaces** e clique no botão **`+`** ou **Create codespace on main**.
4. *Dica de Ouro:* Nas configurações da máquina, escolha a opção **4-core** para que os algoritmos de alinhamento processem mais rapidamente.

#### Opção B: Clonar para o seu Computador Local (Linux/Mac)

Se preferir rodar o pipeline na sua própria máquina (precisa do GATK e Samtools instalados), faça o clone direto do repositório através do terminal:

```bash
git clone https://github.com/renatopuga/somatico.git
cd somatico
```

## Passo 2: Preparação

### Preparação do Genoma de Referência (hg19)

O algoritmo GATK é estrito quanto à formatação. Ele requer um mapa genómico perfeitamente limpo e indexado para alinhar as leituras de ADN.

### Descarregar os cromossomas alvo da UCSC:

```
bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz
```

### Concatenar e limpar o cabeçalho:

Dica: O GATK prefere cromossomas numerados diretamente (ex: >13) em vez do prefixo da UCSC (ex: >chr13). O comando sed resolve isso automaticamente.

```bash
zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa
```

### Gerar índices e dicionários:

```bash
samtools faidx hg19.fa
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R hg19.fa -O hg19.interval_list -OT ACGT
```

## Passo 3: Execução do Pipeline 

### Paciente WP017

A análise "Padrão Ouro" compara o tecido tumoral com o tecido saudável do próprio paciente para filtrar, com elevada precisão, as mutações genéticas germinativas (de nascença).

#### Chamada de Variantes (Mutect2):

```bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  -I tumor_wp018.bam \
  -normal WP018 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -O WP017.somatic.vcf.gz \
  -L hg19.interval_list
```

#### Avaliação de Contaminação Laboratorial:
Calcula matematicamente a percentagem de ADN externo ou contaminação técnica na amostra.

##### Sumário do Tumor

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.table
```

##### Sumário do Tecido Normal

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp018.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP018.table
```

##### Cálculo Final

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.table \
  -matched WP018.table \
  -O WP017.contamination.table
```

#### Filtragem Final:
Aplica os carimbos de aprovação (PASS) ou motivos de rejeição cruzando as variantes com o cálculo de contaminação.

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.vcf.gz \
  --contamination-table WP017.contamination.table \
  -O WP017.filtered.vcf.gz
```

### Paciente WP190

Para analisar a segunda amostra, aplicamos exatamente a mesma lógica biológica. Para não tornar este documento demasiado longo, os comandos estão ocultos abaixo.

#### Chamada de Variantes (Mutect2):

```bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  -I tumor_wp191.bam \
  -normal WP191 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -O WP190.somatic.vcf.gz \
  -L hg19.interval_list
```

#### Avaliação de Contaminação Laboratorial:
Calcula matematicamente a percentagem de ADN externo ou contaminação técnica na amostra.

##### Sumário do Tumor

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.table
```

##### Sumário do Tecido Normal

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp191.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP191.table
 ``` 

##### Cálculo Final

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.table \
  -matched WP191.table \
  -O WP190.contamination.table
```

#### Filtragem Final

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.vcf.gz \
  --contamination-table WP190.contamination.table \
  -O WP190.filtered.vcf.gz
```

## Passo 4: Verificação de Resultados

Depois de gerar os seus ficheiros filtered.vcf.gz, pode ver o impacto prático do pipeline contando quantas variantes "verdadeiras" sobreviveram aos rigorosos filtros do GATK (aquelas que recebem a etiqueta PASS).

Para a amostra WP017:

```bash
zgrep -v "^#" WP017.filtered.vcf.gz | grep -w "PASS" | wc -l
```

Para a amostra WP190:

```bash
zgrep -v "^#" WP190.filtered.vcf.gz | grep -w "PASS" | wc -l
```


