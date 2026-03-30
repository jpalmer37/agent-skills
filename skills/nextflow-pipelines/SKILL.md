---
name: Nextflow Pipelines
description: Create production-quality, containerized bioinformatics pipelines with Nextflow DSL2. Use when building portable pipelines with container support, running workflows on HPC/cloud platforms, or following nf-core best practices for reproducible analysis.
---

# Nextflow Pipelines

Build scalable, reproducible bioinformatics pipelines using Nextflow DSL2. This skill guides you through creating production-quality workflows with proper container management, version tracking, and nf-core conventions.

## When to Use This Skill

- Building multi-step bioinformatics workflows (RNA-seq, variant calling, etc.)
- Creating portable pipelines that run on local machines, HPC clusters, or cloud
- Following nf-core community standards and best practices
- Managing tool dependencies with containers (Docker/Singularity)
- Tracking provenance and software versions for reproducibility

## Core Concepts

**Process** - Self-contained execution unit with inputs, outputs, and a script
**Channel** - Data flow pipeline connecting processes
**Module** - Reusable process definition
**Subworkflow** - Composed set of processes
**Profile** - Execution configuration (docker, singularity, cluster)

## Version Compatibility

Examples tested with: Nextflow 23.10+, FastQC 0.12+, MultiQC 1.21+, Salmon 1.10+, fastp 0.23+, BCFtools 1.20+, Samtools 1.20+

Before using code patterns, verify installed versions:
```bash
nextflow -version
<tool> --version
```

If code fails due to version mismatch, check `<tool> --help` to confirm flags and adapt accordingly.

## Quick Start

### Minimal Pipeline

```groovy
#!/usr/bin/env nextflow

params.reads = "data/*_{1,2}.fq.gz"
params.outdir = "results"

process FASTQC {
    publishDir "${params.outdir}/qc", mode: 'copy'

    input:
    tuple val(sample_id), path(reads)

    output:
    path("*.html"), emit: html
    path("*.zip"),  emit: zip

    script:
    """
    fastqc ${reads}
    """
}

workflow {
    Channel.fromFilePairs(params.reads)
        | FASTQC
}
```

Run with:
```bash
nextflow run main.nf
nextflow run main.nf -resume  # Resume after failure
```

## Production Process Template

Every process should follow the nf-core pattern with container management and version tracking:

```groovy
process SAMTOOLS_SORT {
    tag "$meta.id"
    label 'process_medium'

    conda 'bioconda::samtools=1.20'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/samtools:1.20--h143571b_1'
    } else {
        container 'quay.io/biocontainers/samtools:1.20--h143571b_1'
    }

    input:
    tuple val(meta), path(bam)

    output:
    tuple val(meta), path("*.sorted.bam"), emit: bam
    path "versions.yml"                   , emit: versions

    script:
    def prefix = task.ext.prefix ?: "${meta.id}"
    """
    samtools sort \\
        -@ ${task.cpus} \\
        -o ${prefix}.sorted.bam \\
        ${bam}

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        samtools: \$(samtools --version 2>&1 | head -n1 | sed 's/^.*samtools //; s/ .*\$//')
    END_VERSIONS
    """
}
```

**Key elements:**
- `tag` - Track sample in execution logs
- `label` - Resource allocation category
- Tri-container pattern (conda/singularity/docker)
- `versions.yml` output for provenance
- `meta` map for sample metadata
- `task.ext.prefix` for configurable output names
- `task.cpus` for resource-aware execution

See **[references/container-management.md](references/container-management.md)** for container patterns.
See **[references/version-tracking.md](references/version-tracking.md)** for provenance tracking.
See **[references/best-practices.md](references/best-practices.md)** for nf-core conventions.

## Modular Pipeline Structure

Organize processes into reusable modules:

```groovy
// modules/local/fastqc.nf
process FASTQC {
    tag "$meta.id"
    label 'process_low'
    
    conda 'bioconda::fastqc=0.12.1'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/fastqc:0.12.1--hdfd78af_0'
    } else {
        container 'quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0'
    }

    input:
    tuple val(meta), path(reads)

    output:
    tuple val(meta), path("*.html"), emit: html
    tuple val(meta), path("*.zip") , emit: zip
    path "versions.yml"             , emit: versions

    script:
    """
    fastqc -t ${task.cpus} ${reads}

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        fastqc: \$(fastqc --version | sed 's/FastQC v//')
    END_VERSIONS
    """
}
```

```groovy
// main.nf
#!/usr/bin/env nextflow

include { FASTQC } from './modules/local/fastqc'
include { FASTP  } from './modules/local/fastp'

workflow {
    reads_ch = Channel.fromFilePairs(params.reads)
        .map { id, files -> [[ id: id, single_end: false ], files] }
    
    FASTQC(reads_ch)
    FASTP(reads_ch)
}
```

