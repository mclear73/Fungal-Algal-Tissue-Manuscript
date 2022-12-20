# Processing Raw RNAseq Data #

## Running FastQC on the raw files ##
- Navigate into the project folder that you created where your split reads are located

      cd Franken_Lichen

- Create a new fastQC_Raw.pbs file

      nano fastQC_Raw.pbs

- This will open up an empty text file
- Insert the following code in to the fastQC_Raw.pbs text file:

      #Initial QC run     
      #!/bin/bash
      #PBS -N QC_RAW
      #PBS -l ncpus=4
      #PBS -l mem=8gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com
    
      cd $PBS_O_WORKDIR

      mkdir FastQC_Raw_Output

      for f in $(ls *.fastq.gz)
      do
      zcat ${f} | ~/FastQC/fastqc -t 4 -o FastQC_Raw_Output ${f}
      done 

-  press ctrl + x
-  press `y`
-  press enter
-  Note: make sure you change the email portion `#PBS -M mclear73@gmail.com`
- This file will loop through all of the files in the directory and put them through fastQC
- To run fastQC_Raw.pbs

      qsub fastQC_Raw.pbs

- To check the progress on the job:

      qstat -a

### Running MultiQC on the FastQC output ###
- To easily view all of the multiQC output at the same time. First navigate to the directory where you deposited all of the fastQC output files. 

      cd fastQC_Raw_Output

- Create a new multiQC.pbs file

      nano runMultiQC.pbs

- Once in the appropriate directory, make a .pbs file according to the following template:

      #Code for MultiQC.pbs
      #!/bin/bash
      #PBS -N MultiQC
      #PBS -l ncpus=4
      #PBS -l mem=8gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      cd $PBS_O_WORKDIR

      source activate py3.7

      multiqc .

- Make sure you change the email portion of the script to your email
- press ctrl + x
- press `y`
- press enter
- Make sure you are in the directory where the FastQC output files are located
- To run multiQC

      qsub runMultiQC.pbs

- This will create multiple output files, but we would like to download the `multiqc_report.html` file
- Download the file to your local machine and open it in your browser to view the output

## Running Trimmomatic on the raw files ##
- Create a new file for running Trimmomatic

      nano run_trimm.pbs

- This will open a blank text file
- Copy and paste the following text into the blank template. Make sure you change the email!:

      #Code for trimmomatic.pbs
      #!/bin/bash
      #PBS -N clean_adapters
      #PBS -l ncpus=8
      #PBS -l mem=16gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com
    
      cd $PBS_O_WORKDIR
      
      for f in $(ls *1.fastq.gz | sed 's/1.fastq.gz//' | sort -u)
      do
      java -jar ~/Trimmomatic-0.39/trimmomatic-0.39.jar PE -phred33 \
      ${f}1.fastq.gz ${f}2.fastq.gz \
      ${f}1_paired.fq.gz ${f}1_unpaired.fq.gz \
      ${f}2_paired.fq.gz ${f}2_unpaired.fq.gz \
      ILLUMINACLIP:/home/centos/Trimmomatic-0.39/adapters/TruSeq3-PE.fa:2:30:10:2:True \
      LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
      done

### Running FastQC and MultiQC on the trimmed and cleaned files ###
- Make a new file for running FastQC

      nano  fastQC_trimmed.pbs

- This will open a blank text document. Copy and paste the following code into the document. Make sure you change the email address!

      #Initial QC run     
      #!/bin/bash
      #PBS -N QC_Trim
      #PBS -l ncpus=4
      #PBS -l mem=8gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com
    
      cd $PBS_O_WORKDIR

      mkdir FastQC_Trim_Output

      for f in $(ls *.fq.gz)
      do
      zcat ${f} | ~/FastQC/fastqc -t 4 -o FastQC_Trim_Output ${f}
      done 

