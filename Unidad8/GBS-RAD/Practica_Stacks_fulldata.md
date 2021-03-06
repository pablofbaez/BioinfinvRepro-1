# Práctica Stacks

Esta prática de Stacks está basada en:

* El protocolo Rochette, N. C. & Catchen, J. M. Deriving genotypes from RAD-seq short-read data using Stacks. Nature Protocols 12, 2640–2659 (2017).

* Los materiales (datos y scripts) que lo acompañan en: https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/)

* El [Manual de stacks](http://catchenlab.life.illinois.edu/stacks/manual)

* La versión v2.2 de Stacks

**NOTAS IMPORTANTES:** 

1) dependiendo de cómo esté instalado Stacks en tu servidor, quizá tengas que correr los comandos de Stacks con `stacks` antes de cada comando. Por ejemplo: `stacks process_radtags` en vez de `process_radtags`. 

2) Algunos flags dentro de los parámetros de Stacks cambiaron entre la versión que Rochette & Catchen (2017) utilizaron y la actualmente disponible. Por lo tanto los modifiqué.  **El [manual de Stacks](http://catchenlab.life.illinois.edu/stacks/manual/) de la versión que tengas instalada siempre debe de ser tu referencia** de los comandos de Stacks.

3) Los comandos de abajo son ligeramente distintos a los de Rochette debido a las diferencias del punto 2 y a que ajusté las rutas relativas.

