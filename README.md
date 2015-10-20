16S Taxonomic Assignment Workflow
===
Copyright (c) 2015, Katherine D. McMahon and Lackeys  
URL: https://mcmahonlab.wisc.edu/  
URL: https://github.com/McMahonLab/  
All rights reserved.

Overview
---

This workflow assigns taxonomy to a fasta file of OTU sequences using both a
small, custom taxonomy database and a large general database. The workflow was developed specifically for the classification of freshwater OTUs against the McMahonLab's [curated freshwater database](https://github.com/mcmahon-uw/FWMFG). Your mileage may vary.

Background
---
Our original taxonomy assignment workflow was straightforward and more simple:
	everything was classified to 70% using our FW database
	all classifications that weren't "unclassified" at the lineage level were kept
	everything else was re-classified using green genes to 60%

However this workflow was flawed due to a misunderstanding of the classification p values:
	-The 70% bootstrap value refers to the repeatability of the taxonomy assignment,
	NOT to it's accuracy.
	-A classification that is given at 70% confidence means that 70% of the time
	when you feed that sequence into that database it goes into that group.
	-The classifier sill puts anything you feed into it a classification,
	even it the true classification is not included in the database.
	-The GG database is huge, if you put something into it that the database doesn't have,
	it will likely put it in random different clusters each time and it will be "unclassified"
	-Our database is small.  If you put something into it that it doesn't have,
	it fill find whatever it's closest to and consistently call it that because there are
	not as many dissimilar options to spread it between.
		consistently. meaning a high bootstrap value.
	-simple example: if you give our FW database an archaea sequence, 100% of the time
	it will say it is a bacteria- because the database only has bacteria in it.

Consequences of the flaw:
	-we were forcing sequences that do not match well to be called our favorite FW taxa
		-we may have missed other taxa that may be important
		-we may have muddied the relationships of our key taxa by adding in unrelated ones

Possible solutions and why they don't work:
	-Classify in GG first
		-then reclassify the "freshwater" phylums.
		-this is basically what Jason did to solve the problem
		-but phyla are too broad in GG, can't assume everything in phylum is in our database
	-Classify in GG first
		-then reclassify the "freshwater" orders (or some lower level)
		-but GG lacks the resolution to identify those orders,
		we'd miss many sequences b/c GG would call them unclassified
	-Combine the two taxonomy databases and classify all at once
		-if something is present in both databases, it will be split during classification,
		50% going into one, the other 50% into the other, and remain unclassified
	-Classify our sequences with green genes, remove those green genes sequences, and then
	combine the databases and classify all at once
		-the taxonomy databases are highly curated, adding something new in is not that simple.
			-the structure of the two databases may differ at higher taxonomic resolution
			-the green genes classification might not actually be a good match if the freshwater
			sequence just doesn't exist in green genes, so then we'd be removing an unrelated
			sequence that we shouldn't.
				-how can you differentiate if the fw sequence does not exist in gg
				or if the fw sequence exists in gg but is only named to class level.
	-Classify the sequences in the freshwater database with a different method, like BLAST,
	since it's a small database anyway.
		-the taxonomy assignment algorithm takes into consideration phylogeny, which is a
		better classification that by just using sequence similarity.

Our solution/this workflow (that we think works):
	-Classify in FW first, but with a cutoff not based on clustering bootstrap values
		-use a BLAST cutoff to identify highly similar sequences to those in
		our freshwater sequence database
		-classify those hightly similar sequences with a classification algorithm and our db
		-classify the remaining sequences with a large database

Workflow Summary
---
This workflow assigns taxonomy to a fasta file of otu sequences using both a
small, custom taxonomy database and a large general database.

Summary of Steps and Commands (all commands entered in terminal window):

-1. background on why we chose this workflow

0. format files (textwrangler or bash)
	depends on your starting file formats

1. make BLAST database file (blast)
	makeblastdb -dbtype nucl -in custom.fasta -input_type fasta -parse_seqids -out custom.db

2. run BLAST (blast)
	blastn -query otus.fasta -task megablast -db custom.db -out otus.custom.blast -outfmt 11 -max_target_seqs 1

3. reformat blast results (blast)
	blast_formatter -archive otus.custom.blast -outfmt "6 qseqid pident length qlen qstart qend" -out otus.custom.blast.table

4. filter BLAST results (R)
	Rscript find_seqIDs_with_pident.R otus.custom.blast.table ids.above.98 98 TRUE
	Rscript find_seqIDs_with_pident.R otus.custom.blast.table ids.below.98 98 FALSE

