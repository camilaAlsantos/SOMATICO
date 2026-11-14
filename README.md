# Análise Somática com GATK4 Mutect2 (Tumor-Normal)
Este repositório fornece um tutorial prático para realizar a chamada de variantes somáticas utilizando a abordagem "Padrão Ouro" (emparelhamento Tumor-Normal) do GATK 4 Mutect2.
Para manter o processamento leve e rápido em ambientes de estudo, o foco deste guia será exclusivamente nos cromossomas 13 e 19 do genoma humano (referência hg19), com dados provenientes do UCSC Genome Browser.

## _Índice_
### Pré-Requisitos: https://gemini.google.com/#-pr%C3%A9-requisitos 
### Passo 1: Preparação do Genoma (hg19) : https://gemini.google.com/#1%EF%B8%8F%E2%83%A3-passo-1-prepara%C3%A7%C3%A3o-do-genoma-de-refer%C3%AAncia-hg19
### Passo 2: Execução do Pipeline (WP017) : https://gemini.google.com/#2%EF%B8%8F%E2%83%A3-passo-2-execu%C3%A7%C3%A3o-do-pipeline-paciente-wp017
### Passo 3: Analisar o Paciente WP190 : https://gemini.google.com/#4%EF%B8%8F%E2%83%A3-passo-4-verifica%C3%A7%C3%A3o-de-resultados
### Passo 4: Verificação de Resultados : https://gemini.google.com/#4%EF%B8%8F%E2%83%A3-passo-4-verifica%C3%A7%C3%A3o-de-resultados 

## Pré-Requisitos

Antes de começar, certifique-se de que o seu ambiente tem os seguintes recursos prontos:

[x] Ambiente: GitHub Codespaces (Recomenda-se máquina de 4-cores para maior velocidade).

[x] Ferramentas Instaladas: samtools e gatk-4.2.2.0.

[x] Ficheiros BAM de Entrada:

Paciente 1: tumor_wp017.bam (Tumor) e tumor_wp018.bam (Normal)

Paciente 2: tumor_wp190.bam (Tumor) e tumor_wp191.bam (Normal)

[x] Recursos Populacionais: Ficheiro af-only-gnomad-chr13-chr19.vcf.gz (usado para filtrar mutações comuns na população).

### 📥 Passo 0: Obtendo os Ficheiros BAM (Amostras)

Os ficheiros de sequenciação brutos (`.bam` e `.bai`) são pesados e já se encontram preparados e alojados no repositório oficial do curso. 

Para aceder a estes dados, os alunos têm duas opções:

### Opção A: Utilizar o GitHub Codespaces (Recomendado)</b></summary>
<br>
Esta é a forma mais fácil e rápida, pois não requer instalação no seu computador.

1. Aceda ao repositório base: [renatopuga/somatico](https://github.com/renatopuga/somatico).
2. Clique no botão verde **`<> Code`**.
3. Mude para o separador **Codespaces** e clique no botão **`+`** ou **Create codespace on main**.
4. *Dica de Ouro:* Nas configurações da máquina, escolha a opção **4-core** para que os algoritmos de alinhamento processem mais rapidamente.

### Opção B: Clonar para o seu Computador Local (Linux/Mac)</b></summary>
<br>
Se preferir rodar o pipeline na sua própria máquina (precisa do GATK e Samtools instalados), faça o clone direto do repositório através do terminal:

```diff
+ Esta linha inteira vai ficar verde no GitHub!
- Se usar um sinal de menos, a linha fica vermelha.
  E sem sinal, fica com a cor normal.
bash
# Clonar o repositório com os ficheiros BAM
git clone [https://github.com/renatopuga/somatico.git](https://github.com/renatopuga/somatico.git)
# Entrar na pasta do projeto
cd somatico
```

## Passo 1: Preparação

## Passo 1: Preparação do Genoma de Referência (hg19)

O algoritmo GATK é estrito quanto à formatação. Ele requer um mapa genómico perfeitamente limpo e indexado para alinhar as leituras de ADN.

### Descarregar os cromossomas alvo da UCSC:

wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz

### Concatenar e limpar o cabeçalho:

Dica: O GATK prefere cromossomas numerados diretamente (ex: >13) em vez do prefixo da UCSC (ex: >chr13). O comando sed resolve isso automaticamente.

zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa

### Gerar índices e dicionários:

samtools faidx hg19.fa
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R hg19.fa -O hg19.interval_list -OT ACGT


## Passo 2: Execução do Pipeline (Paciente WP017)

A análise "Padrão Ouro" compara o tecido tumoral com o tecido saudável do próprio paciente para filtrar, com elevada precisão, as mutações genéticas germinativas (de nascença).

### Chamada de Variantes (Mutect2):

./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  -I tumor_wp018.bam \
  -normal WP018 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -O WP017.somatic.vcf.gz \
  -L hg19.interval_list


### Avaliação de Contaminação Laboratorial:
Calcula matematicamente a percentagem de ADN externo ou contaminação técnica na amostra.

# Sumário do Tumor
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.table

# Sumário do Tecido Normal
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp018.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP018.table

# Cálculo Final
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.table \
  -matched WP018.table \
  -O WP017.contamination.table


## Filtragem Final:
Aplica os carimbos de aprovação (PASS) ou motivos de rejeição cruzando as variantes com o cálculo de contaminação.

./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.vcf.gz \
  --contamination-table WP017.contamination.table \
  -O WP017.filtered.vcf.gz


## Passo 3: Analisar o Paciente WP190

Para analisar a segunda amostra, aplicamos exatamente a mesma lógica biológica. Para não tornar este documento demasiado longo, os comandos estão ocultos abaixo.

### 1. Mutect2
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  -I tumor_wp191.bam \
  -normal WP191 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -O WP190.somatic.vcf.gz \
  -L hg19.interval_list

### 2. Resumos de Contaminação
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.table

./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp191.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP191.table

./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.table \
  -matched WP191.table \
  -O WP190.contamination.table

### 3. Filtragem Final
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.vcf.gz \
  --contamination-table WP190.contamination.table \
  -O WP190.filtered.vcf.gz


## Passo 4: Verificação de Resultados

Depois de gerar os seus ficheiros filtered.vcf.gz, pode ver o impacto prático do pipeline contando quantas variantes "verdadeiras" sobreviveram aos rigorosos filtros do GATK (aquelas que recebem a etiqueta PASS).

Para a amostra WP017:

zgrep -v "^#" WP017.filtered.vcf.gz | grep -w "PASS" | wc -l


Para a amostra WP190:

zgrep -v "^#" WP190.filtered.vcf.gz | grep -w "PASS" | wc -l


# Curiosidade: Experimente remover a secção | grep -w "PASS" dos comandos acima para ver o total absoluto de variantes encontradas inicialmente. A diferença entre esse número e o resultado final prova o excelente trabalho do GATK a limpar falsos positivos e "ruído" genético!