- Press ctrl + x
- Press `y`
- Press Enter
- To run the new FastQC script:

      qsub fastQC_trimmed.pbs

- To view the output navigate into the output directory, first copy the runMultiQC.pbs file into your new output folder

      cd FastQC_Raw_Output
      cp runMultiQC.pbs ../FastQC_Trimmed_Output

- Navigate into your new output directory

      cd ../FastQC_Trimmed_Output

- Run MultiQC

      qsub runMultiQC.pbs

- When completed, download the `multiqc_report.html` file to your local machine and view on a browser
- Make sure that sequence quality looks good and ensure that adapters have been removed

## Mapping the cleaned reads to reference genome(s) ##
### Creating a STAR indexed reference genome (*A. nidulans* with *C. reinhardtii* example) ###
- Create a new pbs file for generating the M. truncatula indexed genome for STAR mapping

      nano AspChlamComb_STAR_index.pbs

- Copy and paste the following into the empty text file, make sure you change the email!		

      #STAR medicagoIndex.pbs
      #!/bin/bash
      #PBS -N medicago_Index
      #PBS -l ncpus=8
      #PBS -l mem=58gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      cd $PBS_O_WORKDIR

      STAR --runMode genomeGenerate \
      --genomeDir ~/Genomes_Indexed/Asp_Chlam_Comb \
      --genomeFastaFiles ~/Genomes/Aspergillus/Asp_Chlam.fasta \
      --sjdbGTFfile ~/Genomes/Mtrunc_v5/Asp_Chlam.gff3 \
      --sjdbGTFtagExonParentGene ID \
      --sjdbOverhang 100 \
      --genomeSAindexNbases 13 \
      --runThreadN 8

- Close and save the file
- To generate the index file run

      qsub AspChlamComb_STAR_index.pbs

### Mapping reads to indexed reference genome ###
- Create a new pbs file for mapping the Aspergillus reads to the indexed genome file

      nano STAR_map_AspChlam.pbs

- Copy and paste the following into the empty text file, make sure you change the email!

      #STAR.pbs
      #!/bin/bash
      #PBS -N STAR_align
      #PBS -l ncpus=8
      #PBS -l mem=32gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      cd $PBS_O_WORKDIR

      for i in $(ls *R1_paired.fq.gz | sed 's/R1_paired.fq.gz//')
      do
      STAR --runMode alignReads \
      --readFilesCommand zcat \
      --outSAMtype BAM Unsorted \
      --genomeDir ~/Genomes_Indexed/Asp_Chlam_Comb \
      --readFilesIn ${i}R1_paired.fq.gz  ${i}R2_paired.fq.gz \
      --outReadsUnmapped Fastx \
      --runThreadN 8 \
      --quantMode GeneCounts \
      -–sjdbGTFfile ~/Genomes/Aspergillus/Asp_Chlam.gff3 \
      --sjdbGTFfeatureExon CDS \
      --sjdbGTFtagExonParentGene ID \
      --outFileNamePrefix ${i}AspChlam \
      --alignIntronMax 3000
      done

- Close and save the file
- To run the STAR mapping script

      qsub STAR_map_AspChlam.pbs

## Creating count files from STAR output (*.bam) files ##
### Sorting *.bam files with samtools ###
- Open a new text file for a samsort .pbs script

      nano samSort.pbs

- Copy and paste the following into the empty text file, makes sure you change the email!

      #SAMsort.pbs
      #!/bin/bash
      #PBS -N SAM_SORT
      #PBS -l ncpus=4
      #PBS -l mem=8gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      cd $PBS_O_WORKDIR

      for i in $(ls *Aligned.out.bam | sed 's/.bam//')
      do
      samtools sort ${i}.bam  -n -o \
      ${i}.sorted.bam
      done

- Close and save the file
- To run the samsort pbs file:

      qsub samSort.pbs

### Creating count files with HTseq ###
- Open a new file called htseq_counts.pbs

      sudo nano htseq_counts.pbs

