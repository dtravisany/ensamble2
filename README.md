# ensamble
Práctico de Ensamble


## Resumen

En este práctico vamos a correr la revisión de unos reads secuenciados mediante dos plataformas de secuenciación Illumina y PacBio y luego utilizaremos dos ensambladores para realizar ensambles de los reads y obtener el genoma bacteriano, revisaremos las diversas métricas de ensamble y evaluaremos la calidad de un ensamble `de novo` utilizando los dos paradigmas más empleados en ensamble. A cada ensamble le realizaremos la predicción y posterior anotación de los genes.


## Materiales

### Software
 1. Quality Check:
  
- [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
- [LongQC](https://github.com/yfukasawa/LongQC)

  2. Ensamble:

- [Canu](https://canu.readthedocs.io/en/latest/)
- [Spades](https://cab.spbu.ru/software/spades/). 

  2.1 Revisión Ensamble:
-[assembly-stats](https://github.com/sanger-pathogens/assembly-stats)

  3. Predicción:

Luego, realizaremos la predicción de [CDS](https://www.uniprot.org/help/cds_protein_definition) a partir de los ensambles con la herramienta:
- [Prodigal](https://github.com/hyattpd/prodigal/wiki)
4. Anotación:
  
- Los péptidos predichos con Prodigal, se les asignará función putativa con el programa [BLAST+](https://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastDocs&DOC_TYPE=Download) y la base de datos [Swiss-Prot](https://www.uniprot.org/statistics/Swiss-Prot).

- Finalmente, utilizaremos una herramienta que combina diferentes bases de datos para la anotación basado en los "[_Cluster of Orthologous Groups_](https://www.ncbi.nlm.nih.gov/research/cog)", esta herramienta utiliza la base de datos _ponche de huevo_ por "[_eggNOG_](https://academic.oup.com/nar/article/47/D1/D309/5173662)" ([web](http://eggnog5.embl.de/))  y se llama [`eggnog-mapper`](https://github.com/eggnogdb/eggnog-mapper) 


#### Input de datos:

 Cada grupo tendrá dos conjuntos de lecturas de secuenciación, correspondientes a un genoma desconocido, el cual tendrán que inferir con los análisis.

- il_1 e il_2: [Librería Paired End](https://www.illumina.com/science/technology/next-generation-sequencing/plan-experiments/paired-end-vs-single-read.html).

- Pacbio: [Secuencias Pacbio](https://www.pacb.com/products-and-services/sequel-system/) Utilizaremos secuencias largas para ensamblar con Canu.
 
## Objetivos del Práctico: 

- Familiarizarse con los conceptos de ensamble, prediccón y anotación.
- Conocer el funcionamiento de herramientas bioinformáticas de ensamble, predicción y/mnt/biostore/curso/data/ anotación.
- Utilizar los servidores de blast.
- Adquirir práctica en entorno Unix. 


## Trabajo preliminar:

Conectarse al servidor.

Debido a que los cálculos que realizaremos en este práctico requieren un poder de cómputo moderado, nos conectaremos a un servidor privado. 
Si están en `Linux/MacOS` o `Windows 10`, puede utilizar la terminal (consola), en (Windows10 en el [Símbolo del sistema](https://es.wikipedia.org/wiki/S%C3%ADmbolo_del_sistema) (`cmd`).


Una vez abierta la terminal, cada grupo debe escribir lo siguiente:

	 ssh  grupoN@servidor

Si están en Windows (anterior a Windows 10) deben instalar un programa llamado [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). 


Las credenciales ya fueron entregadas. 
En el nombre de usuario y la contraseña, la `N` debe
ser reemplazada por el número del grupo que fue asignado.

#### Nombre de las carpetas:

Al conectarnos al servidor, entramos directamente al directorio home.
De acá nos tendremos que mover a nuestra carpeta de trabajo donde estan alojadas las secuencias:

	cd readsgN

En este directorio están los archivos de lecturas de secuenciación que se le
asigno a cada grupo. 

Para poder ver estos archivos debemos escribir lo siguiente:

	ls
Ahora, sobre cada archivo de secuencia Illumina deberá ejecutar el siguiente comando:

	fastqc -t 4 nombre_archivo.fastq
	
Donde `nombre_archivo` corresponde a los archivos de la secuenciación por Illumina.

Sobre los archivos pacbio deberá ejecutar:

	python /opt/PythonCurso/LongQC/longQC.py sampleqc --ncpu 8 -m 2 -o pacbioLongQC -x pb-rs2   pacbio.fastq
	
  

## Ensamble de Genomas:

### Canu la evolución de Celera - wgs-assembler. 

[Celera](http://wgs-assembler.sourceforge.net/wiki/index.php?title=Main_Page) es un ensamblador que utiliza [OLC](https://www.ncbi.nlm.nih.gov/pubmed/22184334). Consta de una fase de corrección de lecturas, una de sobrelape, de generación de contigs y finalmente de scaffolding. 

Fue el encargado de ensamblar proyectos emblematicos como el genoma humano, actualmente como su sitio lo indica, ha sido descontinuado y reemplazado por [Canu](https://canu.readthedocs.io/en/latest/), Canu ha sido diseñado para ensamblar lecturas que tienen mucho ruido.


#### Ensamblar las lecturas:

Debido a que el proceso de ensamblar lecturas utilizando OLC puede tomar un tiempo prolongado,
ejecutar el comando de ensamble de la manera habitual es inconveniente, ya que si
cerramos la ventana de la consola, el proceso terminará también, por ende,
tendríamos que esperar que el ensamble terminara para cerrar la consola. En el caso/mnt/biostore/curso/data/
de que nos desconectaramos de internet/red, perderíamos lo que llevamos
ejecutando. Una solución a este problema es el comando [screen](https://linux.die.net/man/1/screen).


		screen -S grupoN_canu

Luego, Ejecutamos el comando [canu](https://canu.readthedocs.io/en/latest/tutorial.html) de `Canu` para ensamblar, trate de estimar el tamaño del genoma, por defecto puse `5m` (5 megabases):


		canu -d canu_grupoN -p grupoN genomeSize=5m -pacbio-raw pacbio.fastq


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
 
  	spades -o spades_grupoN -t 16 -k21,33,43,55,65,77,87,99 -1 il_1.fastq -2 il_2.fastq --pacbio pacbio.fastq 
	

Donde N es su grupo 

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

dentro podremos ubicar un archivo fasta llamado `btN.contigs.fasta`, lo abriremos y con la barra de espacio lo recorremos:

	less grupoN.contigs.fasta

Apretamos `q` para salir de less.
	
  También podemos hacer un `grep` 

	grep ">" grupoN.contigs.fasta
	
o para saber el número de scaffolds 

	grep -c ">" grupoN.contigs.fasta

 Utilizaremos el programa `assembly-stats` para obtener estadísticas de nuestros ensambles:

 	assembly-stats -l 1000 -t grupoN.contigs.fasta

Ahora vaya a la carpeta de spades y ejecute:

  	assembly stats -l 1000 -t scaffolds.fasta
   
¿Existen diferencias entre los ensambles?
