# Container Dependency Management

Nextflow processes should declare dependencies via conda, Docker, and Singularity to ensure
portability across execution environments. Follow the nf-core tri-container pattern for
every process.

## Table of Contents

- [Tri-Container Pattern](#tri-container-pattern)
- [Conda Specification](#conda-specification)
- [Container Registries](#container-registries)
- [BioContainers Lookup](#biocontainers-lookup)
- [Multi-Tool Containers](#multi-tool-containers)
- [Custom Containers](#custom-containers)
- [Config-Level Container Settings](#config-level-container-settings)
- [Common Pitfalls](#common-pitfalls)

## Tri-Container Pattern

Every process MUST declare all three dependency mechanisms in this order:

```groovy
process TOOL_NAME {
    tag "$sample_id"
    label 'process_medium'

    conda 'bioconda::toolname=1.2.3'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/toolname:1.2.3--h1234567_0'
    } else {
        container 'quay.io/biocontainers/toolname:1.2.3--h1234567_0'
    }

    input:
    // ...

    output:
    // ...

    script:
    // ...
}
```

**Why this pattern:**
- `conda` enables local development and environments without containers
- The `if/else` block selects Singularity (HPC) or Docker (cloud/local) at runtime
- `params.singularity_pull_docker_container` allows forcing Docker pull even under Singularity
- BioContainers provides pre-built images for most bioinformatics tools

## Conda Specification

### Single tool
```groovy
conda 'bioconda::bcftools=1.20'
```

### Multiple tools (space-separated)
```groovy
conda 'bioconda::bcftools=1.20 conda-forge::gsl=2.7'
```

### From environment file
```groovy
conda "${projectDir}/envs/mytools.yml"
```

Example `envs/mytools.yml`:
```yaml
name: mytools
channels:
  - bioconda
  - conda-forge
  - defaults
dependencies:
  - bcftools=1.20
  - samtools=1.20
  - gsl=2.7
```

### Pin exact versions
Always pin tool versions to ensure reproducibility. Never use:
```groovy
// BAD - unpinned version
conda 'bioconda::samtools'
```

## Container Registries

### BioContainers (preferred for bioinformatics)
```groovy
// Docker
container 'quay.io/biocontainers/samtools:1.20--h143571b_1'

// Singularity
container 'https://depot.galaxyproject.org/singularity/samtools:1.20--h143571b_1'
```

### Docker Hub
```groovy
container 'staphb/fastp:0.23.4'
```

### GitHub Container Registry
```groovy
container 'ghcr.io/org/tool:1.0.0'
```

### Amazon ECR Public
```groovy
container 'public.ecr.aws/biocontainers/samtools:1.20--h143571b_1'
```

## BioContainers Lookup

To find the correct container tag for a bioinformatics tool:

1. **Search BioContainers**: `https://biocontainers.pro/tools/<toolname>`
2. **Quay.io tags page**: `https://quay.io/repository/biocontainers/<toolname>?tab=tags`
3. **Galaxy Singularity depot**: `https://depot.galaxyproject.org/singularity/`

The container tag format is: `toolname:version--build_string`

Example: `bcftools:1.20--h8b25389_0`
- Tool: bcftools
- Version: 1.20
- Build hash: h8b25389
- Build number: 0

## Multi-Tool Containers

When a process requires multiple tools, use a mulled container:

```groovy
process COMPLEX_ANALYSIS {
    conda 'bioconda::samtools=1.20 bioconda::bcftools=1.20 bioconda::bedtools=2.31.1'
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container 'https://depot.galaxyproject.org/singularity/mulled-v2-HASH:TAG'
    } else {
        container 'quay.io/biocontainers/mulled-v2-HASH:TAG'
    }
    // ...
}
```

Find mulled container hashes at: `https://github.com/BioContainers/multi-package-containers`

When no mulled container exists, build a custom Dockerfile:

```dockerfile
FROM condaforge/mambaforge:latest
RUN mamba install -y -c bioconda -c conda-forge \
    samtools=1.20 \
    bcftools=1.20 \
    bedtools=2.31.1 \
    && mamba clean -a
```

## Custom Containers

For in-house tools or custom combinations:

```groovy
process CUSTOM_TOOL {
    conda "${projectDir}/envs/custom.yml"
    if (workflow.containerEngine == 'singularity' && !params.singularity_pull_docker_container) {
        container "${params.singularity_cache}/custom_tool-1.0.sif"
    } else {
        container 'registry.example.com/team/custom_tool:1.0'
    }
    // ...
}
```

Build and push workflow:
```bash
# Build Docker image
docker build -t registry.example.com/team/custom_tool:1.0 .
docker push registry.example.com/team/custom_tool:1.0

# Convert to Singularity
singularity build custom_tool-1.0.sif docker://registry.example.com/team/custom_tool:1.0
```

## Config-Level Container Settings

### nextflow.config
```groovy
// Global container settings
docker {
    enabled = true
    runOptions = '-u $(id -u):$(id -g)'
    temp = 'auto'
}

singularity {
    enabled = true
    autoMounts = true
    cacheDir = "${params.singularity_cache ?: 'work/singularity'}"
}

// Pull and cache containers before execution
params.singularity_pull_docker_container = false
```

### Profile-based container selection
```groovy
profiles {
    docker {
        docker.enabled = true
        singularity.enabled = false
    }
    singularity {
        singularity.enabled = true
        singularity.autoMounts = true
        docker.enabled = false
    }
    conda {
        conda.enabled = true
        docker.enabled = false
        singularity.enabled = false
    }
}
```

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Unpinned tool versions | Always specify exact versions: `tool=1.2.3` |
| Missing build string in container tag | Include full tag: `tool:1.2.3--h1234567_0` |
| Singularity can't pull from quay.io | Use Galaxy depot URL for Singularity images |
| Container doesn't have required tool | Verify tool is in container before using |
| conda and container versions mismatch | Keep conda spec and container tag versions aligned |
| Forgetting `autoMounts` for Singularity | Set `singularity.autoMounts = true` in config |
