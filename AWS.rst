Getting started with bcbio and gemini for human genomics
========================================================

Below are instructions for getting started with some basic human
genome variant calling and investigation.  See
http://ivory.idyll.org/blog/2015-pycon-talk.html for background and
additional links.

Setting up a VM to run things on
--------------------------------

Start up an m3.2xlarge machine on AWS, running Ubuntu 14.04.

Attach a 1 TB disk as /dev/sdf.

Log in as ubuntu.

Now update package database::

   sudo apt-get update

Upgrade packages::

   sudo apt-get -y upgrade

...and install a bunch of software::

   sudo apt-get -y install git gcc g++ gfortran unzip zlib1g-dev ruby-dev \
        openjdk-7-jre openjdk-7-jdk make libssl-dev libmysqlclient-dev

Start screen::

   screen

Format and mount the 1 TB disk as /disk::

   sudo mkfs -t ext4 /dev/xvdf
   sudo mkdir /disk
   mount /dev/xvdf /disk
   sudo chmod a+rwxt /disk

Installing and running the bcbio human genome variant calling pipeline
----------------------------------------------------------------------

Go there, and install the `bcbio pipeline
<https://bcbio-nextgen.readthedocs.org/en/latest/>`__::

   cd /disk

   # download installer script
   wget https://raw.github.com/chapmanb/bcbio-nextgen/master/scripts/bcbio_nextgen_install.py

   # run!!
   python bcbio_nextgen_install.py /disk/bcbio --tooldir=/disk/tools \
     --genomes GRCh37 --aligners bwa --aligners bowtie2

This will take a while...

.. #/disk/bcbio/anaconda/bin/bcbio_nextgen.py upgrade --tooldir=/disk/tools --genomes GRCh37 --aligners bwa --aligners bowtie2 --data

After the install finishes (successfully, one hopes!), download some
`data sets
<ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/>`__::

   cd /disk
   curl -O ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/HG-002_Homogeneity-10953946/HG002Run02-11611685/HG002-Run2_S1.bam
   curl -O ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/HG-002_Homogeneity-10953946/HG002Run02-11611685/HG002-Run2_S1.bam.bai

Build a project.csv file::

   cd /disk
   cat >project.csv <<EOF
   sample1,HG002,batch1,normal,male,
   EOF

Build a new project config file::

   bcbio_nextgen.py -w template freebayes-variant project.csv HG002-Run2_S1.bam

And now run it! ::

   cd /disk/project/work
   bcbio_nextgen.py ../config/project.yaml

(This will take approximately 1500 CPU hours.)

Installing and running Gemini, for investigating variants
---------------------------------------------------------

`Gemini <http://gemini.readthedocs.org>`__ is a database system for exploring human variants.

Go to your work directory::

   cd /disk

Download & run the gemini installer::

   wget https://raw.github.com/arq5x/gemini/master/gemini/scripts/gemini_install.py
   python gemini_install.py /disk/tools /disk/share/gemini

   /disk/share/gemini/anaconda/bin/python /disk/share/gemini/gemini/gemini/install-data.py /disk/share/gemini

Install some extra databases::

   gemini update --dataonly --extra cadd_score
   gemini update --dataonly --extra gerp_bp

Also install the Perl MySQL module::

   sudo /disk/tools/bin/cpanm DBD::mysql

Manually install the VEP database::

   cd /disk/tools/Cellar/vep/75_2014-06-12/lib
   wget ftp://ftp.ensembl.org/pub/release-73/variation/VEP/homo_sapiens_vep_73.tar.gz
   tar xzf homo_sapiens*.tar.gz
   ln -fs 73 75

Now, download the human genome to do some VCF fixups::

   cd /disk
   mkdir hg19
   cd hg19
   curl -O http://hgdownload.soe.ucsc.edu/goldenPath/hg19/bigZips/chromFa.tar.gz
   tar xzf chromFa.tar.gz
   rm chr{X,Y,M}.fa
   cat chr?.fa chr??.fa > hg19.fa
   samtools faidx hg19.fa

Go grab some VCF files from an Ashkenazi trio::

   mkdir /disk/trio
   cd /disk/trio

   curl -O ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/HG-002_Homogeneity-10953946/HG002Run02-11611685/HG002-Run2_S1.genome.vcf.gz

   curl -O ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/HG-003_Homogeneity-12389378/HG003Run03-13288282/HG003Run03_S1.genome.vcf.gz

   curl -O ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NISTAshkenazimTrio/HG-004_Homogeneity-14572558/HG004run02-15332344/HG004run02_S1.genome.vcf.gz

   gunzip *.gz

Fix 'em up::

   for vcf in *.genome.vcf
   do
      zless $vcf \
       | sed 's/ID=AD,Number=./ID=AD,Number=R/' \
       | vt decompose -s - \
       | vt normalize -r ../hg19/hg19.fa > ${vcf}.norm

      variant_effect_predictor.pl -i ${vcf}.norm \
       --cache \
       --sift b \
       --polyphen b \
       --symbol \
       --numbers \
       --biotype \
       --total_length \
       -o $(basename $vcf .vcf).vep.vcf \
       --vcf \
       --fields Consequence,Codons,Amino_acids,Gene,SYMBOL,Feature,EXON,PolyPhen,SIFT,Protein_position,BIOTYPE
   done

   for i in *.vep.vcf; do
      gemini load -v $i -t VEP $(basename $i .vcf).db --cores 8
   done

Correlating with specific 'rs' IDs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I haven't yet found a good way to extract variants connected to a
particular 'rs' ID (which is, for example, what `SNPedia indexes
around <http://snpedia.com>`__ without dumping to a text file and
searching that. While it's ugly, the following works... ::

   gemini query -q "select * from variants" HG002*.vep.db > HG002.var
   cut -f1,15 HG002.var | grep -v None$ > HG002.var.subset

This last file, ``HG002.var.subset``, can be grepped for specific rs IDs -
for example, ::

   grep rs12948217 HG002.var.subset

will pull out the `Canavan disease SNP <http://www.snpedia.com/index.php/Canavan_disease>`__ that I reference in the talk.

