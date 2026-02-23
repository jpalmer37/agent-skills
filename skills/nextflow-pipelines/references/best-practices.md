# Nextflow Best Practices

Production-quality Nextflow pipelines follow conventions established by the nf-core community.
Apply these patterns to build robust, portable, and maintainable workflows.

## Table of Contents

- [Project Structure](#project-structure)
- [Process Design](#process-design)
- [Naming Conventions](#naming-conventions)
- [Input/Output Patterns](#inputoutput-patterns)
- [Parameter Design](#parameter-design)
- [Error Handling and Retry](#error-handling-and-retry)
- [Logging and Reporting](#logging-and-reporting)
- [Testing](#testing)
- [nf-core Conventions](#nf-core-conventions)

## Project Structure

Standard layout for a production pipeline:

```
my-pipeline/
├── main.nf                    # Entry point workflow
├── nextflow.config            # Default configuration
├── nextflow_schema.json       # Parameter validation schema (optional)
├── modules/
│   ├── local/                 # Pipeline-specific modules
│   │   ├── fastp.nf
│   │   └── custom_script.nf
│   └── nf-core/               # Imported nf-core modules
│       ├── fastqc/
│       └── samtools/sort/
├── subworkflows/
│   ├── local/
│   │   └── input_check.nf
│   └── nf-core/
├── conf/
│   ├── base.config            # Base resource defaults
│   ├── modules.config         # Module-specific publishDir/ext settings
│   └── test.config            # Test profile parameters
├── lib/                       # Groovy helper classes/functions
│   └── WorkflowMain.groovy
├── bin/                       # Executable scripts (added to PATH)
│   └── check_samplesheet.py
├── assets/                    # Static assets (schemas, adapters)
│   └── samplesheet_schema.json
└── docs/
    └── usage.md
```

### Key directories
- `bin/` scripts are automatically added to `$PATH` in process containers
- `lib/` Groovy classes are auto-loaded and available in workflow scripts
- `conf/` separates config concerns (resources, modules, test data)

## Process Design

### Atomic processes
Each process should do ONE thing well:

```groovy
// GOOD: Single responsibility
process SAMTOOLS_INDEX {
    input:
    tuple val(sample_id), path(bam)
    output:
    tuple val(sample_id), path("*.bai"), emit: bai
    // ...
}

// BAD: Too many responsibilities
process ALIGN_SORT_INDEX_MARKDUP {
    // Too much in one process - hard to debug, cache, and reuse
}
```

### Use tag for tracking
```groovy
process ALIGN {
    tag "$sample_id"         // Simple sample tracking
    tag "$sample|$lane"      // Multi-value tracking
}
```

### Use label for resource allocation
```groovy
process FASTQC {
    label 'process_low'      // 2 CPUs, 4 GB, 1h
}
process BWA_MEM {
    label 'process_high'     // 16 CPUs, 64 GB, 12h
}
```

### publishDir patterns
```groovy
process FASTP {
    publishDir "${params.outdir}/trimmed", mode: 'copy', pattern: '*.fq.gz'
    publishDir "${params.outdir}/qc",      mode: 'copy', pattern: '*.json'
    publishDir "${params.outdir}/qc",      mode: 'copy', pattern: '*.html'
    // Separate patterns for different output types
}
```

Use `mode: 'copy'` for final outputs, `mode: 'symlink'` (default) for intermediate files.

### Script block conventions
```groovy
script:
def prefix = task.ext.prefix ?: "${sample_id}"
def args   = task.ext.args   ?: ''
"""
samtools sort \\
    -@ ${task.cpus} \\
    ${args} \\
    -o ${prefix}.sorted.bam \\
    ${bam}
"""
```

- Use `task.ext.prefix` and `task.ext.args` for configurable process behavior
- Use `\\` line continuation for readability
- Reference `task.cpus`, `task.memory` for resource-aware commands

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Process | UPPER_SNAKE_CASE | `SAMTOOLS_SORT` |
| Workflow | PascalCase or UPPER_SNAKE_CASE | `QC_WORKFLOW` |
| Channel | lower_snake_case with `_ch` suffix | `reads_ch` |
| Parameter | lower_snake_case | `params.outdir` |
| Module file | lowercase matching process | `samtools/sort/main.nf` |
| Label | `process_` prefix | `process_high` |

## Input/Output Patterns

### Tuple-based I/O (preferred)
Carry metadata through the pipeline:

```groovy
// Input: tuple with metadata map
input:
tuple val(meta), path(reads)

// meta is a Groovy map: [id: 'sample1', single_end: false, strandedness: 'reverse']

// Output: preserve metadata
output:
tuple val(meta), path("*.bam"), emit: bam
```

### Samplesheet parsing
```groovy
Channel.fromPath(params.input)
    .splitCsv(header: true)
    .map { row ->
        def meta = [id: row.sample, single_end: row.single_end.toBoolean()]
        def reads = meta.single_end
            ? [file(row.fastq_1, checkIfExists: true)]
            : [file(row.fastq_1, checkIfExists: true), file(row.fastq_2, checkIfExists: true)]
        [meta, reads]
    }
    .set { reads_ch }
```

### Optional inputs
```groovy
input:
tuple val(meta), path(bam), path(bai)
path(target_bed)  // can be empty: pass [] from workflow

script:
def bed_arg = target_bed ? "-L ${target_bed}" : ''
"""
gatk HaplotypeCaller ${bed_arg} -I ${bam} -O output.vcf
"""
```

## Parameter Design

### Use params block in nextflow.config
```groovy
params {
    // Input
    input       = null
    outdir      = './results'
    genome      = null

    // Tool options
    skip_qc     = false
    save_trimmed = false

    // Resource limits
    max_cpus    = 16
    max_memory  = '128.GB'
    max_time    = '240.h'
}
```

### Validate required parameters
```groovy
// In main.nf or lib/WorkflowMain.groovy
if (!params.input) {
    error "Parameter 'input' is required. Use --input <samplesheet.csv>"
}
```

### Parameter schema validation
Use `nextflow_schema.json` for automatic validation and help text generation:
```bash
nf-core schema build   # Interactive schema builder
nextflow run main.nf --help  # Auto-generated help from schema
```

## Error Handling and Retry

### Dynamic resource retry
```groovy
process MEMORY_INTENSIVE {
    label 'process_high'
    errorStrategy { task.exitStatus in [137, 140, 143] ? 'retry' : 'finish' }
    maxRetries 3
    memory { 16.GB * task.attempt }
    time   { 4.h  * task.attempt }

    // Exit codes: 137=OOM killed, 140=SIGTERM timeout, 143=SIGTERM
}
```

### Error strategy options
```groovy
errorStrategy 'terminate'   // Stop pipeline on first error (default)
errorStrategy 'finish'      // Complete running tasks, then stop
errorStrategy 'ignore'      // Skip failed tasks, continue pipeline
errorStrategy 'retry'       // Retry failed tasks
```

### Resource capping
```groovy
// In conf/base.config
def check_max(obj, type) {
    if (type == 'memory') {
        return obj.compareTo(params.max_memory as nextflow.util.MemoryUnit) == 1
            ? params.max_memory as nextflow.util.MemoryUnit : obj
    } else if (type == 'time') {
        return obj.compareTo(params.max_time as nextflow.util.Duration) == 1
            ? params.max_time as nextflow.util.Duration : obj
    } else if (type == 'cpus') {
        return Math.min(obj, params.max_cpus as int)
    }
}
```

## Logging and Reporting

### Pipeline header
```groovy
log.info """\
    M Y   P I P E L I N E
    ======================
    input    : ${params.input}
    outdir   : ${params.outdir}
    genome   : ${params.genome}
    profile  : ${workflow.profile}
    """
    .stripIndent()
```

### Completion handler
```groovy
workflow.onComplete {
    log.info "Pipeline completed at : ${workflow.complete}"
    log.info "Duration              : ${workflow.duration}"
    log.info "Success               : ${workflow.success}"
    log.info "Work dir              : ${workflow.workDir}"
    log.info "Exit status           : ${workflow.exitStatus}"
    if (!workflow.success) {
        log.error "Pipeline failed. Check .nextflow.log for details."
    }
}
```

### Enable built-in reports
```bash
nextflow run main.nf -with-report report.html -with-timeline timeline.html -with-dag dag.png
```

## Testing

### Test profile
```groovy
// conf/test.config
params {
    input  = 'https://raw.githubusercontent.com/org/repo/main/assets/test_samplesheet.csv'
    genome = 'GRCh38'
    outdir = 'results/test'
    max_cpus   = 2
    max_memory = '6.GB'
    max_time   = '6.h'
}
```

```bash
nextflow run main.nf -profile test,docker
```

### nf-test for unit testing
```groovy
// tests/modules/samtools_sort.nf.test
nextflow_process {
    name "SAMTOOLS_SORT"
    script "modules/local/samtools_sort.nf"
    process "SAMTOOLS_SORT"

    test("should sort BAM file") {
        when {
            process {
                """
                input[0] = [
                    [id: 'test'],
                    file('https://github.com/nf-core/test-datasets/raw/modules/data/genomics/sarscov2/illumina/bam/test.paired_end.sorted.bam')
                ]
                """
            }
        }
        then {
            assert process.success
            assert process.out.bam
        }
    }
}
```

## nf-core Conventions

### Module structure (nf-core style)
```
modules/nf-core/samtools/sort/
├── main.nf          # Process definition
├── meta.yml         # Module metadata
└── tests/
    ├── main.nf.test
    └── tags.yml
```

### Using nf-core tools
```bash
# Install a module
nf-core modules install samtools/sort

# List available modules
nf-core modules list remote

# Create a new module
nf-core modules create samtools/sort

# Lint a module
nf-core modules lint samtools/sort
```

### nf-core module main.nf template
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

    when:
    task.ext.when == null || task.ext.when

    script:
    def prefix = task.ext.prefix ?: "${meta.id}"
    def args   = task.ext.args   ?: ''
    """
    samtools sort \\
        -@ ${task.cpus} \\
        ${args} \\
        -o ${prefix}.sorted.bam \\
        ${bam}

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        samtools: \$(samtools --version 2>&1 | head -n1 | sed 's/^.*samtools //; s/ .*\$//')
    END_VERSIONS
    """
}
```

This template demonstrates all nf-core conventions:
- Tri-container pattern (conda/singularity/docker)
- `task.ext.prefix` and `task.ext.args` for configurability
- `task.ext.when` for conditional execution
- `versions.yml` provenance tracking
- Metadata map (`meta`) for sample tracking