## Configuration File

```groovy
// nextflow.config
params {
    // Input/output
    input  = null
    outdir = 'results'
    
    // Reference
    genome = null
    
    // Options
    skip_qc = false
    
    // Resource limits
    max_cpus   = 16
    max_memory = '128.GB'
    max_time   = '240.h'
}

process {
    // Default resources
    cpus   = 2
    memory = 4.GB
    time   = 1.h
    
    // Resource labels
    withLabel: 'process_single' {
        cpus   = 1
        memory = 4.GB
        time   = 1.h
    }
    withLabel: 'process_low' {
        cpus   = 2
        memory = 4.GB
        time   = 1.h
    }
    withLabel: 'process_medium' {
        cpus   = 8
        memory = 16.GB
        time   = 4.h
    }
    withLabel: 'process_high' {
        cpus   = 16
        memory = 64.GB
        time   = 12.h
    }

    // Cap resources to system/cluster limits
    resourceLimits = [
        cpus:   params.max_cpus,
        memory: params.max_memory,
        time:   params.max_time
    ]
}

profiles {
    docker {
        docker.enabled = true
        docker.runOptions = '-u $(id -u):$(id -g)'
    }
    singularity {
        singularity.enabled = true
        singularity.autoMounts = true
    }
    conda {
        conda.enabled = true
    }
    slurm {
        process.executor = 'slurm'
        process.queue = 'normal'
    }
}
```

## Container Dependency Management

**Always use the tri-container pattern** for maximum portability:

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
    consensus = "${prefix}.bcftools.consensus.fasta"
    sequenceID = "${sample}_${segment}"
    """
    # Filter low coverage regions
    zcat $mosdepth_per_base | awk '\$4<${low_coverage}' > low_cov.bed

    # Remove ambiguous bases from reference
    awk '/^>/ {print; next} {gsub(/[RYSWKMBDHVryswkmbdhv]/, "N"); print}' $fasta > ${fasta}.no_ambiguous.fasta

    # Filter variants by allele frequency
    bcftools filter \\
        -Oz \\
        -o no_low_af_indels.vcf.gz \\
        -e "TYPE != 'SNP' && FMT/VAF < ${major_allele_fraction}" \\
        $vcf

    tabix no_low_af_indels.vcf.gz

    # Generate consensus
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

**Why this pattern:**
- `conda` - Enables local development without containers
- `if/else` block - Selects Singularity (HPC) or Docker (cloud/local) at runtime
- Pinned versions - Ensures reproducibility
- BioContainers - Pre-built images for most bioinformatics tools

**Find container tags:**
- BioContainers: `https://biocontainers.pro/tools/<toolname>`
- Quay.io: `https://quay.io/repository/biocontainers/<toolname>?tab=tags`

See **[references/container-management.md](references/container-management.md)** for detailed patterns including multi-tool containers and custom images.

## Version Tracking (Provenance)

Every process **must** emit `versions.yml` for reproducibility:

```groovy
output:
path "versions.yml", emit: versions

script:
"""
# ... process commands ...

cat <<-END_VERSIONS > versions.yml
"${task.process}":
    samtools: \$(samtools --version 2>&1 | head -n1 | sed 's/^.*samtools //; s/ .*\$//')