5. recover sequence IDs left out of blast (python, bash)
	python fetch_seqIDs_blast_removed.py otus.fasta otus.FW.blast.table ids.missing
	cat ids.below.98 ids.missing > ids.below.98.all

6. create fasta files of desired sequence IDs (python)
	python fetch_fastas_with_seqIDs.py ids.above.98 otus.fasta otus.above.98.fasta
	python fetch_fastas_with_seqIDs.py ids.below.98.all otus.fasta otus.below.98.fasta

7. assign taxonomy (mothur)
	~/mothur/mothur
	classify.seqs(fasta=otus.above.98.fasta, template=custom.fasta,  taxonomy=custom.taxonomy, method=wang, probs=T, processors=2)
	classify.seqs(fasta=otus.below.98.fasta, template=general.fasta, taxonomy=general.taxonomy, method=wang, probs=T, processors=2)
	quit()

8. combine taxonomy files (terminal)
	cat otus.above.98.custom.wang.taxonomy otus.below.98.general.wang.taxonomy > otus.taxonomy

9. assign taxonomy without custom database (mothur, bash)
	~/mothur/mothur
	classify.seqs(fasta=otus.fasta, template=general.fasta, taxonomy=general.taxonomy, method=wang, probs=T, processors=2)
	quit()
	cat otus.general.wang.taxonomy > otus.general.taxonomy

10. reformat taxonomy files (textwrangler or bash)
	find tab, replace semicolon

11. compare taxonomy files (R)
	Rscript find_classification_disagreements.R custom.plus.general.tax.file.path general.only.tax.file.path results.folder.path taxonomy.bootstrap.cutoff


Detailed Workflow Instructions
---
__________________________________________________________________________________________

0. format files (textwrangler or bash)

Reformat fasta file of your sequences to have no whitespace in the seqID line.
BLAST will call some parts the seqID and some parts comments if they're separated.
Having BLAST seqIDs that don't match the full >comment line of the fasta file will
throw off the python script that fetches the chosen sequences.

In this workflow I will refer to this file as otus.fasta

Reformat your taxonomy files to be compatible with mothur.  mothur requires
seqID tab level;level;level;level;
no white space except the tab between ID and taxonomy, and the semicolon on the end too!

In this workflow I will call the taxonomy files:
	custom.fasta		
	custom.taxonomy
	general.fasta
	general.taxonomy
custom is the small specific database you want to use before the large general database.
(For me custom is our freshwater database and general is the green genes database.)
the .fasta are the sequences w/ IDs, the .taxonomy are the IDs w/ taxonomic names.

__________________________________________________________________________________________

1. make BLAST database file (blast)

Use the command makeblastdb to create a blast database out of the FW taxonomy fasta files.
Need to create a database because:
	1. BLAST will run faster
	2. Having a database is necessary for some of the output formats

Full Command (type in terminal):

makeblastdb -dbtype nucl -in custom.fasta -input_type fasta -parse_seqids -out custom.db

What the filenames are:
	custom.fasta		this is the path to the small custom taxonomy file you want to use to assign
						taxonomy before using a larger taxonomy database
						(i.e. the freshwater taxonomy database fasta file)
						note: if the file path has spaces in it it needs to be ' "/path/double quoted" '
	custom.db			this is the name you choose for the blast database you are creating.
						It makes 6 files with different final extensions, but all start with this.  
						Default is your -in file name with those extensions, but it's
						less confusing to add .db so you know what the weird extension files are
