# Version Tracking (versions.yml)

Every Nextflow process should emit a `versions.yml` file capturing the exact tool versions
used during execution. This enables full provenance tracking and reproducibility auditing.

## Table of Contents

- [Version Tracking (versions.yml)](#version-tracking-versionsyml)
  - [Table of Contents](#table-of-contents)
  - [Why versions.yml](#why-versionsyml)
  - [Basic Pattern](#basic-pattern)
  - [Format Specification](#format-specification)
    - [YAML structure](#yaml-structure)
    - [Key rules](#key-rules)
    - [Correct heredoc syntax](#correct-heredoc-syntax)
  - [Version Extraction Commands](#version-extraction-commands)
    - [Custom script version tracking](#custom-script-version-tracking)
  - [Multi-Tool Processes](#multi-tool-processes)
    - [Process using R and Python scripts](#process-using-r-and-python-scripts)
  - [Collecting Versions Pipeline-Wide](#collecting-versions-pipeline-wide)
    - [Custom version collector process](#custom-version-collector-process)
    - [Collecting in the workflow block](#collecting-in-the-workflow-block)
  - [Complete Examples](#complete-examples)
    - [Minimal process with versions](#minimal-process-with-versions)

## Why versions.yml

- Captures runtime tool versions, not just declared versions
- Enables reproducibility auditing across pipeline runs
- Required by nf-core module standards
- Allows automated software bill of materials (SBOM) generation
- Detects version drift between environments

## Basic Pattern

Every process MUST include `versions.yml` as an output and generate it in the script block:

```groovy
process SAMTOOLS_SORT {
    tag "$sample_id"
    label 'process_medium'

    conda 'bioconda::samtools=1.20'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/samtools:1.20--h143571b_1'
    } else {
        container 'quay.io/biocontainers/samtools:1.20--h143571b_1'
    }

    input:
    tuple val(sample_id), path(bam)

    output:
    tuple val(sample_id), path("*.sorted.bam"), emit: bam
    path "versions.yml"                        , emit: versions

    script:
    """
    samtools sort -@ ${task.cpus} -o ${sample_id}.sorted.bam ${bam}

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        samtools: \$(samtools --version 2>&1 | head -n1 | sed 's/^.*samtools //; s/ .*\$//')
    END_VERSIONS
    """
}
```

## Format Specification

### YAML structure
```yaml
"PROCESS_NAME":
    toolname: "version_string"
```

### Key rules
1. The process name key MUST use `${task.process}` (resolves to the fully qualified process name)
2. Tool names use lowercase
3. Version strings are plain strings (no leading `v`)
4. One entry per tool used in the process
5. Use heredoc `cat <<-END_VERSIONS` with leading tab stripping (`<<-` not `<<`)
6. Escape `$` in shell commands with `\$` since Nextflow interpolates `$` in script blocks

### Correct heredoc syntax
```groovy
script:
"""
# ... main commands ...

cat <<-END_VERSIONS > versions.yml
"${task.process}":
    toolname: \$(toolname --version 2>&1 | head -n1 | sed 's/^.*toolname //; s/ .*\$//')
END_VERSIONS
"""
```

**Important:** The `END_VERSIONS` closing marker must be at the start of a line (no leading spaces).
Indentation inside the heredoc uses tabs (stripped by `<<-`).

## Version Extraction Commands

Common patterns for extracting version strings from bioinformatics tools:

| Tool | Version Command |
|------|----------------|
| samtools | `samtools --version 2>&1 \| head -n1 \| sed 's/^.*samtools //; s/ .*\$//'` |
| bcftools | `bcftools --version 2>&1 \| head -n1 \| sed 's/^.*bcftools //; s/ .*\$//'` |
| bwa | `bwa 2>&1 \| grep Version \| sed 's/Version: //'` |
| bwa-mem2 | `bwa-mem2 version 2>&1 \| head -n1` |
| fastp | `fastp --version 2>&1 \| sed -e 's/fastp //'` |
| fastqc | `fastqc --version \| sed -e 's/FastQC v//'` |
| gatk | `gatk --version 2>&1 \| head -n1 \| sed 's/.*GATK) v//; s/ .*\$//'` |
| picard | `picard MarkDuplicates --version 2>&1 \| sed 's/Version://'` |
| salmon | `salmon --version \| sed -e 's/salmon //'` |
| star | `STAR --version \| sed -e 's/STAR_//'` |
| multiqc | `multiqc --version \| sed -e 's/.*version //'` |
| bedtools | `bedtools --version \| sed -e 's/bedtools v//'` |
| trimmomatic | `trimmomatic -version 2>&1 \| head -n1` |
| bowtie2 | `bowtie2 --version \| head -n1 \| sed 's/.*version //'` |
| hisat2 | `hisat2 --version \| head -n1 \| sed 's/.*version //'` |
| minimap2 | `minimap2 --version` |
| snpeff | `snpEff -version 2>&1 \| head -n1 \| sed 's/SnpEff //; s/ .*\$//'` |
| mosdepth | `mosdepth --version 2>&1 \| sed 's/mosdepth //'` |
| tabix | `tabix --version 2>&1 \| head -n1 \| sed 's/.*) //; s/ .*\$//'` |
| python | `python --version \| sed 's/Python //'` |
| r-base | `R --version \| head -n1 \| sed 's/.*version //; s/ .*\$//'` |

### Custom script version tracking
For in-house scripts, embed a version variable:

```groovy
script:
"""
# ... script commands ...

cat <<-END_VERSIONS > versions.yml
"${task.process}":
    python: \$(python --version | sed 's/Python //')
    custom_script: "1.0.0"
END_VERSIONS
"""
```

## Multi-Tool Processes

When a process uses multiple tools, list all of them:

```groovy
process BCF_CONSENSUS {
    tag "$sample|$segment|$ref_id"
    label 'process_medium'

    conda 'bioconda::bcftools=1.20 conda-forge::gsl=2.7'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/bcftools:1.20--h8b25389_0'
    } else {
        container 'quay.io/biocontainers/bcftools:1.20--h8b25389_0'
    }

    input:
    tuple val(sample), val(segment), val(ref_id), path(fasta), path(vcf), path(mosdepth_per_base)
    val(low_coverage)
    val(major_allele_fraction)

    output:
    tuple val(sample), path(consensus), emit: fasta
    path "versions.yml"                , emit: versions

    script:
    def prefix = fluPrefix(sample, segment, ref_id)
    consensus    = "${prefix}.bcftools.consensus.fasta"
    sequenceID   = "${sample}_${segment}"
    """
    zcat $mosdepth_per_base | awk '\$4<${low_coverage}' > low_cov.bed

    awk '/^>/ {print; next} {gsub(/[RYSWKMBDHVryswkmbdhv]/, "N"); print}' $fasta > ${fasta}.no_ambiguous.fasta

    bcftools filter \\
        -Oz \\
        -o no_low_af_indels.vcf.gz \\
        -e "TYPE != 'SNP' && FMT/VAF < ${major_allele_fraction}" \\
        $vcf

    tabix no_low_af_indels.vcf.gz

    bcftools consensus \\
        -f ${fasta}.no_ambiguous.fasta \\
        -m low_cov.bed \\
        no_low_af_indels.vcf.gz > $consensus

    sed -i -E "s/^>(.*)/>$sequenceID/g" $consensus

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        bcftools: \$(bcftools --version 2>&1 | head -n1 | sed 's/^.*bcftools //; s/ .*\$//')
    END_VERSIONS
    """
}
```

### Process using R and Python scripts
```groovy
process DESEQ2 {
    // ... container directives ...

    output:
    path "*.csv"        , emit: results
    path "versions.yml" , emit: versions

    script:
    """
    Rscript deseq2_analysis.R

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        r-base: \$(R --version | head -n1 | sed 's/.*version //; s/ .*\$//')
        bioconductor-deseq2: \$(Rscript -e 'cat(as.character(packageVersion("DESeq2")))')
    END_VERSIONS
    """
}
```

## Collecting Versions Pipeline-Wide

### Custom version collector process
```groovy
process CUSTOM_DUMPSOFTWAREVERSIONS {
    publishDir "${params.outdir}/pipeline_info", mode: 'copy'

    input:
    path versions

    output:
    path "software_versions.yml"    , emit: yml
    path "software_versions_mqc.yml", emit: mqc_yml

    script:
    """
    #!/usr/bin/env python3
    import yaml
    import platform
    from collections import defaultdict

    versions = defaultdict(dict)
    for f in sorted("${versions}".split()):
        if f == 'versions.yml' or f.endswith('versions.yml'):
            with open(f) as fh:
                data = yaml.safe_load(fh)
                if data:
                    for process, tools in data.items():
                        for tool, version in tools.items():
                            versions[tool][process] = str(version)

    unique_versions = {}
    for tool, procs in sorted(versions.items()):
        unique_versions[tool] = sorted(set(procs.values()))[-1]

    unique_versions['Nextflow'] = "${nextflow.version}"
    unique_versions['Python'] = platform.python_version()

    with open('software_versions.yml', 'w') as f:
        yaml.dump(unique_versions, f, default_flow_style=False)

    # MultiQC-compatible format
    mqc = {
        'id': 'software_versions',
        'section_name': 'Software Versions',
        'plot_type': 'html',
        'data': '<dl>' + ''.join(
            f'<dt>{k}</dt><dd>{v}</dd>' for k, v in sorted(unique_versions.items())
        ) + '</dl>'
    }
    with open('software_versions_mqc.yml', 'w') as f:
        yaml.dump(mqc, f, default_flow_style=False)
    """
}
```

### Collecting in the workflow block
```groovy
workflow {
    // ... process calls ...

    // Collect all versions
    ch_versions = Channel.empty()
    ch_versions = ch_versions.mix(FASTP.out.versions.first())
    ch_versions = ch_versions.mix(BWA_MEM.out.versions.first())
    ch_versions = ch_versions.mix(SAMTOOLS_SORT.out.versions.first())
    ch_versions = ch_versions.mix(GATK_HAPLOTYPECALLER.out.versions.first())

    CUSTOM_DUMPSOFTWAREVERSIONS(ch_versions.collectFile(name: 'collated_versions.yml'))
}
```

**Note:** Use `.first()` when collecting versions from processes that run per-sample to avoid
duplicating the same version info. Use `.collect()` only if versions may differ across samples.

## Complete Examples

### Minimal process with versions
```groovy
process FASTQC {
    tag "$sample_id"
    label 'process_low'

    conda 'bioconda::fastqc=0.12.1'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/fastqc:0.12.1--hdfd78af_0'
    } else {
        container 'quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0'
    }

    input:
    tuple val(sample_id), path(reads)

    output:
    tuple val(sample_id), path("*.html"), emit: html
    tuple val(sample_id), path("*.zip") , emit: zip
    path "versions.yml"                  , emit: versions

    script:
    """
    fastqc --threads ${task.cpus} ${reads}

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        fastqc: \$(fastqc --version | sed -e 's/FastQC v//')
    END_VERSIONS
    """
}
```
