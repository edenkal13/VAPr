# VAPr 
## Variant Annotation and Prioritization package

This package is aimed at providing a way of retrieving variant information using [ANNOVAR](http://annovar.openbioinformatics.org/en/latest/) and [myvariant.info](http://myvariant.info/). In particular, it is suited for bioinformaticians interested in aggregating variant information into a single NoSQL database (MongoDB solely at the moment). 

# Updates and Latest Improvements (v.2.0.5)

1. Optimizations

- Variant querying from MyVariant.info with asynchronous requests 
- Parallel parsing to MongoDB using multiprocessing library

These improvements resulted in a 4-10x speedup for the entire processing pipeline.
 
2. Usability

- Possibility of providing any number of vcf files in a folder and sub-folders with auto-detection of samples/sample names in each file
- Usage of a design file for users that have their vcf files grouped by sample name
- Multi-sample vcf files support

3. Features Added

- Filters for deleterious heterozygote variants
- Trio analysis filters for de novo mutations

## Background

VAPr was developed to simplify the steps required to get mutation data from a VCF file to a downstream analysis process.
 A query system was implemented allowing users to quickly slice the genomic variant (GV) data and select variants 
 according to their characteristics, allowing researchers to focus their analysis only on the subset of data that contains meaningful 
 information. Further, this query system allows the user to select the format in which the data can be retrieved. 
 Most notably, CSV or VCF files can be retrieved from the database, allowing any researcher to quickly filter variants 
 and retrieve them in commonly used formats. 
The package can also be installed and used without having to download ANNOVAR. In that case, variant data can be 
retrieved solely by MyVariant.info and rapidly parsed to the MongoDB instance. 

### Annovar


ANNOVAR, along with at least some of its data sets, should to be installed. We can not provide it with this package since
 downloading ANNOVAR requires the user to register on [their website](http://www.openbioinformatics.org/annovar/annovar_download_form.php).
 The data sets required to benefit from the full functionalities of the package are the following:

- knownGene
- tfbsConsSites
- cytoBand
- genomicSuperDups
- esp6500siv2_all
- 1000g2015aug_all
- popfreq_all
- clinvar_20140929
- cosmic70
- nci60

### Available Methods
The package offers two different ways to obtain variant data. One requires annovar, while the other is based solely on the
use of the MyVariant.info service. The latter can be used without having to worry about downloading and installing annovar
databases, but it tends to return partial or no information for some variants. 
Calling the methods is extremely straightforward since the syntax is the same. The main difference is in the arguments 
passed to the class VariantParsing.

### Data Models
Intuitively, variant data could be stored in SQL-like databases, since annotation files are usually produced in VCF or
CSV formats. However, a different approach may be more fruitful. As explained on our paper (currently under review), 
the abundance and diversity of genomic variant data causes SQL schemas to perform poorly for variant storage and querying. 
As it can be the case for many variants, the number of different fields and sub-fields it can have can be over 500, 
with even more diverse nested sub-fields. Creating a pre-defined schema (as required by SQL-like engined) becomes rather 
impossible: representing such variant in a table format would thus result in a highly sparse and inefficient storage. 
Representing instead a variant **atomically**, that is, as a standalone JSON object having no pre-defied schema, it is 
possible to compress the rich data into a more manageable format. A sample entry in the Mongo Database will look like 
[this](https://github.com/ucsd-ccbb/VAPr/blob/master/sample_variant_document). The variety of data that can be retrieved 
from the sources results from the richness of databases that can be accessed through MyVariant.info. However, not every 
variant will have such data readily available. In some cases, the data will be restricted to what can be inferred from 
the vcf file and the annotation carried out with Annovar. In that case, the entries that will be found in the document 
will be the following:

    {
      "_id": ObjectId("5887c6d3bc644e51c028971a"),
      "chr": "chr19",
      "cytoband": {
        "Region": "13",
        "Sub_Band": "41",
        "Chromosome": 19,
        "Band": "q",
        "Name": "19q13.41"
            },
            
      "genotype": {
        "alleles": [
            0,
            2
            ],
        "genotype_likelihoods": [
              89.0,
              6.0,
              0.0
                ],
        "filter_passing_reads_count": 2,
        "genotype": "1/1"
          },
          
      "end": 51447065,
      "vcf": {
        "alt": "G",
        "position": "25194768",
        "ref": "A"
            }
        
      "hgvs_id": "chr20:g.25194768A>G",
      "esp6500siv2_all": "0.71"
    }


## Filtering 

Five different pre-made filters that allow for the retrieval of specific variants have been implemented. The filters 
are in the form of MongoDB queries, and are designed to provide the user with a set of relevant variants. In case the 
user would like to define her or his own query, a template is provided.
The output of the queries is a list of dictionaries (JSON documents), where each dictionary contains data relative to 
one variant. 
The data can be retrieved by querying any field that is contained in MyVariant.info or ANNOVAR annotation databases. 
Since there are more than 700 possible keys that can be associated to a variant, it is out of the scope of this
document to list them all. They are however fully available online at the bottom of [this page](http://docs.myvariant.info/en/latest/doc/data.html). If the queries are
done instead on the ANNOVAR data fields, then the possible keys to be queried upon are the following:

- `chr`   
- `cytoband.Region` 
- `cytoband.Sub_Band`
- `cytoband.Chromosome`
- `cytoband.Band`
- `cytoband.Name` 
- `genotype.alleles`
- `genotype.genotype_lieklihoods`
- `genotype.filter_passing_reads_count`
- `genotype.genotype`
- `start`
- `end`
- `vcf.alt`
- `vcf.position`
- `vcf.ref`
- `hgvs_id`
- `esp6500siv2_all`

These are entries that are guaranteed to be present for every variant. Their typical values can be retrieved from the 
dictionary displayed above, which can serve as a guide to build effective queries that target the desired values. 


### Filter #1: specifying cancer-specific rare variants

- filter 1: ThousandGenomeAll < 0.05 or info not available
- filter 2: ESP6500siv2_all < 0.05 or info not available
- filter 3: cosmic70 information is present
- filter 4: Func_knownGene is exonic, splicing, or both
- filter 5: ExonicFunc_knownGene is not "synonymous SNV"
- filter 6: Read Depth (DP) > 10

### Filter #2: specifying rare disease-specific (rare) variants

- filter 1: ThousandGenomeAll < 0.05 or info not available
- filter 2: ESP6500siv2_all < 0.05 or info not available
- filter 3: cosmic70 information is present
- filter 4: Func_knownGene is exonic, splicing, or both
- filter 5: ExonicFunc_knownGene is not "synonymous SNV"
- filter 6: Read Depth (DP) > 10
- filter 7: Clinvar data is present 

### Filter #3: specifying rare disease-specific (rare) variants with high impact

- filter 1: ThousandGenomeAll < 0.05 or info not available
- filter 2: ESP6500siv2_all < 0.05 or info not available
- filter 3: cosmic70 information is present
- filter 4: Func_knownGene is exonic, splicing, or both
- filter 5: ExonicFunc_knownGene is not "synonymous SNV"
- filter 6: Read Depth (DP) > 10
- filter 7: Clinvar data is present 
- filter 8: cadd.phred > 10

### Filter #4: specifying deleterious compound heterozygote variants

- filter 1: genotype heterozyogosity subclass is compound
- filter 2: cadd.phred > 10

### Filter #5: finding De Novo Mutations

- A de novo variant is defined as a variant that occurs only in the specified sample (sample1) and not on the other two 
(sample2, sample3). Occurrence is defined as having allele frequencies greater than `[0, 0] ([REF, ALT])` for multi-sample
vcf files. 
-  For the case in which the variants come from multiple files, de novo variants are the ones that occur only 
once on the sample specified (usually found in one file), and that do not occur on the other samples (found in the other files)

### Create your own filter

As long as you have a MongoDB instance running, filtering can be performed through pymongo as shown by the code below. 
If a list is intended to be created from it, simply add: `filter2 = list(filter2)`

If you'd like to customize your filters, a good idea would be to look at the available fields to be filtered. 
Looking at the myvariant.info [documentation](http://docs.myvariant.info/en/latest/doc/data.html), you can see what are 
all the fields available and can be used for filtering.

## Output Files

### Create unfiltered annotated vcf and csv files 

The package lets you create unfiltered and filtered annotated csv and vcf files. See the docs on available methods.


**Citations**

* Wang K, Li M, Hakonarson H. [ANNOVAR: Functional annotation of genetic variants from next-generation sequencing data Nucleic Acids Research](http://nar.oxfordjournals.org/content/38/16/e164) , 38:e164, 2010
* Xin J, Mark A, Afrasiabi C, Tsueng G, Juchler M, Gopal N, Stupp GS, Putman TE, Ainscough BJ, Griffith OL, Torkamani A, Whetzel PL, Mungall CJ, Mooney SD, Su AI, Wu C (2016) [High-performance web services for querying gene and variant annotation.](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-016-0953-9) Genome Biology 17(1):1-7