What each flag does:
	-dbtype nucl		a required argument, says the database input file is nucleic acids (nucl)
	-in custom.fasta	specify path of the input file that it makes the database out of
	-input_type fasta	tells it the input file is a fasta file (that's the default value too)
	-parse_seqids		this tells it to include information in the database that will later
						allow you to pull sequences back out of it.  
						This is necessary for using blast_formatter later.
	-out custom.db		specify path of the database files this command created  

These 6 files are created:
	custom.db.nhr
	custom.db.nog
	custom.db.nsd
	custom.db.nsi
	custom.db.nsq
but later when you tell it which database file to use you just say custom.db and it figures the rest out

__________________________________________________________________________________________

2. run BLAST (blast)

Use the command blastn to run a megablast that returns the best hit in the taxonomy database (subject)
for each of your OTU sequences (queries). Megablast is optimized for finding very similar matches with
sequences longer than 30 bp.  This workflow was made for ~100 bp sequences and may need to be re-optimized
for sequences of different lengths.

Full Command (type in terminal):

blastn -query otus.fasta -task megablast -db custom.db -out otus.custom.blast -outfmt 11 -max_target_seqs 1

What the filenames are:
	otus.fasta				the query file. the fasta file of your OTU sequences you want classified
							this is what you reformatted in step 0.
	custom.db				the subject file. the blast database file you made from the taxonomy
							database fasta sequences in step 1.
	otus.custom.blast		the BLAST output file. Its format is unreadable by you,
							but it's the detailed BLAST format the blast_formatter accepts.
							filename example here is query.subject.blast

What each flag does:						
	-query otus.fasta 		specify path of query file
	-task megablast 		this is already optimized by smart BLAST people for high similarity hits.
							It is also the blastn default task. for more info see
							http://www.ncbi.nlm.nih.gov/Class/MLACourse/Modules/BLAST/nucleotide_blast.html
	-db custom.db 			the name of the blast database created previously (without the additional file extensions)
	-out otus.custom.blast	specify path of the blast output file (this is the file you're creating)
	-outfmt 11 				need this format in order to use blast_formatter command
	-max_target_seqs 1		only keep the best hit for each query sequence

Note: Not good if high %id shorter sequences are being chosen over lower%id longer sequences that still meet %id cutoff.
This is something that could be examined to decide if blastn custom scoring should be used instead of megablast.
However, we are only keeping very similar sequences anyway, and these sequences are very short, and
megablast is optimized with complicated statistics to find closely related sequences which we want.
The R script in step 4 will tell you alignment length vs. query length stats so keep an eye out for this.
If many alignments are shorter then this should be examined more carefully.

__________________________________________________________________________________________

3. reformat BLAST results (blast)

The blast_formatter function in blast takes the full outformat 11 file and reformats it to any other
possible format.  Here we reformat to a custom table format to feed into the R script the pulls out
matching and nonmatching sequence IDs.
However, using blast_formatter you can look at your blast results from the previous step any way you'd like.

Full Command (type in terminal):

blast_formatter -archive otus.custom.blast -outfmt "6 qseqid pident length qlen qstart qend" -out otus.custom.blast.table

What the filenames are:
	otus.custom.blast			The blast result file generated in step 2. This is in the ASN.1 blast file format.
	otus.custom.blast.table		The blast result reformatted into a 6 column table that the R script accepts.
								The tab-delimited columns are: qseqid pident length qlen qstart qend

What each flag does:		
	-archive otus.custom.blast 		Specify path to the blast result file you are reformatting
	-outfmt "6 qseqid pident length qlen qstart qend"
									6 is tabular format without headers or other info btwn rows of data,
									the rest specifies what goes in each tab-delimited column:
										1 qseqid: query (OTU) sequence ID
										2 pident: percent identity (# of matches / # "columns" in the sequence)
										3 length: length of alignment
										4 qlen: full length of query sequence
										5 qstart: index of beginning of alignment on query sequence
										6 qend: index of end of alignment on query sequence
	-out otus.custom.blast.table	Specify path to the output file of formatted blast results

__________________________________________________________________________________________

4. filter BLAST results (R)

The find_seqIDs_with_pident.R script takes the formatted blast file and
calculates a "true" pident value that corrects the value for the entire length of the query.
I could not find any command that forces blast use the entire query sequence.
The "true" pident is a worst case scenario that assumes any edge gaps are mismatches.
Calculation:
	"true pident" = pident * length / (length - (qend - qstart) + qlen)
The R script is run twice, once to generate the sequence ID's above/equal to the "true pident"
cutoff that are destined for taxonomy assignment in the small custom database, once to generate
the sequence ID's below the "true pident" cutoff that are destined for taxonomy assignment
in the large general database.

Full Two Commands (type in terminal):

Rscript find_seqIDs_with_pident.R otus.custom.blast.table ids.above.98 98 TRUE
Rscript find_seqIDs_with_pident.R otus.custom.blast.table ids.below.98 98 FALSE

These arguments must be in the correct order.
Rscript sources the .R script using the arguments supplied after it in the terminal.
Separate all arguments with a space.

What each argument is:
	1st: script.R	the path to this script.  The script is called find_seqIDs_with_pident.R
	2nd: blastfile	the path to the formatted blast file. This is the otus.custom.blast.table file from step 3
	3rd: outputfile	the path the to file you are creating. This is the list of sequence ID's matching your criteria.
	4th: cutoff#	the cutoff percent identity to use. Note this is the calculated "true" full length percent identity,
					not necessarily the value blast returns for the match.  this is a number.
	5th: matches	want matches? TRUE or FALSE. TRUE: return seqID's >= cutoff, FALSE: return seqID's < cutoff

What each file is:
	find_seqIDs_with_pident.R	the r script.  source from the command line by typing
								Rscript, not R.  Rscript accepts arguments after the script.
	otus.custom.blast.table		the formatted blast output from step 3
	ids.above.98				the output file containing seqIDs at or above your cutoff value
	ids.below.98				the output file containing seqIDs below your cutoff value
								the format of these output files are \n delimited seqIDs.

Note: You may need to choose a different "true pident" cutoff for your sequence data.
I chose mine based on values that gave me classifications consistent to the class level
between the small and large databases. You can use a supplementary R script & workflow to
repeat this.  One thing that might affect this choice is the length of your sequences.


__________________________________________________________________________________________

5. recover sequence IDs left out of blast (python, bash)

Blast has a built in reporting cutoff based on evalue.  The blast expect value depends on
the length of the hit and the size of the database, and it reflects how frequently you
would see a hit of that quality by chance. The default evalue cutoff is 10, which means
blast does not report a match that you'd see 10 or more times by chance.  For more about
the evalue statistics, see: http://www.ncbi.nlm.nih.gov/BLAST/tutorial/Altschul-1.html

The python script fetch_seqIDs_blast_removed.py is used to find all of the
sequence IDs in the original fasta file that do not appear in the blast output.  The python
script then creates a new file in the same format as step 4's R script output file that
is a newline-delimited list of the missing sequence IDs.

The bash command cat concatenates these missing ids with the ids below the chosen cutoff
pident.  That is because the ids blast didn't report hits for had even worse pidents than
the ones that didn't make the R script cutoff.

Full Two Commands (type in terminal):

python fetch_seqIDs_blast_removed.py otus.fasta otus.FW.blast.table ids.missing
cat ids.below.98 ids.missing > ids.below.98.all

These arguments in the python script must be in the correct order.
python sources the .py script using the arguments supplied after it in the terminal.
Separate all arguments with a space.

What each argument is- 1st command (python):
	1st: script.py		path to this script. The script is called fetch_seqIDs_blast_removed.py
	2nd: fastafile		path to the fasta file containing all the seqIDs
	3rd: blastfile		path to the blast results in table format (col1 = seqID)				
	4th: outputfile		path to the file with \n delimited seqIDs the script creates

What the syntax is- 2nd command (bash):
	cat file1 file2 > file3		means "combine file1 and file2 into file 3"

What each file is:
	fetch_seqIDs_blast_removed.py	python script you're sourcing
	otus.fasta						original fasta file of your sequences from step 0
	otus.FW.blast.table				reformatted blast results from step 3
	ids.missing						missing seqIDs file created in this step with python
	ids.below.98					seqIDs below your cutoff from R script in step 4
	ids.below.98.all				all seqIDs below your cutoff created in this step with bash

Note: It's a little concerning that so many sequences were not reported by blast.  They are
16S sequences so shouldn't they all be pretty close?? This may mean we should tweak the
penalty values.  It definitely deserves another look. Note that these sequences are already
curated to remove bad reads using mothur & Alex's pipeline.

__________________________________________________________________________________________

6. create fasta files of desired sequence IDs (python)

The fetch_fastas_with_seqIDs.py takes the sequence IDs selected from the blast output
and finds them in the original query fasta file.  
It then creates a new fasta file containing just the desired sequences.

Full Two Commands (type in terminal):

python fetch_fastas_with_seqIDs.py ids.above.98 otus.fasta otus.above.98.fasta
python fetch_fastas_with_seqIDs.py ids.below.98.all otus.fasta otus.below.98.fasta

These arguments must be in the correct order.
python sources the .py script using the arguments supplied after it in the terminal.
Separate all arguments with a space.

What each argument is:
	1st: script.py	the path to this script. The script is called fetch_fastas_with_seqIDs.py
	2nd: idfile		the path to the file with \n separated seqIDs that was generated in step 4
	3rd: fastafile	the path to the otu fasta file containing all the seqIDs from step 0
	4th: outputfile the path to the new fasta file the script generates
					NOTE: if this file already exist the script will delete it before starting.

What each file is:
	fetch_fastas_with_seqIDs.py		the python script you're sourcing
	ids.above.98					the file of seqIDs at or above your cutoff from step 4
	ids.below.98.all				the file of seqIDs below your cutoff from step 5
	otus.fasta						the original fasta file of your otu sequences
	otus.above.98.fasta				the output file with fasta sequences at or above your cutoff
	otus.below.98.fasta				the output file with fasta sequences below your cutoff

__________________________________________________________________________________________

7. assign taxonomy (mothur)

The classify.seqs() command in mothur classifies sequences using a specified algorithm
and a provided taxonomy database.  I use the default algorithm (wang with kmer size 8),
and ask it to show bootstrap values. The output file is a list of sequence ID's next to
their assigned taxonomy.

Full Four Commands (type in terminal):

~/mothur/mothur
classify.seqs(fasta=otus.above.98.fasta, template=custom.fasta,  taxonomy=custom.taxonomy, method=wang, probs=T, processors=2)
classify.seqs(fasta=otus.below.98.fasta, template=general.fasta, taxonomy=general.taxonomy, method=wang, probs=T, processors=2)
quit()

What the first and last commands do:
	~/mothur/mothur		this is the path to the mothur program installed on your computer.
						~/mothur/mothur is the default place to instal it.
						You must open mothur to use the mothur command classify.seqs().
						When in the mothur program " mothur > " appears instead of $
	quit()				exits the mothur program and returns you back to bash in the terminal.

What the filenames are:
	otus.above.98.fasta		this is the fasta file containing only seqIDs >= your cutoff, from step 5
	otus.below.98.fasta		this is the fasta file containing only seqIDs < your cutoff, from step 5
	custom.fasta			this is the fasta file for your small custom taxonomy database (ie freshwater)
	general.fasta			this is the fasta file for your large general taxonomy database (ie green genes)
	custom.taxonomy			this is the .taxonomy file for your small custom taxonomy database (ie freshwater)
	general.taxonomy		this is the .taxonomy file for your large general taxonomy database (ie freshwater)

What each flag does:
	fasta=		path to the .fasta file you want classified
	template=	path to the .fasta file of the taxonomy database
	taxonomy=	path to the taxonomy file of the taxonomy database
	method=		algorithm for assigning taxonomy. default is wang.
	probs=		T or F, show the bootstrap probabilities or not?
	cutoff=		minimum bootstrap value for getting a name instead of unclassified
				typically minimum is 60%
				note: the default is no cutoff, full reporting.  I have an R script that
				let's you apply a bootstrap cutoff after the fact if you want to see everything 1st
	processors=	the number of processors on your computer to run it on
				note: this did not work on my mac, still ran on only 1 core.

What the output files are (note you have no control over the name extensions added):
	otus.above.98.custom.wang.taxonomy
	otus.above.98.custom.wang.tax.summary
	otus.below.98.general.wang.taxonomy
	otus.below.98.general.wang.tax.summary

NOTE: these bootstrap percent confidence values are NOT the confidence that the taxonomy
assignment is *correct*, just that it is *repeatable* in that database.  This is another
paremeter that we could explore changing more in the future.  It likely should be different
in the different databases also because of the different database sizes.  I left cutoff out
in this command so that you can explore the different results of it later, in step 11.


__________________________________________________________________________________________

8. combine taxonomy files (terminal)

Concatenate the two taxonomy files to create one complete one.  You can very simply just
combine them because there are no duplicate sequences between them.  The cat command in bash
concatenates two files into one.  You also choose your final file name here.

Full Command (type in terminal):

cat otus.above.98.custom.wang.taxonomy otus.below.98.general.wang.taxonomy > otus.taxonomy

Command syntax:
	cat file1 file2 > file3		means "combine file1 and file2 into file 3"

What the filenames are:
	otus.above.98.custom.wang.taxonomy		the taxonomy file for sequences classified with custom database
	otus.below.98.general.wang.taxonomy		the taxonomy file for sequences classified with general database
	otus.taxonomy							the name you choose for the output complete taxonomy file for all
											your sequences.


__________________________________________________________________________________________

Steps 9-11 are an optional check.
__________________________________________________________________________________________

9. assign taxonomy with general database only (mothur, bash)

Get a large, general database classification of your otus.fasta file to compare too.  
Green Genes, our general database choice, is a huge database (1,262,986 sequences), so the
taxonomy assignment clustering algorithm is likely to only settle on a given taxonomic
assignment if it is unambiguously correct.  Therefore, we trust the upper level Green Genes
assignments more than our custom database, even though we trust the lower level taxonomic
assignments using our custom database more.

So in this step you assign taxonomy with the general database using mothur, and then
rename the output file something easy to work with using bash.

Full Four Commands (type in terminal):

~/mothur/mothur
classify.seqs(fasta=otus.fasta, template=general.fasta, taxonomy=general.taxonomy, method=wang, probs=T, processors=2)
quit()
cat otus.general.wang.taxonomy > otus.general.taxonomy

What the commands do:
	See step 7 for a detailed explanation of these four commands and their arguments.

What the filenames are:
	otus.fasta			the original fasta file of your OTU sequences
	general.fasta		the fasta file of the large, general database
	general.taxonomy	the taxonomy file of the large, general database


__________________________________________________________________________________________

10. reformat taxonomy files (textwrangler or bash)

The R script in step 11 requires semicolon delimited taxonomy files.  The mothur output
.taxonomy files are delimited with both tabs and semicolons.

Reformat both files you will compare:
	the otus.taxonomy file created in step 8
	the otus.general.taxonomy file created in step 9

Find: tab
Replace: semicolon

You can do this in a text editor, or using bash.

For bash, Full Four Commands (Type in Terminal):

sed 's/[[:blank:]]/\;/' <otus.taxonomy >otus.taxonomy.reformatted
mv otus.taxonomy.reformatted otus.taxonomy
sed 's/[[:blank:]]/\;/' <otus.general.taxonomy >otus.genearl.taxonomy.reformatted
mv otus.general.taxonomy.reformatted otus.general.taxonomy

Syntax of commands and what each argument does:
	sed 's/find/replace/ <input >output
		sed		is a "stream editor," a function for editing streams of text in the terminal
		's		tells it you are doing a substitution
		find	this is the character string you are finding
		replace	this is what you are replacing it with
		input	this is the file sed searches through, here it is
				otus.taxonomy
				otus.general.taxonomy
		output	this is the file sed creates (note it must have a different name than input)

	mv filename1 filename2
		mv		this is a function to move (aka rename) a file from name1 to name2
				simply keeping the edited file the same name  


***I should use mv above when I rename files too, to keep the total # of crap down
instead of cat that I think leaves the old file there still too...


__________________________________________________________________________________________

11. compare taxonomy files (R)


The R script find_classification_disagreements.R  creates a folder containing a file for
each upper taxonomic level (kingdom, phylum, class, order, lineage) that lists all of the
classification disagreements at that taxonomic level between the custom + general taxonomy
database workflow and the general only taxonomy database workflow.

This script also allows the user to choose a bootstrap %confidence cutoff under which all
the lower assignments are unclassified.  The script does not include unclassified names in
its reporting of disagreements.  This allows you ignore taxonomy assignment conflicts when
you don't trust the assignment anyway, and you can decide what you trust (60% is generally
considered the minimum cutoff you should use.)

Note: you must create a new folder to save these results in before running the script.
	include the pident cutoff in your folder name, b/c the file names will not include that.

Full Command (type in terminal):

Rscript find_classification_disagreements.R fw.plus.gg.tax.file.path gg.only.tax.file.path results.folder.path taxonomy.bootstrap.cutoff

Note: you must enter all the arguments in this order.

What the arguments are:
	Rscript									this open R to accept arguments from the command line
	find_classification_disagreements.R		this is the R script you're calling
	fw.plus.gg.tax.file.path				this is the path to your custom taxonomy file
											i.e. the path to otus.taxonomy created in step 8
	gg.only.tax.file.path					this is the path to the general-only taxonomy file
											i.e. the path to otus.general.taxonomy
	results.folder.path						this is the path to the folder you want the R script
											to save the .csv results files in.
											NOTE: you must create this folder before running script!
	taxonomy.bootstrap.cutoff				this is the % confidence cutoff you want to use
											on the general database assignments

What the generated files in your folder are:
	kingdom_conflicts.csv
	phylum_conflicts.csv
	class_conflicts.csv
	order_conflicts.csv
	lineage_conflicts.csv

****should add a way to separately specify the bootstrap cutoff for each database-
because it makes sense to make our database cutoff much higher than the gg one
buuuuttt, maybe doesn't matter.  maybe just something to do for the final file version.
like I should just write that part of the script into a separate script that gives ppl
a clean file with only the cutoff they want showing with names.
