# 如何运行VQSR

原文链接: [ (howto) Recalibrate variant quality scores = run VQSR](https://software.broadinstitute.org/gatk/documentation/article?id=2805)

内容有点多，估计跟GATK最佳实践也是很大部分重合着的

#### Objective

Recalibrate variant quality scores and produce a callset filtered for the desired levels of sensitivity and specificity.

#### Prerequisites

- TBD

#### Caveats

This document provides a typical usage example including parameter values. However, the values given may not be representative of the latest Best Practices recommendations. When in doubt, please consult the [FAQ document on VQSR training sets and parameters](https://www.broadinstitute.org/gatk/guide/article?id=1259), which overrides this document. See that document also for caveats regarding exome vs. whole genomes analysis design.

#### Steps

1. Prepare recalibration parameters for SNPs
   a. Specify which call sets the program should use as resources to build the recalibration model
   b. Specify which annotations the program should use to evaluate the likelihood of Indels being real
   c. Specify the desired truth sensitivity threshold values that the program should use to generate tranches
   d. Determine additional model parameters
2. Build the SNP recalibration model
3. Apply the desired level of recalibration to the SNPs in the call set
4. Prepare recalibration parameters for Indels a. Specify which call sets the program should use as resources to build the recalibration model b. Specify which annotations the program should use to evaluate the likelihood of Indels being real c. Specify the desired truth sensitivity threshold values that the program should use to generate tranches d. Determine additional model parameters
5. Build the Indel recalibration model
6. Apply the desired level of recalibration to the Indels in the call set

------

### 1. Prepare recalibration parameters for SNPs

#### a. Specify which call sets the program should use as resources to build the recalibration model

For each training set, we use key-value tags to qualify whether the set contains known sites, training sites, and/or truth sites. We also use a tag to specify the prior likelihood that those sites are true (using the Phred scale).

- True sites training resource: HapMap

This resource is a SNP call set that has been validated to a very high degree of confidence. The program will consider that the variants in this resource are representative of true sites (truth=true), and will use them to train the recalibration model (training=true). We will also use these sites later on to choose a threshold for filtering variants based on sensitivity to truth sites. The prior likelihood we assign to these variants is Q15 (96.84%).

- True sites training resource: Omni

This resource is a set of polymorphic SNP sites produced by the Omni genotyping array. The program will consider that the variants in this resource are representative of true sites (truth=true), and will use them to train the recalibration model (training=true). The prior likelihood we assign to these variants is Q12 (93.69%).

- Non-true sites training resource: 1000G

This resource is a set of high-confidence SNP sites produced by the 1000 Genomes Project. The program will consider that the variants in this resource may contain true variants as well as false positives (truth=false), and will use them to train the recalibration model (training=true). The prior likelihood we assign to these variants is Q10 (90%).

- Known sites resource, not used in training: dbSNP

This resource is a SNP call set that has not been validated to a high degree of confidence (truth=false). The program will not use the variants in this resource to train the recalibration model (training=false). However, the program will use these to stratify output metrics such as Ti/Tv ratio by whether variants are present in dbsnp or not (known=true). The prior likelihood we assign to these variants is Q2 (36.90%).

*The default prior likelihood assigned to all other variants is Q2 (36.90%). This low value reflects the fact that the philosophy of the GATK callers is to produce a large, highly sensitive callset that needs to be heavily refined through additional filtering.*

#### b. Specify which annotations the program should use to evaluate the likelihood of SNPs being real

These annotations are included in the information generated for each variant call by the caller. If an annotation is missing (typically because it was omitted from the calling command) it can be added using the VariantAnnotator tool.

- [Coverage (DP)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_Coverage.php)

Total (unfiltered) depth of coverage. Note that this statistic should not be used with exome datasets; see caveat detailed in the [VQSR arguments FAQ doc](https://software.broadinstitute.org/gatk/documentation/(https://www.broadinstitute.org/gatk/guide/article?id=1259)).

- [QualByDepth (QD)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_QualByDepth.php)

Variant confidence (from the QUAL field) / unfiltered depth of non-reference samples.

- [FisherStrand (FS)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php)

Measure of strand bias (the variation being seen on only the forward or only the reverse strand). More bias is indicative of false positive calls. This complements the [StrandOddsRatio (SOR)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php) annotation.

- [StrandOddsRatio (SOR)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php)

Measure of strand bias (the variation being seen on only the forward or only the reverse strand). More bias is indicative of false positive calls. This complements the [FisherStrand (FS)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_FisherStrand.php) annotation.

- [MappingQualityRankSumTest (MQRankSum)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_MappingQualityRankSumTest.php)

The rank sum test for mapping qualities. Note that the mapping quality rank sum test can not be calculated for sites without a mixture of reads showing both the reference and alternate alleles.

- [ReadPosRankSumTest (ReadPosRankSum)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_ReadPosRankSumTest.php)

The rank sum test for the distance from the end of the reads. If the alternate allele is only seen near the ends of reads, this is indicative of error. Note that the read position rank sum test can not be calculated for sites without a mixture of reads showing both the reference and alternate alleles.

- [RMSMappingQuality (MQ)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_RMSMappingQuality.php)

Estimation of the overall mapping quality of reads supporting a variant call.

- [InbreedingCoeff](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_InbreedingCoeff.php)

Evidence of inbreeding in a population. See caveats regarding population size and composition detailed in the [VQSR arguments FAQ doc](https://software.broadinstitute.org/gatk/documentation/(https://www.broadinstitute.org/gatk/guide/article?id=1259)).

#### c. Specify the desired truth sensitivity threshold values that the program should use to generate tranches

- First tranche threshold 100.0
- Second tranche threshold 99.9
- Third tranche threshold 99.0
- Fourth tranche threshold 90.0

Tranches are essentially slices of variants, ranked by VQSLOD, bounded by the threshold values specified in this step. The threshold values themselves refer to the sensitivity we can obtain when we apply them to the call sets that the program uses to train the model. The idea is that the lowest tranche is highly specific but less sensitive (there are very few false positives but potentially many false negatives, i.e. missing calls), and each subsequent tranche in turn introduces additional true positive calls along with a growing number of false positive calls. This allows us to filter variants based on how sensitive we want the call set to be, rather than applying hard filters and then only evaluating how sensitive the call set is using post hoc methods.

------

### 2. Build the SNP recalibration model

#### Action

Run the following GATK command:

```
java -jar GenomeAnalysisTK.jar \ 
    -T VariantRecalibrator \ 
    -R reference.fa \ 
    -input raw_variants.vcf \ 
    -resource:hapmap,known=false,training=true,truth=true,prior=15.0 hapmap.vcf \ 
    -resource:omni,known=false,training=true,truth=true,prior=12.0 omni.vcf \ 
    -resource:1000G,known=false,training=true,truth=false,prior=10.0 1000G.vcf \ 
    -resource:dbsnp,known=true,training=false,truth=false,prior=2.0 dbsnp.vcf \ 
    -an DP \ 
    -an QD \ 
    -an FS \ 
    -an SOR \ 
    -an MQ \
    -an MQRankSum \ 
    -an ReadPosRankSum \ 
    -an InbreedingCoeff \
    -mode SNP \ 
    -tranche 100.0 -tranche 99.9 -tranche 99.0 -tranche 90.0 \ 
    -recalFile recalibrate_SNP.recal \ 
    -tranchesFile recalibrate_SNP.tranches \ 
    -rscriptFile recalibrate_SNP_plots.R 
```

#### Expected Result

This creates several files. The most important file is the recalibration report, called `recalibrate_SNP.recal`, which contains the recalibration data. This is what the program will use in the next step to generate a VCF file in which the variants are annotated with their recalibrated quality scores. There is also a file called `recalibrate_SNP.tranches`, which contains the quality score thresholds corresponding to the tranches specified in the original command. Finally, if your installation of R and the other required libraries was done correctly, you will also find some PDF files containing plots. These plots illustrated the distribution of variants according to certain dimensions of the model.

For detailed instructions on how to interpret these plots, please refer to the [VQSR method documentation](https://www.broadinstitute.org/gatk/guide/article?id=39) and [presentation videos](https://www.broadinstitute.org/gatk/guide/presentations).

------

### 3. Apply the desired level of recalibration to the SNPs in the call set

#### Action

Run the following GATK command:

```
java -jar GenomeAnalysisTK.jar \ 
    -T ApplyRecalibration \ 
    -R reference.fa \ 
    -input raw_variants.vcf \ 
    -mode SNP \ 
    --ts_filter_level 99.0 \ 
    -recalFile recalibrate_SNP.recal \ 
    -tranchesFile recalibrate_SNP.tranches \ 
    -o recalibrated_snps_raw_indels.vcf 
```

#### Expected Result

This creates a new VCF file, called `recalibrated_snps_raw_indels.vcf`, which contains all the original variants from the original `raw_variants.vcf` file, but now the SNPs are annotated with their recalibrated quality scores (VQSLOD) and either `PASS` or `FILTER` depending on whether or not they are included in the selected tranche.

Here we are taking the second lowest of the tranches specified in the original recalibration command. This means that we are applying to our data set the level of sensitivity that would allow us to retrieve 99% of true variants from the truth training sets of HapMap and Omni SNPs. If we wanted to be more specific (and therefore have less risk of including false positives, at the risk of missing real sites) we could take the very lowest tranche, which would only retrieve 90% of the truth training sites. If we wanted to be more sensitive (and therefore less specific, at the risk of including more false positives) we could take the higher tranches. In our Best Practices documentation, we recommend taking the second highest tranche (99.9%) which provides the highest sensitivity you can get while still being acceptably specific.

------

### 4. Prepare recalibration parameters for Indels

#### a. Specify which call sets the program should use as resources to build the recalibration model

For each training set, we use key-value tags to qualify whether the set contains known sites, training sites, and/or truth sites. We also use a tag to specify the prior likelihood that those sites are true (using the Phred scale).

- Known and true sites training resource: Mills

This resource is an Indel call set that has been validated to a high degree of confidence. The program will consider that the variants in this resource are representative of true sites (truth=true), and will use them to train the recalibration model (training=true). The prior likelihood we assign to these variants is Q12 (93.69%).

The default prior likelihood assigned to all other variants is Q2 (36.90%). This low value reflects the fact that the philosophy of the GATK callers is to produce a large, highly sensitive callset that needs to be heavily refined through additional filtering.

#### b. Specify which annotations the program should use to evaluate the likelihood of Indels being real

These annotations are included in the information generated for each variant call by the caller. If an annotation is missing (typically because it was omitted from the calling command) it can be added using the VariantAnnotator tool.

- [Coverage (DP)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_Coverage.php)

Total (unfiltered) depth of coverage. Note that this statistic should not be used with exome datasets; see caveat detailed in the [VQSR arguments FAQ doc](https://software.broadinstitute.org/gatk/documentation/(https://www.broadinstitute.org/gatk/guide/article?id=1259)).

- [QualByDepth (QD)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_QualByDepth.php)

Variant confidence (from the QUAL field) / unfiltered depth of non-reference samples.

- [FisherStrand (FS)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php)

Measure of strand bias (the variation being seen on only the forward or only the reverse strand). More bias is indicative of false positive calls. This complements the [StrandOddsRatio (SOR)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php) annotation.

- [StrandOddsRatio (SOR)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_StrandOddsRatio.php)

Measure of strand bias (the variation being seen on only the forward or only the reverse strand). More bias is indicative of false positive calls. This complements the [FisherStrand (FS)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_FisherStrand.php) annotation.

- [MappingQualityRankSumTest (MQRankSum)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_MappingQualityRankSumTest.php)

The rank sum test for mapping qualities. Note that the mapping quality rank sum test can not be calculated for sites without a mixture of reads showing both the reference and alternate alleles.

- [ReadPosRankSumTest (ReadPosRankSum)](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_ReadPosRankSumTest.php)

The rank sum test for the distance from the end of the reads. If the alternate allele is only seen near the ends of reads, this is indicative of error. Note that the read position rank sum test can not be calculated for sites without a mixture of reads showing both the reference and alternate alleles.

- [InbreedingCoeff](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_annotator_InbreedingCoeff.php)

Evidence of inbreeding in a population. See caveats regarding population size and composition detailed in the [VQSR arguments FAQ doc](https://software.broadinstitute.org/gatk/documentation/(https://www.broadinstitute.org/gatk/guide/article?id=1259)).

#### c. Specify the desired truth sensitivity threshold values that the program should use to generate tranches

- First tranche threshold 100.0
- Second tranche threshold 99.9
- Third tranche threshold 99.0
- Fourth tranche threshold 90.0

Tranches are essentially slices of variants, ranked by VQSLOD, bounded by the threshold values specified in this step. The threshold values themselves refer to the sensitivity we can obtain when we apply them to the call sets that the program uses to train the model. The idea is that the lowest tranche is highly specific but less sensitive (there are very few false positives but potentially many false negatives, i.e. missing calls), and each subsequent tranche in turn introduces additional true positive calls along with a growing number of false positive calls. This allows us to filter variants based on how sensitive we want the call set to be, rather than applying hard filters and then only evaluating how sensitive the call set is using post hoc methods.

#### d. Determine additional model parameters

- Maximum number of Gaussians (`-maxGaussians`) 4

This is the maximum number of Gaussians (*i.e.* clusters of variants that have similar properties) that the program should try to identify when it runs the variational Bayes algorithm that underlies the machine learning method. In essence, this limits the number of different ”profiles” of variants that the program will try to identify. This number should only be increased for datasets that include very many variants.

------

### 5. Build the Indel recalibration model

#### Action

Run the following GATK command:

```
java -jar GenomeAnalysisTK.jar \ 
    -T VariantRecalibrator \ 
    -R reference.fa \ 
    -input recalibrated_snps_raw_indels.vcf \ 
    -resource:mills,known=false,training=true,truth=true,prior=12.0 Mills_and_1000G_gold_standard.indels.b37.sites.vcf \
    -resource:dbsnp,known=true,training=false,truth=false,prior=2.0 dbsnp.b37.vcf \
    -an QD \
    -an DP \ 
    -an FS \ 
    -an SOR \ 
    -an MQRankSum \ 
    -an ReadPosRankSum \ 
    -an InbreedingCoeff
    -mode INDEL \ 
    -tranche 100.0 -tranche 99.9 -tranche 99.0 -tranche 90.0 \ 
    --maxGaussians 4 \ 
    -recalFile recalibrate_INDEL.recal \ 
    -tranchesFile recalibrate_INDEL.tranches \ 
    -rscriptFile recalibrate_INDEL_plots.R 
```

#### Expected Result

This creates several files. The most important file is the recalibration report, called `recalibrate_INDEL.recal`, which contains the recalibration data. This is what the program will use in the next step to generate a VCF file in which the variants are annotated with their recalibrated quality scores. There is also a file called `recalibrate_INDEL.tranches`, which contains the quality score thresholds corresponding to the tranches specified in the original command. Finally, if your installation of R and the other required libraries was done correctly, you will also find some PDF files containing plots. These plots illustrated the distribution of variants according to certain dimensions of the model.

For detailed instructions on how to interpret these plots, please refer to the online GATK documentation.

------

### 6. Apply the desired level of recalibration to the Indels in the call set

#### Action

Run the following GATK command:

```
java -jar GenomeAnalysisTK.jar \ 
    -T ApplyRecalibration \ 
    -R reference.fa \ 
    -input recalibrated_snps_raw_indels.vcf \ 
    -mode INDEL \ 
    --ts_filter_level 99.0 \ 
    -recalFile recalibrate_INDEL.recal \ 
    -tranchesFile recalibrate_INDEL.tranches \ 
    -o recalibrated_variants.vcf 
```

#### Expected Result

This creates a new VCF file, called `recalibrated_variants.vcf`, which contains all the original variants from the original `recalibrated_snps_raw_indels.vcf` file, but now the Indels are also annotated with their recalibrated quality scores (VQSLOD) and either `PASS` or `FILTER` depending on whether or not they are included in the selected tranche.

Here we are taking the second lowest of the tranches specified in the original recalibration command. This means that we are applying to our data set the level of sensitivity that would allow us to retrieve 99% of true variants from the truth training sets of HapMap and Omni SNPs. If we wanted to be more specific (and therefore have less risk of including false positives, at the risk of missing real sites) we could take the very lowest tranche, which would only retrieve 90% of the truth training sites. If we wanted to be more sensitive (and therefore less specific, at the risk of including more false positives) we could take the higher tranches. In our Best Practices documentation, we recommend taking the second highest tranche (99.9%) which provides the highest sensitivity you can get while still being acceptably specific.