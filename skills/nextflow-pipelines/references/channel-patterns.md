# Advanced Channel Patterns

Channels are the backbone of Nextflow data flow. Master these patterns to build
flexible, efficient pipelines.

## Table of Contents

- [Channel Factories](#channel-factories)
- [Operators for Transformation](#operators-for-transformation)
- [Joining and Combining](#joining-and-combining)
- [Branching and Filtering](#branching-and-filtering)
- [Collecting and Grouping](#collecting-and-grouping)
- [Metadata Map Pattern](#metadata-map-pattern)
- [Common Pipeline Patterns](#common-pipeline-patterns)

## Channel Factories

### fromFilePairs
```groovy
// Standard paired-end reads
Channel.fromFilePairs("data/*_{1,2}.fq.gz", checkIfExists: true)
// Emits: [sample_id, [read1.fq.gz, read2.fq.gz]]

// Custom grouping size
Channel.fromFilePairs("data/*_{1,2,3}.fq.gz", size: 3)

// Flat (no grouping)
Channel.fromFilePairs("data/*_{1,2}.fq.gz", flat: true)
// Emits: [sample_id, read1.fq.gz, read2.fq.gz]
```

### fromPath
```groovy
Channel.fromPath("data/*.bam", checkIfExists: true)
Channel.fromPath(["data/*.bam", "data/*.cram"])  // Multiple patterns
```

### fromSamplesheet (nf-schema plugin)
```groovy
Channel.fromSamplesheet("input")  // reads from params.input using schema
```

> **Plugin note:** `Channel.fromSamplesheet` was provided by the `nf-validation` plugin, which is now deprecated. Use the `nf-schema` plugin instead — it provides the same API with improvements. Add to `nextflow.config`:
> ```groovy
> plugins {
>     id 'nf-schema@2.1.0'
> }
> ```

### of / value / empty
```groovy
Channel.of('chr1', 'chr2', 'chr3')        // Multiple items
Channel.value(file("ref/genome.fa"))       // Single reusable value
Channel.empty()                            // Empty channel for mixing
```

## Operators for Transformation

### map
```groovy
// Add metadata
Channel.fromPath("data/*.bam")
    .map { bam ->
        def id = bam.baseName.replaceAll(/\.sorted$/, '')
        tuple(id, bam)
    }

// Transform tuple elements
reads_ch.map { sample_id, reads ->
    tuple([id: sample_id, single_end: reads instanceof Path], reads)
}
```

### flatMap
```groovy
// Expand grouped items
Channel.of([1, ['a', 'b']], [2, ['c', 'd']])
    .flatMap { id, items -> items.collect { [id, it] } }
// Emits: [1, 'a'], [1, 'b'], [2, 'c'], [2, 'd']
```

### transpose
```groovy
// Unpack grouped outputs
PROCESS.out.results  // [sample_id, [file1, file2, file3]]
    .transpose()     // [sample_id, file1], [sample_id, file2], [sample_id, file3]
```

### splitCsv / splitText
```groovy
Channel.fromPath("samplesheet.csv")
    .splitCsv(header: true, sep: ',')
    .map { row -> tuple(row.sample, file(row.fastq_1), file(row.fastq_2)) }
```

## Joining and Combining

### join (key-based merge)
```groovy
// Join two channels by first element (key)
bam_ch   = Channel.of(['s1', 'a.bam'], ['s2', 'b.bam'])
index_ch = Channel.of(['s1', 'a.bai'], ['s2', 'b.bai'])

bam_ch.join(index_ch)
// Emits: ['s1', 'a.bam', 'a.bai'], ['s2', 'b.bam', 'b.bai']

// Join with remainder handling
bam_ch.join(index_ch, remainder: true)  // Keep unmatched entries
bam_ch.join(index_ch, failOnMismatch: true)  // Fail if keys don't match
```

### combine (cartesian product)
```groovy
// Combine every sample with a reference
reads_ch.combine(Channel.value(file("ref/genome.fa")))
// If reads_ch has 3 items, emits 3 tuples each with genome.fa

// Combine by key
ch1.combine(ch2, by: 0)  // Match on first element
```

### cross
```groovy
// Match items between channels by key
ch1 = Channel.of(['s1', 'a.bam'], ['s2', 'b.bam'])
ch2 = Channel.of(['s1', 'metadata1'], ['s2', 'metadata2'])
ch1.cross(ch2).map { a, b -> tuple(a[0], a[1], b[1]) }
```

### mix
```groovy
// Merge multiple channels (no ordering guarantee)
ch_versions = Channel.empty()
ch_versions = ch_versions.mix(FASTQC.out.versions.first())
ch_versions = ch_versions.mix(FASTP.out.versions.first())
ch_versions = ch_versions.mix(BWA_MEM.out.versions.first())
```

## Branching and Filtering

### branch
```groovy
reads_ch.branch {
    single_end: it[0].single_end
    paired_end: true  // default/catch-all
}
.set { branched }

ALIGN_SE(branched.single_end)
ALIGN_PE(branched.paired_end)
```

### filter
```groovy
// Filter by closure
reads_ch.filter { meta, reads -> meta.strandedness == 'reverse' }

// Filter by value
Channel.of(1, 2, 3, 4, 5).filter { it > 3 }
```

### multiMap
```groovy
// Create multiple named outputs from one channel
reads_ch.multiMap { meta, reads ->
    fastqc: tuple(meta, reads)
    fastp:  tuple(meta, reads)
}
.set { multi_reads }

FASTQC(multi_reads.fastqc)
FASTP(multi_reads.fastp)
```

## Collecting and Grouping

### collect
```groovy
// Gather all items into a single list
FASTQC.out.zip.collect()  // [file1.zip, file2.zip, ...]

// Collect with flat: false preserves nesting
ch.collect(flat: false)
```

### collectFile
```groovy
// Concatenate files
ch_versions.collectFile(name: 'all_versions.yml', newLine: true)

// Collect to files by key
results_ch.collectFile { meta, file ->
    ["${meta.group}.txt", file.text + '\n']
}
```

### groupTuple
```groovy
// Group by key
Channel.of(['s1', 'a.bam'], ['s1', 'b.bam'], ['s2', 'c.bam'])
    .groupTuple()
// Emits: ['s1', ['a.bam', 'b.bam']], ['s2', ['c.bam']]

// Group by specific index
ch.groupTuple(by: [0, 1])  // Group by first two elements
```

### toSortedList
```groovy
Channel.of(3, 1, 2).toSortedList()  // [[1, 2, 3]]
```

## Metadata Map Pattern

The nf-core convention is to pass a metadata map (`meta`) as the first tuple element:

```groovy
// Create meta map from samplesheet
Channel.fromPath(params.input)
    .splitCsv(header: true)
    .map { row ->
        def meta = [
            id:            row.sample,
            single_end:    row.single_end.toBoolean(),
            strandedness:  row.strandedness ?: 'auto',
            group:         row.group ?: 'default'
        ]
        def reads = meta.single_end
            ? [file(row.fastq_1)]
            : [file(row.fastq_1), file(row.fastq_2)]
        [meta, reads]
    }

// Access meta in process
process ALIGN {
    tag "$meta.id"

    input:
    tuple val(meta), path(reads)

    script:
    def se_flag = meta.single_end ? '-U' : '-1'
    """
    aligner ${se_flag} ${reads} -o ${meta.id}.bam
    """
}
```

### Updating meta through the pipeline
```groovy
ALIGN.out.bam
    .map { meta, bam ->
        def new_meta = meta + [aligned: true, bam_size: bam.size()]
        [new_meta, bam]
    }
```

## Common Pipeline Patterns

### Fan-out by chromosome
```groovy
chromosomes = Channel.of(1..22, 'X', 'Y').map { "chr${it}" }
bam_ch.combine(chromosomes)
    .set { bam_by_chr }

CALL_VARIANTS(bam_by_chr)
// Runs variant calling per sample per chromosome

// Fan-in: group results
CALL_VARIANTS.out.vcf
    .groupTuple()  // Group VCFs by sample
    .set { vcfs_by_sample }

MERGE_VCFS(vcfs_by_sample)
```

### Conditional process execution
```groovy
workflow {
    FASTP(reads_ch)

    if (!params.skip_qc) {
        FASTQC(FASTP.out.reads)
    }

    // Or using channel emptiness
    ch_bed = params.target_bed ? Channel.fromPath(params.target_bed) : Channel.empty()
}
```

### Reference file handling
```groovy
// Value channel for reference files (reused across all samples)
genome     = Channel.value(file(params.genome_fasta, checkIfExists: true))
genome_idx = Channel.value(file(params.genome_fai,   checkIfExists: true))

// Combine sample data with references
reads_ch.combine(genome).combine(genome_idx)
```

### Waiting for upstream processes
```groovy
// Use .collect() or .first() to synchronize
INDEX(genome)
ALIGN(reads_ch, INDEX.out.index.first())  // .first() converts to value channel
```