END_VERSIONS
"""
```

**Collect versions pipeline-wide:**

```groovy
workflow {
    ch_versions = Channel.empty()
    ch_versions = ch_versions.mix(FASTQC.out.versions.first())
    ch_versions = ch_versions.mix(BWA_MEM.out.versions.first())
    ch_versions = ch_versions.mix(SAMTOOLS_SORT.out.versions.first())
    
    CUSTOM_DUMPSOFTWAREVERSIONS(
        ch_versions.collectFile(name: 'collated_versions.yml')
    )
}
```

See **[references/version-tracking.md](references/version-tracking.md)** for extraction commands and collector process examples.

## Channel Operations

### Creating channels

```groovy
// Paired-end reads
Channel.fromFilePairs("data/*_{1,2}.fq.gz", checkIfExists: true)
    .map { id, files -> [[id: id, single_end: false], files] }

// From samplesheet (nf-core pattern)
Channel.fromPath(params.input)
    .splitCsv(header: true)
    .map { row ->
        def meta = [
            id: row.sample,
            single_end: row.single_end.toBoolean(),
            strandedness: row.strandedness ?: 'auto'
        ]
        def reads = meta.single_end 
            ? [file(row.fastq_1)] 
            : [file(row.fastq_1), file(row.fastq_2)]
        [meta, reads]
    }

// Single reference file
genome_ch = Channel.value(file(params.genome_fasta))
```

### Transforming channels

```groovy
// Join channels by key
bam_ch.join(bai_ch)  // [[sample, bam.file], [sample, bai.file]] -> [sample, bam, bai]

// Combine with reference
reads_ch.combine(genome_ch)

// Group by metadata
results_ch.groupTuple()  // Groups by first element (key)

// Branch by condition
reads_ch.branch {
    single: it[0].single_end
    paired: true
}
```

See **[references/channel-patterns.md](references/channel-patterns.md)** for advanced operations (transpose, multiMap, cross, etc.).

## Subworkflows

Group related processes into reusable subworkflows:

```groovy
// subworkflows/local/qc.nf
include { FASTQC  } from '../../modules/local/fastqc'
include { MULTIQC } from '../../modules/local/multiqc'

workflow QC_WORKFLOW {
    take:
    reads    // channel: [ meta, [ reads ] ]

    main:
    ch_versions = Channel.empty()
    
    FASTQC(reads)
    ch_versions = ch_versions.mix(FASTQC.out.versions.first())
    
    MULTIQC(
        FASTQC.out.zip.map { meta, zip -> zip }.collect()
    )
    ch_versions = ch_versions.mix(MULTIQC.out.versions)

    emit:
    html     = MULTIQC.out.html
    versions = ch_versions
}
```

```groovy
// main.nf
include { QC_WORKFLOW   } from './subworkflows/local/qc'
include { ALIGN_WORKFLOW } from './subworkflows/local/align'

workflow {
    reads_ch = Channel.fromPath(params.input)
        .splitCsv(header: true)
        .map { row -> [[id: row.sample], [file(row.r1), file(row.r2)]] }
    
    QC_WORKFLOW(reads_ch)
    ALIGN_WORKFLOW(reads_ch)
    
    // Collect all versions
    ch_versions = Channel.empty()
        .mix(QC_WORKFLOW.out.versions)
        .mix(ALIGN_WORKFLOW.out.versions)
}
```

## HPC and Cloud Execution

### SLURM cluster

```groovy
// conf/slurm.config
process {
    executor = 'slurm'
    queue = 'normal'
    clusterOptions = '--account=myproject'
    
    errorStrategy { task.exitStatus in [137,140,143] ? 'retry' : 'finish' }
    maxRetries = 3
}

executor {
    queueSize = 100
    submitRateLimit = '10 sec'
}
```

```bash
nextflow run main.nf -profile slurm,singularity
```

### AWS Batch

```groovy
// conf/awsbatch.config
process {
    executor = 'awsbatch'
    queue = 'my-batch-queue'
}

aws {
    region = 'us-east-1'
    batch.cliPath = '/usr/local/bin/aws'
}
```

```bash
nextflow run main.nf -profile awsbatch -bucket-dir s3://my-bucket/work
```

### Google Cloud Batch

```groovy
// conf/google.config
process {
    executor = 'google-batch'
}

google {
    batch.location = 'us-central1'
    project        = 'my-project-id'
}
```

```bash
nextflow run main.nf -profile google -bucket-dir gs://my-bucket/work
```

> **Note:** `google-lifesciences` was replaced by `google-batch` (Google Cloud Batch) in Nextflow v22.07. The config key also changed from `google.region` to `google.batch.location`.

## Resource Management

### Resource labels

Define standard resource tiers in `nextflow.config`:

```groovy
process {
    withLabel: 'process_single' {
        cpus   = 1
        memory = 4.GB
        time   = 1.h
    }
    withLabel: 'process_low' {
        cpus   = 2
        memory = 4.GB
        time   = 1.h
    }
    withLabel: 'process_medium' {
        cpus   = 8
        memory = 16.GB
        time   = 4.h
    }
    withLabel: 'process_high' {
        cpus   = 16
        memory = 64.GB
        time   = 12.h
    }
}
```

Apply labels in processes:

```groovy
process BWA_MEM {
    label 'process_high'
    // Automatically gets 16 CPUs, 64 GB, 12h
}
```

### Dynamic retry with increasing resources

```groovy
process MEMORY_INTENSIVE {
    label 'process_high'
    errorStrategy { task.exitStatus in [137,140,143] ? 'retry' : 'finish' }
    maxRetries 3
    memory { 16.GB * task.attempt }  // Scales linearly: 16 GB → 32 GB → 48 GB
    time   { 4.h * task.attempt }
}
```

Since Nextflow v24.10 you can scale based on *actual measured metrics* from the previous attempt rather than just the attempt count:

```groovy
process MEMORY_INTENSIVE {
    memory { task.previousTrace?.memory ? task.previousTrace.memory * 2 : 16.GB }
}
```

## Error Handling and Resume

### Error strategies

```groovy
process CRITICAL_STEP {
    errorStrategy 'terminate'  // Stop entire pipeline (default)
}

process RISKY_STEP {
    errorStrategy 'retry'
    maxRetries 3
    memory { 8.GB * task.attempt }
}

process OPTIONAL_STEP {
    errorStrategy 'ignore'  // Skip failures, continue pipeline
}
```

### Resume from failure

```bash
# Resume execution using cached results
nextflow run main.nf -resume

# View previous run logs
nextflow log

# Clean work directory
nextflow clean -f

# View specific run details
nextflow log <run_name> -f script,status,exit
```

Nextflow caches all successfully completed tasks. Use `-resume` to skip them after fixing errors.

## Logging and Reporting

### Pipeline header

```groovy
log.info """\
    R N A - S E Q   P I P E L I N E
    ================================
    input        : ${params.input}
    outdir       : ${params.outdir}
    genome       : ${params.genome}
    profile      : ${workflow.profile}
    """
    .stripIndent()
```

### Completion handler

```groovy
workflow.onComplete {
    log.info "Pipeline completed at: ${workflow.complete}"
    log.info "Duration            : ${workflow.duration}"
    log.info "Success             : ${workflow.success}"
    log.info "Work directory      : ${workflow.workDir}"
    log.info "Exit status         : ${workflow.exitStatus}"
}
```

### Execution reports

```bash
# Generate HTML report
nextflow run main.nf -with-report report.html

# Generate timeline
nextflow run main.nf -with-timeline timeline.html

# Generate DAG visualization
nextflow run main.nf -with-dag flowchart.png

# All reports
nextflow run main.nf \\
    -with-report report.html \\
    -with-timeline timeline.html \\
    -with-dag dag.png
```

## Dry-Run Testing (`stub:`)

The `stub:` block runs instead of `script:` when the pipeline is invoked with `-stub` or `-stub-run`. It lets you verify workflow logic, channel flow, and file-name expectations without running expensive jobs — useful for rapid iteration and CI checks.

```groovy
process BWA_MEM {
    input:
    tuple val(meta), path(reads)
    path genome

    output:
    tuple val(meta), path("*.bam"), emit: bam

    stub:
    """
    touch ${meta.id}.bam
    """

    script:
    """
    bwa mem -t ${task.cpus} ${genome} ${reads} | samtools view -bS - > ${meta.id}.bam
    """
}
```

```bash
nextflow run main.nf -stub        # runs stub blocks, skips real computation
```

The stub block should create the expected output files so downstream processes can continue. This is much faster than a test profile running real data.

## Complete Example: RNA-seq Pipeline

See **[examples/rnaseq.nf](examples/rnaseq.nf)** for a production-quality RNA-seq pipeline with:
- Container management (tri-container pattern)
- Version tracking (versions.yml)
- Quality control (FastQC, MultiQC)
- Read trimming (fastp)
- Quantification (Salmon)
- Proper channel handling
- Resource labels

## Best Practices Checklist

When building Nextflow pipelines, ensure:

- [ ] Use DSL2 (the default since v22.03 — no need to add `nextflow.enable.dsl=2`)
- [ ] Apply tri-container pattern to all processes (conda + docker + singularity)
- [ ] Emit `versions.yml` from every process
- [ ] Use `meta` map for sample metadata (nf-core pattern)
- [ ] Add `tag` to processes for execution tracking
- [ ] Use `label` for resource allocation
- [ ] Set `publishDir` for final outputs
- [ ] Use `checkIfExists: true` when creating channels from files
- [ ] Organize modules in `modules/local/` directory
- [ ] Use subworkflows for related process groups
- [ ] Define profiles for different execution environments
- [ ] Add pipeline header logging
- [ ] Implement workflow completion handler
- [ ] Test with `-resume` to verify caching works
- [ ] Validate parameter inputs

## Reference Documentation

For detailed information, see:

- **[references/container-management.md](references/container-management.md)** - Tri-container pattern, BioContainers, custom containers
- **[references/version-tracking.md](references/version-tracking.md)** - Provenance tracking, version extraction, collectors
- **[references/best-practices.md](references/best-practices.md)** - nf-core conventions, project structure, testing
- **[references/channel-patterns.md](references/channel-patterns.md)** - Advanced channel operations, joining, branching

## Additional Resources

- nf-core modules: `https://nf-co.re/modules`
- nf-core pipelines: `https://nf-co.re/pipelines`
- Nextflow documentation: `https://www.nextflow.io/docs/latest/`
- BioContainers: `https://biocontainers.pro/`

## Related Skills

- `bioskills` - 425+ bioinformatics tools and workflows
- Workflow alternatives: Snakemake, CWL, WDL
