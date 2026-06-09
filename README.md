# ensamble
Práctico de Ensamble


## Resumen

En este práctico vamos a correr la revisión de unos reads secuenciados mediante dos plataformas de secuenciación Illumina/ BGI y PacBio / ONT y luego utilizaremos dos ensambladores para realizar ensambles de los reads y obtener el genoma bacteriano, revisaremos las diversas métricas de ensamble y evaluaremos la calidad de un ensamble `de novo` utilizando los dos paradigmas más empleados en ensamble. La predicción de genes y su posterior anotación funcional de estos ensambles se abordarán en el próximo práctico.


## Materiales

### Software
 1. Quality Check:
  
   - [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
   - [LongQC](https://github.com/yfukasawa/LongQC)

 2. Trimming y filtrado:

   - [fastp](https://github.com/OpenGene/fastp)
   - [Filtlong](https://github.com/rrwick/Filtlong)

 3. Ensamble:

  - [Canu](https://canu.readthedocs.io/en/latest/)
  - [Spades](https://cab.spbu.ru/software/spades/). 

 4. Revisión del ensamble:

  - [assembly-stats](https://github.com/sanger-pathogens/assembly-stats)

> **Nota:** la predicción de [CDS](https://www.uniprot.org/help/cds_protein_definition) con [Prodigal](https://github.com/hyattpd/prodigal/wiki) y la anotación funcional con [BLAST+](https://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastDocs&DOC_TYPE=Download) / [Swiss-Prot](https://www.uniprot.org/statistics/Swiss-Prot) y [`eggnog-mapper`](https://github.com/eggnogdb/eggnog-mapper) se realizarán en el **próximo práctico**, a partir de los ensambles obtenidos aquí.


#### Input de datos:

 Cada grupo tendrá dos conjuntos de lecturas de secuenciación, correspondientes a un genoma desconocido, el cual tendrán que inferir con los análisis.

- il_1 e il_2: [Librería Paired End](https://www.illumina.com/science/technology/next-generation-sequencing/plan-experiments/paired-end-vs-single-read.html).

- long: secuencias largas (`long.fastq.gz`) que usaremos para ensamblar con Canu y como apoyo en el ensamble híbrido de SPAdes. Pueden revisar el tipo de secuencia (PacBio u ONT) en [este archivo](https://docs.google.com/spreadsheets/d/1ZGyQLNEZHWMnDa9r0mFZvZxzRa1nnAPsVIAY_qEfj5s/edit?usp=sharing).
 
## Objetivos del Práctico: 

- Familiarizarse con los conceptos de ensamble de novo y evaluación de la calidad de ensambles.
- Conocer el funcionamiento de herramientas bioinformáticas de control de calidad y ensamble (FastQC, LongQC, Canu, SPAdes, assembly-stats).
- Comparar los dos paradigmas de ensamble (OLC vs grafo de Bruijn) sobre datos reales.
- Adquirir práctica en entorno Unix. 


## Trabajo preliminar:

Conectarse al servidor.

Debido a que los cálculos que realizaremos en este práctico requieren un poder de cómputo moderado, nos conectaremos a un servidor privado. 
Si están en `Linux/MacOS` o `Windows 10`, puede utilizar la terminal (consola), en (Windows10 en el [Símbolo del sistema](https://es.wikipedia.org/wiki/S%C3%ADmbolo_del_sistema) `cmd`).


Una vez abierta la terminal, cada grupo debe escribir lo siguiente:

	 ssh  <usuario>@<servidor>
Donde `<usuario>`: es el nombre de usuario asignado y `<servidor>`: es el server donde conectarán.
Si están en Windows (anterior a Windows 10) deben instalar un programa llamado [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). 


Las credenciales ya fueron entregadas. 

#### Nombre de las carpetas:

Al conectarnos al servidor, entramos directamente al directorio home.
De acá nos tendremos que mover a nuestra carpeta de trabajo donde estan alojadas las secuencias:

	cd /opt/reads/gNreads

En este directorio están los archivos de lecturas de secuenciación que se le
asigno a cada grupo. 

Para poder ver estos archivos debemos escribir lo siguiente:

	ls
Los archivos vienen comprimidos (`.fastq.gz`); todas las herramientas de este práctico los aceptan así, no es necesario descomprimirlos.

**Entornos `conda`:** todas las herramientas del práctico están instaladas en el entorno **`bioinfo2026`**, salvo **LongQC**, que está en su propio entorno **`longqc`**. Active el entorno principal antes de empezar:

	mamba activate bioinfo2026

Solo deberá cambiarse al entorno `longqc` para el paso de LongQC (se indica más abajo) y luego volver a `bioinfo2026`.

Ahora, sobre los dos archivos de secuencia Illumina deberá ejecutar el siguiente comando:

	fastqc -t 4 il_1.fastq.gz il_2.fastq.gz
	
Donde `il_1.fastq.gz` e `il_2.fastq.gz` corresponden a las lecturas Paired End de Illumina.

Sobre el archivo de lecturas largas deberá ejecutar LongQC. Como está en otro entorno, primero actívelo y, al terminar, vuelva al entorno principal:

	mamba activate longqc
	longQC.py sampleqc --ncpu 8 -m 2 -o longqc.out -x <tipoSecuenciaLong>   long.fastq.gz
	mamba activate bioinfo2026
	
  Donde `<tipoSecuenciaLong>` corresponde a la tecnología de secuenciación utilizada en su archivo `long.fastq.gz` (revísela en el [spreadsheet](https://docs.google.com/spreadsheets/d/1ZGyQLNEZHWMnDa9r0mFZvZxzRa1nnAPsVIAY_qEfj5s/edit?usp=sharing) y use el preset correspondiente: `pb-rs2` para PacBio RS o `pb-sequel` para PacBio Sequel/Sequel II —incluido HiFi, ya que LongQC no tiene preset propio para HiFi—; `ont-ligation` o `ont-rapid` para Nanopore según el kit).

## Trimming y filtrado de lecturas

Antes de ensamblar conviene limpiar las lecturas: recortar adaptadores y bases de baja calidad en las cortas, y descartar las lecturas largas más cortas o de peor calidad. Esto reduce las sub-estructuras del grafo (tips, bubbles, arcos espurios) y mejora el ensamble, tal como vimos en la teoría.

### Lecturas cortas (Illumina) con `fastp`

[`fastp`](https://github.com/OpenGene/fastp) recorta adaptadores y calidad en un solo paso y genera un reporte antes/después:

	fastp -i il_1.fastq.gz -I il_2.fastq.gz -o il_1.trim.fastq.gz -O il_2.trim.fastq.gz --detect_adapter_for_pe --thread 4 --html fastp_grupoN.html --json fastp_grupoN.json

Esto genera las lecturas limpias `il_1.trim.fastq.gz` e `il_2.trim.fastq.gz` (que usaremos en SPAdes) y un reporte `fastp_grupoN.html` que pueden abrir para compararlo con el FastQC inicial.

### Lecturas largas con `filtlong`

[`filtlong`](https://github.com/rrwick/Filtlong) filtra las lecturas largas por largo y calidad. Descartaremos las menores a 1000 bp y el peor 10% restante:

	filtlong --min_length 1000 --keep_percent 90 long.fastq.gz | gzip > long.filt.fastq.gz

Esto genera `long.filt.fastq.gz`, que usaremos en Canu (y como apoyo en SPAdes). Tenga presente que Canu además realiza su propia corrección y trimming interno, por lo que aquí solo aplicamos un filtrado ligero.

## Ensamble de Genomas:

### Canu la evolución de Celera - wgs-assembler. 

[Celera](http://wgs-assembler.sourceforge.net/wiki/index.php?title=Main_Page) es un ensamblador que utiliza [OLC](https://www.ncbi.nlm.nih.gov/pubmed/22184334). Consta de una fase de corrección de lecturas, una de sobrelape, de generación de contigs y finalmente de scaffolding. 

Fue el encargado de ensamblar proyectos emblematicos como el genoma humano, actualmente como su sitio lo indica, ha sido descontinuado y reemplazado por [Canu](https://canu.readthedocs.io/en/latest/), Canu ha sido diseñado para ensamblar lecturas que tienen mucho ruido.


#### Ensamblar las lecturas:

Debido a que el proceso de ensamblar lecturas utilizando OLC puede tomar un tiempo prolongado,
ejecutar el comando de ensamble de la manera habitual es inconveniente, ya que si
cerramos la ventana de la consola, el proceso terminará también, por ende,
tendríamos que esperar que el ensamble terminara para cerrar la consola. En el caso
de que nos desconectaramos de internet/red, perderíamos lo que llevamos
ejecutando. Una solución a este problema es el comando [screen](https://linux.die.net/man/1/screen).


		screen -S grupoN_canu

Luego, Ejecutamos el comando [canu](https://canu.readthedocs.io/en/latest/tutorial.html) de `Canu` para ensamblar. Canu necesita un tamaño aproximado del genoma (`genomeSize`) para estimar la cobertura: **no tiene que ser exacto**, basta con el tamaño esperado de su organismo, que encontrará en la columna **`tamaño esperado`** del [spreadsheet](https://docs.google.com/spreadsheets/d/1ZGyQLNEZHWMnDa9r0mFZvZxzRa1nnAPsVIAY_qEfj5s/edit?usp=sharing).

El flag de la tecnología depende del tipo de lectura larga que le tocó (misma planilla). En Canu 2.x use `-pacbio` si son lecturas PacBio CLR, `-pacbio-hifi` si son HiFi, o `-nanopore` si son Oxford Nanopore (las antiguas `-pacbio-raw`/`-nanopore-raw` ya no se usan):


		canu -d canu_grupoN -p grupoN genomeSize=<tamañoEsperado> -nanopore long.filt.fastq.gz

Reemplace `<tamañoEsperado>` por el valor de la planilla (p. ej. `4.6m`, `9m`) y `-nanopore` por `-pacbio` o `-pacbio-hifi` según corresponda a su tipo de lectura larga. Note que usamos `long.filt.fastq.gz`, el archivo ya filtrado con `filtlong`.


Para cerrar la consola sin matar el proceso, tecleamos `Ctrl`+ `a` + `d`. 

Si queremos recuperar la consola donde lanzamos el programa 
escribimos lo siguiente:

		screen -r grupoN_canu
		
		
Para salir nuevamente tecleamos `Ctrl`+ `a` + `d` 

### SPAdes

[SPAdes](https://cab.spbu.ru/software/spades/) es un ensamblador de lecturas cortas que utiliza el [grafo de Bruijn](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5531759/).

SPAdes se ejecuta relativamente rápido, así que dependiendo de sus lecturas asignaremos los `k`-mers en base al largo de los reads, deben ser menor que el largo e impares.

Crearemos un `screen ` para spades:

		screen -S spades
		
 Luego dentro del screen ejecutamos:
 
  	spades.py -o spades_grupoN -t 16 -k 21,33,43,55,65,77,87,99 -1 il_1.trim.fastq.gz -2 il_2.trim.fastq.gz --nanopore long.filt.fastq.gz
	

Donde N es su grupo. El ejecutable es `spades.py` y usamos las lecturas ya limpias (`il_1.trim.fastq.gz`, `il_2.trim.fastq.gz`, `long.filt.fastq.gz`). Igual que en Canu, el flag de las lecturas largas depende de su tecnología: use `--pacbio` si son PacBio o `--nanopore` si son Oxford Nanopore. Recuerde que los `k` deben ser impares y menores que el largo de sus reads Illumina.

### Revisar los ensambles:

El ensamble de SPAdes si ha seguido el tutorial, debería estar en la carpeta `spades_grupoN`:

  entramos a la carpeta del resultado, path absoluto:
  
  	cd spades_grupoN
	
dentro podremos ubicar un archivo fasta llamado scaffolds, lo abriremos y con la barra de espacio lo recorremos:

	less scaffolds.fasta

Apretamos `q` para salir de less.
	
  También podemos hacer un `grep` 

	grep ">" scaffolds.fasta
	
o para saber el número de scaffolds 

	grep -c ">" scaffolds.fasta


Podemos hacer lo mismo una vez que haya terminado el ensamblador canu


Nos dirigimos a `canu_grupoN/grupoN` 

	cd canu_grupoN/grupoN

Recuerde reemplazar las N por el número de su grupo.

dentro podremos ubicar un archivo fasta llamado `grupoN.contigs.fasta`, lo abriremos y con la barra de espacio lo recorremos:

	less grupoN.contigs.fasta

Apretamos `q` para salir de less.
	
  También podemos hacer un `grep` 

	grep ">" grupoN.contigs.fasta
	
o para saber el número de contigs 

	grep -c ">" grupoN.contigs.fasta

 Utilizaremos el programa `assembly-stats` para obtener estadísticas de nuestros ensambles:

 	assembly-stats -l 1000 -t grupoN.contigs.fasta

Ahora vaya a la carpeta de spades y ejecute:

  	assembly-stats -l 1000 -t scaffolds.fasta
   
¿Existen diferencias entre los ensambles?
