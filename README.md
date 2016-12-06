# FALCON-integrate - millanek (Sanger) fork
This is a slightly modified version of the PacBio repository to allow more customisation,
specifically to make it easier to run the FALCON assembler on the Wellcome Trust Sanger Institute's
compute cluster.

## set-up
```sh
git clone https://github.com/millanek/FALCON-integrate.git
cd FALCON-integrate
git submodule update --init
```
I suggest you install the Python code under your own Python installation:
e.g. 
```sh
cd FALCON
~/programs/Python-2.7.12/python setup.py install --user
```

To run the FALCON code you will also need to compile the modified DALIGNER code and have the folder in your PATH:
```sh
cd ../DAZZ_DB
make
cd ../DALIGNER
make
PATH=/path/to/FALCON-integrate/DALIGNER:$PATH
```

The FALCON\_unzip pipeline here is in its original PacBio version - needs some modification to get it run at Sanger
I have the pipeline running but it is not public on gitHub; please contact me directly (see below)

## contact

This fork is maintained by Milan Malinsky
Any issues with getting FALCON work at Sanger, please let me know at mm21@sanger.ac.uk

Some more information is on the PacBio [wiki](https://github.com/PacificBiosciences/FALCON-integrate/wiki).

## submodules
In case you are unfamiliar with [**git-submodules**](http://www.git-scm.com/book/en/v2/Git-Tools-Submodules), they are quite easy to use from the command-line:
```sh
git submodule update --init
```
If that fails, you can update your **git**, or try this:
```sh
git submodule init
git submodule update
```
which is *almost* the same thing.


## example FALCON assembly workflow (after complete DAZZLER pipeline):
these are just examples of parameters that worked reasonably well for one particular assembly; much more info on parameter tuning is here: 
https://github.com/PacificBiosciences/FALCON/wiki/Manual
### error-correct
```sh
for i in {1..N}; do bsub -G yourGroup -o correct_o_%J -e correct_e_%J -R'select[mem>15000] rusage[mem=15000]' -M15000 sh -c 'LA4Falcon -H 5000 -fo DATABASE_patched.db DATABASE_patched.${1}.las | fc_consensus.py --output_multi --n_core 0 --min_cov 6 --max_cov_aln 60 --max_n_read 200 > DATABASE_patched.${1}.corrected_max_cov_aln60_multi.fasta' ec $i; done
for i in {1..N}; do echo $i; awk 'BEGIN{ FS ="_"; printThis = 0;}{ if (substr($1,1,1) == ">") { if ($2 > 7000) { printThis = 1; print;} else {printThis = 0;}} else { if (printThis == 1) {print;}} }' DATABASE_patched.${i}.corrected_max_cov_aln60_multi.fasta > DATABASE_patched.${i}.corrected_max_cov_aln60_multi_min7kb.fasta
/path/to/FALCON-integrate/renumber_fasta.sh DATABASE_patched.${i}.corrected_max_cov_aln60_multi_min7kb.fasta
```
### set up new DAZZLER database of corrected reads and build multiple alignments
geting new .las files
### overlap filtering:
```sh
for i in {1..N}; do echo DATABASE_patched.corrected_max_cov_aln60_multi_min7kb.${i}.las >> fofn.txt; done
bsub -G yourGroup -o overlapFilter_o_%J -e overlapFilter_e_%J -R'select[mem>25000] rusage[mem=25000]' -M25000 sh -c 'fc_ovlp_filter.py --db DATABASE_patched.corrected_max_cov_aln60_multi_min7kb.db --fofn fofn.txt --n_core 0 --min_cov 10 --max_cov 120 --bestn 10 --max_diff 90 > filtered_overlaps_DATABASE_patched.corrected_max_cov_aln60_multi_min7kb.ovlp'
```
### do the assembly with different min length cutoffs:
```sh
for minl in 6000 7000 8000 9000 10000 11000 12000 13000 14000 15000 16000 17000; do bsub -G yourGroup -o graphBuild_o_%J -e graphBuild_e_%J -R'select[mem>16000] rusage[mem=16000]' -M16000 sh -c 'fc_ovlp_to_graph.py --min_len $1 --params_fn minl_${1} filtered_overlaps_DATABASE_patched.corrected_max_cov_aln60_multi_min7kb.ovlp; fc_graph_to_contig.py --run_name minl_${1} DATABASE_patched.corrected_max_cov_aln60_multi_min7kb_renumbered_onlyReadID.fasta' assemle $minl; done
```
