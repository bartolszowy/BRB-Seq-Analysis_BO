# BRB-Seq Pipeline (Conda Environment)

This is the refactored BRB-Seq pipeline with:
- 2 YAML config files for SpikeIn and Full Run
- Simplified 3-column mapping file
- Conda environment 
- SLURM dependency chaining 
- Automated repooling calculation 

## Setup (One-Time)

### 1. Build the conda environment

```bash
cd /scratch/bmlab/bolszowy/projects/Conda_BRB_seq_v1
conda env create -f environment.yml
conda activate brb_seq
```

### 2. Build STAR indices (if not already done)

```bash
# Download and decompress genome files
cd /scratch/bmlab/bolszowy/projects/Conda_BRB_seq_v1/index_build

# Mouse
cd Mus_musculus/Ensembl/GRCm39/download/
wget https://ftp.ensembl.org/pub/release-113/fasta/mus_musculus/dna/Mus_musculus.GRCm39.dna.primary_assembly.fa.gz
wget https://ftp.ensembl.org/pub/release-113/gtf/mus_musculus/Mus_musculus.GRCm39.113.gtf.gz
gunzip *.gz

# Human
cd ../../Homo_sapiens/Ensembl/GRCh38/download/
wget https://ftp.ensembl.org/pub/release-113/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz
wget https://ftp.ensembl.org/pub/release-113/gtf/homo_sapiens/Homo_sapiens.GRCh38.113.gtf.gz
gunzip *.gz

# Build indices
cd ../..
sbatch run_star_builds.sh mouse
sbatch run_star_builds.sh human

# When done, copy to LTS (if you have write access):
# rsync -av index_build/ /lts/bmlab/bmlab_1/bolszowy/Index/
```

### 3. Set up a project directory structure

```bash
mkdir -p MyProject/{metadata,scripts,spike_in_analysis,full_run_analysis}
cd MyProject
```

### 4. Copy pipeline scripts to your project

```bash
cp /path/to/downloaded/scripts/*.sh scripts/
cp /path/to/downloaded/scripts/*.py scripts/
chmod +x scripts/*.sh scripts/*.py
```

## Running the Pipeline

### Step 1: Create your config file

Copy `config.yaml` template and fill in your project details:

```bash
cp config.yaml myproject_config.yaml
# Edit with nano/vim/etc.
```

Required fields:
- `project.name` — no hyphens, dots, or spaces
- `project.directory` — full path to output
- `project.run_type` — `spike_in` or `full_run`
- `project.species` — `mouse` or `human`
- `input.samples_file` — path to samples.tsv
- `input.read1` — path to R1 FASTQ (barcode + UMI)
- `input.read2` — path to R2 FASTQ (cDNA)
- `chemistry.preset` — `Alithea`, `PrimeSeq`, or `TripBRB`
- `slurm.mail` — your email

### Step 2: Create your samples file

Create `metadata/samples.tsv` with 3 tab-separated columns:

```
SampleName	Barcode	Well
Sample_WT_1	AAGTAGAGAGTA	A1
Sample_WT_2	CCGTGAGCGGAG	A2
Sample_KO_1	TTGGAGTTGGAG	A3
```

**Sample name rules:**
- Only letters, numbers, and underscores
- No hyphens, dots, or spaces (R will mangle them)

### Step 3: Submit the pipeline

```bash
conda activate brb_seq
bash scripts/submit_pipeline.sh myproject_config.yaml
```

That's it! The script will:
1. Validate sample names
2. Auto-count samples
3. Submit all steps with SLURM dependencies
4. Generate a submission record in `logs/`

Monitor progress:
```bash
squeue -u yourusername
tail -f spike_in_analysis/logs/step1_*.out
```

### Step 4: Review outputs

**Spike-in outputs:**
- `MultiQC/MyProject_QCReport.html` — Quality metrics
- `repooling/repooling_report.tsv` — Per-sample metrics table
- `repooling/epmotion_export.tsv` — Import into epMotion
- `repooling/repooling_summary.txt` — Human-readable summary

**Full run outputs:**
- `MultiQC/MyProject_QCReport.html` — Quality metrics
- `Counts_Files/Raw_Counts.txt` — Gene x sample raw counts
- `Counts_Files/Dedup_Counts.txt` — Gene x sample deduplicated counts


## Troubleshooting

### "Config file not found"
Check the path to `config.yaml`. Use full path or run from project directory.

### "Samples file not found"
Update `input.samples_file` in config to the correct path.

### "Invalid sample name"
Sample names can only contain letters, numbers, and underscores. Remove hyphens, dots, and spaces.

### "Reference paths file not found"
STAR indices haven't been built yet. Run `Build_STAR_Index.sh` first.

### "Job failed at step 1"
Check `PROJECT_DIR/logs/step1_JOBID_TASKID.err` for the specific sample that failed.

### Repooling report missing
Only generated for `run_type: spike_in`. Full runs don't need repooling.

## File Locations After Pipeline Completes

```
PROJECT_DIR/
├── logfile/                     # Per-sample log files (metrics)
├── fastqc/                      # FastQC reports
├── cutadapt/                    # Trimming logs
├── STAR/                        # STAR alignment logs
├── FeatureCounts/               # FeatureCounts outputs
├── MultiQC/                     # Combined QC report (HTML)
├── Counts_Files/                # Count matrices (full run only)
├── repooling/                   # Repooling calculations (spike-in only)
│   ├── repooling_report.tsv
│   ├── epmotion_export.tsv
│   └── repooling_summary.txt
└── logs/                        # SLURM job outputs
    ├── submission_*.txt
    ├── step1_*.out
    ├── step2_*.out
    └── step3_*.out
```

## Getting Help

If you encounter issues:
1. Check the SLURM error logs in `PROJECT_DIR/logs/`
2. Check the per-sample log files in `PROJECT_DIR/logfile/`
3. Verify your config.yaml has all required fields
4. Verify sample names follow the naming rules
5. Ask Brian or check the BRB-Seq documentation