- Copy and paste the following into the blank file

      #HTSeq.pbs
      #!/bin/bash
      #PBS -N HTSeq
      #PBS -l ncpus=4
      #PBS -l mem=8gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      cd $PBS_O_WORKDIR

      source activate py3.7

      for i in $(ls *.sorted.bam | sed 's/.sorted.bam//')
      do
      htseq-count -s no -t CDS -f bam -i ID ${i}.sorted.bam /home/centos/Genomes/Aspergillus/Asp_Chlam.gff3  > ${i}.counts
      done

- Close and save the file
- To run the htseq_couts file:

      qsub htseq_counts.pbs

## Aggregating a count summary file with bash ##
- Create a new text file to run a bash command to aggregate the STAR log files

      nano agg_STAR_log.sh

- Copy and paste the following into the empty text file

      for i in $(grep -l 'Uniquely mapped reads number' *Log.final.out); do 
      export percent=$(sed -n '/Uniquely mapped reads %/p' $i |cut -f2)
      export num=$(sed -n '/Uniquely mapped reads number/p' $i| cut -f2)
      export file=$(paste <(echo "$i") <(echo "$num") <(echo "$percent"))
      echo "$file" >> compiled_STAR_stats.txt
      done

- Save and close the file
- Make the bash script executable

      chmod u+x agg_STAR_log.sh

-While in the directory where all of the STAR output files are located run the bash script

      ./agg_STAR_log.sh

## Aggregate the count files with Python script ##
- Open a new text file to aggregate counts

       nano aggregate_counts.py

- In the empty text document, copy and paste the following code

      import glob
      import pandas as pd
  
      #Creates a list of filenames from files in the working directory that end in .csv
      filenames = glob.glob('*ReadsPerGene.out.tab')
  
      #creates a list of dataframes from the working directoryo
      dfs = [pd.read_csv(filename, names = ['Gene', filename], sep='\t', skiprows=4, usecols=[0,1]) for filename in filenames]
  
      #Combines all dataframes into one
      combinedDF = pd.concat(dfs, axis=1)
      #Drops duplicate gene columns that are remnants from the concat function
      #This works by transposing the dataframe, dropping duplicates, then re-transposing
      combinedDF = combinedDF.T.drop_duplicates().T
      #Sets the index to the gene column
      combinedDF = combinedDF.set_index('Gene')
  
      #Starts a dataFrame with the filenames as the index
      metaDF = pd.DataFrame(filenames).rename(columns={0:'Sample'})
      metaDF = metaDF.set_index('Sample')
  
      #Saves the dataframes in a single excel file in the current directory
      combinedDF.to_csv('CompiledCounts.csv')
      metaDF.to_csv('MetaData.csv')

- To run the python script, make sure that you are operating in the correct conda environment

      conda activate py3.7

- Make sure that the python script is in the directory where all of the HTseq output files are located
- To move the file

      cp aggregate_counts.py /path/to/directory/htseq_counts

- Navigate to the directory where the HTseq counts are located using `cd`
- Run the python script

      python aggregate_counts.py

- This will create a files called '**CompiledCounts.csv**' & '**MetaData.csv**' that can be downloaded to your local machine

## What to do with the remaining data ##
### Checking for contamination with phyloFlash ###
- To run phyloFlash create a pbs script

      nano run_phyloFlash.pbs

- Copy and paste the following into the blank text file **(make sure to change the input file names to your files)**:

      #run_phyloFlash.pbs
      #!/bin/bash
      #PBS -N phyloFlash
      #PBS -l ncpus=8
      #PBS -l mem=32gb
      #PBS -m ae
      #PBS -M mclear73@gmail.com

      source activate py3.7

      cd $PBS_O_WORKDIR

      phyloFlash.pl -lib run01 -read1 reads_F.fq.gz \
      -read2 -reads_R.fq.gz \
      -almosteverything