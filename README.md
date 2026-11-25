# Somaticos---Amostras-Extras
Amostras extras da atividade prática - Aula de somáticos do dia 14/11

## Atualização e instalação das ferramentas

### Atualizar repositório
```sudo apt-get update```

### Instalar o samtools
'''sudo apt-get install samtools'''

### Instalar GATK4
'''docker pull broadinstitute/gatk:4.2.2.0'''

'''wget -c https://github.com/broadinstitute/gatk/releases/download/4.2.2.0/gatk-4.2.2.0.zip'''

'''unzip gatk-4.2.2.0.zip'''

## Amostras Extras

### - WP190 (tumor) e WP191 (normal)
### - WP017 (tumor) e WP018 (normal)
### Nota 1: Utilizar o af-gnomad chr13 e chr19.
### Nota 2: Será preciso baixar o chr13 e chr19 da UCSC.

## Download das referências chr13 e chr19

'''wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz'''

'''wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz'''

### Concatenar os arquivos .fa.gz para hg19 e substituir "chr".

'''zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa'''

### Gerar o index do arquivo hg19.fa

'''samtools faidx hg19.fa'''

### Criar dicionário da referência.

'''./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict'''

### Dividir o genoma de referência em intervalos baseados em regiões com bases indeterminadas.

'''./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R hg19.fa -O hg19.interval_list -OT ACGT'''

### Verificar o nome das amostras (SM).

'''samtools view -H tumor_wp191.bam | grep RG | cut -f5 | sed -e "s/SM://g"'''
'''samtools view -H tumor_wp190.bam | grep RG | cut -f5 | sed -e "s/SM://g"'''
'''samtools view -H tumor_wp018.bam | grep RG | cut -f6 | sed -e "s/SM://g"'''
'''samtools view -H tumor_wp017.bam | grep RG | cut -f6 | sed -e "s/SM://g"'''


### Chamada de SNVs e indels comparando as amostras normal e tumoral.

'''./gatk-4.2.2.0/gatk Mutect2 \
	-R hg19.fa \
	-I tumor_wp191.bam \
	-I tumor_wp190.bam \
	-normal WP191 \
	--germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
	-O wp190_somatic.vcf.gz \
	-L hg19.interval_list'''

'''./gatk-4.2.2.0/gatk Mutect2 \
	-R hg19.fa \
	-I tumor_wp018.bam \
	-I tumor_wp017.bam \
	-normal WP018 \
	--germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
	-O wp017_somatic.vcf.gz \
	-L hg19.interval_list'''

### Tabulação das métricas para inferência de contaminação na amostra tumoral e normal.

'''./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_wp190.bam \
	-V af-only-gnomad-chr13-chr19.vcf.gz \
	-L hg19.interval_list \
	-O tumor_wp190.table'''

'''./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_wp191.bam \
	-V af-only-gnomad-chr13-chr19.vcf.gz \
	-L hg19.interval_list \
	-O tumor_wp191.table'''

'''./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_wp017.bam \
	-V af-only-gnomad-chr13-chr19.vcf.gz \
	-L hg19.interval_list \
	-O tumor_wp017.table'''

'''./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_wp018.bam \
	-V af-only-gnomad-chr13-chr19.vcf.gz \
	-L hg19.interval_list \
	-O tumor_wp018.table'''

### Cálculo da Contaminação: estimativa da porcentagem de contaminação por outras amostras através da comparação da frequência alélica das variantes da amostra normal.

'''./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp190.table \
	-matched tumor_wp191.table \
	-O wp190_contamination.table'''

'''./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp017.table \
	-matched tumor_wp018.table \
	-O wp017_contamination.table'''

### Filtragem das SNVs e indels somáticas.

'''./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V wp190_somatic.vcf.gz \
	--contamination-table wp190_contamination.table \
	-O wp190_filtered.vcf.gz'''

'''./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V wp017_somatic.vcf.gz \
	--contamination-table wp017_contamination.table \
	-O wp017_filtered.vcf.gz'''

### Variantes restantes pós filtragem.
### Resultado esperado: WP190 (5) e WP017 (3).

'''zcat wp190_filtered.vcf.gz | grep -v "\#" | wc -l'''

'''zcat wp017_filtered.vcf.gz | grep -v "\#" | wc -l'''

## Painel de normais (PoN)

### Instalar VEP

'''docker pull ensemblorg/ensembl-vep'''

### Criar diretório

'''mkdir -p vep_output'''

### Altera/concede permissão de leitura, escrita e execução.

'''chmod 777 vep_output'''

### Rodar VEP

'''bash vep-docker.sh'''

### Baixar arquivo vcf e vcf.idx do repositório do Google Cloud Storage

'''wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf'''

'''wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf.idx'''

### Chamada de variantes somáticas, com base no banco de frequências alélicas germinativas e com Painel de Normais (PoN).

'''./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O wp190.somatic.pon.vcf.gz'''

'''./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O wp017.somatic.pon.vcf.gz'''

  
### Cálcula de contaminação

'''./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp190.table \
	-O wp190.contamination.pon.table'''

'''./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp017.table \
	-O wp017.contamination.pon.table'''


### Gerar VCF filtrado com as variantes somáticas candidatas e de baixa contaminação.

'''./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V wp190.somatic.pon.vcf.gz \
	--contamination-table wp190.contamination.pon.table \
	-O wp190.filtered.pon.vcf.gz'''

'''./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V wp017.somatic.pon.vcf.gz \
	--contamination-table wp017.contamination.pon.table \
	-O wp017.filtered.pon.vcf.gz'''

### Variantes candidatas pós filtragem com Painel de Normais (PoN).
### Resultados esperados: WP190 () e WP017 ()

'''zcat wp190.filtered.pon.vcf.gz | grep -v "\#" | grep PASS | wc -l'''

'''zcat wp017.filtered.pon.vcf.gz | grep -v "\#" | grep PASS | wc -l'''