4) El tutorial utiliza corre de forma interactiva ("líneas de código en la terminal" en vez de scripts. Pero recuerda que cuadno lo estés utilizando en tu proyecto deberás **hacer tus scripts** y probablemente correrlos en un cluster con un sistema de colas. 

### Obtener los datos del tutorial:

(para la demostración en vivo yo ya hice estos pasos previamente para ahorrar tiempo pero lo dejo aquí como nota).

`wget http://catchenlab.life.illinois.edu/data/rochette2017_gac_or.tar.gz`

Descomprimir:

`tar -xzvf rochette2017_gac_or.tar.gz`

Verás que dentro del arhivo descomprimido hay un solo directorio, llamadado **top**. La idea es que esta sea el nombre de tu proyecto, puedes dejarlo con algo genérico como *top* o ponerle el nombre corto de tu proyecto, eg *Juniperus_RAD* 

Dentro folder `top` está la estructura típica de un proyecto de Stacks para analizar datos RADseq:

```
$ cd rochette2017_gac_or/top
$ ls 
README      cleaned       genome  raw            stacks.ref    tests.ref
alignments  demo_scripts  info    stacks.denovo  tests.denovo

```

En esta práctica no vamos a seguir todos los pasos a detalle, así que no utilizaremos los scripts de `demo_scripts` sino una versión simplificada de los pasos principales. Sin embargo durante todo el tutorial **asumimos que demo_scritps es tu wd**

Si quieres siguir el protocolo de Rochette completo, recomiendo borrar los scripts de `demo_scripts` y utilizar los que están disponibles en línea, pues tienen pequeñas actualizaciones útiles (puedes bajarlos todos con `wget https://bitbucket.org/rochette/rad-seq-genotyping-demo/get/c22c0c66b8fd.zip`). Pero ojo con tu versión de Stacks.


## ¿Qué va en cada directorio?

* **raw:** aquí van tus secuencias `.fq.gz` crudas, tal cual salidas del secuenciador. (paso 0)

* **cleaned:** aquí van tus secuencias después de hacer un análisis de calidad, limpiarlas y demultiplexearlas. (pasos 1 y 2). Alternativamente puedes tener un directorio `cleaned` y otro `samples` para las lecturas limpias y demultiplexeadas, respectivamente, si haces estos pasos por separado.

* **genome:** aquí vivirá tu genoma de referencia. (paso 3). Este directorio solo existe si corres stacks con genoma de referencia.

* **tests.denovo** aquí van los resultados de tus (varias) pruebas de ensamblado de novo con algunas pocas muestras (paso 4a)

* **stacks.denovo:** aquí van los resultados del ensamblado de novo con los parámetros que elegiste (paso 5a)

* **aligments:** aquí va el resultado de alinear las secuencias de tus muestras vs el genoma de referencia. Este directorio solo existe si corres stacks con genoma de referencia.

* **stacks.ref:** aquí va el resultado de llamar los genotipos 


Vamos a ver paso por paso la pipeline para llegar a eso.


#### Preparar input
El paso 0 de la pipeline es poner nuestras secuencias en `/raw`, y los barcodes y population map en `info`. Los barcodes deben estar en un archivo que se vea así:

`<barcode>TAB<sample name>`

Por ejemplo: 

```
$ cat info/barcodes.lane1.tsv 
CTCGCC	sj_1819.35
GACTCT	sj_1819.31
GAGAGA	sj_1819.32
GATCGT	sj_1819.33
GCAGAT	sj_1819.34
GCCGTA	sj_1819.36
GCGACC	sj_1819.37
GCGCTG	sj_1819.38
GCTCAA	sj_1819.39
...
```

El population map describe a qué población corresponde cada muestra:

Estructura:

`<sample name>TAB<population name>`

Ejemplo:

```
$ cat info/popmap.tsv 
cs_1335.01	cs
cs_1335.02	cs
cs_1335.03	cs
cs_1335.04	cs
pcr_1193.00	pcr
pcr_1193.01	pcr
```

#### Paso 1) Revisar la calidad de los datos

**Nota** recuerda que asumiremos que estamos en `demo_scripts`

Ver si las secuencias empiezan como lo esperamos según nuestros barcodes y nuestra enzima de restricción. Por ejemplo para los datos del tutorial esperamos un barcode de 6 bases + sitio de corte de SbfI tras la ligación (TGCAGG).

Para una muestra:

```
$ zcat ../raw/lane1/lane1_NoIndex_L001_R1_001.fq.gz | head -n 20
@HWI-ST0747:233:C0RH3ACXX:1:1101:10000:10465 1:N:0:
GGAGCAGAGAGGAGACTAAAGGGGAGCAGAGGGAAGACTAAAGGGGAGCAGAGGGAAGACTAAAGGGGAGCAGAGGAGAGACTAAAGGGGAGCAGAGAGGA
+
CCCFFFFFHHHHHJJJJJJIJJJJIJJGIGIIJ@GHIJJJJJJIJJGHGHHHHFF8BCEEECDDDCDDB@BDDCDBA@BDDDDDDDDDDD-<B?C??9?@8
@HWI-ST0747:233:C0RH3ACXX:1:1101:10000:17644 1:N:0:
TTCCGTTGCAGGGACAGGAGCGCAGGAAATGAGCAGAGGAGGGTTTAACGTACCGGAGGCAGAAGCTGGTTCGACAAAGCGAATGTAAACTGTAAAGCCTC
+
BBBBBBFFHHHHHJJIJJHJJJJJIJIJJJJIJIHIGIIEIJJ;DGGEIHHFFFFDCDD@BBBCDDDDD?A@B<B59CCCDDD7AADDEDCCABDEDCDA<
@HWI-ST0747:233:C0RH3ACXX:1:1101:10000:2461 1:N:0:
TTCTAGTGCAGGTCTCCTGCTACCACCATCCCACCTCTGCAGCTCATTCAGAATGAAGCAGCTCCCCCCGTCTTTTTCCTTCCTACTTTCTCCCTCACTCC

```

Ejemplo de cómo hacerlo para todas las muestras:

[1.survey_lanes.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/1.survey_lanes.sh)



#### Paso 2) Limpiar y demultiplexear

Nota: Recuerda dependiendo de cómo esté instalado Stacks en tu servidor, quizá tengas que correr los comandos de Stacks con `stacks` antes de cada comando.

Crear variables con rutas relativas:

```
raw_dir=../raw/lane1/barcodes_file=../info/barcodes.lane1.tsv
out_dir=../cleaned/
```

Correr [process_radtags para demultiplexear](http://catchenlab.life.illinois.edu/stacks/manual/#clean) según tu tipo de barcodes:

```
process_radtags -p $raw_dir -b $barcodes_file -o $out_dir -e sbfI --inline_null -c -q -r 
```

Mientras corre tu terminal debe verse así:

```
Processing single-end data.
Using Phred+33 encoding for quality scores.
Found 10 input file(s).
Searching for single-end, inlined barcodes.
Loaded 24 barcodes (6bp).
Will attempt to recover barcodes with at most 1 mismatches.
Processing file 1 of 10 [lane1_NoIndex_L001_R1_002.fq.gz]
  Processing RAD-Tags...1M...2M...3M...
  3891784 total reads; -327491 ambiguous barcodes; -79841 ambiguous RAD-Tags; +89869 recovered; -354523 low quality reads; 3129929 retained reads.
Processing file 2 of 10 [lane1_NoIndex_L001_R1_007.fq.gz]
  Processing RAD-Tags...1M...2M...3M...
  3906654 total reads; -357482 ambiguous barcodes; -86906 ambiguous RAD-Tags; +120284 recovered; -296963 low quality reads; 3165303 retained reads.
...
```

Puedes ver qué hacen todas las flags en el [manual de process_radtags](http://catchenlab.life.illinois.edu/stacks/comp/process_radtags.php).


Nuestro output al debe generar un archivo `fa.gz` para cada muestra y un archivo `.log` para cada lane:

```
$ ls ../cleaned/
cs_1335.01.fq.gz  cs_1335.15.fq.gz   pcr_1211.02.fq.gz          process_radtags.lane2.oe   sj_1484.06.fq.gz  wc_1218.05.fq.gz
cs_1335.02.fq.gz  cs_1335.16.fq.gz   pcr_1211.03.fq.gz          process_radtags.lane3.log  sj_1484.07.fq.gz  wc_1218.06.fq.gz
cs_1335.03.fq.gz  cs_1335.17.fq.gz   pcr_1211.04.fq.gz          process_radtags.lane3.oe   sj_1819.31.fq.gz  wc_1218.07.fq.gz
cs_1335.04.fq.gz  cs_1335.19.fq.gz   pcr_1211.05.fq.gz          sj_1483.01.fq.gz           sj_1819.32.fq.gz  wc_1219.01.fq.gz
cs_1335.05.fq.gz  pcr_1193.00.fq.gz  pcr_1211.06.fq.gz          sj_1483.02.fq.gz           sj_1819.33.fq.gz  wc_1219.02.fq.gz
cs_1335.06.fq.gz  pcr_1193.01.fq.gz  pcr_1211.07.fq.gz          sj_1483.03.fq.gz           sj_1819.34.fq.gz  wc_1219.04.fq.gz
cs_1335.07.fq.gz  pcr_1193.02.fq.gz  pcr_1211.08.fq.gz          sj_1483.04.fq.gz           sj_1819.35.fq.gz  wc_1219.05.fq.gz
cs_1335.08.fq.gz  pcr_1193.08.fq.gz  pcr_1211.09.fq.gz          sj_1483.05.fq.gz           sj_1819.36.fq.gz  wc_1220.01.fq.gz
cs_1335.09.fq.gz  pcr_1193.09.fq.gz  pcr_1211.10.fq.gz          sj_1483.06.fq.gz           sj_1819.37.fq.gz  wc_1220.02.fq.gz
cs_1335.10.fq.gz  pcr_1193.10.fq.gz  pcr_1211.11.fq.gz          sj_1484.01.fq.gz           sj_1819.38.fq.gz  wc_1220.03.fq.gz
cs_1335.11.fq.gz  pcr_1193.11.fq.gz  pcr_1213.02.fq.gz          sj_1484.02.fq.gz           sj_1819.39.fq.gz  wc_1221.01.fq.gz
cs_1335.12.fq.gz  pcr_1193.12.fq.gz  process_radtags.lane1.log  sj_1484.03.fq.gz           sj_1819.40.fq.gz  wc_1221.02.fq.gz
cs_1335.13.fq.gz  pcr_1210.05.fq.gz  process_radtags.lane1.oe   sj_1484.04.fq.gz           sj_1819.41.fq.gz  wc_1221.04.fq.gz
cs_1335.14.fq.gz  pcr_1211.01.fq.gz  process_radtags.lane2.log  sj_1484.05.fq.gz           wc_1218.04.fq.gz  wc_1222.02.fq.gz
```

¡El log tiene info muy importante! Nos dice cuántas lecturas de cada muestra se recuperaron, y si no se recuperaron, por qué.

```
$ less ../cleaned/process_radtags.lane1.log 

process_radtags v2.2, executed 2020-05-04 17:41:50
/usr/lib/stacks/bin/process_radtags -p ../raw/lane1/ -b ../info/barcodes.lane1.tsv -o ../cleaned/ -e sbfI --inline_null -c -q -r
File    Retained Reads  Low Quality     Barcode Not Found       RAD cutsite Not Found   Total
lane1_NoIndex_L001_R1_001.fq.gz 3136039 352260  321836  78281   3888416
lane1_NoIndex_L001_R1_002.fq.gz 3129929 354523  327491  79841   3891784
lane1_NoIndex_L001_R1_003.fq.gz 3152587 329947  326572  80827   3889933
lane1_NoIndex_L001_R1_004.fq.gz 3195170 292073  317415  78743   3883401
lane1_NoIndex_L001_R1_005.fq.gz 3152970 313188  350822  84884   3901864
lane1_NoIndex_L001_R1_006.fq.gz 3140862 341788  332094  82297   3897041
lane1_NoIndex_L001_R1_007.fq.gz 3165303 296963  357482  86906   3906654
lane1_NoIndex_L001_R1_008.fq.gz 3175402 309017  322006  80990   3887415
lane1_NoIndex_L001_R1_009.fq.gz 3167011 318301  324280  80895   3890487
lane1_NoIndex_L001_R1_010.fq.gz 452235  54472   47744   12124   566575

Total Sequences 35603570
Barcode Not Found       3027742
Low Quality     2962532
RAD Cutsite Not Found   745788
Retained Reads  28867508

Barcode Filename        Total   NoRadTag        LowQuality      Retained
CTCGCC  sj_1819.35      1346898 30273   118707  1197918
GACTCT  sj_1819.31      26185   25574   163     448
GAGAGA  sj_1819.32      1061822 40347   91021   930454
GATCGT  sj_1819.33      1306694 28929   122723  1155042
GCAGAT  sj_1819.34      982500  34682   84821   862997
GCCGTA  sj_1819.36      1180712 24784   108070  1047858
GCGACC  sj_1819.37      977166  18061   85919   873186
GCGCTG  sj_1819.38      1540755 40369   137439  1362947

```

Script completo en el protocolo de Rochette: [2.clean_lanes.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/2.clean_lanes.sh). Nota que una parte del script sirve para analizar los logs.


#### Paso 3) Conseguir genoma de referencia

Si existe genoma de referencia para tu especie. Yey. Si no, sáltate este paso.

Baja el archivo `.fa` o `fa.gz` con el genoma de referencia de tu organismo y guárdalo en `genome`.

El de Gasterosteus aculeatus de este ejemplo puede bajarse de [ENSEMBL](https://www.ensembl.org/Gasterosteus_aculeatus/Info/Index) con

```
$ wget "ftp://ftp.ensembl.org/pub/release-87/fasta/gasterosteus_aculeatus/dna/Gasterosteus_aculeatus.BROADS1.dna.toplevel.fa.gz" || exit
$ mv Gasterosteus_aculeatus.BROADS1.dna.toplevel.fa.gz ../genome
```

Una vez que lo hayas bajado, crea la bd para el alineamiento:

```
genome_fa=../genome/Gasterosteus_aculeatus.BROADS1.dna.toplevel.fa.gz
mkdir ../genome/bwa
bwa index -p ../genome/bwa/gac $genome_fa > ../genome/bwa/bwa_index

```

Script completo en el protocolo de Rochette: [3.genome_db.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/3.genome_db.sh)


#### Paso 4 Hacer pruebas con un set de datos peque

Correr Stacks, tanto de novo como con genoma de referencia, tarda. Y como tendrás que hacer varias pruebas para ajustar parámetros es buena idea hacer estas pruebas con un subset de los datos. Por ejemplo 10-12 muestras representativias de tus taxa/poblaciones y número de secuencias por muestras.

Escoje estas muestras y ponlas en un population map de prueba que viva en `info`y que se vea maso así: 

```
$ cat ../info/popmap.test_samples.tsv
cs_1335.01	cs
cs_1335.13	cs
cs_1335.15	cs
pcr_1193.10	pcr
pcr_1193.11	pcr
pcr_1211.04	pcr
sj_1483.06	sj
sj_1484.07	sj
sj_1819.36	sj
wc_1218.04	wc
wc_1221.01	wc
wc_1222.02	wc
```


**NOTA IMPORTANTE:** Si estás siguiendo el protocolo de Mastretta-Yanes et al (2015) utilizando muestras replicadas, este subset de datos deben ser tus muestras replicadas.


#### Paso 4a) Pruebas con ensamble de novo

Script para el paso 4a completo en el protocolo de Rochette: [4.tests_denovo.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/4.tests_denovo.sh)

* **4a.1: Decide cuál de las [estrategias para escoger parámetros de novo utilizarás](http://catchenlab.life.illinois.edu/stacks/manual/#params)** (en realidad ya tomaste esta decisión al hacer tu trabajo de laboratorio).

* **4a.2: Escoge una serie de combinaciones de valores de los [principales parámetros de Stacks de novo](http://catchenlab.life.illinois.edu/stacks/param_tut.php).** 

* **4a.3: Para cada una de estas combinaciones crea un directorio output dentro de `/tests.denovo`.**

Recuerda ponerles nombres relevantes. Puedes usar a nuestros amigos los for loops para esto.

Ejemplo:

```
$ M_values="1 2 3 4 5 6 7 8 9"
$ for M in $M_values ;do
	mkdir ../tests.denovo/stacks.M$M
done
```

* **4a.4: Corre el wrapper [denovo_map](http://catchenlab.life.illinois.edu/stacks/comp/denovo_map.php) de stacks, que corre en un solo comando  los [pasos de la pipeline de novo](http://catchenlab.life.illinois.edu/stacks/manual/#phand).**

**Nota:** este es de los pasos que más tardan en correr.

Una vez más aprovecha tu sabiduría de los for loops. Por ejemplo para probar valores de M del 1 al 9 dejando fijo n y m:

```
M_values="1 2 3 4 5 6 7 8 9"
popmap=../info/popmap.test_samples.tsv
for M in $M_values ;do
	n=$M
	m=3
	echo "Running Stacks for M=$M, n=$n..."
	reads_dir=../cleaned
	out_dir=../tests.denovo/stacks.M$M
	log_file=$out_dir/denovo_map.oe
	denovo_map.pl --samples $reads_dir -O $popmap -o $out_dir -M $M -n $n -m $m
done
```

Que es lo mismo que poner en tu script (lo cual es super error prone y se ve horrible):

```
denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M1 -M 1 -n 1 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M2 -M 2 -n 2 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M3 -M 3 -n 3 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M4 -M 4 -n 4 -m 3

denovo_map.pl --samples ../cleaned -O ../info/
popmap.test_samples.tsv -o ../tests.denovo/stacks.M5 -M 5 -n 5 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M6 -M 6 -n 6 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M7 -M 7 -n 7 -m 3

denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M8 -M 8 -n 8 -m 3

ref_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M9 -M 9 -n 9 -m 3
```

**Pregunta** ¿qué hace cada uno de los flags de stacks? 

**Ejercicio** modifica el loop anterior **dejando fijo el valor de M** y **variando el valor de n** del 1 al 5.

El ouput dentro de cada uno de tus directorios de resultados se verá parecido a este:

```
$ ls ../tests.denovo/stacks.M1
catalog.alleles.tsv.gz     cs_1335.15.tags.tsv.gz      populations.haplotypes.tsv        sj_1819.36.matches.tsv.gz
catalog.calls              denovo_map.log              populations.hapstats.tsv          sj_1819.36.snps.tsv.gz
catalog.fa.gz              gstacks.log                 populations.log                   sj_1819.36.tags.tsv.gz
catalog.snps.tsv.gz        gstacks.log.distribs        populations.log.distribs          tsv2bam.log
catalog.tags.tsv.gz        pcr_1193.10.alleles.tsv.gz  populations.markers.tsv           wc_1218.04.alleles.tsv.gz
cs_1335.01.alleles.tsv.gz  pcr_1193.10.matches.bam     populations.sumstats.tsv          wc_1218.04.matches.bam
cs_1335.01.matches.bam     pcr_1193.10.matches.tsv.gz  populations.sumstats_summary.tsv  wc_1218.04.matches.tsv.gz
cs_1335.01.matches.tsv.gz  pcr_1193.10.snps.tsv.gz     sj_1483.06.alleles.tsv.gz         wc_1218.04.snps.tsv.gz
cs_1335.01.snps.tsv.gz     pcr_1193.10.tags.tsv.gz     sj_1483.06.matches.bam            wc_1218.04.tags.tsv.gz
cs_1335.01.tags.tsv.gz     pcr_1193.11.alleles.tsv.gz  sj_1483.06.matches.tsv.gz         wc_1221.01.alleles.tsv.gz
cs_1335.13.alleles.tsv.gz  pcr_1193.11.matches.bam     sj_1483.06.snps.tsv.gz            wc_1221.01.matches.bam
cs_1335.13.matches.bam     pcr_1193.11.matches.tsv.gz  sj_1483.06.tags.tsv.gz            wc_1221.01.matches.tsv.gz
cs_1335.13.matches.tsv.gz  pcr_1193.11.snps.tsv.gz     sj_1484.07.alleles.tsv.gz         wc_1221.01.snps.tsv.gz
cs_1335.13.snps.tsv.gz     pcr_1193.11.tags.tsv.gz     sj_1484.07.matches.bam            wc_1221.01.tags.tsv.gz
cs_1335.13.tags.tsv.gz     pcr_1211.04.alleles.tsv.gz  sj_1484.07.matches.tsv.gz         wc_1222.02.alleles.tsv.gz
cs_1335.15.alleles.tsv.gz  pcr_1211.04.matches.bam     sj_1484.07.snps.tsv.gz            wc_1222.02.matches.bam
cs_1335.15.matches.bam     pcr_1211.04.matches.tsv.gz  sj_1484.07.tags.tsv.gz            wc_1222.02.matches.tsv.gz
cs_1335.15.matches.tsv.gz  pcr_1211.04.snps.tsv.gz     sj_1819.36.alleles.tsv.gz         wc_1222.02.snps.tsv.gz
cs_1335.15.snps.tsv.gz     pcr_1211.04.tags.tsv.gz     sj_1819.36.matches.bam            wc_1222.02.tags.tsv.gz
```

En resumen para cada muestra te generó un archivo **tags.tsv**, **snps.tsv**, **alleles.tsv**, **matches.tsv** y **matches,bam**. [Puedes ver que contienen exactamente aquí](http://catchenlab.life.illinois.edu/stacks/manual/#files). 

Y no olvides que los logs también están llenos de información preciada

```
$ less ../tests.denovo/stacks.M1/denovo_map.log 
denovo_map.pl version 2.2 started at 2020-05-04 20:11:32
/usr/lib/stacks/bin/denovo_map.pl --samples ../cleaned -O ../info/popmap.test_samples.tsv -o ../tests.denovo/stacks.M1 -M 1 -n 1 -m 3

ustacks
==========

Sample 1 of 12 'cs_1335.01'
----------
/usr/lib/stacks/bin/ustacks -t gzfastq -f ../cleaned/cs_1335.01.fq.gz -o ../tests.denovo/stacks.M1 -i 1 -m 3 -M 1
ustacks parameters selected:
  Input file: '../cleaned/cs_1335.01.fq.gz'
  Sample ID: 1
  Min depth of coverage to create a stack (m): 3
  Repeat removal algorithm: enabled
  Max distance allowed between stacks (M): 1
  Max distance allowed to align secondary reads: 3
  Max number of stacks allowed per de novo locus: 3
  Deleveraging algorithm: disabled
  Gapped assembly: enabled
  Minimum alignment length: 0.8
  Model type: SNP
  Alpha significance level for model: 0.05

Loading RAD-Tags...1M...

Loaded 1302167 reads; formed:
  59499 stacks representing 1190118 primary reads (91.4%)
  104369 secondary stacks representing 112049 secondary reads (8.6%)
  ...
```


* 4a.5 Exporta loci a analizar con [populations](http://catchenlab.life.illinois.edu/stacks/comp/populations.php)
  
**Nota:** este paso corre rápido.

Si estás siguiendo el método r80 (Paris et al 2017), filtra los loci que solo estén en al menos el 80% de tus muestras:

```
M_values="1 2 3 4 5 6 7 8 9"
for M in $M_values ;do
	stacks_dir=../tests.denovo/stacks.M$M
	out_dir=$stacks_dir/populations.r80
	mkdir $out_dir
	log_file=$out_dir/populations
	populations -P $stacks_dir -O $out_dir -r 0.80
done

```

Si estás siguiendo el protocolo de replicados (Mastretta-Yanes et al 2015) puedes exportar todos los loci, o hacer un filtro como el de arriba. IMPORTANTE: Necesitarás además exportar los datos a plink, agregando el flag `--plink`.

```
$ populations -P $stacks_dir -O $out_dir -r 0.80 --plink
```

* **4a.6. Compara los outputs**.

La info que queremos está en los archivos output del paso anterior (`populations.r80/populations.log`), y para el método de los replicados además en el archivo plink (para método replicados).

*Método r80*

Te interesa comparar: el % de loci compartido entre el 80% de las muestras, el número de SNPs y el número de loci

Puedes hacer tus propias gráficas, o utilizar los scripts de R de Rochette [4.plot_n_loci.R](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/R_scripts/4.plot_n_loci.R) y [4.plot_n_snps_per_locus.R](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/R_scripts/4.plot_n_snps_per_locus.R). En cualquier forma para esto necesitarás sacar los datos del archivo `populations.r80/batch_1.populations.log`, de una sección que se ve así:


```
Removed 20595 loci that did not pass sample/population constraints from 57772 loci.
Kept 37177 loci, composed of 3589615 sites; 4023 of those sites were filtered, 65315 variant sites remained.
Mean genotyped sites per locus: 96.55bp (stderr 0.01).*
```

Te dejo un truquito (imo más sencillo que el de Rochette) para extraer esta info de todos los logs con nuestro amigo `sed` y nuestros infalibles loops.

```
mkdir -p ../tests.denovo/results
results_dir=../tests.denovo/results

# create file to save resutls
echo 'M,n,m,n_loci,n_snps' > $results_dir/n_snps_per_locus.tsv

# feed results with a loop
for M in $M_values ;do
	n=$M
	m=3
	log_file=../tests.denovo/stacks.M$M/populations.r80/populations.log

# get number of loci and snps from log file 
n_loci=$(sed -n 's/Kept \(.*\) loci.*/\1/p' $log_file)
n_snps=$(sed -n 's/.*, \([0-9]\{1,\}\) variant .*/\1/p' $log_file)

# echo results and save them to desired file
echo $M,$n,$m,$n_loci,$n_snps >> $results_dir/n_snps_per_locus.tsv
done
```

**Ejercicio** Revisa el contenido de `$results_dir/n_snps_per_locus.tsv` **sin** cambiar tu wd.


**Método replicados**

(no se puede hacer con este set de datos ejemplo)

Te interesa comparar: el número de loci y SNPs, % de missing data y valores de error de loci, alelos y SNPs, de los cuales el error en los SNPs es el más importante.

Los scripts para comparar esto puedes obteneros en el repo [RAD-error-rates](https://github.com/AliciaMstt/RAD-error-rates)


#### Paso 4b) Pruebas con genoma de referencia

* **4b.1. Escoge un método de alineamiento**

Revisa el Box 2 de Rochette et al para una discusión de método usar. Aquí utilizaremos BWA.

* **4b.2. Escoge parámetros de alineamiento**

La suerte de quienes tienen genoma de referencia es tal que normalmente no hace falta hacer tantas pruebas como con el ensambladod de novo, pues si el genoma de referencia es cercano se obtienen resultados buenos aún variando parámetros.
 
 * **4b.3. Crea los directorios `alignments` y `stacks` para resultados de prueba en el directorio `/tests.ref`**

Por ejemplo para nuestra prueba con bwa:

```
mkdir -p ../tests.ref/alignments.bwa
mkdir -p ../tests.ref/stacks.bwa/
```

* **4b.4. Alinea _algunas_ muestras (guardadas en `/cleaned`) contra el genoma de referencia.**

El prodecimineto base es:

`bwa mem -M <el genoma indexado previamente> <el .fq.gz de la muestra a alinear> | samtools view -b > <el bam output para esa muestra>`

Podemos usar el population map que hicimos antes (`../info/popmap.test_samples.tsv`) con las pocas muestras para las pruebas de_novo dentro de un loop:


```
# extract samples names from pop name to use in for loop
for sample in $(cut -f1 ../info/popmap.test_samples.tsv) ;do

# define variables
	fq_file=../cleaned/$sample.fq.gz
	bam_file=../tests.ref/alignments.bwa/$sample.bam
	log_file=../tests.ref/alignments.bwa/$sample
	db=../genome/bwa/gac
  
# run bwa & samtools 
  bwa mem -M $db $fq_file | samtools view -b | samtools sort --threads 4 > $bam_file
	
done

```

Mientras corre debe verse algo así:

```
[M::bwa_idx_load_from_disk] read 0 ALT contigs
[M::process] read 105264 sequences (10000080 bp)...
[M::process] read 105264 sequences (10000080 bp)...
[M::mem_process_seqs] Processed 105264 reads in 15.612 CPU sec, 15.484 real sec
[M::process] read 105264 sequences (10000080 bp)...
[M::mem_process_seqs] Processed 105264 reads in 15.763 CPU sec, 15.489 real sec
[M::process] read 105264 sequences (10000080 bp)...

```

Nota: en [5.tests_ref.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/5.tests_ref.sh) hay un error en la línea 22, pues hace falta correr `samtools sort` sin lo cual no funciona el siguiente paso.



* **4b.5. Corre `ref_map`**

El script del protocolo de Rochette [5.tests_ref.sh](https://bitbucket.org/rochette/rad-seq-genotyping-demo/src/default/demo_scripts/5.tests_ref.sh) corre los pasos por separado en vez de usando `ref_map`. Así que mejor revisa la [documentación de ref_map](http://catchenlab.life.illinois.edu/stacks/comp/ref_map.php). 

[ref_map](http://catchenlab.life.illinois.edu/stacks/comp/ref_map.php) es un wrapper de la pipeline con genoma de referencia, que al igual que `denovo_map` conjunta en un solo programa todos los pasos de la pipeline (una vez que tenemos las muestras alineadas), incluyendo `populations`. 


Ejemplo:

```
## define variables:

# directory with algimened samples
samples_dir=../tests.ref/alignments.bwa/

# popmap
popmap_file=../info/popmap.test_samples.tsv

# output directory
out_dir=../tests.ref/stacks.bwa/


## Run ref map
ref_map.pl --samples $samples_dir --popmap $popmap_file -o $out_dir 

``` 

Si todo sale bien debes ver algo así:

```
Parsed population map: 12 files in 4 populations and 1 group.
Found 12 sample file(s).

Calling variants, genotypes and haplotypes...
  /usr/lib/stacks/bin/gstacks -I ../tests.ref/alignments.bwa/ -M ../info/popmap.test_samples.tsv -O ../tests.ref/stacks.bwa

Calculating population-level summary statistics
  /usr/lib/stacks/bin/populations -P ../tests.ref/stacks.bwa -M ../info/popmap.test_samples.tsv

ref_map.pl is done.
```

El output de `ref_map` son estos archivos:

```
$ ls ../tests.ref/stacks.bwa/
catalog.calls  gstacks.log.distribs        populations.log           populations.sumstats.tsv
catalog.fa.gz  populations.haplotypes.tsv  populations.log.distribs  populations.sumstats_summary.tsv
gstacks.log    populations.hapstats.tsv    populations.markers.tsv   ref_map.log
```


* **4b.6. Revisa los logs para evaluar los alineamientos**

Revisa con cuidado los logs en tu direcotorio otuput de `ref_map`. En este caso (`../tests.ref/stacks.bwa/`):

* `ref_map.log` tiene información de cómo corrió toda la pipeline.

* En `populations.log` fíjate especialmente en estas líneas:

```
Removed 0 loci that did not pass sample/population constraints from 71209 loci.
Kept 71209 loci, composed of 6803166 sites; 0 of those sites were filtered, 68261 variant sites remained.
    6638037 genomic sites, of which 159500 were covered by multiple loci (2.4%).
Mean genotyped sites per locus: 95.44bp (stderr 0.01).

Population summary statistics (more detail in populations.sumstats_summary.tsv):
  cs: 2.8367 samples per locus; pi: 0.2326; all/variant/polymorphic sites: 4535745/67026/34800; private alleles: 7661
  pcr: 2.8591 samples per locus; pi: 0.2087; all/variant/polymorphic sites: 4482316/66781/30227; private alleles: 5824
  sj: 2.8461 samples per locus; pi: 0.25225; all/variant/polymorphic sites: 4599579/67304/37468; private alleles: 8109
  wc: 2.858 samples per locus; pi: 0.24353; all/variant/polymorphic sites: 5364807/67307/36334; private alleles: 7729
Populations is done.
```

Y **lo más importante de todo** revisa el log `gstacks.log.distribs` pues dice cuántos reads tenía tu muestra (records) y cuántos se mantuvieron (*_kept). Este es un punto clave a comparar entre diferentes alineamientos.

```
BEGIN bam_stats_per_sample
sample  records primary_kept    kept_frac       primary_kept_read2      primary_disc_mapq       primary_disc_sclip      unmapped        secondary       supplementary
cs_1335.01      1310138 1083979 0.827   0       153713  26230   38245   7971    0
cs_1335.13      1103479 919934  0.834   0       127705  19855   30184   5801    0
cs_1335.15      863091  727440  0.843   0       91113   16729   23581   4228    0
pcr_1193.10     1540304 1286836 0.835   0       168965  31028   45027   8448    0
pcr_1193.11     1497374 1246502 0.832   0       167745  29674   44822   8631    0
pcr_1211.04     1115081 925483  0.830   0       130134  21625   31255   6584    0
sj_1483.06      808591  673792  0.833   0       90957   15884   23674   4284    0
sj_1484.07      3177766 2593718 0.816   0       404030  64313   96655   19050   0
sj_1819.36      1052470 877528  0.834   0       119644  19000   31686   4612    0
wc_1218.04      1069490 883633  0.826   0       129614  20573   29938   5732    0
wc_1221.01      1857688 1545467 0.832   0       208903  38252   52919   12147   0
wc_1222.02      1128774 943803  0.836   0       123909  23055   31185   6822    0
END bam_stats_per_sample
...
```

Al igual que con la pipeline *de novo* compara el número de loci y snps obtenidos. 


### Paso 5 Correr Stacks con todas tus muestras

Este paso es esencialmente igual al paso 4, pero con todas tus muestras y los parámetros que elegiste como los mejores. 

Similar a como hicimos con los directorios para las pruebas, si estás trabajando **de novo** el output de tus scripts debes guardarlo en el directorio `/stacks.denovo` y si trabajaste con un **genoma de referencia** debes guardarlo en el directorio `/stacks.ref`. 

### Paso 6 Exporta tus datos con `populations` 

El programa de Stacks [populations](http://catchenlab.life.illinois.edu/stacks/comp/populations.php) te permite:

* Exportar tus datos a otros formatos útiles para análisis filogenéticos o de genética de poblaciones (vcf, plink, phylip, genepop, structure, y otros). 

* calcular *F*st, Pi, *F*is, HWE y otros.

* Repetir los dos puntos anteriores filtrando por loci/individuo 

Recuerda: el principal input de `populations` el el flag `P`, que es **la ruta al directorio** donde están guardados tus resultados de correr `denovo_map` o `ref_map`. Es decir en este tuturial `/stacks.denovo` y `/stacks.ref`.


### Ultimas recomendaciones

* Lee con detenimiento el manual de cada comando de Stacks que corras ¿qué hace cada flag? ¿cuál es el input y el output?

* Cada que corras un comando de Stacks has `ls` en el directorio output y **revisa** cada archivo (apoyándote de la documentación de Stacks). Recuerda que los archivos `.tsv` puedes revisarlos mejor en Excel o Libre Office.

* Levántate de tu silla regularmente y toma agua.

* Lee las recomendaciones y comandos para revisar que cada paso haya corrido correctamente que vienen en Rochette et al.

* **Ajusta Stacks a tus datos** ¿cómo son tus barcodes? ¿secuenciaste single end o pair end? En la documentación viene qué hacer en cada caso, pero debes tenerlo siempre presente. 


## Fin

Lo hiciste muy bien. Te mereces pensar en las maravillas de la evolución viendo este video de [una hydra comiéndose una larva de mosquito](https://youtu.be/SS_HYY97ehw) por haber llegado hasta aquí.  